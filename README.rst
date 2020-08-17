CPython 3.6
##################

主要是3.6版本

不要陷入繁琐的细节中...

解释OR编译
=====================

毫无疑问, python是一个解释性语言, 但是确实也存在"编译"阶段, python也算先"编译"后执行的

这里的编译不是传统的编译为机器码的编译, 而是把代码编译为字节码的过程, 你也可以理解为把代码解释为字节码, 然后执行字节码

加载.py文件, 在交互环境下执行python代码等等都会先把代码编译成字节码再执行的, 对于py文件, 会生成pyc文件

pyc文件将会加速加载速度, 但是没有优化运行效率的功能.


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


.. code-block:: c

    PyObject *
    PyEval_EvalFrameEx(PyFrameObject *f, int throwflag)
    {
        PyThreadState *tstate = PyThreadState_GET();
        // 这里CPython也是称执行字节码的程序为解释器interp(reter)
        return tstate->interp->eval_frame(f, throwflag);
    }


tstate->interp->eval_frame最终会读取字节码, 然后执行LOAD_NAME, ROT_TWO等字节码对应的过程(这个eval_frame我们可以指向我们自己的函数, 做定制用)

.. code-block:: c

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

引用
===========

python中变量和对象的关系是称为引用, 可以看成指向关系

比如a="a", 就是变量名a, a也是一个对象, 是一个str对象, 其指向一个"a"的str对象, 这个"a"对象就被a这个对象给引用了, 我们通过对象a获取其指向值的时候就得到"a"

指向关系存储在字典中, 每一个作用域都是一个dict, 这个dict就是查询指向关系, 也就是变量名和其值的表

所以在python中的函数传参是所谓的引用传递, 传入函数的是变量名指向的对象, 所以传入可变对象的时候会修改对象的值

.. code-block:: python

    In [51]: def test_pass(a):
        ...:     a.append("aaaaaaaaaaa")
        ...:     return
        ...:
    
    In [52]: x=["x"]
    
    In [53]: test_pass(x)
    
    In [54]: x
    Out[54]: ['x', 'aaaaaaaaaaa']

传入x就是传入x指向的对象, 该对象是一个list对象, 函数使用a指向传入的对象, 所以此时a和x指向同一个list对象, 函数中a修改了list对象

函数结束之后, python解除了a和list对象的引用关系, 而x依然指向list, 所以可以看到x指向的对象被修改了


多核并行
==============

CPython的线程是系统线程的一个包装, 调度上还是依赖于操作系统, 只是线程的时候需要获取全局锁, 也就是所谓的GIL, 所以多核下同时只能有一个线程正在运行

但是CPython中有很多操作是释放掉了GIL的, 比如网络请求, sleep等等, 带有这些操作的线程是可以和其他线程同时执行的

还可以把程序写成C代码然后手动释放GIL, 那么这样多核并行也是可以的, 但是要注意任何python代码都要在GIL下运行


compile
============

compile.rst

code object? frame object?

exec和eval有什么区别?

class
=============================

class_xxx.rst

type为什么既可以传入1个参数也可以传入3个参数?

type和类是什么关系?

为什么__new__的返回值既可以是类, 也可以是对象?

查询类/对象属性的时候和继承深度有关? 继承层级越深查询越慢?


GC
========

gc.rst

为什么调用sys.getrefcount得到的引用计数不对?

容器对象一定会被加入GC链表吗?

随着未GC的对象越来越多, 会不会导致GC的时候遍历的时间越来越长?


thread
====================

thread.rst

能不能手动终止线程?


