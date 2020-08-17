Thread
############

CPython的线程是系统线程的一个包装, 调度上还是依赖于操作系统, 只是线程执行之前需要获取全局锁, 所谓的GIL, 所以多核下同时只能有一个线程运行

但是可以把程序写成C代码然后手动释放GIL, 那么这样多核并行也是可以的, 但是要注意任何python代码都要在GIL下运行


创建/启动/线程以及等待线程停止
===================================

当我们调用Thread.start的时候将会去启动一个系统线程, 同时等待线程启动的通知

.. code-block:: python

    class Thread
    
        def start(self):
            try:
                # 系统线程将会调用self._bootstrap
                _start_new_thread(self._bootstrap, ())
            except Exception:
                with _active_limbo_lock:
                    del _limbo[self]
                raise
            # 当线程调用self._bootstrap的时候, self._started.wait就会返回
            self._started.wait()

_start_new_thread则是创建一个bootstate去保存传入的func, args, kwarg等信息, 然后调用系统调用去创建和启动系统

.. code-block:: c

    static PyObject *
    thread_PyThread_start_new_thread(PyObject *self, PyObject *fargs)
    {
        PyObject *func, *args, *keyw = NULL;
        struct bootstate *boot;
        long ident;
    
        boot = PyMem_NEW(struct bootstate, 1);
        if (boot == NULL)
            return PyErr_NoMemory();
        // 保存当前解释器, 所以每个线程都使用同一个解释器
        boot->interp = PyThreadState_GET()->interp;
        boot->func = func;
        boot->args = args;
        boot->keyw = keyw;
        boot->tstate = _PyThreadState_Prealloc(boot->interp);
        if (boot->tstate == NULL) {
            PyMem_DEL(boot);
            return PyErr_NoMemory();
        }
        Py_INCREF(func);
        Py_INCREF(args);
        Py_XINCREF(keyw);
        PyEval_InitThreads(); /* Start the interpreter's thread-awareness */
        // 调用系统调用去创建和启动线程
        ident = PyThread_start_new_thread(t_bootstrap, (void*) boot);
        if (ident == -1) {
            PyErr_SetString(ThreadError, "can't start new thread");
            Py_DECREF(func);
            Py_DECREF(args);
            Py_XDECREF(keyw);
            PyThreadState_Clear(boot->tstate);
            PyMem_DEL(boot);
            return NULL;
        }
        return PyLong_FromLong(ident);
    }

可以看到每个线程对应一个bootstate结构, 其中共享了解释器对象, 然后PyThread_start_new_thread则是创建系统线程, 传入的函数是t_bootstrap

为什么不传入python的函数, 因为系统不能执行python代码呀, 所以要使用另外一个c函数去启动python代码, 这个c函数这里就是t_bootstrap

.. code-block:: c

    static void
    t_bootstrap(void *boot_raw)
    {
        struct bootstate *boot = (struct bootstate *) boot_raw;
        PyThreadState *tstate;
        PyObject *res;
    
        tstate = boot->tstate;
        tstate->thread_id = PyThread_get_thread_ident();
        _PyThreadState_Init(tstate);
        PyEval_AcquireThread(tstate);
        nb_threads++;
        // 执行python代码!!!!!!!!!!!!
        res = PyEval_CallObjectWithKeywords(
            boot->func, boot->args, boot->keyw);
    }

线程中执行我们的代码之前会先做一些准备处理, 在self._bootstrap_inner中

code-block:: python

    def _bootstrap_inner(self):
        try:
            self._set_ident()
            self._set_tstate_lock()
            # 通知self.start方法可以返回了
            self._started.set()
            with _active_limbo_lock:
                _active[self._ident] = self
                del _limbo[self]

            if _trace_hook:
                _sys.settrace(_trace_hook)
            if _profile_hook:
                _sys.setprofile(_profile_hook)

            try:
                self.run()
            except: pass # 省略代码

将会保存当前线程id, 设置该线程启动状态, 这样self.start才能返回, 说明我们的线程正在地被系统调用到了, 最后才会走到self.run中执行我们传入的函数

self.join是等待线程结束, 其是检查线程状态锁self._tstate_lock是否被c代码释放了, 如果被释放了表示线程已经进入了清理操作了

.. code-block:: python

    def join(self):
        if timeout is None:
            self._wait_for_tstate_lock()
        else:
            self._wait_for_tstate_lock(timeout=max(timeout, 0))

    def _wait_for_tstate_lock(self, block=True, timeout=-1):
        lock = self._tstate_lock
        if lock is None:  # already determined that the C code is done
            assert self._is_stopped
        # 拿到锁表示C代码已经终止了
        elif lock.acquire(block, timeout):
            lock.release()
            self._stop()

在C代码中负责释放线程状态锁self._tstate_lock, self._tstate_lock是在self._bootstrap_inner中创建的, 在C代码中将这个锁保存到tstate这个线程信息结构中

.. code-block:: python

    def _set_tstate_lock(self):
        # 调用C函数去创建锁
        self._tstate_lock = _set_sentinel()
        self._tstate_lock.acquire()

    def _bootstrap_inner(self):
        try:
            self._set_ident()
            self._set_tstate_lock()

.. code-block:: c

    static PyObject *
    thread__set_sentinel(PyObject *self)
    {
        PyObject *wr;
        // 获取当前线程的信息, tstate结构
        PyThreadState *tstate = PyThreadState_Get();
        lockobject *lock;
    
        // 创建一个锁
        lock = newlockobject();
        if (lock == NULL)
            return NULL;
        // 把这个系统的锁包装成python对象
        wr = PyWeakref_NewRef((PyObject *) lock, NULL);
        if (wr == NULL) {
            Py_DECREF(lock);
            return NULL;
        }
        // tstate保存好这个锁
        tstate->on_delete_data = (void *) wr;
        // on_delete表示释放的时候调用哪个函数, release_sentinel这个函数就是释放lock的
        tstate->on_delete = &release_sentinel;
        return (PyObject *) lock;
    }

在系统线程要退出的时候, 释放该锁


.. code-block:: c

    static void
    t_bootstrap(void *boot_raw)
    {
        struct bootstate *boot = (struct bootstate *) boot_raw;
        PyThreadState *tstate;
        PyObject *res;
    
        tstate = boot->tstate;
        tstate->thread_id = PyThread_get_thread_ident();
        _PyThreadState_Init(tstate);
        PyEval_AcquireThread(tstate);
        nb_threads++;
        res = PyEval_CallObjectWithKeywords(
            boot->func, boot->args, boot->keyw);
         // 调用结束后
        Py_DECREF(boot->func);
        Py_DECREF(boot->args);
        Py_XDECREF(boot->keyw);
        PyMem_DEL(boot_raw);
        nb_threads--;
        PyThreadState_Clear(tstate);
        // 释放我们的锁!!!!!!!!!!!!
        PyThreadState_DeleteCurrent();
        PyThread_exit_thread();
    }

    // PyThreadState_DeleteCurrent调用到tstate_delete_common, tstate_delete_common则调用on_delete函数, 也就是release_sentinel
    static void
    tstate_delete_common(PyThreadState *tstate)
    {
        // 调用release_sentinel
        if (tstate->on_delete != NULL) {
            tstate->on_delete(tstate->on_delete_data);
        }
        PyMem_RawFree(tstate);
    }

    // 真正释放锁的地方
    static void release_sentinel(void *wr)
    {
        PyObject *obj = PyWeakref_GET_OBJECT(wr);
        lockobject *lock;
        if (obj != Py_None) {
            assert(Py_TYPE(obj) == &Locktype);
            // 获取锁结构
            lock = (lockobject *) obj;
            if (lock->locked) {
                // 释放锁的地方!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
                PyThread_release_lock(lock->lock_lock);
                lock->locked = 0;
            }
        }
        Py_DECREF(wr);
    }

daemon
==========

daemon表示后台线程, 主线程会等待所有非daemon线程结束后直接结束.

.. code-block:: python

    _main_thread = _MainThread()
    
    def _shutdown():
        assert tlock is not None
        assert tlock.locked()
        tlock.release()
        _main_thread._stop()
        t = _pickSomeNonDaemonThread()
        while t:
            t.join()
            t = _pickSomeNonDaemonThread()
        _main_thread._delete()

因为module只会加载一次并且_main_thread是全局变量, 所以主线程加载threading这个mudule的时候主线程肯定被设置未_main_thread

在_shutdown函数中, 会遍历所有非daemon线程, 然后join一下, 最后主线程会调用_delete方法去做清理操作

而daemon线程则判断python的runtime是否有效, 无效就退出了

.. code-block:: python

    import sys as _sys
    class Thread
        def _bootstrap(self):
            try:
                self._bootstrap_inner()
            except:
                # 任何异常同时self._daemonic表示daemon为True, _sys无效了, 那么显然python的runtime无效了
                # 直接退出吧
                if self._daemonic and _sys is None:
                    return
                raise


手动停止线程
===============

https://github.com/Bogdanp/dramatiq/blob/95cd9f6f35f1b40c138124cbb96e3667db088ef4/dramatiq/middleware/threading.py#L62

线程一般不会提供主动停止的功能, 在CPython中threading.Thread中的join方法是等待线程停止, 而_stop方法则是设置Thread的状态, 并且调用_stop的前提是线程已经停止了

但是CPython可以把异常传递给指定线程, 那么当指定线程运行到下一个字节码的时候就会抛出指定的异常, 这样线程就停止了

下面是dramatiq中的例子


.. code-block:: python

    def _raise_thread_exception_cpython(thread_id, exception):
        exctype = (exception if inspect.isclass(exception) else type(exception)).__name__
        thread_id = ctypes.c_long(thread_id)
        exception = ctypes.py_object(exception)
        # 设置指定线程的异常
        count = ctypes.pythonapi.PyThreadState_SetAsyncExc(thread_id, exception)
        if count == 0:
            logger.critical("Failed to set exception (%s) in thread %r.", exctype, thread_id.value)
        elif count > 1:  # pragma: no cover
            logger.critical("Exception (%s) was set in multiple threads.  Undoing...", exctype)
            ctypes.pythonapi.PyThreadState_SetAsyncExc(thread_id, ctypes.c_long(0))

使用ctype调用PyThreadState_SetAsyncExc, 这个函数是把获取指定线程的信息结构tstate, 然后把指定异常设置到该tstate中

.. code-block:: c

    PyThreadState_SetAsyncExc(long id, PyObject *exc) {
        PyInterpreterState *interp = GET_INTERP_STATE();
        PyThreadState *p;
    
        HEAD_LOCK();
        for (p = interp->tstate_head; p != NULL; p = p->next) {
            if (p->thread_id == id) {
                PyObject *old_exc = p->async_exc;
                Py_XINCREF(exc);
                // 把异常设置到tstate上!!!!!!!!!!!!!!!
                p->async_exc = exc;
                HEAD_UNLOCK();
                Py_XDECREF(old_exc);
                // 通知解释器执行字节码之前需要判断一下!!!!!!!!!!
                _PyEval_SignalAsyncExc();
                return 1;
            }
        }
        HEAD_UNLOCK();
        return 0;
    }

    //_PyEval_SignalAsyncExc设置变量eval_break以及pending_async_exec
    #define SIGNAL_ASYNC_EXC() \
        do { \
            pending_async_exc = 1; \
            _Py_atomic_store_relaxed(&eval_breaker, 1); \
        } while (0)
    
     //执行代码的时候判断异常
     if (_Py_atomic_load_relaxed(&eval_breaker)) {
        if (tstate->async_exc != NULL) {
            // 判断有异常!!!!!
            PyObject *exc = tstate->async_exc;
            tstate->async_exc = NULL;
            UNSIGNAL_ASYNC_EXC();
            PyErr_SetNone(exc);
            Py_DECREF(exc);
            // 处理错误, 一般没有捕获的异常直接就终止运行了
            goto error;
        }
    }



那么在指定线程开始运行的时候就会校验到异常, 这样线程就停止了, 理论上这个也不是主动停止线程, 而是主动让函数退出导致线程退出.

这样还需要注意的是如果线程"卡"在某个操作而进行不到下一个字节码的话, 异常是不会被捕获的, 比如网络请求太久了没有返回, C代码一直没有返回等等


