# 目录
* brpc的M:N线程模型
  * :white_check_mark: [bthread基础](docs/bthread_basis.md)
  * :white_check_mark: [多核环境下pthread调度执行bthread的过程](docs/bthread_schedule.md)
  * :white_check_mark: [pthread线程间的Futex同步](docs/futex.md)
  * :white_check_mark: [Butex机制：bthread粒度的挂起与唤醒](docs/butex.md)
* 内存管理
  * :white_check_mark: [ResourcePool：多线程下高效的内存分配与回收](docs/resource_pool.md)
  * :white_check_mark: [I/O读写缓冲区](docs/io_buf.md)
* butil基础库
  * :white_check_mark: [侵入式双向链表](docs/linkedlist.md)
  * :white_check_mark: [FlatMap哈希表](docs/flat_map.md)
  * :white_check_mark: [多线程框架下的定时器](docs/timer.md)
* brpc的实时监控
  * bvar库
  * 常用性能监控指标
* 并发读写TCP连接上的数据
  * [protobuf编程模式](docs/protobuf.md)
  * :white_check_mark: [多线程向同一TCP连接写入数据](docs/io_write.md)
  * :white_check_mark: [从TCP连接读取数据的并发处理](docs/io_read.md)
* Client端执行流程
  * :white_check_mark: [同一RPC过程中各个bthread间的互斥](docs/client_bthread_sync.md)
  * :white_check_mark: [无异常状态下的一次完整RPC请求过程](docs/client_rpc_normal.md)
  * :white_check_mark: [RPC请求可能遇到的多种异常及应对策略](docs/client_rpc_exception.md)
  * :white_check_mark: [重试&Backup Request](docs/client_retry_backup.md)
* Server端执行流程
  * 处理一次RPC请求的完整过程
  * 服务器自动限流
  * 防雪崩
