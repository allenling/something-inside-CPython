CPython 3.6
##################

主要是3.8版本

解释OR编译
=====================

python是一个解释性语言, 但是确实也存在"编译"阶段, python也算先"编译"后执行的

这里的编译不是编译为机器码的编译, 而是把代码编译为字节码的过程, 也就是解析为C代码, 然后再执行C代码

而python语句和C代码之间需要有映射, 那么这些映射就是字节码, 比如访问变量x则是对应LOAD_NAME这个字节码, 这个字节码在C代码中有一块代码块去执行


解释器OR虚拟机
==================

python执行字节码的程序有时候被称为虚拟机, 有时候被称为解释器, 称为虚拟机的话并没有像java VM那么强大, 所以更多的被称为解释器(CPython的代码中也是称为解释器), 一个基于栈的解释器

其工作流程也很简单, 获取当前字节码, 调用当前字节码的处理逻辑, 执行完之后获取下一个字节码继续执行. 而基于栈是说其数据是放在栈上的, 字节码操作数据入栈出栈

比如经典的交换元素的操作

.. code-block:: python

    In [24]: dis.dis("a,b=b,a")
      1           0 LOAD_NAME                0 (b)
                  2 LOAD_NAME                1 (a)
                  4 ROT_TWO
                  6 STORE_NAME               1 (a)
                  8 STORE_NAME               0 (b)
                 10 LOAD_CONST               0 (None)
                 12 RETURN_VALUE

其中会进入到ROT_TWO这个字节码中, 在C实现中我们首先找到当前解释器, 然后调用解释器去执行当前帧(frame, codeobject这些先不要纠结)


.. code-block:: c++

    PyObject *
    PyEval_EvalFrameEx(PyFrameObject *f, int throwflag)
    {
        PyThreadState *tstate = PyThreadState_GET();
        // 这里CPython也是称执行字节码的程序为解释器interp(reter)
        return tstate->interp->eval_frame(f, throwflag);
    }


tstate->interp->eval_frame最终会读取字节码, 然后执行LOAD_NAME, ROT_TWO等字节码对应的过程(这个eval_frame我们可以指向我们自己的函数, 做定制用)

.. code-block:: c++

    switch (opcode) {
        TARGET(LOAD_NAME) {
            PyObject *name = GETITEM(names, oparg);
            PyObject *locals = f->f_locals;
            PyObject *v;
            if (locals == NULL) {
                PyErr_Format(PyExc_SystemError,
                             "no locals when loading %R", name);
                goto error;
            }
            if (PyDict_CheckExact(locals)) {
                v = PyDict_GetItem(locals, name);
                Py_XINCREF(v);
            }else{
              // 根据LEGB原则等获取变量指向的值
            }
            PUSH(v);
            DISPATCH();
        }

        TARGET(ROT_TWO) {
            PyObject *top = TOP();
            PyObject *second = SECOND();
            SET_TOP(second);
            SET_SECOND(top);
            FAST_DISPATCH();
        }


        TARGET(STORE_NAME) {
            PyObject *name = GETITEM(names, oparg);
            PyObject *v = POP();
            PyObject *ns = f->f_locals;
            int err;
            if (ns == NULL) {
                PyErr_Format(PyExc_SystemError,
                             "no locals found when storing %R", name);
                Py_DECREF(v);
                goto error;
            }
            if (PyDict_CheckExact(ns))
                err = PyDict_SetItem(ns, name, v);
            else
                err = PyObject_SetItem(ns, name, v);
            Py_DECREF(v);
            if (err != 0)
                goto error;
            DISPATCH();
        }
    }

1. 首先, LOAD_NAME是先通过GETITEM从names这个存储我们执行需要寻找的变量名的数组中, 简单的理解为name就是变量名数组, 获取第oparg个位置中的名字

   在dis.dis中我们看到LOAD_NAME(0)就是得到变量名字b, LOAD_NAME(1)就是得到变量名字a. 

2. 然后先从f->f_locals, 可以简单理解为局部变量的dict, 然后先从该表中获取名字为a和b的名字指向的值v, 如果没有

   则走else部分是根据LEGB原则去寻找a和b这两个变量在哪里定义了, 也就是a和b指向了哪个对象

   找到a和b指向的值之后, 调用PUSH, 把值入栈. 我们这边把a指向的值标志为va, 而b指向的值为vb. 根据先加载b后加载a这样的顺序, 此时栈的顺序是va在栈顶, vb在栈底


3. ROT_TWO的作用则是调换栈顶的2个元素, 也就是拿到top和second, 然后把second设置为栈顶, top设置为第二个元素, 所以此时顺序则是vb在栈顶而va在第二个位置


4. 最后调用STORE_NAME来调整a和b指向的值, 首先STORE_NAME(1)则是拿到变量名a, 然后弹出栈顶的值, 也就是vb, 然后进行赋值, 此时a就指向vb, 同理, b就指向了ba, 交换完成

.. code-block:: python

    '''
    局部变量名数组 [b,   a]
    局部变量数组   [vb, va]
    
    1. LOAD_NAME(0), 也就是取出vb, 入栈, 此时

       --栈顶  vb
       
    2. LOAD_NAME(1), 取出va, 入栈

       --栈顶 va
              vb
              
    3. ROT_TWO, 交换两者

       --栈顶 va          --栈顶 vb
              vb   ====>        va
            
     4. STORE_NAME, 重新赋值, STORE_NAME(1), 1表示a在局部变量名数组的下标, 所以这里表示把变量名a指向栈顶元素, 所以先把栈顶元素出栈
     
        --栈顶 vb                   --栈顶 va
               va    ===弹出vb====>             =>  a->vb
                      
     5. 同理STORE_NAME(0)表示b指向栈顶元素
 
        --栈顶 va                  --栈顶 
                    ===弹出va==>             =>  b->va
                    
    '''

python中的引用
==================

一句话总结就是: 名字映射(字典)和指针!

先来看一个简单的赋值语句x=1

.. code-block:: python

    In [2]: dis.dis("x=1")
      1           0 LOAD_CONST               0 (1)
                  2 STORE_NAME               0 (x)
                  4 LOAD_CONST               1 (None)
                  6 RETURN_VALUE

LOAD_CONST是加载第0个常量, 也就是1(当然这个1也是python对象, 整形对象), 然后STORE_NAME存储起来

STORE_NAME则是加载第0个变量的名字, 也就是x, 然后将x赋值为1

.. code-block:: C++

        case TARGET(STORE_NAME): {
            // 这里的oparg就是0, names存储得是用到变量名字
            PyObject *name = GETITEM(names, oparg);
            PyObject *v = POP();
            // 这里f->f_locals就是局部变量得存储对象
            PyObject *ns = f->f_locals;
            int err;
            if (ns == NULL) {
                _PyErr_Format(tstate, PyExc_SystemError,
                              "no locals found when storing %R", name);
                Py_DECREF(v);
                goto error;
            }
            if (PyDict_CheckExact(ns))
                // 我们会走到这里, 也就是ns是一个Dict!!
                err = PyDict_SetItem(ns, name, v);
            else
                err = PyObject_SetItem(ns, name, v);
            Py_DECREF(v);
            if (err != 0)
                goto error;
            DISPATCH();
        }

如果我们打印x的值, 那么会走到LOAD_NAME这个字节码

.. code-block:: C++

        case TARGET(LOAD_NAME): {
            // 从变量名字列表中取出第一个, 也就是x
            PyObject *name = GETITEM(names, oparg);
            // 先从局部变量中查找
            PyObject *locals = f->f_locals;
            PyObject *v;
            if (locals == NULL) {
                _PyErr_Format(tstate, PyExc_SystemError,
                              "no locals when loading %R", name);
                goto error;
            }
            if (PyDict_CheckExact(locals)) {
                // 这里获取locals这个字典的x这个key的值!!!!!!!!!!
                v = PyDict_GetItemWithError(locals, name);
                if (v != NULL) {
                    Py_INCREF(v);
                }
                else if (_PyErr_Occurred(tstate)) {
                    goto error;
                }
            }
            // 省略了很多代码
        }

所以在python中, 变量名字和变量对象得值向关系其实是用dict来存储的!!!

如果我们调用函数my_func(x, y), 字节码则是

.. code-block:: python

    In [5]: dis.dis("my_func(x, y)")
      1           0 LOAD_NAME                0 (my_func)
                  2 LOAD_NAME                1 (x)
                  4 LOAD_NAME                2 (y)
                  6 CALL_FUNCTION            2
                  8 RETURN_VALUE

加载my_func, x, y这三个名称, 然后调用函数, 在LOAD_NAME字节码中, 我们看到返回值v是一个PyObject的指针

.. code-block:: C++

    PyObject *v;
    v = PyDict_GetItemWithError(locals, name);

所以我们总是将一个PyObject的指针传入函数, 那么这样调用这个对象的本身的函数时候, 就会改变这个对象, 比如

.. code-block:: python

    def my_func(x, y):
        x[0] = 1
        y["key"] = "value"
        return

    a = ["x"]
    b = {"key": "no value"}
    my_func(a, b)

a和b这两个分别是list和dict类型, 将a和b传入函数my_func其实是把a和b值向的PyObject的指针传入函数

在my_func的赋值语句就改变了对象, 同时a和x, b和y都是通过dict指向同一个对象, 所以在my_func调用结束之后, 访问a和b发现他们的内容变了

如果是传入字符串呢? 字符串和数字这些对象本身不提供函数修改自身, 这些称为不可变对象, 所以传入不可变对象的时候, 也就不会被改变了



为什么列表解析比for语句快?
=================================

遍历上的对比
---------------------

首先来看遍历速度上的区别

.. code-block:: python

    import time
    import dis

    l = 1000 * 1000 * 100  # 100 million loops
    data = [0] * l

    def list_comp():
        start = time.time()
        for i in data:
            x = i
        print(time.time() - start)
        return

    def for_loop():
        start = time.time()
        for i in range(l):
            x = data[i]
        print(time.time() - start)
        return

    def main():
        list_comp()
        for_loop()
        return

    if __name__ == "__main__":
        main()

结果是

.. code-block:: python

    """
    4.239412784576416
    10.272042989730835
    """

同样只是简单的赋值操作, 遍历列表本身和通过下标获取值差距了1倍之多. 查看每个函数的字节码

.. code-block:: python

    "
    这个是遍历列表
         12 GET_ITER
    >>   14 FOR_ITER                 8 (to 24)
         16 STORE_FAST               1 (i)

         18 LOAD_FAST                1 (i)
         20 STORE_FAST               2 (x)
         22 JUMP_ABSOLUTE           14
    >>   24 POP_BLOCK
    这个是for循环
         16 GET_ITER
    >>   18 FOR_ITER                12 (to 32)
         20 STORE_FAST               1 (i)

         22 LOAD_GLOBAL              3 (data)
         24 LOAD_FAST                1 (i)
         26 BINARY_SUBSCR
         28 STORE_FAST               2 (x)
         30 JUMP_ABSOLUTE           18
    >>   32 POP_BLOCK
    """

两者的循环都是从GET_ITER开始, 获取对象的可迭代对象, 列表的可迭代对象是一个叫listiterobject

在通过下标获取值的函数中, GET_ITER其实是range对象的可迭代对象, 两者影响不是很大.

在遍历列表本身的函数中, FOR_ITER则是返回可迭代对象的下一个元素, listiterobject行为则是很直接, 找到列表对象, 返回下一个元素(当然可迭代对象保持了当前的下标)

.. code-block:: C++

    static PyObject *
    listiter_next(listiterobject *it)
    {
        PyListObject *seq;
        PyObject *item;

        assert(it != NULL);
        // 可迭代对象找到对象的列表对象
        seq = it->it_seq;
        if (seq == NULL)
            return NULL;
        assert(PyList_Check(seq));

        if (it->it_index < PyList_GET_SIZE(seq)) {
            // 拿到list上对应的下标元素
            item = PyList_GET_ITEM(seq, it->it_index);
            // 保持下一个元素的下标
            ++it->it_index;
            Py_INCREF(item);
            return item;
        }

        it->it_seq = NULL;
        Py_DECREF(seq);
        return NULL;
    }
    // 而PyList_GET_ITEM很简单, 找到元素列表(数组)， 返回第x个
    #define PyList_GET_ITEM(op, i) (((PyListObject *)(op))->ob_item[i])

而在通过下标获取元素的函数中, 找到列表对应的元素则是通过字节码BINARY_SUBSCR来实现的

该字节码通过函数PyObject_GetItem去查找是否有mp_subscript这样的实现, 如果没有, 又有很多检查.

列表对象是有mp_subscript函数的, 名字叫list_subscript, 该函数再调用一个叫list_item的函数, 该函数只是返回list对象中的元素列表中的对应元素

.. code-block:: C++

    //
    static PyObject *
    list_item(PyListObject *a, Py_ssize_t i)
    {
        // 省略代码
        Py_INCREF(a->ob_item[i]);
        return a->ob_item[i];
    }

所以看到, 遍历列表本身的时候, 只需要找一个函数, 调用就可以了, 而通过下标去得到列表的元素则需要有一个查找链, 其中还有各种检查条件

所以调用函数的个数也是影响因素之一, 因为调用函数也消耗时间, 即使很小但是次数多了就很显著了.


列表解析和for循环append
-------------------------------

再来看列表解析和for循环然后append的对比

https://stackoverflow.com/questions/22108488/are-list-comprehensions-and-functional-functions-faster-than-for-loops

.. code-block:: python

    import time

    iter_count = 10*1000*1000

    def run_for():
        x = []
        for i in range(iter_count):
            x.append(i)
        return

    def run_list_comp():
        x = [i for i in range(iter_count)]
        return

    def main():
        s1 = time.time()
        run_for()
        s2 = time.time()
        run_list_comp()
        s3 = time.time()
        print(s2 - s1, "\n", s3 - s2)
        return

    # 输出结果
    # 2.5292608737945557
    # 1.6167056560516357

列表解析更快一点. 先看看列表解析的循环的字节码(字节码在run_list_comp.__code__.co_consts[1])

.. code-block:: python

    """
          0 BUILD_LIST               0
          2 LOAD_FAST                0 (.0)
    >>    4 FOR_ITER                 8 (to 14)
          6 STORE_FAST               1 (i)
          8 LOAD_FAST                1 (i)
         10 LIST_APPEND              2
         12 JUMP_ABSOLUTE            4
    >>   14 RETURN_VALUE
    """

其中LOAD_FAST是获取run_list_comp.__code__.co_names中第0个下标的名字, 也就是range, 也就是加载range, 然后FOR_ITER则是获取range的可迭代对象

后面STORE_FAST则是把range的返回赋值到名字为i的变量中, 调用调用LIST_APPEND字节码, 也就是直接调用list.append函数了

看看for循环append的字节码(这次字节码直接dis.dis(run_for)就可以了)

.. code-block:: python

    """
         12 GET_ITER
    >>   14 FOR_ITER                14 (to 30)
         16 STORE_FAST               1 (i)

         18 LOAD_FAST                0 (x)
         20 LOAD_ATTR                2 (append)
         22 LOAD_FAST                1 (i)
         24 CALL_FUNCTION            1
         26 POP_TOP
         28 JUMP_ABSOLUTE           14
    >>   30 POP_BLOCK
    """

GET_ITER和FOR_ITER依然是和range有关, 关键是多了一个LOAD_ATTR去查找列表对象的append属性, 然后执行CALL_FUNCTION去执行该函数

所以这里就比起上面直接调用append函数多了两个字节码, 这两个字节码有调用很多额外函数, 有很多额外的检查

**所以总结起来就是python的函数查询/调用也会消耗一定的时间, 当次数多了之后时间消耗就比较明显**

多核并行?并发?
================

CPython的线程是系统线程的一个包装, 调度上还是依赖于操作系统, 只是线程的时候需要获取全局锁, 也就是所谓的GIL, 所以多核下同时只能有一个线程正在运行

但是CPython中有很多操作是释放掉了GIL的, 比如网络请求, sleep等等, 带有这些操作的线程是可以和其他线程同时执行的

还可以把程序写成C代码然后手动释放GIL, 那么这样多核并行也是可以的, 但是要注意任何python代码都要在GIL下运行


