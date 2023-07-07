[Brpc重试](#Brpc重试)

[BackupRequest](#BackupRequest)


## Brpc重试

异常的处理是在OnVersionedRPCReturned函数, 它会在如下场景被调用
   - 收到响应数据 - ControllerPrivateAccessor::OnResponse
   - 发送失败 - HandleSendFailed
   - Socket错误 - HandleSocketFailed

```C++
...
// 在 OnVersionedRPCReturned 中
// 调用_retry_policy判断错误码是否需要重试
if (_retry_policy ? _retry_policy->DoRetry(this)
               : DefaultRetryPolicy()->DoRetry(this)) {
        // The error must come from _current_call because:
        //  * we intercepted error from _unfinished_call in OnVersionedRPCReturned
        //  * ERPCTIMEDOUT/ECANCELED are not retrying error by default.
        CHECK_EQ(current_id(), info.id) << "error_code=" << _error_code;
        ...
        _current_call.OnComplete(this, _error_code, info.responded, false);
        // 添加重试次数, 会刺激call_id改变
        // CallId id = { _correlation_id.value + _current_call.nretry + 1 };
        ++_current_call.nretry;
        ...

        // 重新发起rpc调用
        return IssueRPC(butil::gettimeofday_us());
    }
...

```
brpc默认的重试策略如下，用户可以继承RetryPolicy，实现自己的重试策略
```C++
class RpcRetryPolicy : public RetryPolicy {
public:
    bool DoRetry(const Controller* controller) const {
        const int error_code = controller->ErrorCode();
        if (!error_code) {
            return false;
        }
        return (EFAILEDSOCKET == error_code
                || EEOF == error_code 
                || EHOSTDOWN == error_code 
                || ELOGOFF == error_code
                || ETIMEDOUT == error_code // This is not timeout of RPC.
                || ELIMIT == error_code
                || ENOENT == error_code
                || EPIPE == error_code
                || ECONNREFUSED == error_code
                || ECONNRESET == error_code
                || ENODATA == error_code
                || EOVERCROWDED == error_code
                || EH2RUNOUTSTREAMS == error_code);
    }
};
```

## BackupRequest

1. 什么是[Backup Request](#https://brpc.apache.org/zh/docs/client/backup-request)
    - 为了保证可用性，需要同时访问两路服务，哪个先返回就取哪个

2. 实际使用场景？
    - 个人感觉用在重要接口里面，它在`请求访问下游查询接口时`使用, 会比较好, 一来查询接口没有并行修改的问题, 二来查询接口一般支持的qps高，不易打崩下游，因此可以保证更高概率的查询成功和较低的查询耗时

3. 代码相关实现
```C++
void Channel::CallMethod(const google::protobuf::MethodDescriptor* method,
                         google::protobuf::RpcController* controller_base,
                         const google::protobuf::Message* request,
                         google::protobuf::Message* response,
                         google::protobuf::Closure* done) {
    ...

    // 在初始化channel时设置的
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

        // 起一个定时器，定时器到则开始发送 backup request
        // 其他请求这里类似的位置直接设计超时定时器
        const int rc = bthread_timer_add(
            &cntl->_timeout_id,
            butil::microseconds_to_timespec(
                cntl->backup_request_ms() * 1000L + start_send_real_us),
            HandleBackupRequest, (void*)correlation_id.value);
        if (BAIDU_UNLIKELY(rc != 0)) {
            cntl->SetFailed(rc, "Fail to add timer for backup request");
            return cntl->HandleSendFailed();
        }
    ...
}
```

`HandleBackupRequest` 调用 `HandleSocketFailed`“`设置错误码EBACKUPREQUEST`”后调用 `OnVersionedRPCReturned`

```C++
    // 在 OnVersionedRPCReturned 中
    ...
    
    if (_error_code == EBACKUPREQUEST) {
        // Reset timeout if needed
        // 启动超时定时器
        int rc = 0;
        if (timeout_ms() >= 0) {
            rc = bthread_timer_add(
                    &_timeout_id,
                    butil::microseconds_to_timespec(_deadline_us),
                    HandleTimeout, (void*)_correlation_id.value);
        }
        if (rc != 0) {
            SetFailed(rc, "Fail to add timer");
            goto END_OF_RPC;
        }
        if (!SingleServer()) {
            // 重新选一个server
            if (_accessed == NULL) {
                _accessed = ExcludedServers::Create(
                    std::min(_max_retry, RETRY_AVOIDANCE));
                if (NULL == _accessed) {
                    SetFailed(ENOMEM, "Fail to create ExcludedServers");
                    goto END_OF_RPC;
                }
            }
            _accessed->Add(_current_call.peer_id);
        }
        // _current_call does not end yet.
        CHECK(_unfinished_call == NULL);  // only one backup request now.
        _unfinished_call = new (std::nothrow) Call(&_current_call);
        if (_unfinished_call == NULL) {
            SetFailed(ENOMEM, "Fail to new Call");
            goto END_OF_RPC;
        }
        ++_current_call.nretry;
        add_flag(FLAGS_BACKUP_REQUEST);

        // 发送新的请求
        return IssueRPC(butil::gettimeofday_us());
    } 

    ...
```


















