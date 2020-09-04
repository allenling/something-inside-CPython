Python Coroutine
###################

可迭代对象/迭代器
=========================

https://wiki.python.org/moin/Iterator

一个可迭代对象是实现了__iter__方法, 该__iter__方法返回了一个迭代器对象

一个迭代器对象是实现了__next__方法, 调用__next__方法会返回下一个元素, 最后会引发StopIteration异常表示迭代结束

一般来说, 一个可迭代对象的目的就是迭代其本身的数据, 为什么不直接迭代可迭代对象本身而是引申出一个迭代器对象? 从而在迭代器对象上迭代呢?

比如按照上面的定义, 迭代list对象的时候显然是迭代list.__iter__返回的的迭代器对象而不是迭代list本身, 为什么不是直接迭代list本身呢?

**因为要支持重复迭代.** 这样每次迭代list的时候都是从头开始迭代的. 如果是直接迭代list本身的话, 函数a迭代到list第5个元素, 而函数b想要从头开始迭代list的时候怎么办?

所以要从可迭代对象中抽出一个迭代器对象. 以list为例子, 我们看看字节码过程


.. code-block:: python

    In [123]: x = [1, 2, 3]

    In [124]: dis.dis("for i in x:\n    print(i)")
      1           0 SETUP_LOOP              20 (to 22)
                  2 LOAD_NAME                0 (x)
                  4 GET_ITER
            >>    6 FOR_ITER                12 (to 20)
                  8 STORE_NAME               1 (i)

      2          10 LOAD_NAME                2 (print)
                 12 LOAD_NAME                1 (i)
                 14 CALL_FUNCTION            1
                 16 POP_TOP
                 18 JUMP_ABSOLUTE            6
            >>   20 POP_BLOCK
            >>   22 LOAD_CONST               0 (None)
                 24 RETURN_VALUE

GET_ITER就是获取可迭代对象的迭代器对象, 然后FOR_ITER就是在迭代器对象上进行迭代(调用__next__获取下一个元素)

GET_ITER就是调用对象的__iter__方法

.. code-block:: c++

    TARGET(GET_ITER) {
        /* before: [obj]; after [getiter(obj)] */
        PyObject *iterable = TOP();
        PyObject *iter = PyObject_GetIter(iterable);
    }

    // PyObject_GetIter函数找到__iter__然后调用
    PyObject_GetIter(PyObject *o)
    {
        PyTypeObject *t = o->ob_type;
        getiterfunc f = NULL;
        // 这里的tp_iter就是__iter__方法
        f = t->tp_iter;
        if (f == NULL) {
            // 省略代码
        }
        else {
            // 返回__iter__的结果
            PyObject *res = (*f)(o);
            // 省略代码
            return res;
        }
    }

在list实现中有一个专门的结构称为list迭代器, 用来迭代list对象的

.. code-block:: c++

    PyTypeObject PyList_Type = {
        PyVarObject_HEAD_INIT(&PyType_Type, 0)
        list_iter,                                  /* tp_iter */
    }
    // list_iter函数返回一个叫listiterobject的对象
    static PyObject *
    list_iter(PyObject *seq)
    {
        listiterobject *it;
        if (!PyList_Check(seq)) {
            PyErr_BadInternalCall();
            return NULL;
        }
        // 生成一个listiterobject对象!!!!!!!!!!!!!!!!
        it = PyObject_GC_New(listiterobject, &PyListIter_Type);
        if (it == NULL)
            return NULL;
        it->it_index = 0;
        Py_INCREF(seq);
        // 这个对象保存了list对象!!!!!!!!!!!!
        it->it_seq = (PyListObject *)seq;
        _PyObject_GC_TRACK(it);
        return (PyObject *)it;
    }

可以看到listiterobject对象保存了list对象, 同时其属性index被设置为0, 那么迭代的时候就是根据这个index访问list对象的元素了

FOR_ITER就是迭代的过程, FOR_ITER本质上就是调用__next__取获取下一个元素

.. code-block:: c++

    TARGET(FOR_ITER) {
        /* before: [iter]; after: [iter, iter()] *or* [] */
        PyObject *iter = TOP();
        // 调用listiterobject.tp_iternext方法!!!!得到下一个元素
        PyObject *next = (*iter->ob_type->tp_iternext)(iter);
    }

tp_iternext就是__next__方法, listiterobject的__next__则是根据index取获取list对应的元素

.. code-block:: c++

    PyTypeObject PyListIter_Type = {
        PyVarObject_HEAD_INIT(&PyType_Type, 0)
        // listiterobject.__next__
        (iternextfunc)listiter_next,                /* tp_iternext */
    }

    //
    static PyObject *
    listiter_next(listiterobject *it)
    {
        PyListObject *seq;
        PyObject *item;
        assert(it != NULL);
        // 得到listiterobject中保存的list对象!!!!!!!!!
        seq = it->it_seq;
        if (seq == NULL)
            return NULL;
        assert(PyList_Check(seq));
        if (it->it_index < PyList_GET_SIZE(seq)) {
            // 根据index获取list上的元素!!!!!!!!!!!!!!
            item = PyList_GET_ITEM(seq, it->it_index);
            // 然后index自增!!!!!!!!!!!!!
            ++it->it_index;
            Py_INCREF(item);
            return item;
        }
        it->it_seq = NULL;
        Py_DECREF(seq);
        return NULL;
    }

所以我们看到迭代可迭代对象的时候, 是迭代其对应的迭代器对象. 当然如果__iter__返回的是self, 那么迭代器对象就是本身了

生成器模式和生成器对象
===========================

1. https://wiki.python.org/moin/Generators
2. https://www.programiz.com/python-programming/generator

如果我们要生成n个元素, 一般我们会一次生成所有需要的元素

.. code-block:: python

    # Build and return a list
    def firstn(n):
        num, nums = 0, []
        while num < n:
            nums.append(num)
            num += 1
        return nums
    sum_of_first_n = sum(firstn(1000000))

显然firstn一次就i生成了1000000个元素, 这样是很暂用内存的, 我们希望一次只生成一个元素, 所以我可以借助迭代器对象来实现

.. code-block:: python

    # Using the generator pattern (an iterable)
    class firstn(object):
        def __init__(self, n):
            self.n = n
            self.num = 0
        def __iter__(self):
            return self
        # Python 3 compatibility
        def __next__(self):
            return self.next()
        def next(self):
            if self.num < self.n:
                cur, self.num = self.num, self.num+1
                return cur
            else:
                raise StopIteration()
    sum_of_first_n = sum(firstn(1000000))

我们看到firstn是一个可迭代对象, 因为其实现了__iter__, 其__iter__返回了一个可迭代对象, 这个对象是firstn自己, 因为__iter__返回了self并且firstn实现了__next__

在__next__方法中, 每次都只返回一个元素, 那么这样每次迭代firstn这个对象的时候一次只会返回一个元素, 最后会引发StopIteration来结束迭代

但是这样使用迭代器的方式比较麻烦, python提供了一个更简单更直接(直觉)的方式创建生成器

This will perform as we expect, but we have the following issues:

1. there is a lot of boilerplate
2. the logic has to be expressed in a somewhat convoluted way


.. code-block:: python

    # a generator that yields items instead of returning a list
    def firstn(n):
        num = 0
        while num < n:
            yield num
            num += 1

    sum_of_first_n = sum(firstn(1000000))

这样的firstn函数中不需要__iter__和__next__, 这些都被python自己处理了

参考2中说到: *Simply speaking, a generator is a function that returns an object (iterator) which we can iterate over (one value at a time).*

看看迭代生成的过程是怎么样的

.. code-block:: python

    In [125]: def test_g():
         ...:     for i in range(10):
         ...:         yield i
         ...:     return
         ...:

    In [126]: dis.dis(test_g)
      2           0 SETUP_LOOP              22 (to 24)
                  2 LOAD_GLOBAL              0 (range)
                  4 LOAD_CONST               1 (10)
                  6 CALL_FUNCTION            1
                  8 GET_ITER
            >>   10 FOR_ITER                10 (to 22)
                 12 STORE_FAST               0 (i)

      3          14 LOAD_FAST                0 (i)
                 16 YIELD_VALUE
                 18 POP_TOP
                 20 JUMP_ABSOLUTE           10
            >>   22 POP_BLOCK

      4     >>   24 LOAD_CONST               0 (None)
                 26 RETURN_VALUE

显然迭代生成器依然是遵循迭代协议, 返回对象的迭代器对象, 对迭代器对象进行调用__next__方法

我们看看生成器对象的__iter__和__next__

.. code-block:: c++

    PyTypeObject PyGen_Type = {
        PyVarObject_HEAD_INIT(&PyType_Type, 0)
        "generator",                                /* tp_name */
        PyObject_SelfIter,                          /* tp_iter */
        (iternextfunc)gen_iternext,                 /* tp_iternext */
    }

    // PyObject_SelfIter返回的self
    PyObject *
    PyObject_SelfIter(PyObject *obj)
    {
        Py_INCREF(obj);
        return obj;
    }

    // gen_iternext则是调用send方法
    static PyObject *
    gen_iternext(PyGenObject *gen)
    {
        return gen_send_ex(gen, NULL, 0, 0);
    }
    // 这是生成器在python对象的定义, 也就是generator.send
    PyObject *
    _PyGen_Send(PyGenObject *gen, PyObject *arg)
    {
        return gen_send_ex(gen, arg, 0, 0);
    }

**我们看到生成器的迭代器对象就是自己, 而迭代的__next__本质上调用了generator.send(None), 所以生成器运行的方式本质上只有generator.send一种!**

生成器是一次只需要生成和返回一个元素, 那么显然生成器对象需要记住当前迭代到了哪里? 在迭代器的例子中我们使用类, 把计数记住在对象中, 但是不是所有的生成器都是计数

所以生成器需要有一种方式记住运行到了哪里, 那么yield就是记住生成器执行到哪里的语句. 在dis生成器内部的时候, 会看到有YIELD_VALUE这个字节码

.. code-block:: c++

        TARGET(YIELD_VALUE) {
            retval = POP();

            if (co->co_flags & CO_ASYNC_GENERATOR) {
                PyObject *w = _PyAsyncGenValueWrapperNew(retval);
                Py_DECREF(retval);
                if (w == NULL) {
                    retval = NULL;
                    goto error;
                }
                retval = w;
            }
            // 记住生成器执行到哪个栈了
            f->f_stacktop = stack_pointer;
            why = WHY_YIELD;
            // 退出当前字节码过程
            goto fast_yield;
        }

我们看到这个字节码先把yield的结果包装一下, 然后把当前的frame对象的栈指针指向当前栈, 然后就退出当前字节码过程. 所以yield就是记住当前执行到哪个栈(语句), 然后退出, 返回结果给调用者.

**所以yield理解为return, 只是保存了当前执行到了哪一个字节码**


生成器中定义了send, close, throw这三个接口,

.. code-block:: c++

    PyTypeObject PyGen_Type = {
        PyVarObject_HEAD_INIT(&PyType_Type, 0)
        "generator",                                /* tp_name */
        gen_methods,                                /* tp_methods */
    }

    static PyMethodDef gen_methods[] = {
        {"send",(PyCFunction)_PyGen_Send, METH_O, send_doc},
        {"throw",(PyCFunction)gen_throw, METH_VARARGS, throw_doc},
        {"close",(PyCFunction)gen_close, METH_NOARGS, close_doc},
        {NULL, NULL}        /* Sentinel */
    };

而send中只是执行生成器中的字节码而已

.. code-block:: c++

    static PyObject *
    gen_send_ex(PyGenObject *gen, PyObject *arg, int exc, int closing)
    {
        // 省略了很多很多代码
        PyThreadState *tstate = PyThreadState_GET();
        // 生成器中的字节码
        PyFrameObject *f = gen->gi_frame;
        gen->gi_running = 1;
        // 执行这些字节码
        result = PyEval_EvalFrameEx(f, exc);
        gen->gi_running = 0;
    }

**所以生成器本意上是指每一步迭代只生成和返回一个元素的对象, 最后引发StopIteration来终止迭代. 我们也可以使用迭代器来实现这样的功能, 但是

生成器简化了创建迭代器的过程. 为了下一次迭代的时候能产生下一个元素, 所以生成器必须具备记住当前执行栈位置的能里, 这也是yield的作用.

生成器本质上只有一种运行方式, 就是调用generator.send, 迭代也是调用generator.send**


awaitable对象
===================

1. https://snarky.ca/how-the-heck-does-async-await-work-in-python-3-5/

2. https://www.python.org/dev/peps/pep-0492/

*Any yield from chain of calls ends with a yield. This is a fundamental mechanism of how Futures are implemented.
Since, internally, coroutines are a special kind of generators, every await is suspended by a yield somewhere down the chain of await calls
(please refer to PEP 3156 for a detailed explanation).* -参考2

**理解的关键在于上面一句中每一个await调用链都会最终被yield给暂停掉!**

这是可以理解的, 因为python中能暂停的语句只有yield, 而await语句和yield from一样, 只是一种代理模式而已, 可以理解为

.. code-block:: python

    await obj
    # 等同于
    yield from obj

    # 等同于
    for i in obj:
        yield i

**await和yield from是功能是一样的, 区分只是为了区别协程和生成器, 这样前者是应用在协程中而后者是应用在生成器中**

在参考1中提到

.. code-block:: python

    async def read_data(db):
        data = await db.fetch('SELECT ...')

*await, similarly to yield from, suspends execution of read_data coroutine until db.fetch awaitable completes and returns the result data.*

所以data就是保存db.fetch('SELECT ...')这个语句的结果, 而await就是等待db.fetch返回

而await后面只能跟awaitable对象, 可以是

1. A native coroutine object returned from a native coroutine function.

   协程对象

2. A generator-based coroutine object returned from a function decorated with types.coroutine().

   基于生成器的协程对象

3. An object with an __await__ method returning an iterator.

   一个对象, 其实现了__await__方法, 这个方法返回的是一个迭代器对象

选项3就是awaitable对象的定义了.


而为什么__await__方法返回的是一个迭代器对象?

1. 这个原因和为什么区分可迭代对象和迭代器一样, 都是为了能重复await一个awaitable对象

2. 因为await等同于yield from, 那么显然能yield from后面要跟着一个迭代器对象(生成器也是可迭代对象, 同时生成器自己是自己的迭代器对象)

来看看await的字节码

.. code-block:: python

    In [156]: class MyAwait:
         ...:
         ...:     def __init__(self):
         ...:         self.n = 1
         ...:         return
         ...:

    In [157]: async def test_my():
         ...:     x = MyAwait()
         ...:     result = await x
         ...:     return result
         ...:
         ...:

    In [158]: dis.dis(test_my)
      2           0 LOAD_GLOBAL              0 (MyAwait)
                  2 CALL_FUNCTION            0
                  4 STORE_FAST               0 (x)

      3           6 LOAD_FAST                0 (x)
                  8 GET_AWAITABLE
                 10 LOAD_CONST               0 (None)
                 12 YIELD_FROM
                 14 STORE_FAST               1 (result)

      4          16 LOAD_FAST                1 (result)
                 18 RETURN_VALUE


注意我们定义的MyAwait这个类并没有__await__方法, 这里只是看一下await的字节码. await的字节码为GET_AWAITABLE

.. code-block:: c++

    TARGET(GET_AWAITABLE) {
        PyObject *iterable = TOP();
        // 调用__await__然后检查其返回值
        PyObject *iter = _PyCoro_GetAwaitableIter(iterable);
        Py_DECREF(iterable);
    }

    // 
    PyObject *
    _PyCoro_GetAwaitableIter(PyObject *o)
    {
        unaryfunc getter = NULL;
        PyTypeObject *ot;
        // 这里显然如果是协程对象, 那么协程的__await__返回的就是自己
        if (PyCoro_CheckExact(o) || gen_is_coroutine(o)) {
            /* 'o' is a coroutine. */
            Py_INCREF(o);
            return o;
        }

        ot = Py_TYPE(o);
        if (ot->tp_as_async != NULL) {
            // 这里查询到__await__
            getter = ot->tp_as_async->am_await;
        }
        if (getter != NULL) {
            // 调用__await__!!!!!!!!!!!!!!!!!!
            PyObject *res = (*getter)(o);
            // 校验__await__的返回值是否是
            // 1. 协程对象
            // 2. 可迭代对象/迭代器对象
            if (res != NULL) {
                if (PyCoro_CheckExact(res) || gen_is_coroutine(res)) {
                    /* __await__ must return an *iterator*, not
                       a coroutine or another awaitable (see PEP 492) */
                    PyErr_SetString(PyExc_TypeError,
                                    "__await__() returned a coroutine");
                    Py_CLEAR(res);
                } else if (!PyIter_Check(res)) {
                    PyErr_Format(PyExc_TypeError,
                                 "__await__() returned non-iterator "
                                 "of type '%.100s'",
                                 Py_TYPE(res)->tp_name);
                    Py_CLEAR(res);
                }
            }
            return res;
        }
    }

所以await就是调用__await__以及校验__await__的返回值是否是迭代器对象, 然后接着的字节码是YIELD_FROM, 所有意味着我们要在返回的迭代器对象上进行迭代

下面是一个简单await对象的例子

.. code-block:: python

    class MyAwait:
        def __init__(self):
            self.n = 1
            return
        def send(self, v=None):
            if v is not None:
                print("start counting")
            if self.n != 5:
                old_n = self.n
                self.n += 1
                return "now counter ", old_n
            raise StopIteration
            return

        def __await__(self):
            return self

        def __next__(self):
            print("__next__")
            return self.send()

    async def test_my():
        m = MyAwait()
        await m
        return

我们定义了一个awaitable对象MyAwait, 然后定义了__await__方法, 返回的可迭代迭代器对象是自己, 所以要实现__next__方法, 而__iter__方法不是必须的

然后send方法是生成器必须的方法, 因为await一定要在async中使用, 而async中的yield from是调用send来运行awaitable对象的, 所以必须实现send方法

这里我们运行协程test_my

.. code-block:: python

    x = test_my()
    res = x.send(None)
    print(res)
    res = x.send(1)
    print(res)


结果为

__next__
('now counter ', 1)
start counting
('now counter ', 2)

当然await也可以和yield from 替换, 比如我们把await换成yield from, 也就是另一种写法, 只不过yield from针对的是迭代器

.. code-block:: python

    class MyNoSend:

        def __init__(self):
            self.n = 1
            return
        def __iter__(self):
            return self

        def __next__(self):
            print("__next__")
            if self.n != 5:
                old_n = self.n
                self.n += 1
                return "no send now counter ", old_n
            raise StopIteration
            return

    def test_no_send():
        m = MyNoSend()
        yield from m
        return

    y = test_no_send()
    res = next(y)
    print(res)
    res = next(y)
    print(res)

    '''
    结果为
    __next__
    ('no send now counter ', 1)
    __next__
    ('no send now counter ', 2)
    '''

**所以await后面肯定要接一个awaitable对象, 这个对象既是迭代器对象(实现了__next__), 同时也是一个协程/生成器对象(实现了send方法)**


Coroutine协程
====================

https://en.wikipedia.org/wiki/Coroutine

协程和线程最关键的区别就是线程是操作系统调用的, 而协程则是用户自己调用的, 或者说用户程序调用的, 这个程序称为EventLoop

所以协程切换的时候不会有像线程那样的系统开销(比如进入内核态等等), 而是完完全全只是一个程序切换到另外一个程序而已

在python中, async/await就定义了一个协程, 协程的运行都是通过send接口来实现的. python中的暂停只能是使用yield关键字

所以await链的最后也是yield, 而send也是从外层一直传递到yield层.

而如果调度两个协程, 这个是EventLoop的作用, 在标准库中有asyncio, 其他EventLoop有curio和trio, 本质上核心思路是一样的

比如在curio中

当一个协程进入等待状态, 那么就使用yield暂停, 然后把主动权交给EventLoop, EventLoop根据yield的返回值知道哪个协程需要等待什么样的操作

EventLoop就把协程和io给关联起来, 把这个io操作放到线程池中运行, 同时执行另外一个协程. 当某个io在线程池中完成了, 那么EventLoop得到

通知后把该完成的io对应的协程找到, 然后通过send把结果发送给协程, 这样协程就可以根据send的值又继续执行了


总结下来就是协程通过yield暂停, EventLoop得到io结果只会通过send重新启动协程




