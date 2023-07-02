[传统RPC框架从fd读取数据的方式](#传统RPC框架从fd读取数据的方式)

[brpc实现的读数据方式](#brpc实现的读数据方式)

## 传统RPC框架从fd读取数据的方式
传统的RPC框架一般会区分I/O线程和worker线程，一台机器上处理的所有fd会散列到多个I/O线程上，每个I/O线程在其私有的EventLoop对象上执行类似下面这样的循环：

```C++
while (!stop) {
  int n = epoll_wait();
  for (int i = 0; i < n; ++i) {
    if (event is EPOLLIN) {
      // read data from fd
      // push pointer of reference of data to task queue
    }
  }
}
```

I/O线程从每个fd上读取到数据后，将已读取数据的指针或引用封装在一个Task对象中，再将Task的指针压入一个全局的任务队列，worker线程从任务队列中拿到报文并进行业务处理。

这种方式存在以下几个问题：

1. 一个I/O线程同一时刻只能从一个fd读取数据，数据从fd的inode内核缓存读取到应用层缓冲区是一个相对较慢的操作，读取顺序越靠后的fd，其上面的数据读取、处理越可能产生延迟。实际场景中，如果10个客户端同一时刻分别通过10条TCP连接向一个服务器发送请求，假设服务器只开启一个I/O线程，epoll同时通知10个fd有数据可读，I/O线程只能先读完第一个fd，再读去读第二个fd...可能等到读第9、第10个fd时，客户端已经报超时了。但实际上10个客户端是同一时刻发送的请求，服务器的读取数据顺序却有先有后，这对客户端来说是不公平的；

2. 多个I/O线程都会将Task压入一个全局的任务队列，会产生锁竞争；多个worker线程从全局任务队列中获取任务，也会产生锁竞争，这都会降低性能。并且多个worker线程从长久看很难获取到均匀的任务数量，例如有4个worker线程同时去任务队列拿任务，worker 1竞争锁成功，拿到任务后去处理，然后worker 2拿到任务，接着worker 3拿到任务，再然后worker 1可能已经处理任务完毕，又来和worker 4竞争全局队列的锁，可能worker 1又再次竞争成功，worker 4还是饿着，这就造成了任务没有在各个worker线程间均匀分配；

3. I/O线程从fd的inode内核缓存读数据到应用层缓冲区，worker线程需要处理这块内存上的数据，内存数据同步到执行worker线程的cpu的cacheline上需要耗费时间。

## brpc实现的读数据方式
brpc没有专门的I/O线程，只有worker线程，epoll_wait()也是在bthread中被执行。当一个fd可读时：

1. 读取动作并不是在epoll_wait()所在的bthread 1上执行，而是会通过TaskGroup::start_foreground()新建一个bthread 2，bthread 2负责将fd的inode内核输入缓存中的数据读到应用层缓存区，pthread执行流会立即进入bthread 2，bthread 1会被加入任务队列的尾部，可能会被steal到其他pthread上执行；

2. bthread 2进行拆包时，每解析出一个完整的应用层报文，就会为每个报文的处理再专门创建一个bthread，所以bthread 2可能会创建bthread 3、4、5...这样的设计意图是尽量让一个fd上读出的各个报文也得到最大化的并发处理。


bthread 1的执行函数: 
1. epoll_wait等待
2. 有消息的话调用 StartInputEvent `->` Socket::ProcessEvent 
            `->` _on_edge_triggered_events(InputMessenger::OnNewMessages)
```C++
void EventDispatcher::Run() {
    while (!_stop) {
        epoll_event e[32];
#ifdef BRPC_ADDITIONAL_EPOLL
        // Performance downgrades in examples.
        int n = epoll_wait(_epfd, e, ARRAY_SIZE(e), 0);
        if (n == 0) {
            n = epoll_wait(_epfd, e, ARRAY_SIZE(e), -1);
        }
#else
        const int n = epoll_wait(_epfd, e, ARRAY_SIZE(e), -1);
#endif
        if (_stop) {
            // epoll_ctl/epoll_wait should have some sort of memory fencing
            // guaranteeing that we(after epoll_wait) see _stop set before
            // epoll_ctl.
            break;
        }
        if (n < 0) {
            if (EINTR == errno) {
                // We've checked _stop, no wake-up will be missed.
                continue;
            }
            PLOG(FATAL) << "Fail to epoll_wait epfd=" << _epfd;
            break;
        }
        for (int i = 0; i < n; ++i) {
            if (e[i].events & (EPOLLIN | EPOLLERR | EPOLLHUP)
#ifdef BRPC_SOCKET_HAS_EOF
                || (e[i].events & has_epollrdhup)
#endif
                ) {
                // We don't care about the return value.
                Socket::StartInputEvent(e[i].data.u64, e[i].events,
                                        _consumer_thread_attr);
            }
        }
        for (int i = 0; i < n; ++i) {
            if (e[i].events & (EPOLLOUT | EPOLLERR | EPOLLHUP)) {
                // We don't care about the return value.
                Socket::HandleEpollOut(e[i].data.u64);
            }
        }
    }
}
```

bthread 2的执行函数: 
```C++
void InputMessenger::OnNewMessages(Socket* m) {
    // Notes:
    // - If the socket has only one message, the message will be parsed and
    //   processed in this bthread. nova-pbrpc and http works in this way.
    // - If the socket has several messages, all messages will be parsed (
    //   meaning cutting from butil::IOBuf. serializing from protobuf is part of
    //   "process") in this bthread. All messages except the last one will be
    //   processed in separate bthreads. To minimize the overhead, scheduling
    //   is batched(notice the BTHREAD_NOSIGNAL and bthread_flush).
    // - Verify will always be called in this bthread at most once and before
    //   any process.
    InputMessenger* messenger = static_cast<InputMessenger*>(m->user());
    int progress = Socket::PROGRESS_INIT;

    // Notice that all *return* no matter successful or not will run last
    // message, even if the socket is about to be closed. This should be
    // OK in most cases.
    InputMessageClosure last_msg;
    bool read_eof = false;
    while (!read_eof) {
        const int64_t received_us = butil::cpuwide_time_us();
        const int64_t base_realtime = butil::gettimeofday_us() - received_us;

        // Calculate bytes to be read.
        size_t once_read = m->_avg_msg_size * 16;
        if (once_read < MIN_ONCE_READ) {
            once_read = MIN_ONCE_READ; // 2^12
        } else if (once_read > MAX_ONCE_READ) {
            once_read = MAX_ONCE_READ; // 2^19
        }

        // 从socket的innode里读出数据
        const ssize_t nr = m->DoRead(once_read);
        if (nr <= 0) {
            // 错误处理, Ignore
        }

        // 处理收到的消息
        if (m->_rdma_state == Socket::RDMA_OFF && messenger->ProcessNewMessage(
                    m, nr, read_eof, received_us, base_realtime, last_msg) < 0) {
            return;
        } 
    }

    if (read_eof) {
        m->SetEOF();
    }
}
```

```C++
int InputMessenger::ProcessNewMessage(Socket* m, ssize_t bytes, bool read_eof,
        const uint64_t received_us, const uint64_t base_realtime,
        InputMessageClosure& last_msg) {
    m->AddInputBytes(bytes);

    // Avoid this socket to be closed due to idle_timeout_s
    m->_last_readtime_us.store(received_us, butil::memory_order_relaxed);
    
    size_t last_size = m->_read_buf.length();
    int num_bthread_created = 0;
    while (1) {
        size_t index = 8888;
        // 消息切片
        ParseResult pr = CutInputMessage(m, &index, read_eof);

        // 启动bthread处理消息, 但是不加入调度
        DestroyingPtr<InputMessageBase> msg(pr.message());
        QueueMessage(last_msg.release(), &num_bthread_created,
                            m->_keytable_pool);
        m->ReAddress(&msg->_socket);
        m->PostponeEOF();
        msg->_process = _handlers[index].process; // 实际的消息处理函数
        msg->_arg = _handlers[index].arg;
    }

    // 开始调度QueueMessage起的bthread
    if (num_bthread_created) {
        bthread_flush();
    }
    return 0;
}
```
消息切片
```C++
ParseResult InputMessenger::CutInputMessage(Socket* m, size_t* index, bool read_eof) {
    const int preferred = m->preferred_index();
    const int max_index = (int)_max_index.load(butil::memory_order_acquire);
    // Try preferred handler first. The preferred_index is set on last
    // selection or by client.
    // 通过InputMessenger::AddHandler注册协议
    if (preferred >= 0 && preferred <= max_index && _handlers[preferred].parse != NULL) {
        int cur_index = preferred;  
        _handlers[cur_index].parse(&m->_read_buf, m, read_eof, _handlers[cur_index].arg);
        m->set_preferred_index(cur_index);
        *index = cur_index;
        return result;
    }
}
```

新建bthread处理每个消息
```C++
static void QueueMessage(InputMessageBase* to_run_msg,
                         int* num_bthread_created,
                         bthread_keytable_pool_t* keytable_pool) {
    if (!to_run_msg) {
        return;
    }
    // Create bthread for last_msg. The bthread is not scheduled
    // until bthread_flush() is called (in the worse case).

    bthread_t th;
    bthread_attr_t tmp = (FLAGS_usercode_in_pthread ?
                          BTHREAD_ATTR_PTHREAD :
                          BTHREAD_ATTR_NORMAL) | BTHREAD_NOSIGNAL;
    tmp.keytable_pool = keytable_pool;
    bthread_start_background(&th, &tmp, ProcessInputMessage, to_run_msg)
    ++*num_bthread_created;
}
```
开始调度
```C++
void bthread_flush() {
    bthread::TaskGroup* g = bthread::tls_task_group;
    if (g) {
        // 开始调度
        return g->flush_nosignal_tasks();
    }
    g = bthread::tls_task_group_nosignal;
    if (g) {
        // NOSIGNAL tasks were created in this non-worker.
        bthread::tls_task_group_nosignal = NULL;
        return g->flush_nosignal_tasks_remote();
    }
}
```

bthread 3-n的执行函数:

```C++
void* ProcessInputMessage(void* void_arg) {
    InputMessageBase* msg = static_cast<InputMessageBase*>(void_arg);
    msg->_process(msg);
    return NULL;
}
```

其中_process是注册协议时注册的函数，例如brpc的是
  - ProcessRpcRequest，里面通过CallMethod调用业务代码
  - ProcessRpcResponse，填写response并唤醒调用rpc的bthread