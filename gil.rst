GIL
############

1. http://www.dabeaz.com/python/GIL.pdf

2. http://www.dabeaz.com/python/NewGIL.pdf

3. http://www.dabeaz.com/python/UnderstandingGIL.pdf

线程之间切换必须拿到GIL这样一个全局锁, 所以每次只有一个线程正在运行. 关于GIL请查看参考1, 2, 3, David写的ppt更详细

优化方向: 子解释器/多解释器(https://www.python.org/dev/peps/pep-0554/)?

GIL结构
=========

*The GIL is just a boolean variable (gil_locked) whose access is protected by a mutex (gil_mutex), and whose changes are signalled by a condition
variable (gil_cond). gil_mutex is taken for short periods of time, and therefore mostly uncontended.*


显然GIL只是一个真假值, 访问它需要获取锁, 并且修改该值之后会通过Condtion来通知其他线程, 而其他线程获取到GIl之后会通知当前释放GIl的线程

.. code-block:: c++

    // 这个就是GIl了
    static _Py_atomic_int gil_locked = {-1};
    // 当前拿住GIL的线程是谁
    static _Py_atomic_address gil_last_holder = {0};

只有修改gil_locked的时候才需要锁而读取该值是不需要锁的, 切换gil的时候, 获取gil的线程会设置gil_last_holder为自己线程的tstate结构

.. code-block:: c++

    static COND_T gil_cond;
    static MUTEX_T gil_mutex;

gil_mutext就是访问gil_locked之前必须要拿到的锁, 而gil_cond和gil_mutext两者结合起来就是一个等待机制, 所谓的Condition, 这个机制则是支持多个

线程对同一个锁进行等待, 锁未被释放之前会一直等待着(当然可以设置超时). 一旦该锁被释放, 多个线程则取抢占直到某个线程获取到锁, 其他未抢到锁的线程

则继续等待. 多个线程谁能拿到锁有些实现是抢占, 有些实现是基于排队先到先得(比如CPython中的threading.Condition就是排队)


获取和释放GIL
===================

获取和释放GIL并没有什么特别的, 大概流程就是拿锁, 拿不到则等待gil释放. 在当前的CPython中, 一个线程先等待5ms看看拿住gil的线程是否主动释放了gil

比如拿住gil的线程进入IO什么的就主动释放锁了, 否则设置一个强制释放的变量, gil_drop_request, 为真, 那么拿住gil的线程看到gil_drop_request为真则会

强行释放gil. 所以只有一个线程的话, 并不会发生gil切换, 如果有线程进入, 那么另外一个线程至少能运行5ms(当然是该线程不会进入IO), 这个5ms是可以设置的

.. code-block:: c++

    static void take_gil(PyThreadState *tstate)
    {
        // 拿个锁
        MUTEX_LOCK(gil_mutex);
    
        if (!_Py_atomic_load_relaxed(&gil_locked))
            goto _ready;
    
        // 读取到gil_locked=1, 表示gil还是锁状态
        while (_Py_atomic_load_relaxed(&gil_locked)) {
            int timed_out = 0;
            unsigned long saved_switchnum;
    
            saved_switchnum = gil_switch_number;
            // 这一句会释放gil_mutex, 同时在gil_cond上进行等待, 等待的时间为INTERVAL=5ms
            COND_TIMED_WAIT(gil_cond, gil_mutex, INTERVAL, timed_out);
            /* If we timed out and no switch occurred in the meantime, it is time
               to ask the GIL-holding thread to drop it. */
            // INTERVAL内没有释放gil, 那么显然要设置变量让其他线程强行让出gil
            if (timed_out &&
                _Py_atomic_load_relaxed(&gil_locked) &&
                // 这里判断如果gil_switch_number
                gil_switch_number == saved_switchnum) {
                SET_GIL_DROP_REQUEST();
            }
        }
        // 省略了代码
    }


这里有一个细节就是如果timeout了, 并且gil_locked是1, 那么判断gil_switch_number是否和之前一样, 如果一样表示没有gil切换, gil依然被其他线程拿住没有释放过, 那么继续设置gil_drop_request提醒其释放

否则表示gil已经被释放过了, 只是我们没拿到gil, 被其他线程拿到了, 那么我们继续等待5ms等待其他线程主动释放


释放gil很简单, 把gil_locked设置为0, 然后通知其他线程可以抢锁了

.. code-block:: c++

    static void drop_gil(PyThreadState *tstate)
    {
        if (!_Py_atomic_load_relaxed(&gil_locked))
            Py_FatalError("drop_gil: GIL is not locked");
        /* tstate is allowed to be NULL (early interpreter init) */
        if (tstate != NULL) {
            /* Sub-interpreter support: threads might have been switched
               under our feet using PyThreadState_Swap(). Fix the GIL last
               holder variable so that our heuristics work. */
            _Py_atomic_store_relaxed(&gil_last_holder, (uintptr_t)tstate);
        }
    
        MUTEX_LOCK(gil_mutex);
        // 设置gil_locked为0
        _Py_ANNOTATE_RWLOCK_RELEASED(&gil_locked, /*is_write=*/1);
        _Py_atomic_store_relaxed(&gil_locked, 0);
        // 通知其他线程可以抢占当前multex了
        COND_SIGNAL(gil_cond);
        // unlock之后其他线程进行抢占锁操作
        MUTEX_UNLOCK(gil_mutex);

        // 省略代码
    
    }


而判断什么时候需要take_gil, drop_gil是在解释器执行字节码的之前判断的

.. code-block:: c++

    // 默认执行字节码的程序
    PyObject *
    _PyEval_EvalFrameDefault()
    {
        // 执行字节码之前判断一下是否需要处理中断
        for (;;) {
        
            // eval_breaker表示有中断
            if (_Py_atomic_load_relaxed(&eval_breaker)) {
                // 下面查看中断类型

                // 有系统信号要处理
                if (_Py_atomic_load_relaxed(&pendingcalls_to_do)) {
                    if (Py_MakePendingCalls() < 0)
                        goto error;
                }


                // gil_drop_request为1表示有线程要求释放gil
                if (_Py_atomic_load_relaxed(&gil_drop_request)) {
                    if (PyThreadState_Swap(NULL) != tstate)
                        Py_FatalError("ceval: tstate mix-up");
                    // 那么我们就释放gil
                    drop_gil(tstate);

                    // 同时再次获取gil
                    take_gil(tstate); 

                    if (_Py_Finalizing && _Py_Finalizing != tstate) {
                        drop_gil(tstate);
                        PyThread_exit_thread();
                    }

                    if (PyThreadState_Swap(tstate) != NULL)
                        Py_FatalError("ceval: orphan tstate");
                }

                // 有异常要处理
                if (tstate->async_exc != NULL) {
                }
            }
            // 执行字节码

            switch(opcode){
            }
        }
    }

FORCE SWITCH
==============

在上面我们可以看到释放gil的时候就立即再次获取gil, 那么显然如果一个计算线程释放gil之后立马获取gil, 很可能自己又能再次拿到gil了

因为系统通知其他线程抢占锁之前, 当前线程又执行到take_gil了, 这样其他线程就无法切换了, 所以默认的, CPython中切换gil的时候释放gil的线程会等待其他线程拿到gil

之后drop_gil才完成


.. code-block:: c++

    static void drop_gil(PyThreadState *tstate)
    {
        // 省略释放gil的操作
        // 之后等待其他线程已经拿到gil了
    #ifdef FORCE_SWITCHING
        if (_Py_atomic_load_relaxed(&gil_drop_request) && tstate != NULL) {
            MUTEX_LOCK(switch_mutex);
            /* Not switched yet => wait */
            if ((PyThreadState*)_Py_atomic_load_relaxed(&gil_last_holder) == tstate) {
            RESET_GIL_DROP_REQUEST();
                /* NOTE: if COND_WAIT does not atomically start waiting when
                   releasing the mutex, another thread can run through, take
                   the GIL and drop it again, and reset the condition
                   before we even had a chance to wait for it. */
                COND_WAIT(switch_cond, switch_mutex);
        }
            MUTEX_UNLOCK(switch_mutex);
        }
    #endif
    }


FORCE_SWITCHING模式在CPython中是默认打开的, 然后释放完gil之后还没结束, 需要等待其他线程通知说已经拿到gil了

这里判断gil_drop_request是否还是1, 是1表示其他线程没有修改该值, 所以我们需要等待一下, 同时如果gil_last_holder依然是当前线程的tstate, 那么说明

其他线程没有拿到gil, 所以调用COND_WAIT等待其他线程通知说已经完成获取gil的操作了


在take_gil的时候通知释放gil的线程说自己已经被调度了

.. code-block:: c++

    static void take_gil(PyThreadState *tstate)
    {
    // 拿到了gil
    _ready:
    #ifdef FORCE_SWITCHING
        /* This mutex must be taken before modifying gil_last_holder (see drop_gil()). */
        MUTEX_LOCK(switch_mutex);
    #endif
        /* We now hold the GIL */
        // 拿到gil之后我们设置gil_locked为1
        _Py_atomic_store_relaxed(&gil_locked, 1);
        _Py_ANNOTATE_RWLOCK_ACQUIRED(&gil_locked, /*is_write=*/1);
    
        // 同时拿到gil之后把当前拿住gil的线程设置为自己, gil的切换数加1
        if (tstate != (PyThreadState*)_Py_atomic_load_relaxed(&gil_last_holder)) {
            _Py_atomic_store_relaxed(&gil_last_holder, (uintptr_t)tstate);
            ++gil_switch_number;
        }
    
    #ifdef FORCE_SWITCHING
        // 通知释放gil的线程我们已经完成获取gil之后的获取操作了
        COND_SIGNAL(switch_cond);
        MUTEX_UNLOCK(switch_mutex);
    #endif
        if (_Py_atomic_load_relaxed(&gil_drop_request)) {
            RESET_GIL_DROP_REQUEST();
        }
        if (tstate->async_exc != NULL) {
            _PyEval_SignalAsyncExc();
        }
    
        MUTEX_UNLOCK(gil_mutex);
        errno = err;
    }


手动释放GIL
===============

在CPython中很多IO操作都主动释放了GIL, 比如sleep

.. code-block:: c++

    static int
    pysleep(_PyTime_t secs)
    {
        _PyTime_t deadline, monotonic;
    #ifndef MS_WINDOWS
        struct timeval timeout;
        int err = 0;
    #else
    // 这里是windows平台的变量定义
    #endif
        deadline = _PyTime_GetMonotonicClock() + secs;
        do {
    #ifndef MS_WINDOWS
            if (_PyTime_AsTimeval(secs, &timeout, _PyTime_ROUND_CEILING) < 0)
                return -1;
            // 下面两个包裹着select的宏是先释放gil然后获取gil
            Py_BEGIN_ALLOW_THREADS
            // select系统调用
            err = select(0, (fd_set *)0, (fd_set *)0, (fd_set *)0, &timeout);
            Py_END_ALLOW_THREADS
            if (err == 0)
                break;
            if (errno != EINTR) {
                PyErr_SetFromErrno(PyExc_OSError);
                return -1;
            }
    #else
    // 里面是windows平台的处理
    #endif
            /* sleep was interrupted by SIGINT */
            if (PyErr_CheckSignals())
                return -1;
            monotonic = _PyTime_GetMonotonicClock();
            secs = deadline - monotonic;
            if (secs < 0)
                break;
            /* retry with the recomputed delay */
        } while (1);
        return 0;
    }

CPython中提供了两个宏操作Py_BEGIN_ALLOW_THREADS和Py_END_ALLOW_THREADS, 前者是释放GIL而后者是获取GIL, 两者之间的代码块就不受GIL影响了

.. code-block:: c++

    #define Py_BEGIN_ALLOW_THREADS { \
                            PyThreadState *_save; \
                            _save = PyEval_SaveThread();
    #define Py_END_ALLOW_THREADS    PyEval_RestoreThread(_save); \
                     }

    // 两者组合起来就是

    {
        PyThreadState *_save;
        // 这里释放gil
        _save_ = PyEval_SaveThread();
        // 你的C代码

        // 这里获取gil
        PyEval_RestoreThread(_save);
    }



但是要注意的是 **所有的python代码都需要在gil的保护下操作, 所以这两个block之间的代码要确保只有c代码而没有python代码**



护航效应(Convoy Effect)
=================================

https://bugs.python.org/issue7946

http://www.dabeaz.com/blog/2010/02/revisiting-thread-priorities-and-new.html

当前的GIL是基于定时释放的, 如果一个IO线程和一个CPU密集型线程同时在执行, 一般地IO线程进入IO操作的时候会释放GIL, 此时CPU密集型线程进入执行

然后IO线程返回发起一个获取GIL的请求, 然后计算线程则释放GIL, 如此循环. 但是如果IO线程的IO操作立即返回而不是陷入等待的话, 那么显然释放GIL

会降低IO线程的性能的, 这种效应就是护航效应. 而Python2.x中IO线程的效率不会很低是因为Python2.x的GIl是基于抢占而不是基于等待的, 所以IO线程IO

操作立即完成之后又立即请求GIL, 这样IO线程的效率影响不是那么大, 虽然也有影响同时还增加了线程切换的次数, 这也是一笔开销

链接中的提到这样现象不仅仅是说IO和tcp或者udp有关, 而是只要IO操作(比如写入文件), 或者说不仅仅是IO操作, 而是任何立即完成但是需要释放GIL的操作

这样一个带有上述操作的线程和另外一个计算线程一起执行的时候, "IO"线程不可避免地性能下降

链接中提到了可以在解释器中进入IO的时候如果IO立即返回则不释放GIL(比如设置IO操作为异步, 根据返回的判断码就知道IO是否立即完成)

这个修复是最简单但也是一种sneaky的方式, 同时David Beazley提出了一个基于优先级抢占的策略, 能优化护航效应, 而PyPy使用了一种类似于David的方法避免了类似的问题

**当然结果是还没有定论如何修复, 因为这个不是什么"大问题", 因为并没有很多很多人去抱怨这个问题.....软件开发的"智慧"**


子解释器
=============

https://www.python.org/dev/peps/pep-0554/

https://lwn.net/Articles/820424/

