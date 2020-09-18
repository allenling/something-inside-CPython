concurrent
####################


Future
=============

一个函数(可执行对象)执行的时候是同步的, 也可以是异步的. 异步执行的时候经常就把该函数交给某个线程去执行

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
        time.sleep(3)
        print(a._state)
        print(a._result)
        return

    if __name__ == "__main__":
        main()

这里我们把wait_on_a发送到线程池中运行, 那么运行就是异步的. 异步执行的函数我们无法直接感知其执行状态的但是每个异步函数都和一个Future对象一一对应起来

那么我们可以检查其对应的Future对象的状态Future的状态就是函数执行的状态, 如果Future的状态是正常结束, 那么Future._result就是函数的返回值

例子中输出为

.. code-block::

   RUNNING
   going to return
   FINISHED
   6

Future一开始的状态是RUNNING, 表示函数依然在运行中, 而之后状态变为了FINISHED, 同时Future._result就是函数返回值

**Future是表示异步执行的结果的对象, 当Future的状态变为完成, 取消, 异常的时候. 表示异步执行的结果同样是完成, 取消和异常**

这样对于异步执行的程序, 线程池中的worker线程执行完成之后, 设置Future对象的状态和结果, 而另外一个线程通过在同一个Future对象上进行检查就可以知道异步执行的结果了

你可以对一个Future对象调用set_result, 表示设置Future的状态为FINISHED, 同时设置上函数返回值

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

因为Future对象会被多个线程操作的, 所以使用self._condition这样一个threading.Condition给保护起来, 然后设置self._result和self._state

当异步执行的函数出现异常的时候, 同样有Future.set_exception方法去调用, 过程都是类似的, 设置self._result和self._state

如果我们希望等待Future执行完成的话, Future.result方法会在指定的timeout时间等待Future._state被设置为FINISHED

.. code-block:: python

    def result(self, timeout=None):
        # 先获取self._condition
        with self._condition:
            # 如果self._state不为FINISHED, 那么引发异常
            if self._state in [CANCELLED, CANCELLED_AND_NOTIFIED]:
                raise CancelledError()
            elif self._state == FINISHED:
                # 否则返回结果
                return self.__get_result()

            # 否则说明异步函数还在执行, 那么在self._condition上等待被其他线程唤醒
            self._condition.wait(timeout)

            # 被唤醒说明异步执行有结果了, 校验结果
            if self._state in [CANCELLED, CANCELLED_AND_NOTIFIED]:
                raise CancelledError()
            elif self._state == FINISHED:
                return self.__get_result()
            else:
                raise TimeoutError()


Future对象经常在async io中使用, async io基于事件机制的(底层基于操作的事件通知机制), async io中的任务(需要执行的函数)遇到io操作的时候

是将操作交给其他worker来执行(比如线程worker), 自己则让出EventLoop的时间片, 等待结果完成. 那么EventLoop需要一个对象, 能通过这个对象获取异步

执行的状态和结果, 同时支持超时机制, 那么这个对象就是Future

waiters
==========

对于一些列的Future对象, 如果希望每次有Future完成的话就返回这个完成的Future, concurrent提供了as_completed函数可以实现这个需求

as_completed中思路创建waiter, 该waiter在Future上进行等待, 一旦有Future状态变化, 比如被调用set_result设置状态为FINISHED, 那么Future就会通知waiter

waiter一旦知道Future的状态变化之后, 根据需要通知等待者. 比如as_completed中的waiter会在每一个Future对象状态变为FINISHED的时候通知调用者

.. code-block:: python

    def as_completed(fs, timeout=None):
        if timeout is not None:
            end_time = timeout + time.time()

        fs = set(fs)
        # 这里是for循环去拿住所有Future._condition
        with _AcquireFutures(fs):
            finished = set(
                    f for f in fs
                    if f._state in [CANCELLED_AND_NOTIFIED, FINISHED])
            # 把已经FINISHED的Future给放到finished集合中
            # 注意finished是集合而不是列表, 所以返回给调用者的Future不是有序的
            pending = fs - finished
            # 在Future上进行等待
            waiter = _create_and_install_waiters(fs, _AS_COMPLETED)

        try:
            # 如果已经有Future完成了, 直接返回不需要等待
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

                # 否则在waiter上(waiter.event)上等待
                waiter.event.wait(wait_timeout)

                # 一旦waiter.event返回了, 表示肯定有Future已经FINISHED了
                with waiter.lock:
                    # 拿到FINISHED列表
                    finished = waiter.finished_futures
                    # 清空waiter.finished_futures
                    waiter.finished_futures = []
                    # 重置waiter.event
                    waiter.event.clear()

                for future in finished:
                    # 把完成的future给yield出去
                    yield future
                    pending.remove(future)

        finally:
            for f in fs:
                with f._condition:
                    f._waiters.remove(waiter)

这里关键在于waiter说明时候唤醒waiter.event, 首先, 我们可以选择Future处于什么状态的时候通知我们, 比如上面的例子中就是当Future为FINISHED的时候

把FINISHED的Future返回给我们. 根据这个需求有不同的waiter, 上面的例子中waiter就是_AsCompletedWaiter

.. code-block:: python

    def _create_and_install_waiters(fs, return_when):
        if return_when == _AS_COMPLETED:
            # 根据传入的_AS_COMPLETED, 创建一个_AsCompletedWaiter
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

        # 把这个waiter传入到所有的Future._waiters列表中
        for f in fs:
            f._waiters.append(waiter)

        return waiter

这里关键在于把waiter传入到Future._waiters列表中, 那么在Future表为FINISHED的时候, 会调用Future._waiters中所有的waiter的add_result方法

.. code-block:: python

    class Future(object):
        def set_result(self, result):
            with self._condition:
                self._result = result
                self._state = FINISHED
                # 调用所有waiters的add_result
                for waiter in self._waiters:
                    waiter.add_result(self)
                self._condition.notify_all()
            self._invoke_callbacks()

    class _Waiter(object):
        """Provides the event that wait() and as_completed() block on."""
        def __init__(self):
            self.event = threading.Event()
            self.finished_futures = []

        def add_result(self, future):
            self.finished_futures.append(future)

    class _AsCompletedWaiter(_Waiter):
        """Used by as_completed()."""

        def __init__(self):
            super(_AsCompletedWaiter, self).__init__()
            self.lock = threading.Lock()

        def add_result(self, future):
            with self.lock:
                super(_AsCompletedWaiter, self).add_result(future)
                self.event.set()

每次_AsCompletedWaiter.add_result则是把已完成的Future加入到self.finished_futures列表, 然后调用self.event

所以每次有Future完成, 那么_AsCompletedWaiter总是通知as_completed去获取finished_futures, 而对于_AllCompletedWaiter这个waiter

总是在所有的Future都完成之后才会通知self.event

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
                # 这个self.num_pending_calls就是等待完成的Future个数
                self.num_pending_calls -= 1
                if not self.num_pending_calls:
                    # 当所有的Future都FINISHED之后, 才会去设置self.event
                    self.event.set()
    
        def add_result(self, future):
            super().add_result(future)
            self._decrement_pending_calls()


ThreadPoolExecutor
========================

线程池就是利用了Future来追踪调用状态的

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
                    执行_WorkItem.run
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

ProcessPoolExecutor和ThreadPoolExecutor思路一样, 只不过ProcessPoolExecutor的worker是进程, 那么就需要使用到multiprocessing.Process来包装

大概流程图是这样:

.. code-block::

    main_thread -启动-> management_thread   ----call_queue----->         multiprocessing.Process
                                            (call_queue是multiprocessing.Queue)
                                            (result_queue是multiprocessing.SimpleQueue)
                                            <---result_queue----         multiprocessing.Process
                                                 |
                                                 |
     submit --向result_queue发送一个None唤醒----->




ProcessPoolExecutor先启动多个multiprocessing.Process作为worker, 以及两个multiprocessing.Queue, 分别是call_queue和result_queue(当然实现不太一样, 但是都是queue)

然后启动一个manage thread, 该线程负责从待处理队列中获取一个task, 通过call_queue发送给worker, 然后worker会把结果通过result_queue发送给manage thread

然后manage thread会根据结果, 修改对应的Future的状态. 而当调用ProcessPoolExecutor.submit的时候, 主线程会将task添加到待处理队列, 然后

发送一个None到result_queue, 这样是为了唤醒manage thread, 当manage thread被唤醒的时候会去查看result_queue得到的是否是None, 如果是None

则从待处理队列中获取task, 发送给worker, 否则更新Future的状态


而multiprocessing.Queue的实现则是进程间通信是用单向(非双向)pipe, 同时Queue的容量是使用更底层的_multiprocessing.SemLock来实现计数

其实_multiprocessing.SemLock功能上就是一个Semaphore. 这样一端要调用put的时候, 检查semlock是否能获取, 能获取说明计数没用完, 否则证明queue已经满了

同时单向的pipe使得get的一端无法put, 那么就不影响semlock的计数了. 同时multiprocessing.Queue的put是异步的, 也就是背后开启了一个线程, 称为feed thread, 专门从

待发送的buffer中获取到下一个发送对象, 然后序列化(multiprocess封装的pickle)为bytes, 通过pipe发送出去.

主线程通知feed thread是通过threading.Condition


.. code-block::

    multiprocess.Queue

     main thread   -----threading.Condition通知-->    feed thread(multiprocess.Queue启动的子线程)
          put                                            |
      把obj存储到                                        |
           |                                             |
           ------>----- 存储待发送内容的buffer -->    从bufer中拿到obj  --然后通过单向pipe发送---> pipe --->



而multiprocess.SimpleQueue和multiprocess.Queue的区别在于SimpleQueue, SimpleQueue只是一个同步的读加锁写不加锁的单向pipe

在代码注释中提到 *Simplified Queue type -- really just a locked pipe*


所以在ProcessPoolExecutor中, result_queue是一个SimpleQueue, 这样主进程和worker进程都可以向result_queue写入

