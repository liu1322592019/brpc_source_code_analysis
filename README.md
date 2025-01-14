# 目录
* brpc的M:N线程模型
  * [bthread基础](docs/bthread_basis.md)
  * [多核环境下pthread调度执行bthread的过程](docs/bthread_schedule.md)
  * [pthread线程间的Futex同步](docs/futex.md)
  * [Butex机制：bthread粒度的挂起与唤醒](docs/butex.md)
* 内存管理
  * [ResourcePool：多线程下高效的内存分配与回收](docs/resource_pool.md)
  * [I/O读写缓冲区](docs/io_buf.md)
* butil基础库
  * [侵入式双向链表](docs/linkedlist.md)
  * [FlatMap哈希表](docs/flat_map.md)
  * [多线程框架下的定时器](docs/timer.md)
* brpc的实时监控
  * [bvar库](docs/bvar.md)
* 并发读写TCP连接上的数据
  * :bangbang: [protobuf编程模式](docs/protobuf.md)
  * [多线程向同一TCP连接写入数据](docs/io_write.md)
  * [从TCP连接读取数据的并发处理](docs/io_read.md)
* Client端执行流程
  * [同一RPC过程中各个bthread间的互斥](docs/client_bthread_sync.md)
  * [无异常状态下的一次完整RPC请求过程](docs/client_rpc_normal.md)
  * [RPC请求可能遇到的多种异常及应对策略](docs/client_rpc_exception.md)
  * [重试&Backup Request](docs/client_retry_backup.md)
* Server端执行流程
  * [处理一次RPC请求的完整过程](docs/server_rpc_normal.md)
  * [服务器自动限流&防雪崩](docs/server_rpc_limit.md)
