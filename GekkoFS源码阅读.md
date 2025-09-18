[各源文件作用总结](各源文件作用总结.md)

## GekkoFS 架构

<img src="images\GekkoFS结构.png" alt="GekkoFS结构" style="zoom: 67%;" />

## 关键路径分析

### 创建文件

​	元数据操作，在元数据节点的键值数据库中为新建文件新增一条目。

（1）客户端调用 libc 库

​	调用 open 函数，内部包含系统调用 sys_open。

（2）syscall_intercept 拦截

​	sys_open 系统调用被 system call intercept library 拦截，执行用户自定义的 hook 函数，根据系统调用号 sys_open 进一步调用 hook_openat 函数。程序检测到文件路径位于 GekkoFS 管理范围内，因此调用 GekkoFS 自定义的处理函数 gkfs_open。由于文件不存在，该函数首先调用 gkfs_create 创建新文件，并将该文件记录在 filemap 中。

​	函数调用链：hook_guard_wrapper() -> hook() -> hook_openat() -> gkfs_open() -> gkfs_create()

（3）Margo RPC client 转发请求

​	gkfs_create 函数内部调用了 forward_create 函数与守护进程通信。forward_create  函数首先根据文件路径的 hash 值确定目标节点，然后向该节点发送create RPC 请求。

​	函数调用链：gkfs_create () -> forward_create () -> ld_network_service->post\<gkfs::rpc::create\>()

（4）Margo RPC server 接收请求

​	守护进程收到 create RPC 请求，调用和该消息绑定的 rpc_srv_create 函数。该函数根据客户端发送的数据，构造元数据并写入后端数据库，最后向客户端返回执行情况。

​	函数调用链：rpc_srv_create ()

（5）Metadata Backend 将元数据条目写入键值数据库

​	rpc_srv_create 调用了 gkfs::metadata::create 函数，完善各项元数据信息，然后是层层调用 put 函数，最终抵达 RocksDBBackend 类的 put_impl 函数，该函数通过调用 rocksdb::DB::Merge 将 create 操作加入待合并的操作队列，等待后续 Get() 被调用时 MetadataMergeOperator::FullMergeV2 被触发， create 操作才会被实际执行，即向数据库中新增一键值对。

​	函数调用链：rpc_srv_create () -> gkfs::metadata::create() -> MetadataDB::put() -> RocksDBBackend::put() -> RocksDBBackend::put_impl() -> rocksdb::DB::Merge()

### 写文件

​	先修改元数据中文件大小一项，然后执行实际的数据写入。

（1）客户端调用 libc 库

​	调用write()函数，内部包含系统调用 sys_write。

（2）syscall_intercept 拦截

​	sys_write 系统调用被拦截，实际执行 gkfs_write。如果用户在创建文件的时候未指定 O_APPEND，GekkoFS 会从 pos 处写入指定大小的数据，并更新 pos 的值，否则用户会根据 update_file_size 操作返回的偏移量来写入。这里我们假设是前一种情况。在 gkfs_do_write 中，首先更新文件大小为 max(offset + count, old_size)，若有 cache 则直接把更新后的 size 写入 cache，等到合适的时机再将 cache 的内容写回后端服务器；否则调用 forward_update_metadentry_size 向服务端发送 update size 的请求，过程类似于文件创建。随后调用 forward_write 向服务端发送写请求。

​	函数调用链：hook_guard_wrapper() -> hook() -> hook_write() -> gkfs_write() -> gkfs_write_ws() -> gkfs_do_write()

（3）Margo RPC client 转发请求

​	gkfs_do_write 函数内部调用了 forward_write 函数与守护进程通信。forward_write 首先收集一些 write 操作所需要的信息，包括 chunk 下标范围和总数、所有目标节点的编号、每一个目标节点需要写入的 chunk 集合、chunk 位示图、第一个和最后一个 chunk 对应的目标节点，然后准备发送 RPC 消息，包括创建并向远程主机暴露数据缓冲区，打包输入数据并发送异步 RPC 消息，最后等待所有请求处理完毕并返回结果。注意**客户端会将 chunk 按照目标节点分组，写入同一个节点的多个 chunk 共用同一条 RPC 消息**。

​	函数调用链：gkfs_do_write () -> forward_write () -> ld_network_service->post\<gkfs::rpc::write_data\>()

（4）Margo RPC server 接收请求

​	守护进程收到 write 请求，调用和该消息绑定的 rpc_srv_write 函数。第一步是收集相关信息，包括反序列化 RPC 消息中的数据、获取 Margo 实例、缓冲区大小和位示图。第二步是设置缓冲区，做好接收数据的准备。第三步是分别计算每个 chunk 的实际大小、执行数据传输并启动异步写盘任务。对于每一个定位到本主机的 chunk，计算其大小，并调用 margo_bulk_transfer 进行 RDMA 传输，最后启动异步写入任务。第四步是等待所有任务执行完毕，这时数据已持久化到本地存储，随后向客户端返回执行状态。可以看出，**虽然 RPC 消息是以主机为单位进行发送 ，但同一台主机上的数据传输和持久化还是以 chunk 为单位**。 

​	函数调用链：rpc_srv_write ()

（5）Data Backend 将数据块写入本地存储

​	write_nonblock 为非阻塞调用，创建了一个 Argobots 任务（ABT_task）来执行 write_file_abt 函数，该任务会在 IO 池中异步执行。write_file_abt 函数则调用了 ChunkStorage 类的 write_chunk 函数，write_chunk 则直接调用 libc 库函数和本地文件系统交互，将数据块写入本地存储。

​	函数调用链：rpc_srv_write() -> ChunkWriteOperation::write_nonblock() -> ChunkWriteOperation::write_file_abt() -> ChunkStorage::write_chunk() -> pwrite()

### 读文件

​	从 pos 处读取指定大小的数据，然后更新 pos 的值。

（1）客户端调用 libc 库

​	调用 read 函数，内部包含系统调用 sys_read。

（2）syscall_intercept 拦截

​	最终由 gkfs_do_read 进行实际处理，内部调用了 forward_read 向守护进程发送 read 类型的 RPC 消息。

（3）Margo RPC client 转发请求

​	同样按照目标节点对 chunk 进行分组，随后批量发送 RPC 消息。先从主副本所在服务器读取数据块，如果失败，则随机选取一个副本所在的服务器读取上面的数据。

（4）Margo RPC server 接收请求

​	和 write 不同的是，守护进程在执行 read 时会首先启动异步读取任务，等某一个 chunk 读取完毕，再通过 RDMA 将这部分数据发送到客户端。等待方式包括轮询和阻塞两种类型。

（5）Data Backend 将数据块从本地存储读入缓冲区

​	函数调用链：rpc_srv_read() -> ChunkReadOperation::read_nonblock() -> ChunkReadOperation::read_file_abt() -> ChunkStorage::read_chunk() -> pread64()
