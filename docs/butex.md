[bthread粒度挂起与唤醒的设计原理](#bthread粒度挂起与唤醒的设计原理)

[brpc中Butex的源码解释](#brpc中Butex的源码解释)

官方定义：Provides futex-like semantics which is sequenced wait and wake operations and guaranteed visibilities.
大概可以理解为是brpc实现的在bthread层面的futex

## bthread粒度挂起与唤醒的设计原理
由于bthread任务是在pthread系统线程中执行，在需要bthread间互斥的场景下不能使用pthread级别的锁（如pthread_mutex_lock或者C++的unique_lock等），否则pthread会被挂起，不仅当前的bthread中止执行，pthread私有的TaskGroup的任务队列中其他bthread也无法在该pthread上调度执行。因此需要在应用层实现bthread粒度的互斥机制，一个bthread被挂起时，pthread仍然要保持运行状态，保证TaskGroup任务队列中的其他bthread的正常执行不受影响。

要实现bthread粒度的互斥，方案如下：

1. 在同一个pthread上执行的多个bthread是串行执行的，不需要考虑互斥；

2. 如果位于heap内存上或static静态区上的一个对象A可能会被在不同pthread执行的多个bthread同时访问，则为对象A维护一个互斥锁（一般是一个原子变量）和等待队列，同时访问对象A的多个bthread首先要竞争锁，假设三个bthread 1、2、3分别在pthread 1、2、3上执行，bthread 1、bthread 2、bthread 3同时访问heap内存上的一个对象A，这时就产生了竞态，假设bthread 1获取到锁，可以去访问对象A，bthread 2、bthread 3先将自身必要的信息（bthread的tid等）存入等待队列，然后自动yiled，让出cpu，让pthread 2、pthread 3继续去执行各自私有TaskGroup的任务队列中的下一个bthread，这就实现了bthread粒度的挂起；

3. bthread 1访问完对象A后，通过查询对象A的互斥锁的等待队列，能够得知bthread 2、bthread 3因等待锁而被挂起，bthread 1负责将bthread 2、3的tid重新压入某个pthread（不一定是之前执行执行bthread 2、3的pthread 2、3）的TaskGroup的任务队列，bthread 2、3就能够再次被pthread执行，这就实现了bthread粒度的唤醒。

下面分析下brpc是如何实现bthread粒度的挂起与唤醒的。

## brpc中Butex的源码解释
brpc实现bthread互斥的主要代码在src/bthread/butex.cpp中：

1. 首先解释下Butex、ButexBthreadWaiter等主要的数据结构：

   ```c++
   struct BAIDU_CACHELINE_ALIGNMENT Butex {
       Butex() {}
       ~Butex() {}

       // 锁变量的值。
       butil::atomic<int> value;
       // 等待队列，存储等待互斥锁的各个bthread的信息。
       ButexWaiterList waiters;
       internal::FastPthreadMutex waiter_lock;
   };
   ```
   
   ```c++
   // 等待队列实际上是个侵入式双向链表，增减元素的操作都可在O(1)时间内完成。
   typedef butil::LinkedList<ButexWaiter> ButexWaiterList;
   ```
   
   ```c++
   // ButexWaiter是LinkNode的子类，LinkNode里只定义了指向前后节点的指针。
   struct ButexWaiter : public butil::LinkNode<ButexWaiter> {
       // Butex实现了bthread间的挂起&唤醒，也实现了bthread和pthread间的挂起&唤醒，
       // 一个pthread在需要的时候可以挂起，等待适当的时候被一个bthread唤醒，线程挂起不需要tid，填0即可。
       // pthread被bthread唤醒的例子可参考brpc的example目录下的一些client.cpp示例程序，执行main函数的pthread
       // 会被挂起，某个bthread执行完自己的任务后会去唤醒pthread。
       bthread_t tid;

       // Erasing node from middle of LinkedList is thread-unsafe, we need
       // to hold its container's lock.
       butil::atomic<Butex*> container;
   };
   ```
   
   ```c++
   // bthread需要挂起时，会在栈上创建一个ButexBthreadWaiter对象（对象存储在bthread的私有栈空间内）并加入等待队列。
   struct ButexBthreadWaiter : public ButexWaiter {
       // 执行bthread的TaskMeta结构的指针。
       TaskMeta* task_meta;
       TimerThread::TaskId sleep_id;
       // 状态标记，根据锁变量当前状态是否发生改变，waiter_state会被设为不同的值。
       WaiterState waiter_state;
       // expected_value存储的是当bthread竞争互斥锁失败时锁变量的值，由于从bthread竞争互斥锁失败到bthread挂起
       // 有一定的时间间隔，在这个时间间隔内锁变量的值可能会发生变化，也许锁已经被释放了，那么之前竞争锁失败的bthread
       // 就不应挂起，否则可能永远不会被唤醒了，它应该放弃挂起动作，再去竞争互斥锁。所以一个bthread在执行挂起动作前
       // 一定要再次去查看锁变量的当前最新值，只有锁变量当前最新值等于expected_value时才能真正执行挂起动作。
       int expected_value;
       Butex* initial_butex;
       // 指向全局唯一的TaskControl单例对象的指针。
       TaskControl* control;
       const timespec* abstime;
   };
   ```
   
   ```c++
   // 如果是pthread挂起，则创建ButexPthreadWaiter对象并加入等待队列。
   struct ButexPthreadWaiter : public ButexWaiter {
       butil::atomic<int> sig;
   };
   ```
   
   以上面的bthread 1获得了互斥锁、bthread 2和bthread 3因等待互斥锁而被挂起的场景为例，Butex的内存布局如下图所示，展现了主要的对象间的内存关系，注意ButexBthreadWaiter变量是分配在bthread的私有栈上的：(不画任务函数会好很多，有些杂乱)
   
   <img src="../images/butex_1.png"/>

2. 执行bthread挂起的函数是butex_wait：

   ```c++
    // arg是指向Butex::value锁变量的指针，expected_value是bthread竞争锁失败时锁变量的值, abstime是超市时长
    int butex_wait(void* arg, int expected_value, const timespec* abstime) {
        // 通过arg定位到Butex对象的地址
        Butex* b = container_of(static_cast<butil::atomic<int>*>(arg), Butex, value);

        if (b->value.load(butil::memory_order_relaxed) != expected_value) {
            // 如果锁变量当前最新值不等于expected_value, 则锁的状态发生了变化, 直接返回，在外层代码中继续去竞争锁
            errno = EWOULDBLOCK;
            return -1;
        }

        // 忽略了pthread的情况

        // 创建ButexBthreadWaiter类型的局部变量bbw，bbw是分配在bthread的私有栈空间上的
        ButexBthreadWaiter bbw;
        bbw.tid = g->current_tid();
        bbw.container.store(NULL, butil::memory_order_relaxed);
        bbw.task_meta = g->current_task();
        bbw.sleep_id = 0;
        bbw.waiter_state = WAITER_STATE_READY;
        bbw.expected_value = expected_value;
        bbw.initial_butex = b;
        bbw.control = g->control();
        bbw.abstime = abstime;

        // pthread在执行任务队列中下一个bthread前，会先执行wait_for_butex()将刚创建的bbw对象放入锁的等待队列
        g->set_remained(wait_for_butex, &bbw);
        // 当前bthread yield让出cpu，pthread会从TaskGroup的任务队列中取出下一个bthread去执行
        TaskGroup::sched(&g);

        // 这里是butex_wait()恢复执行时的开始执行点, 可能抢到锁, 也可能定时器到了

        // 自旋取消定时器
        BT_LOOP_WHEN(unsleep_if_necessary(&bbw, get_global_timer_thread()) < 0, 30);
        
        // 和TaskGroup::interrupt(只在bthread_interrupt时会触发)相呼应, 自旋等待current_waiter不是NULL
        BT_LOOP_WHEN(bbw.task_meta->current_waiter.exchange(
                        NULL, butil::memory_order_acquire) == NULL, 30);

        bool is_interrupted = false;
        if (bbw.task_meta->interrupted) {
            bbw.task_meta->interrupted = false;
            is_interrupted = true;
        }
        if (WAITER_STATE_TIMEDOUT == bbw.waiter_state) {
            errno = ETIMEDOUT;
            return -1;
        } else if (WAITER_STATE_UNMATCHEDVALUE == bbw.waiter_state) {
            errno = EWOULDBLOCK;
            return -1;
        } else if (is_interrupted) {
            errno = EINTR;
            return -1;
        }
        return 0;
    }
   ```
   
   ```c++
   static void wait_for_butex(void* arg) {
       ButexBthreadWaiter* const bw = static_cast<ButexBthreadWaiter*>(arg);
       Butex* const b = bw->initial_butex;
       {
           BAIDU_SCOPED_LOCK(b->waiter_lock);
           // 再次判断锁变量的当前最新值是否与expected_value相等。
           if (b->value.load(butil::memory_order_relaxed) != bw->expected_value) {
               // 锁变量的状态发生了变化，bw代表的bthread不能挂起，要去重新竞争锁。
               // 但bw代表的bthread之前已经yield让出cpu了，所以下面要将bw代表的bthread的id再次放入TaskGroup的
               // 任务队列，让它恢复执行。
               // 将bw的waiter_state更改为WAITER_STATE_UNMATCHEDVALUE，表示锁的状态已发生改变。
               bw->waiter_state = WAITER_STATE_UNMATCHEDVALUE;
           } else if (bw->waiter_state == WAITER_STATE_READY/*1*/ &&
                      !bw->task_meta->interrupted) {
               // 将bw加入到锁的等待队列，这才真正完成bthread的挂起，然后直接返回。
               b->waiters.Append(bw);
               bw->container.store(b, butil::memory_order_relaxed);
               if (bw->abstime != NULL) {
                   bw->sleep_id = get_global_timer_thread()->schedule(
                       erase_from_butex_and_wakeup, bw, *bw->abstime);
                   if (!bw->sleep_id) {  // TimerThread stopped.
                       errno = ESTOP;
                       erase_from_butex_and_wakeup(bw);
                   }
               }
               return;
           }
       }
       
       // 锁状态发生变化的情况下，才执行后面的代码。

       // b->container is NULL which makes erase_from_butex_and_wakeup() and
       // TaskGroup::interrupt() no-op, there's no race between following code and
       // the two functions. The on-stack ButexBthreadWaiter is safe to use and
       // bw->waiter_state will not change again.
       // unsleep_if_necessary(bw, get_global_timer_thread());
       // 将bw代表的bthread的tid重新加入TaskGroup的任务队列。
       tls_task_group->ready_to_run(bw->tid);
       // FIXME: jump back to original thread is buggy.

       // // Value unmatched or waiter is already woken up by TimerThread, jump
       // // back to original bthread.
       // TaskGroup* g = tls_task_group;
       // ReadyToRunArgs args = { g->current_tid(), false };
       // g->set_remained(TaskGroup::ready_to_run_in_worker, &args);
       // // 2: Don't run remained because we're already in a remained function
       // //    otherwise stack may overflow.
       // TaskGroup::sched_to(&g, bw->tid, false/*2*/);
   }
   ```

3. 执行bthread唤醒的函数有butex_wake（唤醒正在等待一个互斥锁的一个bthread）、butex_wake_all（唤醒正在等待一个互斥锁的所有bthread）、butex_wake_except（唤醒正在等待一个互斥锁的除了指定bthread外的其他bthread），下面解释butex_wake的源码：

   ```c++
   int butex_wake(void* arg) {
       // arg是Butex::value锁变量的地址, 定位到Butex对象的地址。
       Butex* b = container_of(static_cast<butil::atomic<int>*>(arg), Butex, value);
       ButexWaiter* front = NULL;
       {
           BAIDU_SCOPED_LOCK(b->waiter_lock);
           // 如果锁的等待队列为空，直接返回。
           if (b->waiters.empty()) {
               return 0;
           }
           // 取出锁的等待队列中第一个ButexWaiter对象的指针，并将该ButexWaiter对象从等待队列中移除。
           front = b->waiters.head()->value();
           front->RemoveFromList();
           front->container.store(NULL, butil::memory_order_relaxed);
       }

       // 忽略了pthread的情况

       ButexBthreadWaiter* bbw = static_cast<ButexBthreadWaiter*>(front);

       // 自旋取消定时器
       unsleep_if_necessary(bbw, get_global_timer_thread());

       // 将挂起的bthread的tid压入TaskGroup的任务队列，实现了将挂起的bthread唤醒。
       TaskGroup* g = get_task_group(bbw->control, nosignal);
       if (g == tls_task_group) {
           run_in_local_task_group(g, bbw->tid, nosignal);
       } else {
           g->ready_to_run_remote(bbw->tid, nosignal);
       }
       return 1;
   }
   ```
