
[数据接收过程涉及到的主要数据结构](#数据接收过程涉及到的主要数据结构) 

[数据接收代码解析](#数据接收代码解析) 

## 数据接收过程涉及到的主要数据结构

1. Controller对象：存储一次完整的RPC请求的Context以及各种状态。相对于client端，server端做的事主要是处理请求并做出响应(如需)，可以立即被执行，至于rpc处理过程中的其他调用什么的，则是在另外的数据结构里被处理

## 数据接收代码解析

在["从TCP连接读取数据的并发处理"](io_read.md)这一节里，我们了解到socket接收到请求后判断协议类型, 如果是brpc请求会进入到`ProcessRpcRequest`里, 在这个函数里会进行相应的限流判断，然后执行业务代码，处理请求并返回数据。

```C++
void ProcessRpcRequest(InputMessageBase* msg_base) {
    const int64_t start_parse_us = butil::cpuwide_time_us();
    DestroyingPtr<MostCommonMessage> msg(static_cast<MostCommonMessage*>(msg_base));
    SocketUniquePtr socket_guard(msg->ReleaseSocket());
    Socket* socket = socket_guard.get();
    const Server* server = static_cast<const Server*>(msg_base->arg());
    ScopedNonServiceError non_service_error(server);

    // 解析数据
    RpcMeta meta;
    if (!ParsePbFromIOBuf(&meta, msg->meta)) {
        LOG(WARNING) << "Fail to parse RpcMeta from " << *socket;
        socket->SetFailed(EREQUEST, "Fail to parse RpcMeta from %s",
                          socket->description().c_str());
        return;
    }
    const RpcRequestMeta &request_meta = meta.request();

    SampledRequest* sample = AskToBeSampled();
    if (sample) {
        sample->meta.set_service_name(request_meta.service_name());
        sample->meta.set_method_name(request_meta.method_name());
        sample->meta.set_compress_type((CompressType)meta.compress_type());
        sample->meta.set_protocol_type(PROTOCOL_BAIDU_STD);
        sample->meta.set_attachment_size(meta.attachment_size());
        sample->meta.set_authentication_data(meta.authentication_data());
        sample->request = msg->payload;
        sample->submit(start_parse_us);
    }

    // 生成controller数据管理整个rpc处理过程
    std::unique_ptr<Controller> cntl(new (std::nothrow) Controller);
    if (NULL == cntl.get()) {
        LOG(WARNING) << "Fail to new Controller";
        return;
    }
    std::unique_ptr<google::protobuf::Message> req;
    std::unique_ptr<google::protobuf::Message> res;

    ServerPrivateAccessor server_accessor(server);
    ControllerPrivateAccessor accessor(cntl.get());
    const bool security_mode = server->options().security_mode() &&
                               socket->user() == server_accessor.acceptor();
    if (request_meta.has_log_id()) {
        cntl->set_log_id(request_meta.log_id());
    }
    if (request_meta.has_request_id()) {
        cntl->set_request_id(request_meta.request_id());
    }
    if (request_meta.has_timeout_ms()) {
        cntl->set_timeout_ms(request_meta.timeout_ms());
    }
    cntl->set_request_compress_type((CompressType)meta.compress_type());
    accessor.set_server(server)
        .set_security_mode(security_mode)
        .set_peer_id(socket->id())
        .set_remote_side(socket->remote_side())
        .set_local_side(socket->local_side())
        .set_auth_context(socket->auth_context())
        .set_request_protocol(PROTOCOL_BAIDU_STD)
        .set_begin_time_us(msg->received_us())
        .move_in_server_receiving_sock(socket_guard);

    if (meta.has_stream_settings()) {
        accessor.set_remote_stream_settings(meta.release_stream_settings());
    }

    // Tag the bthread with this server's key for thread_local_data().
    if (server->thread_local_options().thread_local_data_factory) {
        bthread_assign_data((void*)&server->thread_local_options());
    }

    Span* span = NULL;
    if (IsTraceable(request_meta.has_trace_id())) {
        span = Span::CreateServerSpan(
            request_meta.trace_id(), request_meta.span_id(),
            request_meta.parent_span_id(), msg->base_real_us());
        accessor.set_span(span);
        span->set_log_id(request_meta.log_id());
        span->set_remote_side(cntl->remote_side());
        span->set_protocol(PROTOCOL_BAIDU_STD);
        span->set_received_us(msg->received_us());
        span->set_start_parse_us(start_parse_us);
        span->set_request_size(msg->payload.size() + msg->meta.size() + 12);
    }

    MethodStatus* method_status = NULL;
    do {
        // 限流等相关保护，避免被打崩
        if (!server->IsRunning()) {
            cntl->SetFailed(ELOGOFF, "Server is stopping");
            break;
        }

        if (socket->is_overcrowded()) {
            cntl->SetFailed(EOVERCROWDED, "Connection to %s is overcrowded",
                            butil::endpoint2str(socket->remote_side()).c_str());
            break;
        }
        
        // 我曾经遇到过上游数据异常重试打崩下游导致下游无法启动(一启动就cpu超限)的情况, 
        // 设置这个可以保证下游服务可以正常启动, 配合hpa可以尽量恢复服务可用性
        if (!server_accessor.AddConcurrency(cntl.get())) {
            cntl->SetFailed(
                ELIMIT, "Reached server's max_concurrency=%d",
                server->options().max_concurrency);
            break;
        }

        if (FLAGS_usercode_in_pthread && TooManyUserCode()) {
            cntl->SetFailed(ELIMIT, "Too many user code to run when"
                            " -usercode_in_pthread is on");
            break;
        }

        // NOTE(gejun): jprotobuf sends service names without packages. So the
        // name should be changed to full when it's not.
        butil::StringPiece svc_name(request_meta.service_name());
        if (svc_name.find('.') == butil::StringPiece::npos) {
            const Server::ServiceProperty* sp =
                server_accessor.FindServicePropertyByName(svc_name);
            if (NULL == sp) {
                cntl->SetFailed(ENOSERVICE, "Fail to find service=%s",
                                request_meta.service_name().c_str());
                break;
            }
            svc_name = sp->service->GetDescriptor()->full_name();
        }
        const Server::MethodProperty* mp =
            server_accessor.FindMethodPropertyByFullName(
                svc_name, request_meta.method_name());
        if (NULL == mp) {
            cntl->SetFailed(ENOMETHOD, "Fail to find method=%s/%s",
                            request_meta.service_name().c_str(),
                            request_meta.method_name().c_str());
            break;
        } else if (mp->service->GetDescriptor()
                   == BadMethodService::descriptor()) {
            BadMethodRequest breq;
            BadMethodResponse bres;
            breq.set_service_name(request_meta.service_name());
            mp->service->CallMethod(mp->method, cntl.get(), &breq, &bres, NULL);
            break;
        }
        // Switch to service-specific error.
        non_service_error.release();
        method_status = mp->status;
        if (method_status) {
            int rejected_cc = 0;
            if (!method_status->OnRequested(&rejected_cc, cntl.get())) {
                cntl->SetFailed(ELIMIT, "Rejected by %s's ConcurrencyLimiter, concurrency=%d",
                                mp->method->full_name().c_str(), rejected_cc);
                break;
            }
        }
        google::protobuf::Service* svc = mp->service;
        const google::protobuf::MethodDescriptor* method = mp->method;
        accessor.set_method(method);
        if (span) {
            span->ResetServerSpanName(method->full_name());
        }
        const int req_size = static_cast<int>(msg->payload.size());
        butil::IOBuf req_buf;
        butil::IOBuf* req_buf_ptr = &msg->payload;
        if (meta.has_attachment_size()) {
            if (req_size < meta.attachment_size()) {
                cntl->SetFailed(EREQUEST,
                    "attachment_size=%d is larger than request_size=%d",
                     meta.attachment_size(), req_size);
                break;
            }
            int body_without_attachment_size = req_size - meta.attachment_size();
            msg->payload.cutn(&req_buf, body_without_attachment_size);
            req_buf_ptr = &req_buf;
            cntl->request_attachment().swap(msg->payload);
        }

        CompressType req_cmp_type = (CompressType)meta.compress_type();
        req.reset(svc->GetRequestPrototype(method).New());
        if (!ParseFromCompressedData(*req_buf_ptr, req.get(), req_cmp_type)) {
            cntl->SetFailed(EREQUEST, "Fail to parse request message, "
                            "CompressType=%s, request_size=%d", 
                            CompressTypeToCStr(req_cmp_type), req_size);
            break;
        }
        
        res.reset(svc->GetResponsePrototype(method).New());
        // 由此可知, done->Run()的过程，!!!实际上会执行响应发送!!!
        // `socket' will be held until response has been sent
        google::protobuf::Closure* done = ::brpc::NewCallback<
            int64_t, Controller*, const google::protobuf::Message*,
            const google::protobuf::Message*, const Server*,
            MethodStatus*, int64_t>(
                &SendRpcResponse, meta.correlation_id(), cntl.get(), 
                req.get(), res.get(), server,
                method_status, msg->received_us());

        // optional, just release resource ASAP
        msg.reset();
        req_buf.clear();

        if (span) {
            span->set_start_callback_us(butil::cpuwide_time_us());
            span->AsParent();
        }

        // 执行业务代码
        if (!FLAGS_usercode_in_pthread) {
            return svc->CallMethod(method, cntl.release(), 
                                   req.release(), res.release(), done);
        }
        if (BeginRunningUserCode()) {
            svc->CallMethod(method, cntl.release(), 
                            req.release(), res.release(), done);
            return EndRunningUserCodeInPlace();
        } else {
            return EndRunningCallMethodInPool(
                svc, method, cntl.release(),
                req.release(), res.release(), done);
        }
    } while (false);
    
    // `cntl', `req' and `res' will be deleted inside `SendRpcResponse'
    // `socket' will be held until response has been sent
    SendRpcResponse(meta.correlation_id(), cntl.release(), 
                    req.release(), res.release(), server,
                    method_status, msg->received_us());
}
```