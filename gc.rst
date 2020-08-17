GCCCCCCCCCCCCCCCCCCC
###########################

1. http://www.wklken.me/posts/2015/09/29/python-source-gc.html

2. https://docs.python.org/3/faq/design.html#how-does-python-manage-memory

3. https://rushter.com/blog/python-garbage-collector/

4. http://www.arctrix.com/nas/python/gc/

CPython的对象管理是引用计数为主, 当引用计数为0的时候, 触发回收(这里的回收统称, 不区分回收到os还是回收到内存池), 标记清除为辅去清理循环引用, 清理循环引用的过程也叫GC

如果确定程序没有循环引用的情况或者不在乎, 那么可以关掉GC(gc.disbale)

大体思路是在参考4中

1. For each container object, set gc_refs equal to the object's reference count.

2. For each container object, find which container objects it references and decrement the referenced container's gc_refs field.

3. All container objects that now have a gc_refs field greater than one are referenced from outside the set of container objects.
   We cannot free these objects so we move them to a different set.

4. Any objects referenced from the objects moved also cannot be freed. We move them and all the objects reachable from them too.

5. Objects left in our original set are referenced only by objects within that set (ie. they are inaccessible from Python and are garbage).
   We can now go about freeing these objects.

优化的方向有

1. 尽量在需要加入GC链表的时候才加入, 能不加入就不加入

2. GC频率可以通过threshold, 以及如果待GC对象和当前所有对象的比例小于1/4的话, 就两个GC策略使得即使即使有很多未GC对象也会频繁增加GC时间

PyGC_Head
============

python对象分为GC对象和非GC对象, GC对象包括自定义类, 容器(比如list, tuple, dict等等), 非GC对象则是int, str, float等类型

而非GC对象会使用引用计数法去回收内存, 而GC对象除了也使用引用计数法, 还会使用标记清除法解决循环引用问题.

所有的GC对象都是使用双链表串联起来

.. code-block:: python

    '''
    
        ........A <-> B <-> C <-> D..............
    
    '''

双链表的prev和next是在PyGC_Head这个结构中

.. code-block:: c

    typedef union _gc_head {
        struct {
            union _gc_head *gc_next;
            union _gc_head *gc_prev;
            Py_ssize_t gc_refs;
        } gc;
        double dummy;  /* force worst-case alignment */
    } PyGC_Head;

其中的gc_next和gc_prev就是链表的指针了, 注意的是, gc_refs并不是"实际的"引用计数, 实际的引用计数保存在PyObject这个结构中, 这个引用计数是复制一份真实的引用计数

这样gc的过程中操作引用计数的时候就不影响实际的引用计数了


在我们创建一个对象的时候, 如果该对象是GC对象, 那么会将其加入到GC链表中

.. code-block:: c

    PyObject *
    PyType_GenericAlloc(PyTypeObject *type, Py_ssize_t nitems)
    {
        PyObject *obj;
        const size_t size = _PyObject_VAR_SIZE(type, nitems+1);
        /* note that we need to add one, for the sentinel */
    
        if (PyType_IS_GC(type))
            // 加入GC链表!!!!!!!!!!!!!
            obj = _PyObject_GC_Malloc(size);
        else
            obj = (PyObject *)PyObject_MALLOC(size);
    
    }

    // _PyObject_GC_Malloc会调用到_PyObject_GC_Alloc

    static PyObject *
    _PyObject_GC_Alloc(int use_calloc, size_t basicsize)
    {
        PyObject *op;
        PyGC_Head *g;
        size_t size;
        if (basicsize > PY_SSIZE_T_MAX - sizeof(PyGC_Head))
            return PyErr_NoMemory();
        // 要分配的对象的内存大小要额外加上PyGC_Head这个结构的大小
        size = sizeof(PyGC_Head) + basicsize;
        if (use_calloc)
            g = (PyGC_Head *)PyObject_Calloc(1, size);
        else
            g = (PyGC_Head *)PyObject_Malloc(size);

        // 省略了初始化的代码

	if (PyType_IS_GC(type))
            // 这里将会把对象加入到GC链表中
	    _PyObject_GC_TRACK(obj);

        op = FROM_GC(g);
        return op;
    }
    

注意到分配内存的时候额外分配了一个PyGC_Head的结果, 同时FROM_GC表示指针移动1格然后强制转换为PyObject*

.. code-block:: c

    #define FROM_GC(g) ((PyObject *)(((PyGC_Head *)g)+1))

    /* Get an object's GC head 而AS_GC则是当前PyObject指针往上移动一格然后得到GC头 */
    #define AS_GC(o) ((PyGC_Head *)(o)-1)

这说明了一个GCd对象额外加了PyGC_Head

.. code-block:: python

    '''
    GC对象的实际结构, x, y, z就是其中的属性, 然后在代码中一般拿到的是PyObject指针, 需要获取GC头的时候, 调用AS_GC, PyObject指针往上移动一格
   
        PyGC_Head
  ----> PyObject 
        xxx
        yyy
        zzz
    
    '''


引用计数
============

引用计数是对象被引用的次数, 引用计数不为0, 那么显然该对象不能被释放掉, 引用计数在对象创建的时候被设置为0, 每次被引用, 则引用计数加1, 解除引用, 引用计数减少1

当引用计数为0的时候会调用__del__方法

.. code-block:: python

    In [1]: import sys
    
    In [2]: class A:
       ...:     def __del__(self):
       ...:         print(self.name, "__del__")
       ...:         return
       ...:     def __init__(self, name):
       ...:         self.name = name
       ...:         return
       ...:
    
    In [3]: def test_refcount():
       ...:     a = A("a")
       ...:     print(sys.getrefcount(a))
       ...:     l = [a]
       ...:     print(sys.getrefcount(a))
       ...:     del a
       ...:     return
       ...:
    
    In [4]: test_refcount()
    2
    3
    a __del__

为什么a的引用计数是2而不是1? 看一下字节码

.. code-block:: python

    In [6]: dis.dis(test_refcount)
      2           0 LOAD_GLOBAL              0 (A)
                  2 LOAD_CONST               1 ('a')
                  4 CALL_FUNCTION            1
                  6 STORE_FAST               0 (a)
    
      3           8 LOAD_GLOBAL              1 (print)
                 10 LOAD_GLOBAL              2 (sys)
                 12 LOAD_ATTR                3 (getrefcount)
                 14 LOAD_FAST                0 (a)
                 16 CALL_FUNCTION            1
                 18 CALL_FUNCTION            1
                 20 POP_TOP
    
      4          22 LOAD_FAST                0 (a)
                 24 BUILD_LIST               1
                 26 STORE_FAST               1 (l)
    
      5          28 LOAD_GLOBAL              1 (print)
                 30 LOAD_GLOBAL              2 (sys)
                 32 LOAD_ATTR                3 (getrefcount)
                 34 LOAD_FAST                0 (a)
                 36 CALL_FUNCTION            1
                 38 CALL_FUNCTION            1
                 40 POP_TOP
    
      6          42 DELETE_FAST              0 (a)
    
      7          44 LOAD_CONST               0 (None)
                 46 RETURN_VALUE

首先我们创建实例a的时候, 字节码是CALL_FUNCTION, 其中会调用PyType_GenericAlloc去分配一个GC对象并且初始化引用计数为1

.. code-block:: c

    PyObject *
    PyType_GenericAlloc(PyTypeObject *type, Py_ssize_t nitems)
    {
        PyObject *obj;
        // 分配obj的内存
    
        // 所有属性设置为0
        memset(obj, '\0', size);
    
        if (type->tp_flags & Py_TPFLAGS_HEAPTYPE)
            Py_INCREF(type);
    
        if (type->tp_itemsize == 0)
            // 这里和下面都会去初始化引用计数
            (void)PyObject_INIT(obj, type);
        else
            (void) PyObject_INIT_VAR((PyVarObject *)obj, type, nitems);
    
        // 加入GC链表中
        if (PyType_IS_GC(type))
            _PyObject_GC_TRACK(obj);
        return obj;
    }

    // PyObject_INIT最终会调用到 _Py_NewReference函数
    void
    _Py_NewReference(PyObject *op)
    {
        _Py_INC_REFTOTAL;
        // 初始化引用计数为1
        op->ob_refcnt = 1;
        _Py_AddToAllObjects(op, 1);
        _Py_INC_TPALLOCS(op);
    }

然后STORE_FAST使得局部变量名a指向新生成的实例

.. code-block:: c

   PREDICTED(STORE_FAST);
   TARGET(STORE_FAST) {
       PyObject *value = POP();
       SETLOCAL(oparg, value);
       FAST_DISPATCH();
   }

    #define GETLOCAL(i)     (fastlocals[i])
    // SETLOCAL只是减少a原本对象的引用计数, 如果a没有指向过任何对象, 那么就不进行任何操作
    #define SETLOCAL(i, value)      do { PyObject *tmp = GETLOCAL(i); \
                                         GETLOCAL(i) = value; \
                                         Py_XDECREF(tmp); } while (0)

在SETLOCAL中并没有提升value, 也就是实例A("a")的引用计数, 所以此时A("a")的引用计数依然是1

然后我们在调用sys.getrefcount的时候, 需要先加载a, 字节码是LOAD_FAST

.. code-block:: c

    TARGET(LOAD_FAST) {
        PyObject *value = GETLOCAL(oparg);
        if (value == NULL) {
            format_exc_check_arg(PyExc_UnboundLocalError,
                                 UNBOUNDLOCAL_ERROR_MSG,
                                 PyTuple_GetItem(co->co_varnames, oparg));
            goto error;
        }
        Py_INCREF(value);
        PUSH(value);
        FAST_DISPATCH();
    }


我们看到LOAD_FAST的时候会把对象的引用计数增加1, 此时a的引用计数就是2了, 所以sys.getrefcount的结果就是2, 当sys.getrefcount调用结束之后

会调用POP_TOP, 把栈顶对象给pop掉, 此时栈顶的对象则是a, 那么此时a的引用计数就又回到了1

.. code-block:: c

    TARGET(POP_TOP) {
        PyObject *value = POP();
        // 减少引用计数
        Py_DECREF(value);
        FAST_DISPATCH();
    }


**所以python中加载对象总是会临时增加引用计数, 所以sys.getrefcount会不准**

最后当一个对象的引用计数为0之后, 那么就会调用到__del__了, 并且把对象从GC链表中移除

.. code-block:: c

    #define Py_DECREF(op)                                   \
        do {                                                \
            PyObject *_py_decref_tmp = (PyObject *)(op);    \
            if (_Py_DEC_REFTOTAL  _Py_REF_DEBUG_COMMA       \
            // 判断引用计数是否为0
            --(_py_decref_tmp)->ob_refcnt != 0)             \
                _Py_CHECK_REFCNT(_py_decref_tmp)            \
            else                                            \
                // 如果为0则调用到__del__方法, 并且把对象从gc链表中移除
                _Py_Dealloc(_py_decref_tmp);                \
        } while (0)


GC
========

CPython的GC处理并不是周期性执行的, 而是当某一代的对象个数达到设置的阈值上限的时候, 就触发GC过程. 

而在CPython中GC对象被分配到3个"世代"中, 一开始对象会被移动到0世代, 然后0世代的GC对象个数达到阈值之后触发GC, GC之后剩下的没有被回收掉的就会被加入到

1世代, 1世代的阈值达到之后就把剩下的移动到2世代, 那么2世代也达到阈值之后呢? 那么计数直接置0, 当然对象依然在2世代链表中只是计数置0, 重新统计


.. code-block:: python

    '''
      每个世代的头也是GC对象
            generation0_head <-> .......A0 <-> B0 <-> C0 <-> D0..............
            generation1_head <-> .......A1 <-> B1 <-> C1 <-> D1..............
            generation2_head <-> .......A2 <-> B2 <-> C2 <-> D2..............
    
    '''


每个世代的对象个数阈值可以在gc模块中得到

.. code-block:: python

    In [38]: gc.get_threshold()
    Out[38]: (700, 10, 10)
    
    In [41]: gc.set_threshold(500, 20, 20)
    
    In [42]: gc.get_threshold()
    Out[42]: (500, 20, 20)

这样说明0世代的个数达到700个之后, 触发gc, 移动剩下的对下个到1世代, 然后1世代计数加1, 2世代也一样, 所以也就是说没700个对象使1世代计数加1, 没7000个对象使得2世代计数加1

所以70000个对象能填满2世代, 接着0, 1, 2世代清0, 继续下一次gc.


当要分配一个新的GC对象, 会检查0世代的对象个数, 达到阈值就执行GC

.. code-block:: c

    static PyObject *
    _PyObject_GC_Alloc(int use_calloc, size_t basicsize)
    {
        // 创建对象的过程省略

        // 设置引用计数为0
        g->gc.gc_refs = 0;
        _PyGCHead_SET_REFS(g, GC_UNTRACKED);
        generations[0].count++; /* number of allocated GC objects */
        if (generations[0].count > generations[0].threshold &&
            enabled &&
            generations[0].threshold &&
            !collecting &&
            !PyErr_Occurred()) {
            // 如果0世代的对象个数达到了阈值, 启动GC
            collecting = 1;
            collect_generations();
            collecting = 0;
        }
        op = FROM_GC(g);
        return op;
    }


**注意这里collect_generations是GC的实现, 里面没有释放GIL的逻辑, 意味着一旦执行了GC, 那么整个进程都停止了工作, Stop the World**

GC的过程就是参考4中提到的过程, 当我们遍历到gc链表上的某个对象的时候:

1. For each container object, set gc_refs equal to the object's reference count. 复制一份对象的引用计数, 保存到PyGC_Head中

2. For each container object, find which container objects it references and decrement the referenced container's gc_refs field.
   找到该对象引用的其他对象, 将其他对象的引用计数减1

3. All container objects that now have a gc_refs field greater than one are referenced from outside the set of container objects.
   We cannot free these objects so we move them to a different set.
   所有引用计数大于0的对象都是还被外部对象引用着, 不能释放

4. Any objects referenced from the objects moved also cannot be freed. We move them and all the objects reachable from them too.
   所有不能删除的对象所引用的对象也不能释放

5. Objects left in our original set are referenced only by objects within that set (ie. they are inaccessible from Python and are garbage).
   We can now go about freeing these objects.
   此时剩下的引用计数等于0的对象就可以释放了

可达, 暂时不可达和不可达
===========================

GC的主要逻辑就是区分对象是否是可达的, 一个对象可达意味着该对象引用计数不为0, 或者虽然引用计数为0但是其他引用计数不为0的对象引用着它, 不可达表示没有引用任何对象同时也没有被任何对象引用

遍历GC链表上的对象的时候, 引用计数减少1不是减少对象本身的引用计数, 而是减少该对象引用的对象的计数, A和B互相引用, 当遍历到A的时候, 找到A引用的对象

B, 然后减少B的引用计数, 遍历到B, 找到B引用的A, 然后减少A的引用计数, A和B的引用计数最后都为0, 回收两个对象

.. code-block:: python

    '''
    
        A(1) <-> B(1)
    
    '''


但是要考虑到这样的情况, A和B互相引用, 同时C引用了B, 那么最后A的计数为0, 但是A不能被释放, 因为B不能释放并且B引用着A, 这个时候A称为暂时不可达对象, 是不可以释放的


.. code-block:: python

    '''
    
        A(1) <-> B(2) <-C
    
    '''


gc的主函数是gc.collect

.. code-block:: c

    static Py_ssize_t
    collect(int generation, Py_ssize_t *n_collected, Py_ssize_t *n_uncollectable,
            int nofail)
    {

        update_refs(young);
        subtract_refs(young);
    
        gc_list_init(&unreachable);
        move_unreachable(young, &unreachable);
    
    }


update_refs是复制引用计数到PyGC_Head中, subtract_refs则是找到对象的引用对象, 然后减少引用对象的引用计数

.. code-block:: c

    static void
    subtract_refs(PyGC_Head *containers)
    {
        traverseproc traverse;
        PyGC_Head *gc = containers->gc.gc_next;
        for (; gc != containers; gc=gc->gc.gc_next) {
            traverse = Py_TYPE(FROM_GC(gc))->tp_traverse;
            // 调用对象的遍历方法, 遍历该对象所有引用的对象, 每遍历到一个对象, 调用visit_decref函数
            (void) traverse(FROM_GC(gc),
                           (visitproc)visit_decref,
                           NULL);
        }
    }

    // visit_decref则是直接把对象的引用计数减少1
    static int
    visit_decref(PyObject *op, void *data)
    {
        assert(op != NULL);
        if (PyObject_IS_GC(op)) {
            PyGC_Head *gc = AS_GC(op);
            /* We're only interested in gc_refs for objects in the
             * generation being collected, which can be recognized
             * because only they have positive gc_refs.
             */
            assert(_PyGCHead_REFS(gc) != 0); /* else refcount was too small */
            if (_PyGCHead_REFS(gc) > 0)
                _PyGCHead_DECREF(gc);
        }
        return 0;
    }

然后gc_list_init则是初始化一个为unreachable的gc链表, 初始化其为空, move_unreachable则是根据引用计数来判断对象的状态, 也就是上面流程中的3, 4


.. code-block:: c

    static void
    move_unreachable(PyGC_Head *young, PyGC_Head *unreachable)
    {
        PyGC_Head *gc = young->gc.gc_next;
    
        while (gc != young) {
            PyGC_Head *next;
    
            if (_PyGCHead_REFS(gc)) {
                PyObject *op = FROM_GC(gc);
                traverseproc traverse = Py_TYPE(op)->tp_traverse;
                assert(_PyGCHead_REFS(gc) > 0);
                _PyGCHead_SET_REFS(gc, GC_REACHABLE);
                (void) traverse(op,
                                (visitproc)visit_reachable,
                                (void *)young);
                next = gc->gc.gc_next;
                if (PyTuple_CheckExact(op)) {
                    _PyTuple_MaybeUntrack(op);
                }
            }
            else {
                next = gc->gc.gc_next;
                gc_list_move(gc, unreachable);
                _PyGCHead_SET_REFS(gc, GC_TENTATIVELY_UNREACHABLE);
            }
            gc = next;
        }
    }


其中的逻辑就是

1. 如果引用计数大于0, 那么遍历其所有的引用对象, 把它们从unreachable列表移回来, 并且把它们的状态设置为可达GC_REACHABLE

2. 如果引用计数为0, 那么暂时把它放入到unreachable列表中

GC_TENTATIVELY_UNREACHABLE状态表示该对象当前引用计数为0, 但是后面有可能有某个可达对象引用了该对象, 该对象到时候会从unreachable列表移除的, 所以称为暂时不可达


下面就是A先被设置为暂时不可达, 然后遍历到B之后, A会被设置为可达

.. code-block:: python

    '''
    
        A(1) <-> B(1)
    
    '''

内部容器可以不加入GC链表
==================================

在源码的注释中提到, 有些时候对象可以不加入gc链表的

Certain types of container cannot participate in a reference cycle, and so do not need to be tracked by the garbage collector.

Untracking these objects reduces the cost of garbage collections. However, determining which objects may be untracked is not free, and the costs must be

weighed against the benefits for garbage collection.

某些容器对象在一开始创建的情况下不可能产生循环引用, 那么这些对象没必要一开始就加入GC链表, 因为加入GC链表的话会使GC过程变慢(对象多了就慢了嘛)

但是判断对象是否应该不被加入GC链表也有耗时的, 所以需要权衡

There are two possible strategies for when to untrack a container:

以下两种情况不需要加入GC链表

i) When the container is created.
ii) When the container is examined by the garbage collector.

1. 容器刚刚开始被创建的时候

2. 


第一种情况比如创建一个dict的时候, 一个新的dict怎么能和其他对象互相引用呢? 因为之前该dict不存在, 所有其他对象不可能引用它, 所以创建一个新的dict的时候, 无须加入gc链表

.. code-block:: c

    static PyObject *
    new_dict(PyDictKeysObject *keys, PyObject **values)
    {
        // 省略了很多代码

        // 创建新的dict并没有使用PyType_GenericAlloc而是使用下面这个
        mp = PyObject_GC_New(PyDictObject, &PyDict_Type);
    }


    // PyObject_GC_New最终会调用到下面
    PyObject *
    _PyObject_GC_New(PyTypeObject *tp)
    {
        // 该函数仅仅GC malloc了而已, 并没有加入GC链表
        PyObject *op = _PyObject_GC_Malloc(_PyObject_SIZE(tp));
        if (op != NULL)
            op = PyObject_INIT(op, tp);
        return op;
    }


以及只包含非GC对象的tuple也没必要加入GC链表, 因为tuple不可变, 显然空tuple也不需要加入GC链表

Tuples containing only immutable objects (integers, strings etc, and recursively, tuples of immutable objects) do not need to be tracked.

但是tuple是否需要加入GC链表并不是在tuple创建的时候决定的, 因为tuple对象也有自己的缓存池, 预分配的tuple显然无法决定之后是否被加入GC链表

比如往tuple内添加的元素可能是GC对象(在C层面, tuple可以动态添加元素)

Dictionaries containing only immutable objects also do not need to be tracked. Dictionaries are untracked when created.

dict同样在创建的时候可以不加入GC链表, 同时只包含非GC对象的dict也可以不加入GC链表. 同时GC遍历dict的时候如果当前dict只包含非GC对象, 那么将会把该dict从gc链表中移除



GC阈值和频率
===============

https://bugs.python.org/issue4074

https://mail.python.org/pipermail/python-dev/2008-June/080579.html

如果存在大量未被GC的GC对象的时候, CPython的早期版本的GC遍历时间会指数级上升, 当然这个问题已经被python3给修复了

如果仅仅是按照上面的gc逻辑的话, 一旦存在非常多的未GC对象, 那么显然会导致我们遍历的对象个数越来越多, 那么GC的时间就越来越久

.. code-block:: python

    class A:
        pass
    
    def list_of_class():
        l = []
        prev_time = time.time()
        for i in range(10000000):
            if i % 1000000 == 0:
                cur_time = time.time()
                print(i, cur_time - prev_time)
                prev_time = cur_time
            l.append(A())

    def list_of_tuples():
        l = []
        prev_time = time.time()
        for i in range(10000000):
            if i % 1000000 == 0:
                cur_time = time.time()
                print(i, cur_time - prev_time)
                prev_time = cur_time
            l.append((i % 2, i % 12))

    def list_of_dicts():
        l = []
        prev_time = time.time()
        for i in range(10000000):
            if i % 1000000 == 0:
                cur_time = time.time()
                print(i, cur_time - prev_time)
                prev_time = cur_time
            l.append({i % 2: {i: 12}})

上面的例子中list_of_dicts和list_of_tuples都不会增加GC的遍历时间, 是因为根据上一节中提到, dict和tuple如果其中的元素只有非GC对象的话, 这些dict和tuple都会从gc链表中移除的

所以dict和tuple并不会增加GC链表上的对象个数, 而list_of_class中我们append的是A这个类, 必然会加入GC链表的, 但是我们的时间也不是指数级别的, 这是因为CPython中也尽量避免巨大对象集合下的GC

当long_lived_pending < long_lived_total / 4的时候, 说明当前需要GC的个数/当前总的对象个数的比例小于1/4, 那么CPython就不gc了, 也就是随着对象个数越多, CPython越不可能去GC

一开始我们把新对象加入到0世代, 然后把0世代的计数加1, 这个计数也是0世代的对象个数, 而1世代, 2世代的计数和对象个数分别都是0)

.. code-block:: python

    '''
    
                      0  500
    对象链表及个数    1  0
    计数                 0
    对象链表及个数    2  0
    计数                 0

    '''

一般情况下, 0代的对象每达到701个, 触发在0代上的GC, 此时遍历的个数是常数级别的, 然后把剩下的可达对象加入到1代中, 然后一致重复


.. code-block:: python

    '''
    
    0  701            0  0     0世代的个数清0
    1  0              1  680   这个680就是701个对象中还剩多少个是可达的, 把它们都合并到1世代中
       0     ====>    1
    2  0              2  0
       0                 0

    '''


当0代和1代的计数满了之后, 此时会触发1代的gc, 先把0代和1代的对象合并起来, 然后进行GC, 同时把剩下的可达对象给合并到2代链表中

.. code-block:: python

    '''

    0  701               0  0     0世代的计数清0
    1  5901              1  0     1世代的计数和对象链表也清0
       11     ====>         0
    2  0                 2  6100  把1世代和0世代的对象都合并到2世代中
       0                    1     2世代的计数加1

    '''

接着继续进行0世代和1世代的gc过程, 最后2世代也满之后

1. 把0世代, 1世代的对象合并到2世代上的gc对象链表中

2. 遍历2世代对象链表, 进行GC

3. 把0, 1, 2世代的计数清0, 这样还可以继续把对象加入到gc链表, 同时记录着当前所有对象的总链表, 也就是2世代链表


.. code-block:: python

    '''

    0  701                   0  0
    1  6450                  1  0    0世代和1世代的所有待GC对象(6450+701)进行GC之后, 加入到2世代中(也就是总的GC对象链表中)
       11      ====>            0
    2  30000                 2  30458  这个计数是当前总的GC对象链表及其长度
       11                    0         2世代的计数清0了

    '''

既然2世代的遍历是遍历所有的对象, 那么显然如果我们一直不断地创建对象不释放, 那么2世代上要遍历的对象个数就越来越多, 时间就i越来越慢, 呈指数增长

但是在CPython中, 一旦当前待gc的对象long_lived_pending, 也就是当前1世代和0世代的对象总数, 和当前等待GC对象的总数, 也就是世代2的链表对象总数long_lived_total. 两者的

比例小于1/4的话, 就不GC了.

所以我们创建了很多很多的对象然后不GC的话, 那么显然long_lived_total越来越大, 而long_lived_pending则是有上界的(因为long_lived_pending)收到0世代和1世代的阈值影响

所以对象越多, long_lived_total越大, 分母越大, 那么我们就越不会遍历总对象链表, 那么此时时间是常数级的, 等于0世代和1世代的链表长度


weakref
===========

弱引用不会增加引用计数, 所以弱引用不会影响GC, 即使存在弱引用互相引用, 由于不会增加引用计数, 所以当del x之后, 即使有一个weakref引用x, 但是python依然能把x给GC掉

.. code-block:: python

    In [1]:     import weakref
       ...:     import sys
       ...:
       ...:     class M:
       ...:         def __init__(self, name):
       ...:             self.name = name
       ...:
       ...:
       ...:     x=M('x')
       ...:
       ...:
       ...:     print(sys.getrefcount(x))
       ...:
       ...:     r=weakref.ref(x)
       ...:     print(r)
       ...:     print(sys.getrefcount(x))
       ...:
    2
    <weakref at 0x0000024ED8E966D8; to 'M' at 0x0000024ED8E75CC0>
    2

The trouble with Finalizers
========================================

GC和析构的问题, 如果两个循环引用的对象中的__del__又互相引用了对方, 假设A, B互相引用, 然后A的__del__引用了B, B.__del__又引用了A, 那么调用A的__del__之后A就被释放了

但是后续如果调用了B.__del__就访问了一个已经无效的内存, 这样的情况怎么办? 参考4中说这无解, 应该避免, 所以GC不会清理掉具有__del__的对象的(或者说具有finalizers的对象)

*Since there is no good solution to this problem, cycles that are referenced from objects with finalizers are not freed.*

同时定义__del__看起来不是不是很安全的样子


总结起来就是当python的__del__方法不存在就好了, 定义一个release方法显式释放资源




