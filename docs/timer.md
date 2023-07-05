[TimerThread设计理念](#TimerThread设计理念)

[代码实现](#代码实现)

## TimerThread设计理念

- 一个TimerThread而不是多个。
- 创建的timer散列到多个Bucket以降低线程间的竞争，默认13个Bucket。
- Bucket内不使用小顶堆管理时间，而是链表 + nearest_run_time字段，当插入的时间早于nearest_run_time时覆盖这个字段，之后去和全局nearest_run_time（和Bucket的nearest_run_time不同）比较，如果也早于这个时间，修改并唤醒TimerThread。链表节点在锁外使用ResourcePool分配。
- 删除时通过id直接定位到timer内存结构，修改一个标志，timer结构总是由TimerThread释放。
- TimerThread被唤醒后首先把全局nearest_run_time设置为几乎无限大(max of int64)，然后取出所有Bucket内的链表，并把Bucket的nearest_run_time设置为几乎无限大(max of int64)。TimerThread把未删除的timer插入小顶堆中维护，这个堆就它一个线程用。在每次运行回调或准备睡眠前都会检查全局nearest_run_time， 如果全局更早，说明有更早的时间加入了，重复这个过程。

## 代码实现

添加新的定时器
```C++
TimerThread::TaskId TimerThread::schedule(
    void (*fn)(void*), void* arg, const timespec& abstime) {
    if (_stop.load(butil::memory_order_relaxed) || !_started) {
        // Not add tasks when TimerThread is about to stop.
        return INVALID_TASK_ID;
    }
    // 根据pthread_id哈希找到bucket, 可以保证几乎无冲突
    // Hashing by pthread id is better for cache locality.
    const Bucket::ScheduleResult result = 
        _buckets[butil::fmix64(pthread_numeric_id()) % _options.num_buckets]
        .schedule(fn, arg, abstime);
    if (result.earlier) {
        bool earlier = false;
        const int64_t run_time = butil::timespec_to_microseconds(abstime);
        {
            BAIDU_SCOPED_LOCK(_mutex);
            if (run_time < _nearest_run_time) {
                _nearest_run_time = run_time;
                ++_nsignals;
                earlier = true;
            }
        }
        if (earlier) {
            // 过期时间更早, 唤醒timer thread
            futex_wake_private(&_nsignals, 1);
        }
    }
    return result.task_id;
}

TimerThread::Bucket::ScheduleResult
TimerThread::Bucket::schedule(void (*fn)(void*), void* arg,
                              const timespec& abstime) {
    butil::ResourceId<Task> slot_id;
    Task* task = butil::get_resource<Task>(&slot_id);
    if (task == NULL) {
        ScheduleResult result = { INVALID_TASK_ID, false };
        return result;
    }
    task->next = NULL;
    task->fn = fn;
    task->arg = arg;
    task->run_time = butil::timespec_to_microseconds(abstime);
    uint32_t version = task->version.load(butil::memory_order_relaxed);
    if (version == 0) {  // skip 0.
        task->version.fetch_add(2, butil::memory_order_relaxed);
        version = 2;
    }
    const TaskId id = make_task_id(slot_id, version);
    task->task_id = id;
    bool earlier = false;
    {
        BAIDU_SCOPED_LOCK(_mutex);
        // 把新任务挂在链表头
        task->next = _task_head;
        _task_head = task;
        if (task->run_time < _nearest_run_time) {
            _nearest_run_time = task->run_time;
            earlier = true;
        }
    }
    ScheduleResult result = { id, earlier };
    return result;
}
```

取消定时器
```C++
int TimerThread::unschedule(TaskId task_id) {
    const butil::ResourceId<Task> slot_id = slot_of_task_id(task_id);
    Task* const task = butil::address_resource(slot_id);
    if (task == NULL) {
        LOG(ERROR) << "Invalid task_id=" << task_id;
        return -1;
    }
    const uint32_t id_version = version_of_task_id(task_id);
    uint32_t expected_version = id_version;
    // This CAS is rarely contended, should be fast.
    // The acquire fence is paired with release fence in Task::run_and_delete
    // to make sure that we see all changes brought by fn(arg).
    if (task->version.compare_exchange_strong(
            expected_version, id_version + 2,
            butil::memory_order_acquire)) {
        return 0;
    }
    return (expected_version == id_version + 1) ? 1 : -1;
}
```

定时器pthread线程函数:

```C++
void TimerThread::run() {
    ...

    int64_t last_sleep_time = butil::gettimeofday_us();

    // min heap of tasks (ordered by run_time)
    std::vector<Task*> tasks;
    tasks.reserve(4096);

    ...

    while (!_stop.load(butil::memory_order_relaxed)) {
        // 把全局nearest_run_time设置为几乎无限大(max of int64)
        // Clear _nearest_run_time before consuming tasks from buckets.
        // This helps us to be aware of earliest task of the new tasks before we
        // would run the consumed tasks.
        {
            BAIDU_SCOPED_LOCK(_mutex);
            _nearest_run_time = std::numeric_limits<int64_t>::max();
        }
        
        // 定时器从bucket里取出扔到小顶堆里
        // Pull tasks from buckets.
        for (size_t i = 0; i < _options.num_buckets; ++i) {
            Bucket& bucket = _buckets[i];
            for (Task* p = bucket.consume_tasks(); p != nullptr; ++nscheduled) {
                // p->next should be kept first
                // in case of the deletion of Task p which is unscheduled
                Task* next_task = p->next;

                // 删除定时器, 归还资源到resource pool
                if (!p->try_delete()) { // remove the task if it's unscheduled
                    // 扔到小顶堆里
                    tasks.push_back(p);
                    std::push_heap(tasks.begin(), tasks.end(), task_greater);
                }
                p = next_task;
            }
        }

        bool pull_again = false;
        while (!tasks.empty()) {
            Task* task1 = tasks[0];  // the about-to-run task
            if (butil::gettimeofday_us() < task1->run_time) {  // not ready yet.
                break;
            }
            // Each time before we run the earliest task (that we think), 
            // check the globally shared _nearest_run_time. If a task earlier
            // than task1 was scheduled during pulling from buckets, we'll
            // know. In RPC scenarios, _nearest_run_time is not often changed by
            // threads because the task needs to be the earliest in its bucket,
            // since run_time of scheduled tasks are often in ascending order,
            // most tasks are unlikely to be "earliest". (If run_time of tasks
            // are in descending orders, all tasks are "earliest" after every
            // insertion, and they'll grab _mutex and change _nearest_run_time
            // frequently, fortunately this is not true at most of time).
            {
                BAIDU_SCOPED_LOCK(_mutex);
                if (task1->run_time > _nearest_run_time) {
                    // 可能从bucket取出定时器的时候插入了执行时间更早的定时器
                    // a task is earlier than task1. We need to check buckets.
                    pull_again = true;
                    break;
                }
            }
            // task1已经超时了, 立即执行, 执行完归还资源
            std::pop_heap(tasks.begin(), tasks.end(), task_greater);
            tasks.pop_back();
            // !!!注意是在pthread上直接执行的!!!, 很危险
            // 实际使用的时候可以考虑起一个bthread去实际执行定时器函数
            if (task1->run_and_delete()) {
                ++ntriggered;
            }
        }
        if (pull_again) {
            BT_VLOG << "pull again, tasks=" << tasks.size();
            continue;
        }

        // The realtime to wait for.
        int64_t next_run_time = std::numeric_limits<int64_t>::max();
        if (!tasks.empty()) {
            next_run_time = tasks[0]->run_time;
        }
        // Similarly with the situation before running tasks, we check
        // _nearest_run_time to prevent us from waiting on a non-earliest
        // task. We also use the _nsignal to make sure that if new task 
        // is earlier that the realtime that we wait for, we'll wake up.
        // 可能执行上面操作的时候插入了执行时间更早的定时器
        int expected_nsignals = 0;
        {
            BAIDU_SCOPED_LOCK(_mutex);
            if (next_run_time > _nearest_run_time) {
                // a task is earlier that what we would wait for.
                // We need to check buckets.
                continue;
            } else {
                _nearest_run_time = next_run_time;
                expected_nsignals = _nsignals;
            }
        }
        timespec* ptimeout = NULL;
        timespec next_timeout = { 0, 0 };
        const int64_t now = butil::gettimeofday_us();
        if (next_run_time != std::numeric_limits<int64_t>::max()) {
            next_timeout = butil::microseconds_to_timespec(next_run_time - now);
            ptimeout = &next_timeout;
        }
        busy_seconds += (now - last_sleep_time) / 1000000.0;
        // sleep || 等待其他线程唤醒
        futex_wait_private(&_nsignals, expected_nsignals, ptimeout);
        last_sleep_time = butil::gettimeofday_us();
    }
    BT_VLOG << "Ended TimerThread=" << pthread_self();
}
```
