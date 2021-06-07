multiprocessing
#####################


fork和CreateProcess的区别
==============================

https://stackoverflow.com/questions/13839935/forking-and-createprocess

fork是Posix系统下使用的系统函数, 而Windows下使用的是_winapi.CreateProcess

使用fork的时候, 父子进程都会从fork点执行, 需要根据fork返回值判断是父进程还是子进程, 从而执行不同的代码, 而CreateProcess则是创建一个新的进程, 但是其会加载python文件

CreateProcess如果传递要执行的python函数?

而关于fork和CreateProcess的内部机制, 比如父子进程的copy-on-write等等, 不做讨论, 这里只是整理python如何使用fork和CreateProcess函数去支持多进程的

Context和Process, Popen
==========================

不同的系统使用的创建进程的函数不一样, 在multiprocessing中只用multiprocessing.Process这个类来代表进程, 区别点就在于Process._popen中

.. code-block:: python

    def f(x):
        print("child process", x, os.getpid())
        time.sleep(0.1)
        return x*x
    def main():
        p = Process(target=f, args=(1, ))
        p.start()
        p.join()
        return

    # multiprocess.Process
    class Process(process.BaseProcess):
        _start_method = None
        @staticmethod
        def _Popen(process_obj):
            return _default_context.get_context().Process._Popen(process_obj)

    # 看看process.BaseProcess
    class BaseProcess(object):
        '''
        Process objects represent activity that is run in a separate process

        The class is analogous to `threading.Thread`
        '''
        def _Popen(self):
            raise NotImplementedError

        def __init__(self, group=None, target=None, name=None, args=(), kwargs={},
                     *, daemon=None):
            # 和Thread一样, 不支持group
            assert group is None, 'group argument must be None for now'
            self._parent_pid = os.getpid()
            # 这里!!!!!
            self._popen = None
            # 这里保存了target和args, 和thread一样
            self._target = target
            self._args = tuple(args)
            self._kwargs = dict(kwargs)
            # 名字
            self._name = name or type(self).__name__ + '-' + \
                         ':'.join(str(i) for i in self._identity)

        # init看不出来, 去看start
        def start(self):
            '''
            Start child process
            '''
            # 这里self._popen不能为空
            assert self._popen is None, 'cannot start a process twice'
            assert self._parent_pid == os.getpid(), \
                   'can only start a process object created by current process'
            assert not _current_process._config.get('daemon'), \
                   'daemonic processes are not allowed to have children'
            _cleanup()
            # 调用self._Popen
            self._popen = self._Popen(self)


所以在start的时候去校验了self._popen, 如果self._popen不为空, 表示进程已经启动了, 同时self._popen是通过self._Popen函数创建的

而在multiprocess.Process中, _Popen定义为

.. code-block:: python

        @staticmethod
        def _Popen(process_obj):
            return _default_context.get_context().Process._Popen(process_obj)


所以就是去context中获取具体的Process, 然后调用其中的_Popen. 所以这里只有multiprocessing.Process一个类, 不同的系统是通过context中的Process来区分具体使用什么Process类的

.. code-block:: python

    if sys.platform != 'win32':
        # Posix下的配置
        pass
    else:
        # Windows下的配置
        class SpawnProcess(process.BaseProcess):
            _start_method = 'spawn'
            @staticmethod
            def _Popen(process_obj):
                from .popen_spawn_win32 import Popen
                return Popen(process_obj)

        class SpawnContext(BaseContext):
            _name = 'spawn'
            Process = SpawnProcess

        _concrete_contexts = {
            'spawn': SpawnContext(),
        }

        _default_context = DefaultContext(_concrete_contexts['spawn'])


Windows下, 使用的Context是SpawnContext, 然后其Process配置是SpawnProcess, 所以当我们在Windows下, 调用multiprocessing.Process.start的时候, 调用

_default_context.get_context().Process._Popen就是等于得到SpawnContext.Process._Popen, 也就是SpawnProcess._Popen

所以multiprocessing中使用不同的Context去指定调用不同的函数去创建进程. multiprocessing.Process只是一个壳, multiprocessing.Process._Popen

的作用是找到Context, 调用真正的Process._Popen, 在Windows下, 这个Process._Popen就是SpawnProcess._Popen, 而SpawnProcess._Popen才是真正创建进程的地方



SpawnProcess/CreateProcess
=============================

.. code-block:: python

    class SpawnProcess(process.BaseProcess):
        _start_method = 'spawn'
        @staticmethod
        def _Popen(process_obj):
            from .popen_spawn_win32 import Popen
            return Popen(process_obj)

真正的_Popen是popen_spawn_win32下的Popen.

.. code-block:: python

    class Popen(object):
        '''
        Start a subprocess to run the code of a process object
        '''
        method = 'spawn'

        def __init__(self, process_obj):
            prep_data = spawn.get_preparation_data(process_obj._name)
    
            # read end of pipe will be "stolen" by the child process
            # -- see spawn_main() in spawn.py.
            # 创建pipe
            rhandle, whandle = _winapi.CreatePipe(None, 0)
            wfd = msvcrt.open_osfhandle(whandle, 0)
            # 拿到命令行参数
            cmd = spawn.get_command_line(parent_pid=os.getpid(),
                                         pipe_handle=rhandle)
            cmd = ' '.join('"%s"' % x for x in cmd)
    
            with open(wfd, 'wb', closefd=True) as to_child:
                # start process
                try:
                    # 调用CreateProcess
                    hp, ht, pid, tid = _winapi.CreateProcess(
                        spawn.get_executable(), cmd,
                        None, None, False, 0, None, None, None)
                    _winapi.CloseHandle(ht)
                except:
                    _winapi.CloseHandle(rhandle)
                    raise
    
                # set attributes of self
                self.pid = pid
                self.returncode = None
                self._handle = hp
                self.sentinel = int(hp)
                util.Finalize(self, _winapi.CloseHandle, (self.sentinel,))
    
                set_spawning_popen(self)
                # send information to child
                try:
                    # 这里这里!!!!!
                    # 发送pre_data给子进程!!!!!!!!!!!!!!!!!
                    reduction.dump(prep_data, to_child)
                    # 发送process_obj给子进程
                    reduction.dump(process_obj, to_child)
                finally:
                    set_spawning_popen(None)

在Windows中, 启动子进程是要执行一个命令的, 类似于执行bash命令, 无法在代码中通过返回值执行执行代码, 而是执行一个cmd(bash)命令. 这里的spawn.get_command_line就是得到子进程要执行的命令

假设我们的python文件叫test_multiprocessing.py, 那么我们得到的命令就是

.. code-block::

    "python.exe" "-B" "-c" "from multiprocessing.spawn import spawn_main; spawn_main(parent_pid=25916, pipe_handle=1112)" "--multiprocessing-fork"

**所以我们看到, Windows下子进程一开始执行的函数是spawn_main, 而不是我们的target函数**

.. code-block:: python

    def spawn_main(pipe_handle, parent_pid=None, tracker_fd=None):
        '''
        Run code specified by data received over pipe
        '''
        assert is_forking(sys.argv)
        if sys.platform == 'win32':
            import msvcrt
            new_handle = reduction.steal_handle(parent_pid, pipe_handle)
            fd = msvcrt.open_osfhandle(new_handle, os.O_RDONLY)
        else:
            from . import semaphore_tracker
            semaphore_tracker._semaphore_tracker._fd = tracker_fd
            fd = pipe_handle
        # 执行_main
        exitcode = _main(fd)
        sys.exit(exitcode)

    # 主要还是在_main
    def _main(fd):
        with os.fdopen(fd, 'rb', closefd=True) as from_parent:
            process.current_process()._inheriting = True
            try:
                # 这里, 接收preparation_data和对象!!!!!!!!!
                preparation_data = reduction.pickle.load(from_parent)
                prepare(preparation_data)
                self = reduction.pickle.load(from_parent)
            finally:
                del process.current_process()._inheriting
        # 然后启动self._bootstrap
        return self._bootstrap()

这里最关键一点就是子进程通过pipe, 接收父进程发送过来的preparation_data和Process实例, 然后执行Process实例的_bootstrap方法!!!!!!!!

在父进程中, 我们看到SpawnProcess.Popen中有

.. code-block:: python

    try:
        # 这里这里!!!!!
        # 发送pre_data和process_obj给子进程!!!!!!!!!!!!!!!!!
        reduction.dump(prep_data, to_child)
        reduction.dump(process_obj, to_child)
    finally:
        set_spawning_popen(None)

所以这里发送了prep_data和process_obj. reduction.dump其实就是pickle.dump, 把prep_data和process_obj写入to_child这个pipe

prep_data是调用prep_data = spawn.get_preparation_data(process_obj._name)得到的. prep_data是一个字典, 其中的key有

.. code-block:: python

    dict_keys(['log_to_stderr', 'authkey', 'name', 'sys_path', 'sys_argv', 'orig_dir', 'dir', 'start_method', 'init_main_from_path'])


在子进程接收到pre_data之后, 调用 prepare(preparation_data), 这个函数的作用则是类似于import我们的py文件.

处理完pre_data之后, 接收到process_obj, 然后调用process_obj._bootstrap函数, 这个函数和Thread一样, 都是去执行我们传入的目标函数

fork
=========

而fork就相对简单一点了, 调用os.fork然后执行multiprocess.Process._bootstrap函数就好了

这是fork下的context

.. code-block:: python

    if sys.platform != 'win32':
    
        class ForkProcess(process.BaseProcess):
            _start_method = 'fork'
            @staticmethod
            def _Popen(process_obj):
                from .popen_fork import Popen
                return Popen(process_obj)
    
        class SpawnProcess(process.BaseProcess):
            _start_method = 'spawn'
            @staticmethod
            def _Popen(process_obj):
                from .popen_spawn_posix import Popen
                return Popen(process_obj)
    
        class ForkServerProcess(process.BaseProcess):
            _start_method = 'forkserver'
            @staticmethod
            def _Popen(process_obj):
                from .popen_forkserver import Popen
                return Popen(process_obj)
    
        class ForkContext(BaseContext):
            _name = 'fork'
            Process = ForkProcess
    
        class SpawnContext(BaseContext):
            _name = 'spawn'
            Process = SpawnProcess
    
        class ForkServerContext(BaseContext):
            _name = 'forkserver'
            Process = ForkServerProcess
            def _check_available(self):
                if not reduction.HAVE_SEND_HANDLE:
                    raise ValueError('forkserver start method not available')
    
        _concrete_contexts = {
            'fork': ForkContext(),
            'spawn': SpawnContext(),
            'forkserver': ForkServerContext(),
        }
        _default_context = DefaultContext(_concrete_contexts['fork'])

尽管Posix下支持fork, spawn和forkserver, 但是其实找的还是fork, 也就是ForkContext.

ForkContext下的Popen则是popen_fork.Popen

.. code-block:: python

    class Popen(object):
        method = 'fork'

        def __init__(self, process_obj):
            sys.stdout.flush()
            sys.stderr.flush()
            self.returncode = None
            self._launch(process_obj)

        def _launch(self, process_obj):
            code = 1
            parent_r, child_w = os.pipe()
            # 直接调用os.fork
            self.pid = os.fork()
            if self.pid == 0:
                try:
                    os.close(parent_r)
                    # 子进程下初始化随机种子
                    if 'random' in sys.modules:
                        import random
                        random.seed()
                    # 子进程下执行我们的target函数
                    code = process_obj._bootstrap()
                finally:
                    os._exit(code)
            else:
                os.close(child_w)
                util.Finalize(self, os.close, (parent_r,))
                self.sentinel = parent_r


queue
==========

pipe, named pip: https://www.tutorialspoint.com/inter_process_communication/inter_process_communication_pipes.htm, https://www.tutorialspoint.com/inter_process_communication/inter_process_communication_named_pipes.htm

1. pipe是单向的, 只能在父子进程之间使用

2. named pipe, aka FIFO, 单向(named pipe也是文件, 可以修改读写模式达到双工https://www.tutorialspoint.com/inter_process_communication/inter_process_communication_named_pipes.htm), 是可以在非父子进程之间使用

unix domain socket vs ip socket: https://serverfault.com/questions/124517/what-is-the-difference-between-unix-sockets-and-tcp-ip-sockets

multiprocessing中的Queue是基于上面IPC机制来实现的

.. code-block:: python

    class Queue(object):
    
        def __init__(self, maxsize=0, *, ctx):
            if maxsize <= 0:
                # Can raise ImportError (see issues #3770 and #23400)
                from .synchronize import SEM_VALUE_MAX as maxsize
            self._maxsize = maxsize
            # 拿到pipe
            self._reader, self._writer = connection.Pipe(duplex=False)

这里的connection.Pipe是根据系统和是否双工, 取不同的类的, Posix下

.. code-block:: python

    if sys.platform != 'win32':
    
        def Pipe(duplex=True):
            '''
            Returns pair of connection objects at either end of a pipe
            '''
            if duplex:
                # 如果双工的话, 使用unix domain socket
                s1, s2 = socket.socketpair()
                s1.setblocking(True)
                s2.setblocking(True)
                # 因为子进程会继承父进程的fd, 所以这里要关闭掉某一个socket
                c1 = Connection(s1.detach())
                c2 = Connection(s2.detach())
            else:
                # 如果是单向的传输, 那么使用两个pipe实现
                fd1, fd2 = os.pipe()
                c1 = Connection(fd1, writable=False)
                c2 = Connection(fd2, readable=False)
    
            return c1, c2

这里需要双工交互的话就使用unix domain socket, 单向的就直接使用pipe了. 而multiprocessing.Queue总是使用pipe, 因为queue是单向的.

put
---------

.. code-block:: python

    class Queue(object):
    
        def __init__(self, maxsize=0, *, ctx):
            self._after_fork()
            return
        
        def _after_fork(self):
            debug('Queue._after_fork()')
            self._notempty = threading.Condition(threading.Lock())
            return
        
        def put(self, obj, block=True, timeout=None):
            assert not self._closed
            if not self._sem.acquire(block, timeout):
                raise Full
        
            with self._notempty:
                if self._thread is None:
                    self._start_thread()
                self._buffer.append(obj)
                self._notempty.notify()

        def _start_thread(self):
            debug('Queue._start_thread()')
        
            # Start thread which transfers data from buffer to pipe
            self._buffer.clear()
            self._thread = threading.Thread(
                # 这里才是真正发送的地方!!!!!!!!!!!!!!!!!!!!
                target=Queue._feed,
                args=(self._buffer, self._notempty, self._send_bytes,
                      self._wlock, self._writer.close, self._ignore_epipe),
                name='QueueFeederThread'
                )
            self._thread.daemon = True
        
            debug('doing self._thread.start()')
            self._thread.start()
            debug('... done self._thread.start()')

启动一个传输线程, 多个线程也可以通过这个Queue发送对象给另外一个进程, put的时候把需要传输的对象加入到self._buffer, 然后通过self._notempty这个锁提供线程安全

真正的发送是在Queue._feed中

.. code-block:: python

    @staticmethod
    def _feed(buffer, notempty, send_bytes, writelock, close, ignore_epipe):
        debug('starting thread to feed data to pipe')
        nacquire = notempty.acquire
        nrelease = notempty.release
        nwait = notempty.wait
        bpopleft = buffer.popleft
        sentinel = _sentinel
        if sys.platform != 'win32':
            wacquire = writelock.acquire
            wrelease = writelock.release
        else:
            wacquire = None

        while 1:
            try:
                nacquire()
                try:
                    if not buffer:
                        nwait()
                finally:
                    nrelease()
                # 全面都是等待通知有对象要传输
                try:
                    while 1:
                        # 拿到对象
                        obj = bpopleft()
                        if obj is sentinel:
                            debug('feeder thread got sentinel -- exiting')
                            close()
                            return

                        # serialize the data before acquiring the lock
                        # 调用pickle.dumps
                        obj = _ForkingPickler.dumps(obj)
                        if wacquire is None:
                            # 发送, 这里的send_bytes最终调用了os.write
                            send_bytes(obj)

get
----------

.. code-block:: python

    def get(self, block=True, timeout=None):
        if block and timeout is None:
            with self._rlock:
                res = self._recv_bytes()
            self._sem.release()
        else:
            if block:
                deadline = time.time() + timeout
            if not self._rlock.acquire(block, timeout):
                raise Empty
            try:
                if block:
                    timeout = deadline - time.time()
                    if timeout < 0 or not self._poll(timeout):
                        raise Empty
                elif not self._poll():
                    raise Empty
                res = self._recv_bytes()
                self._sem.release()
            finally:
                self._rlock.release()
        # unserialize the data after having released the lock
        return _ForkingPickler.loads(res)


这里无非是如果是block模式, 那么拿到读锁, 直接阻塞等待, 否则如果有timeout的话, 如果在n1时间内拿到锁, 那么我们必须在n2=timeout-n1时间内调用self._poll去等待有数据可读的通知

注意的是, self._poll是self._reader.poll, 而最终会调用select


Queue和SimpleQueue的区别
=============================


两种的区别在代码中可以看到, SimpleQueue是没有在读写的时候进行线程安全保护的, 所以如果你明确是一人读一人写, 那么使用SimpleQueue就好了

如果多人读写的话, 使用Queue吧


Pool
==========




