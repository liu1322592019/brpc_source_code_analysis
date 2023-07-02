[数据发送过程涉及到的主要数据结构](#数据发送过程涉及到的主要数据结构) 

[数据发送过程中的系统内存布局与多线程执行状态](#数据发送过程中的系统内存布局与多线程执行状态) 

[数据发送代码解析](#数据发送代码解析) 

## 数据发送过程涉及到的主要数据结构
1. Channel对象：表示客户端与一台服务器或一组服务器的连接通道。

2. Controller对象：存储一次完整的RPC请求的Context以及各种状态。

3. Butex对象：实现bthread粒度的互斥锁，管理锁的挂起等待队列。

4. Id对象：同步一次RPC过程中的各个bthread（发送数据、处理服务器的响应、处理超时均由不同的bthread完成）。

## 数据发送过程中的系统内存布局与多线程执行状态
以brpc自带的实例程序example/multi_threaded_echo_c++/client.cpp为例，结合Client端内存布局的变化过程和多线程执行过程，阐述无异常状态下（所有发送数据都及时得到响应，没有超时、没有因服务器异常等原因引发的请求重试）的Client发送请求直到处理响应的过程。

该程序运行后，会与单台服务器建立一条TCP长连接，创建thread_num个bthread（后续假设thread_num=3）在此TCP连接上发送、接收数据，不涉及连接池、负载均衡。RPC使用同步方式，这里的同步是指bthread间的同步：负责发送数据的bthread完成发送操作后，不能结束，而是需要挂起，等待负责接收服务器响应的bthread将其唤醒后，再恢复执行。挂起时会让出cpu，pthread可继续执行任务队列中的其他bthread。

具体运行过程为：

1. 在main函数的栈上创建一个Channel对象，并初始化Channel的协议类型、连接类型、RPC超时时间、请求重试次数等参数，上述参数后续会被赋给所有通过此Channel的RPC请求；

2. 在main函数中调用bthread_start_background创建3个bthread，此时TaskControl、TaskGroup对象都并不存在，所以此时需要在heap内存上创建它们（惰性初始化方式，不是在程序启动时就创建所有的对象，而是到对象确实要被用到时才去创建, 调用栈: bthread_start_background -> start_from_non_worker -> get_or_new_task_control）：
   - 一个TaskControl单例对象
   - N个TaskGroup对象（后续假设N=4），每个TaskGroup对应一个系统线程pthread，是pthread的线程私有对象，每个pthread启动后以自己的TaskGroup对象的run_main_task函数作为主工作函数，在该函数内执行无限循环，不断地从TaskGroup的任务队列中取得bthread id、通过id找到bthread对象、去执行bthread任务函数。
   
3. 在TaskMeta对象池中创建3个TaskMeta对象（每个TaskMeta等同一个bthread），每个TaskMeta的fn函数指针指向client.cpp中定义的static类型函数sender，sender就是bthread的任务处理函数。每个TaskMeta创建完后，按照散列规则找到一个TaskGroup对象，并将tid（也就是bthread的唯一标识id）压入该TaskGroup对象的_remote_rq队列中（TaskGroup所属的pthread线程称为worker线程，worker线程自己产生的bthread的tid会被压入自己私有的TaskGroup对象的_rq队列，本实例中的main函数所在线程不属于worker线程，所以main函数所在的线程生成的bthread的tid会被压入找到的TaskGroup对象的_rq队列）；

4. main函数执行到这里，不能直接结束（否则Channel对象会被马上析构，所有RPC无法进行），必须等待3个bthread全部执行sender函数结束后，main才能结束。

   - main函数所在线程挂起的实现机制是，将main函数所在线程的信息存储在ButexPthreadWaiter中，并加入到TaskMeta对象的version_butex指针所指的Butex对象的等待队列waiters中，TaskMeta的任务函数fn执行结束后，会从waiters中查找到“之前因等待TaskMeta的任务函数fn执行结束而被挂起的”pthread线程，再将其唤醒。关于Butex机制的细节，可参见[这篇文章](butex.md)；
   
   - main函数所在的系统线程在join bthread 1的时候就被挂起，等待在wait_pthread函数处。bthread 1执行sender函数结束后，唤醒main函数的线程，main函数继续向下执行，去join bthread 2。如果此时bthread 2仍在运行，则再将存储了main函数所在线程信息的一个新的ButexPthreadWaiter加入到bthread 2对应的TaskMeta对象的version_butex指针所指的Butex对象的等待队列waiters中，等到bthread 2执行完sender函数后再将main函数所在线程唤醒。也可能当main函数join bthread 2的时候bthread 2已经运行完成，则join操作直接返回，接着再去join bthread 3；
   
   - 只有当三步join操作全部返回后，main函数才结束。  
   
5. 此时Client进程内部的线程状态是：

   - bthread状态：三个bthread 1、2、3已经创建完毕，各自的bthread id已经被分别压入不同的TaskGroup对象（设为TaskGroup 1、2、3）的任务队列_remote_rq中；
   
   - pthread状态：此时进程中存在5个pthread线程：3个pthread即将从各自私有的TaskGroup对象的_remote_rq中拿到bthread id，将要执行bthread id对应的TaskMeta对象的任务函数；1个pthread仍然阻塞在run_main_task函数上，等待新任务到来通知；main函数所在线程被挂起，等待bthread 1执行结束。
   
6. 此时Client进程内部的内存布局如下图所示，由于bthread 1、2、3还未开始运行，未分配任何局部变量，所以此时各自的私有栈都是空的：
   
    <img src="../images/client_send_req_1.png" width="70%" height="70%"/>

7. TaskGroup 1、2、3分别对应的3个pthread开始执行各自拿到的bthread的任务函数，即client.cpp中的static类型的sender函数。由于各个bthread有各自的私有栈空间，所以sender中的局部变量request、response、Controller对象均被分配在bthread的私有栈内存上；

8. 根据protobuf的标准编程模式，3个执行sender函数的bthread都会执行Channel的CallMethod函数，CallMethod负责的工作为：

   - CallMethod函数的入参为各个bthread私有的request、response、Controller，CallMethod内部会为Controller对象的相关成员变量赋值，包括RPC起始时间戳、最大重试次数、RPC超时时间、Backup Request超时时间、标识一次RPC过程的唯一id correlation_id等等。Controller对象可以认为是存储了一次RPC过程的所有Context上下文信息;
   
   - 在CallMethod函数中不存在线程间的竞态，CallMethod本身是线程安全的。而Channel对象是main函数的栈上对象，main函数所在线程已被挂起，直到3个bthread全部执行完成后才会结束，所以Channel对象的生命期贯穿于整个RPC过程;
   
   - 构造Controller对象相关联的Id对象，Id对象的作用是同步一次RPC过程中的各个bthread，因为在一次RPC过程中，发送请求、接收响应、超时处理均是由不同的bthread负责，各个bthread可能运行在不同的pthread上，因此这一次RPC过程的Controller对象可能被上述不同的bthread同时访问，也就是相当于被不同的pthread并发访问，产生竞态。此时不能直接让某个pthread去等待线程锁，那样会让pthread挂起，阻塞该pthread私有的TaskGroup对象的任务队列中其他bthread的执行。因此如果一个bthread正在访问Controller对象，此时位于不同pthread上的其他bthread若想访问Controller，必须将自己的bthread信息加入到一个等待队列中，yield让出cpu，让pthread继续去执行任务队列中下一个bthread。正在访问Controller的bthread让出访问权后，会从等待队列中找到挂起的bthread，并将其bthread id再次压入某个TaskGroup的任务队列，这样就可让原先为了等待Controller访问权而挂起的bthread得以从yield点恢复执行。这就是bthread级别的挂起-唤醒的基本原理，这样也保证所有pthread是wait-free的。
   
   - 在CallMethod中会通过将Id对象的butex指针指向的Butex结构的value值置为“locked_ver”表示Id对象已被锁，即当前发送数据的bthread正在访问Controller对象。在本文中假设发送数据后正常接收到响应，不涉及重试、RPC超时等，所以不深入阐述Id对象，关于Id的细节请参考[这篇文章](client_bthread_sync.md)。

9. pthread线程执行流程接着进入Controller的IssueRPC函数，在该函数中：

   - 按照指定协议格式将RPC过程的首次请求的call_id、RPC方法名、实际待发送数据打包成报文；
   
   - 调用Socket::Write函数执行实际的发送数据过程。Socket对象表示Client与单台Server的连接。向fd写入数据的细节过程参考[这篇文章](io_write.md)；
   
   - 在实际发送数据前需要先建立与Server的TCP长连接，并惰性初始化event_dispatcher_num个EventDispatcher对象（假设event_dispatcher_num=2），从而新建2个bthread 4和5，并将它们的任务处理函数设为static类型的EventDispatcher::RunThis函数，当bthread 4、5得到pthread执行时，会调用epoll_wait检测是否有I/O事件触发。brpc是没有专门的I/O pthread线程的；
   
   - 从Socket::Write函数返回后，调用bthread_id_unlock释放对Controller对象的独占访问。
   
10. 因为RPC使用synchronous同步方式，所以bthread完成数据发送后调用bthread_id_join将自身挂起，让出cpu，等待负责接收服务器响应的bthread来唤醒。此时Client进程内部的线程状态是：bthread 1、2、3都已挂起，执行bthread任务的pthread 1、2、3分别跳出了bthread 1、2、3的任务函数，回到TaskGroup::run_main_task函数继续等待新的bthread任务，因为在向fd写数据的过程中通常会新建一个KeepWrite bthread（bthread 6），假设这个bthread的id被压入到TaskGroup 4的任务队列中，被pthread 4执行，所以pthread 1、2、3此时没有新bthread可供执行，处于系统线程挂起状态。

11. 此时Client进程内部的内存布局如下图所示，注意各个类型对象分配在不同的内存区，比如Butex对象、Id对象分配在heap上，Controller对象、ButexBthreadWaiter对象分配在bthread的私有栈上：

    <img src="../images/client_send_req_2.png" width="100%" height="100%"/>

12. KeepWrite bthread完成工作后，3个请求都被发出，假设服务器正常返回了3个响应，由于3个响应是在一个TCP连接上接收的，所以bthread 4、5二者只会有一个通过epoll_wait()检测到fd可读，并新建一个bthread 7去负责将fd的inode输入缓存中的数据读取到应用层，在拆包过程中，解析出一条Response，就为这个Response的处理再新建一个bthread，目的是实现响应读取+处理的最大并发。因此Response 1在bthread 8中被处理，Response 2在bthread 9中被处理，Response 3在bthread 7中被处理（最后一条Response不需要再新建bthread了，直接在bthread 7本地处理即可）。bthread 8、9、7会将Response 1、2、3分别复制到相应Controller对象的response中，这时应用程序就会看到响应数据了。bthread 8、9、7也会将挂起的bthread 1、2、3唤醒，bthread 1、2、3会恢复执行，可以对Controller对象中的response做一些操作，并开始发送下一个RPC请求。

## 数据发送代码解析

rpc调用函数

```C++
void Channel::CallMethod(const google::protobuf::MethodDescriptor* method,
                         google::protobuf::RpcController* controller_base,
                         const google::protobuf::Message* request,
                         google::protobuf::Message* response,
                         google::protobuf::Closure* done) {
    const int64_t start_send_real_us = butil::gettimeofday_us();
    Controller* cntl = static_cast<Controller*>(controller_base);
    cntl->OnRPCBegin(start_send_real_us);

    // 从channel像controller拷贝各种数据

    // Override max_retry first to reset the range of correlation_id
    if (cntl->max_retry() == UNSET_MAGIC_NUM) {
        cntl->set_max_retry(_options.max_retry);
    }
    if (cntl->max_retry() < 0) {
        // this is important because #max_retry decides #versions allocated
        // in correlation_id. negative max_retry causes undefined behavior.
        cntl->set_max_retry(0);
    }
    // HTTP needs this field to be set before any SetFailed()
    cntl->_request_protocol = _options.protocol;
    if (_options.protocol.has_param()) {
        CHECK(cntl->protocol_param().empty());
        cntl->protocol_param() = _options.protocol.param();
    }
    if (_options.protocol == brpc::PROTOCOL_HTTP && (_scheme == "https" || _scheme == "http")) {
        URI& uri = cntl->http_request().uri();
        if (uri.host().empty() && !_service_name.empty()) {
            uri.SetHostAndPort(_service_name);
        }
    }
    cntl->_preferred_index = _preferred_index;
    cntl->_retry_policy = _options.retry_policy;
    if (_options.enable_circuit_breaker) {
        cntl->add_flag(Controller::FLAGS_ENABLED_CIRCUIT_BREAKER);
    }
    const CallId correlation_id = cntl->call_id();
    // 使用cntl前, 上锁，避免不同的bthread同时访问
    const int rc = bthread_id_lock_and_reset_range(
                    correlation_id, NULL, 2 + cntl->max_retry());
    if (rc != 0) {
        // 理论上进不来这里, 此时不应该有任何的访问
        CHECK_EQ(EINVAL, rc);
        if (!cntl->FailedInline()) {
            cntl->SetFailed(EINVAL, "Fail to lock call_id=%" PRId64,
                            correlation_id.value);
        }
        LOG_IF(ERROR, cntl->is_used_by_rpc())
            << "Controller=" << cntl << " was used by another RPC before. "
            "Did you forget to Reset() it before reuse?";
        // Have to run done in-place. If the done runs in another thread,
        // Join() on this RPC is no-op and probably ends earlier than running
        // the callback and releases resources used in the callback.
        // Since this branch is only entered by wrongly-used RPC, the
        // potentially introduced deadlock(caused by locking RPC and done with
        // the same non-recursive lock) is acceptable and removable by fixing
        // user's code.
        if (done) {
            done->Run();
        }
        return;
    }
    cntl->set_used_by_rpc();

    if (cntl->_sender == NULL && IsTraceable(Span::tls_parent())) {
        const int64_t start_send_us = butil::cpuwide_time_us();
        const std::string* method_name = NULL;
        if (_get_method_name) {
            method_name = &_get_method_name(method, cntl);
        } else if (method) {
            method_name = &method->full_name();
        } else {
            const static std::string NULL_METHOD_STR = "null-method";
            method_name = &NULL_METHOD_STR;
        }
        Span* span = Span::CreateClientSpan(
            *method_name, start_send_real_us - start_send_us);
        span->set_log_id(cntl->log_id());
        span->set_base_cid(correlation_id);
        span->set_protocol(_options.protocol);
        span->set_start_send_us(start_send_us);
        cntl->_span = span;
    }
    // Override some options if they haven't been set by Controller
    if (cntl->timeout_ms() == UNSET_MAGIC_NUM) {
        cntl->set_timeout_ms(_options.timeout_ms);
    }
    // Since connection is shared extensively amongst channels and RPC,
    // overriding connect_timeout_ms does not make sense, just use the
    // one in ChannelOptions
    cntl->_connect_timeout_ms = _options.connect_timeout_ms;
    if (cntl->backup_request_ms() == UNSET_MAGIC_NUM) {
        cntl->set_backup_request_ms(_options.backup_request_ms);
    }
    if (cntl->connection_type() == CONNECTION_TYPE_UNKNOWN) {
        cntl->set_connection_type(_options.connection_type);
    }
    cntl->_response = response;
    cntl->_done = done;
    cntl->_pack_request = _pack_request;
    cntl->_method = method;
    cntl->_auth = _options.auth;

    if (SingleServer()) {
        cntl->_single_server_id = _server_id;
        cntl->_remote_side = _server_address;
    }

    // Share the lb with controller.
    cntl->_lb = _lb;

    // Ensure that serialize_request is done before pack_request in all
    // possible executions, including:
    //   HandleSendFailed => OnVersionedRPCReturned => IssueRPC(pack_request)
    // 序列化成为buf, 用于消息发送, brpc为SerializeRequestDefault[brpc/protocol.cpp]
    _serialize_request(&cntl->_request_buf, cntl, request);
    if (cntl->FailedInline()) {
        // Handle failures caused by serialize_request, and these error_codes
        // should be excluded from the retry_policy.
        return cntl->HandleSendFailed();
    }
    if (FLAGS_usercode_in_pthread &&
        done != NULL &&
        TooManyUserCode()) {
        cntl->SetFailed(ELIMIT, "Too many user code to run when "
                        "-usercode_in_pthread is on");
        return cntl->HandleSendFailed();
    }

    if (cntl->_request_stream != INVALID_STREAM_ID) {
        // Currently we cannot handle retry and backup request correctly
        cntl->set_max_retry(0);
        cntl->set_backup_request_ms(-1);
    }

    if (cntl->backup_request_ms() >= 0 &&
        (cntl->backup_request_ms() < cntl->timeout_ms() ||
         cntl->timeout_ms() < 0)) {
        // Setup timer for backup request. When it occurs, we'll setup a
        // timer of timeout_ms before sending backup request.

        // _deadline_us is for truncating _connect_timeout_ms and resetting
        // timer when EBACKUPREQUEST occurs.
        if (cntl->timeout_ms() < 0) {
            cntl->_deadline_us = -1;
        } else {
            cntl->_deadline_us = cntl->timeout_ms() * 1000L + start_send_real_us;
        }
        // 起个定时器用于超时
        const int rc = bthread_timer_add(
            &cntl->_timeout_id,
            butil::microseconds_to_timespec(
                cntl->backup_request_ms() * 1000L + start_send_real_us),
            HandleBackupRequest, (void*)correlation_id.value);
        if (BAIDU_UNLIKELY(rc != 0)) {
            cntl->SetFailed(rc, "Fail to add timer for backup request");
            return cntl->HandleSendFailed();
        }
    } else if (cntl->timeout_ms() >= 0) {
        // Setup timer for RPC timetout

        // _deadline_us is for truncating _connect_timeout_ms
        cntl->_deadline_us = cntl->timeout_ms() * 1000L + start_send_real_us;
        // 起个定时器用于超时
        const int rc = bthread_timer_add(
            &cntl->_timeout_id,
            butil::microseconds_to_timespec(cntl->_deadline_us),
            HandleTimeout, (void*)correlation_id.value);
        if (BAIDU_UNLIKELY(rc != 0)) {
            cntl->SetFailed(rc, "Fail to add timer for timeout");
            return cntl->HandleSendFailed();
        }
    } else {
        cntl->_deadline_us = -1;
    }

    // 发起RPC调用
    cntl->IssueRPC(start_send_real_us);

    if (done == NULL) {
        // MUST wait for response when sending synchronous RPC. It will
        // be woken up by callback when RPC finishes (succeeds or still
        // fails after retry)
        Join(correlation_id);
        if (cntl->_span) {
            cntl->SubmitSpan();
        }
        cntl->OnRPCEnd(butil::gettimeofday_us());
    }
}
```

实际处理请求发送

```C++
void Controller::IssueRPC(int64_t start_realtime_us) {
    _current_call.begin_time_us = start_realtime_us;
    
    // If has retry/backup request，we will recalculate the timeout,
    if (_real_timeout_ms > 0) {
        _real_timeout_ms -= (start_realtime_us - _begin_time_us) / 1000;
    }

    // Clear last error, Don't clear _error_text because we append to it.
    _error_code = 0;

    // Make versioned correlation_id.
    // call_id         : unversioned, mainly for ECANCELED and ERPCTIMEDOUT
    // call_id + 1     : first try.
    // call_id + 2     : retry 1
    // ...
    // call_id + N + 1 : retry N
    // All ids except call_id are versioned. Say if we've sent retry 1 and
    // a failed response of first try comes back, it will be ignored.
    const CallId cid = current_id();

    // Intercept IssueRPC when _sender is set. Currently _sender is only set
    // by SelectiveChannel.
    if (_sender) {
        if (_sender->IssueRPC(start_realtime_us) != 0) {
            return HandleSendFailed();
        }
        CHECK_EQ(0, bthread_id_unlock(cid));
        return;
    }

    // 根据指定的负载均衡策略选择服务端
    // Pick a target server for sending RPC
    _current_call.need_feedback = false;
    _current_call.enable_circuit_breaker = has_enabled_circuit_breaker();
    SocketUniquePtr tmp_sock;
    if (SingleServer()) {
        // Don't use _current_call.peer_id which is set to -1 after construction
        // of the backup call.
        const int rc = Socket::Address(_single_server_id, &tmp_sock);
        if (rc != 0 || (!is_health_check_call() && !tmp_sock->IsAvailable())) {
            SetFailed(EHOSTDOWN, "Not connected to %s yet, server_id=%" PRIu64,
                      endpoint2str(_remote_side).c_str(), _single_server_id);
            tmp_sock.reset();  // Release ref ASAP
            return HandleSendFailed();
        }
        _current_call.peer_id = _single_server_id;
    } else {
        LoadBalancer::SelectIn sel_in =
            { start_realtime_us, true,
              has_request_code(), _request_code, _accessed };
        LoadBalancer::SelectOut sel_out(&tmp_sock);
        const int rc = _lb->SelectServer(sel_in, &sel_out);
        if (rc != 0) {
            std::ostringstream os;
            DescribeOptions opt;
            opt.verbose = false;
            _lb->Describe(os, opt);
            SetFailed(rc, "Fail to select server from %s", os.str().c_str());
            return HandleSendFailed();
        }
        _current_call.need_feedback = sel_out.need_feedback;
        _current_call.peer_id = tmp_sock->id();
        // NOTE: _remote_side must be set here because _pack_request below
        // may need it (e.g. http may set "Host" to _remote_side)
        // Don't set _local_side here because tmp_sock may be not connected
        // here.
        _remote_side = tmp_sock->remote_side();
    }
    if (_stream_creator) {
        _current_call.stream_user_data =
            _stream_creator->OnCreatingStream(&tmp_sock, this);
        if (FailedInline()) {
            return HandleSendFailed();
        }
        // remote_side can't be changed.
        CHECK_EQ(_remote_side, tmp_sock->remote_side());
    }

    Span* span = _span;
    if (span) {
        if (_current_call.nretry == 0) {
            span->set_remote_side(_remote_side);
        } else {
            span->Annotate("Retrying %s",
                           endpoint2str(_remote_side).c_str());
        }
    }
    // 根据类型处理或获取TCP连接
    // Handle connection type
    if (_connection_type == CONNECTION_TYPE_SINGLE ||
        _stream_creator != NULL) { // let user decides the sending_sock
        // in the callback(according to connection_type) directly
        _current_call.sending_sock.reset(tmp_sock.release());
        // TODO(gejun): Setting preferred index of single-connected socket
        // has two issues:
        //   1. race conditions. If a set perferred_index is overwritten by
        //      another thread, the response back has to check protocols one
        //      by one. This is a performance issue, correctness is unaffected.
        //   2. thrashing between different protocols. Also a performance issue.
        _current_call.sending_sock->set_preferred_index(_preferred_index);
    } else {
        int rc = 0;
        if (_connection_type == CONNECTION_TYPE_POOLED) {
            rc = tmp_sock->GetPooledSocket(&_current_call.sending_sock);
        } else if (_connection_type == CONNECTION_TYPE_SHORT) {
            rc = tmp_sock->GetShortSocket(&_current_call.sending_sock);
        } else {
            tmp_sock.reset();
            SetFailed(EINVAL, "Invalid connection_type=%d", (int)_connection_type);
            return HandleSendFailed();
        }
        if (rc) {
            tmp_sock.reset();
            SetFailed(rc, "Fail to get %s connection",
                      ConnectionTypeToString(_connection_type));
            return HandleSendFailed();
        }
        // Remember the preferred protocol for non-single connection. When
        // the response comes back, InputMessenger calls the right handler
        // w/o trying other protocols. This is a must for (many) protocols that
        // can't be distinguished from other protocols w/o ambiguity.
        _current_call.sending_sock->set_preferred_index(_preferred_index);
        // Set preferred_index of main_socket as well to make it easier to
        // debug and observe from /connections.
        if (tmp_sock->preferred_index() < 0) {
            tmp_sock->set_preferred_index(_preferred_index);
        }
        tmp_sock.reset();
    }
    if (_tos > 0) {
        _current_call.sending_sock->set_type_of_service(_tos);
    }
    if (is_response_read_progressively()) {
        // Tag the socket so that when the response comes back, the parser will
        // stop before reading all body.
        _current_call.sending_sock->read_will_be_progressive(_connection_type);
    }

    // Handle authentication
    const Authenticator* using_auth = NULL;
    if (_auth != NULL) {
        // Only one thread will be the winner and get the right to pack
        // authentication information, others wait until the request
        // is sent.
        int auth_error = 0;
        if (_current_call.sending_sock->FightAuthentication(&auth_error) == 0) {
            using_auth = _auth;
        } else if (auth_error != 0) {
            SetFailed(auth_error, "Fail to authenticate, %s",
                      berror(auth_error));
            return HandleSendFailed();
        }
    }
    // Make request
    butil::IOBuf packet;
    SocketMessage* user_packet = NULL;
    _pack_request(&packet, &user_packet, cid.value, _method, this,
                  _request_buf, using_auth);
    // TODO: PackRequest may accept SocketMessagePtr<>?
    SocketMessagePtr<> user_packet_guard(user_packet);
    if (FailedInline()) {
        // controller should already be SetFailed.
        if (using_auth) {
            // Don't forget to signal waiters on authentication
            _current_call.sending_sock->SetAuthentication(ErrorCode());
        }
        return HandleSendFailed();
    }

    timespec connect_abstime;
    timespec* pabstime = NULL;
    if (_connect_timeout_ms > 0) {
        if (_deadline_us >= 0) {
            connect_abstime = butil::microseconds_to_timespec(
                std::min(_connect_timeout_ms * 1000L + start_realtime_us,
                         _deadline_us));
        } else {
            connect_abstime = butil::microseconds_to_timespec(
                _connect_timeout_ms * 1000L + start_realtime_us);
        }
        pabstime = &connect_abstime;
    }

    // Write会调用StartWrite写入数据到TCP，实际发送
    Socket::WriteOptions wopt;
    wopt.id_wait = cid;
    wopt.abstime = pabstime;
    wopt.pipelined_count = _pipelined_count;
    wopt.auth_flags = _auth_flags;
    wopt.ignore_eovercrowded = has_flag(FLAGS_IGNORE_EOVERCROWDED);
    int rc;
    size_t packet_size = 0;
    if (user_packet_guard) {
        if (span) {
            packet_size = user_packet_guard->EstimatedByteSize();
        }
        rc = _current_call.sending_sock->Write(user_packet_guard, &wopt);
    } else {
        packet_size = packet.size();
        rc = _current_call.sending_sock->Write(&packet, &wopt);
    }
    if (span) {
        if (_current_call.nretry == 0) {
            span->set_sent_us(butil::cpuwide_time_us());
            span->set_request_size(packet_size);
        } else {
            span->Annotate("Requested(%lld) [%d]",
                           (long long)packet_size, _current_call.nretry + 1);
        }
    }
    if (using_auth) {
        // For performance concern, we set authentication to immediately
        // after the first `Write' returns instead of waiting for server
        // to confirm the credential data
        _current_call.sending_sock->SetAuthentication(rc);
    }

    // 解除对bthread_id_t的锁，确保收到响应时可以唤起
    CHECK_EQ(0, bthread_id_unlock(cid));
}
```

收到响应后, 主要调用栈如下
 - ProcessRpcResponse
   - ControllerPrivateAccessor::OnResponse
      - Controller::OnVersionedRPCReturned
         - bthread_id_about_to_destroy
         - EndRPC
            - bthread_timer_del
            - bthread_id_unlock_and_destroy - 唤醒调用rpc的bthread 