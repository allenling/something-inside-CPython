concurrent
####################


Future
=============

一个函数(可执行对象)执行的时候是同步的, 也可以是异步的. 异步执行的时候经常就把该函数交给某个线程/进程去执行

但是这个函数执行完只会的结果我们怎么去获取呢? 我们可以使用queue, 同步机制等等, **Future对象同样是为了帮助我们获取这些异步执行的结果**

.. code-block:: python

    from concurrent.futures import ThreadPoolExecutor
    import time

    def wait_on_a():
        time.sleep(2)
        print("going to return")
        return 6

    def main():
        executor = ThreadPoolExecutor(max_workers=2)
        a = executor.submit(wait_on_a)
        print(a._state)
        print(a.result(timeout=3))
        print(a._state)
        return

    if __name__ == "__main__":
        main()

这里我们把wait_on_a发送到线程池中运行, 那么运行就是异步的. 异步执行的函数我们无法直接感知其执行状态的, 但是我们可以把每个异步函数都和一个Future对象一一对应起来

那么我们可以检查其对应的Future对象的状态Future的状态就是函数执行的状态

例子中输出为

.. code-block::

   RUNNING
   going to return
   6
   FINISHED

Future第一次打印的时候状态是RUNNING, 表示函数依然在运行中, 而之后状态变为了FINISHED, 同时Future.result调用返回的就是函数返回值

**Future是表示异步可执行的状态和对象**. Future就和执行单位(Thread/Process)剥离出来了, 作为一个单独的组件, 但是其状态代表的则是分配给它的可执行对象的状态

所以如果谈到Future, 其实也是指对应的可执行对象

Future._state
==================

一个Future的状态有下面几个

.. code-block:: python

    # Possible future states (for internal use by the futures package).
    PENDING = 'PENDING'
    RUNNING = 'RUNNING'
    # The future was cancelled by the user...
    CANCELLED = 'CANCELLED'
    # ...and _Waiter.add_cancelled() was called by a worker.
    CANCELLED_AND_NOTIFIED = 'CANCELLED_AND_NOTIFIED'
    FINISHED = 'FINISHED'

    _FUTURE_STATES = [
        PENDING,
        RUNNING,
        CANCELLED,
        CANCELLED_AND_NOTIFIED,
        FINISHED
    ]

PENDING是初始状态, 在Future.__init__中Future._state被初始化为PENDING

RUNNING状态是在Executor执行Future对应的可执行对象的时候设置的, RUNNING只能是从PENDING状态转移过来, 只能通过调用set_running_or_notify_cancel方法来设置

.. code-block:: python

    def set_running_or_notify_cancel(self):
        """Mark the future as running or process any cancel notifications.

        Should only be used by Executor implementations and unit tests.

        If the future has been cancelled (cancel() was called and returned
        True) then any threads waiting on the future completing (though calls
        to as_completed() or wait()) are notified and False is returned.

        If the future was not cancelled then it is put in the running state
        (future calls to running() will return True) and True is returned.

        This method should be called by Executor implementations before
        executing the work associated with this future. If this method returns
        False then the work should not be executed.

        Returns:
            False if the Future was cancelled, True otherwise.

        Raises:
            RuntimeError: if this method was already called or if set_result()
                or set_exception() was called.
        """
        with self._condition:
            if self._state == CANCELLED:
                self._state = CANCELLED_AND_NOTIFIED
                for waiter in self._waiters:
                    waiter.add_cancelled(self)
                # self._condition.notify_all() is not necessary because
                # self.cancel() triggers a notification.
                return False
            elif self._state == PENDING:
                # 如果Future为PENDING状态, 那么设置为RUNNING状态
                self._state = RUNNING
                return True
            else:
                LOGGER.critical('Future %s in unexpected state: %s',
                                id(self),
                                self._state)
                raise RuntimeError('Future in unexpected state')

通过注释可知, 能把一个Future对象设置为RUNNING状态的应该只能是Executor, 在这里提供了ThreadPoolExecutor和ProcessPoolExecutor两种执行者

比如在ThreadPoolExecutor, 一个WorkItem被执行的是调用set_running_or_notify_cancel去设置Future的状态为RUNNING

.. code-block:: python

    class _WorkItem(object):
        def run(self):
            if not self.future.set_running_or_notify_cancel():
                return

            # 在当前线程中执行可执行对象fn
            # 当前线程和submit线程不一样
            try:
                result = self.fn(*self.args, **self.kwargs)
            except BaseException as e:
                # 设置异常
                self.future.set_exception(e)
            else:
                # 否则设置结果
                self.future.set_result(result)

而FINISHED状态则是表示可执行对象已经结束, 此时可能是成功或者异常, 调用set_result来设置返回值, 或者调用set_exception来设置发生的异常

.. code-block:: python

    def set_result(self, result):
        with self._condition:
            # 如果异步执行完成了, 那么设置self._state为FINISHED
            # 同时self._result为函数返回值
            self._result = result
            self._state = FINISHED
            for waiter in self._waiters:
                waiter.add_result(self)
            self._condition.notify_all()
        self._invoke_callbacks()

    def set_exception(self, exception):
        with self._condition:
            # 如果异步执行出现异常, 那么self._state依然是FINISHED
            # 只是self._exception记录了出现的异常
            self._exception = exception
            self._state = FINISHED
            for waiter in self._waiters:
                waiter.add_exception(self)
            self._condition.notify_all()
        self._invoke_callbacks()

CANCELLED和CANCELLED_AND_NOTIFIED都是表示Future是否被取消.

**取消操作只能是在Future没有执行, 也就是处于PENDING状态的时候执行, 一旦Future为RUNNING状态之后, 就不能取消了**

.. code-block:: python

    def cancel(self):
        """Cancel the future if possible.

        Returns True if the future was cancelled, False otherwise. A future
        cannot be cancelled if it is running or has already completed.
        """
        with self._condition:
            # 如果为RUNNING或者FINISHED, 那么返回False, 表示Future没有取消
            if self._state in [RUNNING, FINISHED]:
                return False

            # 如果早已取消, 那么返回True, 表示Future早已取消
            if self._state in [CANCELLED, CANCELLED_AND_NOTIFIED]:
                return True

            # 这里只有_state为PENDING才能走到
            self._state = CANCELLED
            self._condition.notify_all()

        self._invoke_callbacks()
        return True

所以状态转移就是

.. code-block:: 

    PENDING -> RUNNING -> FINISHED

            -> CANCELLED/CANCELLED_AND_NOTIFIED

为什么无法取消Future(可执行对象)
==================================

**为什么不能取消Future(或者说异步执行对象)? 那是因为你无法通知可执行对象终止.**

比如线程, 在操作系统级别有中断, 在Python提供的ThreadException, 但是这些都是在指令级别上的操作, 前者是检查CPU指令, 后者是检查字节码, 同时后者必须执行到下一个字节码的时候才会去

查询是否需要终止线程, 如果你在某个字节码上阻塞住了(比如time.sleep), 那么你依然无法终止线程.

所以如果你不能在指令级别上进行操作, 那么你无法终止一个可执行对象. 比如一个函数一直加1, 你如何终止呢?

一般我们需要在可执行对象上查看某些状态来判断我们是否需要退出, 比如使用nonblock模式去读取一个queue, 如果没有得到终止指令, 那么就继续执行, 否则退出

.. code-block:: python

    while True:
        try:
            cmd = cmd_queue.get(block=False)
        except Empty:
            # run your code
            pass
        else:
            if cmd == "quit":
                return

又比如C++中的boost的thread库提供interruption_point, 每次线程执行到interruption_point的时候回去检查是否需要退出, 这个和python的方法一样.

**所以, 网络操作一定要记得加上timeout!**

Future.result/Future.exception
=====================================

因为Future对象会被多个线程操作的, 所以使用Future._condition这样一个threading.Condition给保护起来

调用Future.result则等待Future执行完成然后获取返回值, 在timeout时间等待Future._state被设置为FINISHED, 注意如果是发生了异常, 那么Future.result则会引发异常

.. code-block:: python

    def result(self, timeout=None):
        # 先获取self._condition
        with self._condition:
            # 如果被取消, 那么引发异常
            if self._state in [CANCELLED, CANCELLED_AND_NOTIFIED]:
                raise CancelledError()
            elif self._state == FINISHED:
                # 如果早就终止了, 那么返回结果
                return self.__get_result()

            # 否则说明异步函数要么是PENDING, 要么是RUNNING, 那么在self._condition上等待被其他线程唤醒
            self._condition.wait(timeout)

            # 如果此时是CANCELLED状态, 说明之前是PENDING状态
            if self._state in [CANCELLED, CANCELLED_AND_NOTIFIED]:
                raise CancelledError()
            elif self._state == FINISHED:
                # 否则之前是RUNNING状态, 那么获取结果返回
                return self.__get_result()
            else:
                # 否则依然在PENDING或者RUNNING状态, 那么就超时了
                raise TimeoutError()


    def __get_result(self):
        if self._exception:
            # 如果执行有异常, 那么引发异常
            raise self._exception
        else:
            # 否则返回结果
            return self._result

wait
==========

Future显然是一个可等待对象(自创), 对于一些Future对象, 有时候希望每次有Future完成的话就返回这个完成的Future, 或则只要有某个出现异常, 就返回这样的需求, concurrent也提供了这些功能

concurrent中wait函数提供了不同的需求的支持, 包括

1. as_completed, 即只要某些future的状态变为FINISHED, 那么返回, 然后继续在未结束的future上等待

2. first_completed, 当第一个future变为FINISHED的时候, 返回, 同时不会在其他future上等待了

3. all_completed, 等待所有的future都FINISHED之后才返回

4. first_exception, 等待第一个future状态为FINISHED, 同时其Future._exception不为空的话, 返回, 并且终止等待其他future

这里最基础的功能就是需要Future对象在状态变更的时候, 提供通知机制, 所以在Future中使用了condition来通知状态变更

Future._waiters
--------------------

虽然Future中使用condition来限制多线程的操作, 但是这里Future._waiters却不是Future._condition的waiters, Future._condition是Threading.Condition, Condition和Condition._waiters其参考其他

这里Future._waiters则是说等待该Future状态变化, 而不参与Future._condition的竞争(毕竟Future._condition是内部使用的, Future._waiters是通知外部的)

比如调用Future.set_result就把状态设置为FINISHED, 则Future就通知Future._waiters了

.. code-block:: python

    def set_result(self, result):
        """Sets the return value of work associated with the future.

        Should only be used by Executor implementations and unit tests.
        """
        with self._condition:
            self._result = result
            self._state = FINISHED
            # 通知Future._waiters
            for waiter in self._waiters:
                waiter.add_result(self)
            self._condition.notify_all()
        self._invoke_callbacks()

所以waiter要实现add_result方法, 不仅仅是add_result, 还有其他方法, 包括add_exception, add_cancelled, 你也可以实现自己的Waiter, 但是至少要实现这3个方法

这3个方法分别是在状态变更的时候被调用

.. code-block:: python

    class _Waiter(object):
        """Provides the event that wait() and as_completed() block on."""
        def __init__(self):
            self.event = threading.Event()
            self.finished_futures = []
    
        def add_result(self, future):
            self.finished_futures.append(future)
    
        def add_exception(self, future):
            self.finished_futures.append(future)
    
        def add_cancelled(self, future):
            self.finished_futures.append(future)


**所以Future只告诉你我状态变更了, 而什么时候通知用户, 是waiter自己根据需要来实现的**


比如我们只需要第一个状态变为FINISHED的future, 那么我们可以这样

.. code-block:: python

    class _FirstCompletedWaiter(_Waiter):
        """Used by wait(return_when=FIRST_COMPLETED)."""
    
        def add_result(self, future):
            super().add_result(future)
            self.event.set()
    
        def add_exception(self, future):
            super().add_exception(future)
            self.event.set()
    
        def add_cancelled(self, future):
            super().add_cancelled(future)
            self.event.set()

只要有Future调用了set_result, 那么就会调用_FirstCompletedWaiter.add_result, 那么直接就通过self.event.set来通知上一层

如果我们需要所有的Future都FINISHED了才通知上一层, 那么可以有

.. code-block:: python

    class _AllCompletedWaiter(_Waiter):
        """Used by wait(return_when=FIRST_EXCEPTION and ALL_COMPLETED)."""
    
        def __init__(self, num_pending_calls, stop_on_exception):
            self.num_pending_calls = num_pending_calls
            self.stop_on_exception = stop_on_exception
            self.lock = threading.Lock()
            super().__init__()
    
        def _decrement_pending_calls(self):
            with self.lock:
                # 计算还有多少个Future没有FINISHED
                self.num_pending_calls -= 1
                if not self.num_pending_calls:
                    # 如果没有, 那么通知上一层
                    self.event.set()
    
        def add_result(self, future):
            super().add_result(future)
            self._decrement_pending_calls()

first_completed, first_exception, all_completed都是在concurrent.future.wait这个函数中支持的

.. code-block:: python

    def wait(fs, timeout=None, return_when=ALL_COMPLETED):
        """Wait for the futures in the given sequence to complete.
    
        Args:
            fs: The sequence of Futures (possibly created by different Executors) to
                wait upon.
            timeout: The maximum number of seconds to wait. If None, then there
                is no limit on the wait time.
            return_when: Indicates when this function should return. The options
                are:
    
                FIRST_COMPLETED - Return when any future finishes or is
                                  cancelled.
                FIRST_EXCEPTION - Return when any future finishes by raising an
                                  exception. If no future raises an exception
                                  then it is equivalent to ALL_COMPLETED.
                ALL_COMPLETED -   Return when all futures finish or are cancelled.
    
        Returns:
            A named 2-tuple of sets. The first set, named 'done', contains the
            futures that completed (is finished or cancelled) before the wait
            completed. The second set, named 'not_done', contains uncompleted
            futures.
        """
        with _AcquireFutures(fs):
            done = set(f for f in fs
                       if f._state in [CANCELLED_AND_NOTIFIED, FINISHED])
            not_done = set(fs) - done
    
            if (return_when == FIRST_COMPLETED) and done:
                return DoneAndNotDoneFutures(done, not_done)
            elif (return_when == FIRST_EXCEPTION) and done:
                if any(f for f in done
                       if not f.cancelled() and f.exception() is not None):
                    return DoneAndNotDoneFutures(done, not_done)
    
            if len(done) == len(fs):
                return DoneAndNotDoneFutures(done, not_done)
    
            # 根据类型创建不同的waiter
            waiter = _create_and_install_waiters(fs, return_when)
    
        # 在waiter上的event等待
        waiter.event.wait(timeout)
        # 返回就移除waiter
        for f in fs:
            with f._condition:
                f._waiters.remove(waiter)
    
        done.update(waiter.finished_futures)
        return DoneAndNotDoneFutures(done, set(fs) - done)


    def _create_and_install_waiters(fs, return_when):
        # 根据不同的类型状态不同的waiter对象
        if return_when == _AS_COMPLETED:
            waiter = _AsCompletedWaiter()
        elif return_when == FIRST_COMPLETED:
            waiter = _FirstCompletedWaiter()
        else:
            pending_count = sum(
                    f._state not in [CANCELLED_AND_NOTIFIED, FINISHED] for f in fs)
    
            if return_when == FIRST_EXCEPTION:
                waiter = _AllCompletedWaiter(pending_count, stop_on_exception=True)
            elif return_when == ALL_COMPLETED:
                waiter = _AllCompletedWaiter(pending_count, stop_on_exception=False)
            else:
                raise ValueError("Invalid return condition: %r" % return_when)
    
        # 这里!!!!!!
        # 把waiter加入到Future._waiters中
        for f in fs:
            f._waiters.append(waiter)
    
        return waiter

这3个的共同点就是只会等待一次, 要么第一个Future状态变化就返回, 要么所有的Future都FINISHED只会才返回, 而as_complted则需要持续等待


as_completed
------------------

既然as_completed需要在状态未变化的Future上继续等待, 那么我们就需要重置waiter.event了, 要重置event, 就需要lock去保护, 所以为什么

_AsCompletedWaiter上带了个lock的原因, 比如在我们重置event的时候, 在重置finished_futures列表之前, 剩下的future都完成了

那么我们此时把finished_futures重置为空列表, 那么如果继续在该event上等待的话, 不加超时就永远不会返回了

.. code-block:: python

    def as_completed(fs, timeout=None):
        """An iterator over the given futures that yields each as it completes.
    
        Args:
            fs: The sequence of Futures (possibly created by different Executors) to
                iterate over.
            timeout: The maximum number of seconds to wait. If None, then there
                is no limit on the wait time.
    
        Returns:
            An iterator that yields the given Futures as they complete (finished or
            cancelled). If any given Futures are duplicated, they will be returned
            once.
    
        Raises:
            TimeoutError: If the entire result iterator could not be generated
                before the given timeout.
        """
        if timeout is not None:
            end_time = timeout + time.time()
    
        fs = set(fs)
        with _AcquireFutures(fs):
            finished = set(
                    f for f in fs
                    if f._state in [CANCELLED_AND_NOTIFIED, FINISHED])
            pending = fs - finished
            waiter = _create_and_install_waiters(fs, _AS_COMPLETED)
        
        # 上面的流程依然是创建waiter
        # 然后先计算一下当前已经完成的Future
    
        try:
            # 返回一开始就完成的Future
            yield from finished
    
            while pending:
                if timeout is None:
                    wait_timeout = None
                else:
                    wait_timeout = end_time - time.time()
                    if wait_timeout < 0:
                        raise TimeoutError(
                                '%d (of %d) futures unfinished' % (
                                len(pending), len(fs)))
    
                # 在waiter.event上等待
                waiter.event.wait(wait_timeout)
    
                # 抢锁, 防止我们无法返回
                # 重置event, 继续等待未完成的Future!!!!!!!!!!!!!!!!!!!!
                with waiter.lock:
                    finished = waiter.finished_futures
                    waiter.finished_futures = []
                    waiter.event.clear()
    
                for future in finished:
                    yield future
                    pending.remove(future)
    
        finally:
            for f in fs:
                with f._condition:
                    f._waiters.remove(waiter)


ThreadPoolExecutor
========================

线程池创建多个threading.Thread来执行可执行对象, 通过queue.Queue来传递要执行的可执行对象, 使用Future来通知用户

.. code-block:: python

    class ThreadPoolExecutor(_base.Executor):
        # 预分配线程
        def _adjust_thread_count(self):
            # When the executor gets lost, the weakref callback will wake up
            # the worker threads.
            def weakref_cb(_, q=self._work_queue):
                q.put(None)
            # TODO(bquinlan): Should avoid creating new threads if there are more
            # idle threads than items in the work queue.
            num_threads = len(self._threads)
            if num_threads < self._max_workers:
                # 预先生成self._max_workers个数的线程
                thread_name = '%s_%d' % (self._thread_name_prefix or self,
                                         num_threads)
                # 每个线程都是调用_worker这个函数
                t = threading.Thread(name=thread_name, target=_worker,
                                     args=(weakref.ref(self, weakref_cb),
                                           self._work_queue))
                # 把这些worker线程都设置为daemon
                t.daemon = True
                t.start()
                self._threads.add(t)
                # 所有的线程都从同一个queue(self._work_queue)中获取数据!!!!!!!!!!!!!
                _threads_queues[t] = self._work_queue


而submit就只是把任务发送到self._work_queue中, 这样当有线程空闲的时候就会从queue中拿到任务来执行


.. code-block:: python

    def submit(self, fn, *args, **kwargs):
        with self._shutdown_lock:
            if self._shutdown:
                raise RuntimeError('cannot schedule new futures after shutdown')
            # 这里创建一个Future
            f = _base.Future()
            # _WorkItem就是报错fn的参数, 以及把Future和fn给绑定起来
            w = _WorkItem(f, fn, args, kwargs)

            # 直接把w放入到self._work_queue中
            self._work_queue.put(w)
            self._adjust_thread_count()
            return f

当_WorkItem.fn结束之后, 更新Future的状态

.. code-block:: python

    def _worker(executor_reference, work_queue):
        try:
            while True:
                # 一直从work_queue中获取_WorkItem
                work_item = work_queue.get(block=True)
                if work_item is not None:
                    # 执行_WorkItem.run
                    work_item.run()
                    # Delete references to object. See issue16284
                    del work_item
                    continue
                executor = executor_reference()
                # Exit if:
                #   - The interpreter is shutting down OR
                #   - The executor that owns the worker has been collected OR
                #   - The executor that owns the worker has been shutdown.
                # 这里是判断主线程是否已经退出了, 如果主线程退出了, 那么显然executor就是None
                if _shutdown or executor is None or executor._shutdown:
                    # Notice other workers
                    # 通知其他线程退出
                    work_queue.put(None)
                    return
                del executor
        except BaseException:
            _base.LOGGER.critical('Exception in worker', exc_info=True)

更新Future状态是在_WorkItem.run中

.. code-block:: python

    class _WorkItem(object):
        def __init__(self, future, fn, args, kwargs):
            self.future = future
            self.fn = fn
            self.args = args
            self.kwargs = kwargs

        def run(self):
            if not self.future.set_running_or_notify_cancel():
                return

            try:
                result = self.fn(*self.args, **self.kwargs)
            except BaseException as e:
                # 有异常, 那么调用set_exception
                self.future.set_exception(e)
            else:
                # 没有错误, 那么调用set_result
                self.future.set_result(result)

ProcessPoolExecutor
======================

concurrent.futures.ProcessPoolExecutor并没有使用multiprocess.Pool

multiprocessing.Pool和ProcessPoolExecutor都是worker进程会一直在queue监听拿到queue中的任务一直执行, 直到异常或者收到通知需要退出才会退出

没有使用multiprocessing.Pool可能觉得multiprocessing.Pool中太复杂了吧?

ProcessPoolExecutor和ThreadPoolExecutor思路一样, 只不过ProcessPoolExecutor的worker是进程(multiprocess.Process), 而queue也是multiprocess.Queue



.. code-block::

    
                                                                                                   P1
    
                  WorkItem(fn, Future)                       CallItem(worker_id, fn)
    ProcessPool -----管理进程------------> manager_thread  ------call_queue------------->          P2
    
    
                                                           <-----result_queue----------->          P3
                                                              ResultItem(worker_id, result)



WorkItem, CallItem, ResultItem               
------------------------------------

在ThreadPoolExecutor中, 只有WorkItem, WorkItem和Future关联在一起, 执行的时候得到WorkItem, 然后更新WorkItem中的Future的状态

而ProcessPoolExecutor中不仅仅有WorkItem, 还有CallItem和ResultItem


.. code-block:: python

    class _WorkItem(object):
        def __init__(self, future, fn, args, kwargs):
            self.future = future
            self.fn = fn
            self.args = args
            self.kwargs = kwargs

    class _ResultItem(object):
        def __init__(self, work_id, exception=None, result=None):
            self.work_id = work_id
            self.exception = exception
            self.result = result

    class _CallItem(object):
        def __init__(self, work_id, fn, args, kwargs):
            self.work_id = work_id
            self.fn = fn
            self.args = args
            self.kwargs = kwargs

WorkItem依然是Future和fn关联, 然后CallItem在而是worker_id和fn相关联, 而ResultItem则是worker_id和result相关联

向进程发送的是CallItem, 因为进程不需要直到Future(因为它直到Future也没用呀), 所以进程worker只需要直到发送给自己的fn就好了, 同时

进程执行完fn只会, 发送ResultItem给主进程, 主进程需要通过worker_id去查找WorkItem, 然后更新Future.result或者Future.exception


submit
-------------

submit的过程和ThreadPoolExecutor一样

.. code-block:: python

    class ProcessPoolExecutor(_base.Executor):
    
        def __init__(self, max_workers=None):

            # 创建_call_queue和_result_queue
            self._call_queue = multiprocessing.Queue(self._max_workers +
                                                     EXTRA_QUEUED_CALLS)
            self._call_queue._ignore_epipe = True
            self._result_queue = SimpleQueue()
            return

        def submit(self, fn, *args, **kwargs):
            with self._shutdown_lock:
                if self._broken:
                    raise BrokenProcessPool('A child process terminated '
                        'abruptly, the process pool is not usable anymore')
                if self._shutdown_thread:
                    raise RuntimeError('cannot schedule new futures after shutdown')
    
                f = _base.Future()
                # 创建WorkItem
                w = _WorkItem(f, fn, args, kwargs)
    
                # 这里拿住了self._shutdown_lock
                # 所以一次只能submit一次
                # 但是因为submit应该是很快的操作, 所以一次一个不是问题
                # 存储待执行的item
                self._pending_work_items[self._queue_count] = w
                self._work_ids.put(self._queue_count)
                # 下标加1
                self._queue_count += 1
                # 这里要唤醒_result_queue去把self._pending_work_items中的WorkItem发送给worker
                # Wake up queue management thread
                self._result_queue.put(None)
    
                self._start_queue_management_thread()
                return f

queue_management_thread
----------------------------

这里是启动一个管理线程去把任务发送给worker, 同时接受worker的结果, 这里为了简便, 发送任务的通知也使用了self._result_queue, 不然又多出一个queue, 有点麻烦

.. code-block:: python

    def _start_queue_management_thread(self):
        # When the executor gets lost, the weakref callback will wake up
        # the queue management thread.
        def weakref_cb(_, q=self._result_queue):
            q.put(None)
        if self._queue_management_thread is None:
            # 启动_queue_management_thread这个管理线程
            # Start the processes so that their sentinels are known.
            self._adjust_process_count()
            self._queue_management_thread = threading.Thread(
                    target=_queue_management_worker,
                    args=(weakref.ref(self, weakref_cb),
                          self._processes,
                          self._pending_work_items,
                          self._work_ids,
                          self._call_queue,
                          self._result_queue))
            self._queue_management_thread.daemon = True
            self._queue_management_thread.start()
            _threads_queues[self._queue_management_thread] = self._result_queue


    def _queue_management_worker(executor_reference,
                                 processes,
                                 pending_work_items,
                                 work_ids_queue,
                                 call_queue,
                                 result_queue):
        reader = result_queue._reader
    
        while True:
            # 把pending_work_items中的work_item发送给进程
            _add_call_item_to_queue(pending_work_items,
                                    work_ids_queue,
                                    call_queue)
    
            # 监听自己的result_queue和worker进程的sentinels
            sentinels = [p.sentinel for p in processes.values()]
            assert sentinels
            ready = wait([reader] + sentinels)
            # 如果wait返回, 说明要么有数据需要读, 要么pipe已经broken了
            if reader in ready:
                result_item = reader.recv()
            else:
                # Mark the process pool broken so that submits fail right now.
                # 这里是shutdown的过程
                # 这里使得submit直接失败, 同时记录下pipe broken的异常
                return
            if isinstance(result_item, int):
                # 这里如果读取到的是workerd的pid, 那么意味着worker已经退出了, 所以我们要清理资源
                # Clean shutdown of a worker using its PID
                # (avoids marking the executor broken)
                assert shutting_down()
                p = processes.pop(result_item)
                p.join()
                if not processes:
                    shutdown_worker()
                    return
            elif result_item is not None:
                # 如果result_item不是None, 表示是一个ResultItem
                # 那么我们需要查询出对应的WorkerItem
                work_item = pending_work_items.pop(result_item.work_id, None)
                # work_item can be None if another process terminated (see above)
                # 更新WorkerItem中的Future, 然后释放掉WorkerItem
                if work_item is not None:
                    if result_item.exception:
                        work_item.future.set_exception(result_item.exception)
                    else:
                        work_item.future.set_result(result_item.result)
                    # Delete references to object. See issue16284
                    del work_item
            # 下面是校验shutdown的过程


所以这里如果ready是None的话, 就会直到开始下一次循环, 直接调用_add_call_item_to_queue去把pending_work_items中的待执行WorkerItem发送给进程

_add_call_item_to_queue
--------------------------


发送CallItem给进程

.. code-block:: python

    def _add_call_item_to_queue(pending_work_items,
                                work_ids,
                                call_queue):
        while True:
            # 如果call_queue满了, 就不发送了
            if call_queue.full():
                return
            # 拿出worker_id
            try:
                work_id = work_ids.get(block=False)
            except queue.Empty:
                return
            else:
                # 拿到worker_item
                work_item = pending_work_items[work_id]

                # 调用call_queue.put, 发送_CallItem
                if work_item.future.set_running_or_notify_cancel():
                    call_queue.put(_CallItem(work_id,
                                             work_item.fn,
                                             work_item.args,
                                             work_item.kwargs),
                                   block=True)
                else:
                    del pending_work_items[work_id]
                    continue



Worker进程
------------------


创建worker进程


.. code-block:: python

    def _adjust_process_count(self):
        for _ in range(len(self._processes), self._max_workers):
            # 直接使用了multiprocessing.Process
            p = multiprocessing.Process(
                    target=_process_worker,
                    args=(self._call_queue,
                          self._result_queue))
            p.start()
            self._processes[p.pid] = p




    # worker进程启动只会在call_queue上监听
    def _process_worker(call_queue, result_queue):
        while True:
            # 拿到_CallItem
            call_item = call_queue.get(block=True)
            if call_item is None:
                # 拿到的call_item是None, 表示主进程要退出了
                # Wake up queue management thread
                # 所以自己也退出, 通知主进程
                result_queue.put(os.getpid())
                return
            # 否则执行可执行对象
            # 发送结果或者异常
            try:
                r = call_item.fn(*call_item.args, **call_item.kwargs)
            except BaseException as e:
                exc = _ExceptionWithTraceback(e, e.__traceback__)
                result_queue.put(_ResultItem(call_item.work_id, exception=exc))
            else:
                result_queue.put(_ResultItem(call_item.work_id,
                                             result=r))



如果worker卡死怎么办?
==========================


如果一个worker卡死了, 那么我们直接通过os去杀死它就好了, 但是要注意处理进程死了之后, 重新生成新的worker进程

