编译相关的小知识
##########################

1. https://late.am/post/2012/03/26/exploring-python-code-objects

2. https://stackoverflow.com/questions/2220699/whats-the-difference-between-eval-exec-and-compile

eval和exec
========================

eval只能执行单行表达式或者code object, 并且不能有赋值语句(毕竟表达式确实不能有赋值), 同时eval总是返回表达式的值


.. code-block:: python

    In [1]: p=10
    
    In [2]: eval("p+1")
    Out[2]: 11
    
    In [3]: eval("p+1\np-1")
    Traceback (most recent call last):
    SyntaxError: invalid syntax

    In [10]: eval("o=p+1")
    SyntaxError: invalid syntax

exec总是执行多行python语句(包括表达式), 并且exec总是返回None, 所以我们可以再exec中使用赋值记住表达式的值

.. code-block:: python

    In [7]: exec("a=p+1\nb=p+2")
    
    In [8]: a
    Out[8]: 11
    
    In [9]: b
    Out[9]: 12

exec和eval本质上执行的还是codeobject, 如果传入的不是codeobject, 比如是str/bytes, 那么最终都会去调用compile的

.. code-block:: python

    exec(compile(source, '<string>', 'exec'))
    
    eval(compile(source, '<string>', 'eval'))

compile
============

.. code-block:: python

    compile(source, filename, mode, flags=0, dont_inherit=False, optimize=-1)


filename是显示作用, 比如下面例子中, <for_loop>这个命令就显示在code_object的打印字符串中

mode除了eval和exec, 还有一个single, single限制只能使用单个python语句, 这个单个语句包含了for, if-elif-else, if-else, try-except-finally这些block

single中如果有多个block或者多个python语句, 那么报错

.. code-block:: python

    In [61]: x=compile("for i in range(10):\n    print(i)", "<for_loop>", "single")
    
    In [62]: x
    Out[62]: <code object <module> at 0x00000168010130C0, file "<for_loop>", line 1>
    
    In [63]: x=compile("a=1\nfor i in range(10):\n    print(i)", "<for_loop>", "single")
      File "<for_loop>", line 1
        a=1
           ^
    SyntaxError: multiple statements found while compiling a single statement


dis.dis
============

dis.dis是disassemble, 也就是"反编译"codeobject, 理解为把codeobject中的执行过程打印出来, 比如

.. code-block:: python

    In [11]: dis.dis("a+b")
      1           0 LOAD_NAME                0 (a)
                  2 LOAD_NAME                1 (b)
                  4 BINARY_ADD
                  6 RETURN_VALUE

在dis.dis中看到, dis总是去找传入对象的codeobject, 然后获取codeobject中的co_code, 也就是字节码, 然后解析这些字节码

.. code-block:: python

    def dis(x=None, *, file=None):
        """Disassemble classes, methods, functions, generators, or code.
    
        With no argument, disassemble the last traceback.
    
        """
        if x is None:
            distb(file=file)
            return
        if hasattr(x, '__func__'):  # Method
            x = x.__func__
        if hasattr(x, '__code__'):  # Function
            x = x.__code__
        if hasattr(x, 'gi_code'):  # Generator
            x = x.gi_code
        if hasattr(x, '__dict__'):  # Class or module
            items = sorted(x.__dict__.items())
            for name, x1 in items:
                if isinstance(x1, _have_code):
                    print("Disassembly of %s:" % name, file=file)
                    try:
                        dis(x1, file=file)
                    except TypeError as msg:
                        print("Sorry:", msg, file=file)
                    print(file=file)
        elif hasattr(x, 'co_code'): # Code object
            disassemble(x, file=file)
        elif isinstance(x, (bytes, bytearray)): # Raw bytecode
            _disassemble_bytes(x, file=file)
        elif isinstance(x, str):    # Source code
            _disassemble_str(x, file=file)
        else:
            raise TypeError("don't know how to disassemble %s objects" %
                            type(x).__name__)

注意的是如果传参是类, 那么获取类中__dict__中有codeobject的对象, 再进行dis. 这里一半都是类中定义的方法才有codeobject

而字节码是在opcode这个模块中能看到, opcode.opmap中有所有的字节码和名字

codeobject
================

引用参考1中对codeobject的描述

Code objects, then, are Python objects which represent some piece of bytecode, along with all that it needs to execute: a declaration of the expected argument types and counts, a list (not dictionary! more about which later) of locals, information about the source code from which the bytecode was generated (for debugging and printing stack traces), etc -- oh, and also (perhaps obviously), the bytecode itself, as a str (or, in Python3, bytes).

以及python如何执行codeobject

CPython implements a virtual machine that executes a stack-based bytecode. At runtime, executable things (functions, methods, modules, class bodies, lambdas, statements, expressions, etc) are all executed as bytecode by the Python virtual machine.


所以codeobject是一个保存了一个executable thing(functions, method等等)在执行的时候需要的一切信息, 包括参数个数, 使用了哪些常量等等这些信息的对象

执行所需要的信息都存储在codeobject.co_xxx的变量中. 执行codeobject就是拿出其中的bytecode, 和co_xxx这些信息就可以执行了.

dis.dis展示的是输入的语法解析出来的字节码, 所以当我们定义类和定义函数的时候, 类和函数内部的字节码是不会展示出来的, 因为我们传入的是定义类/函数的过程

.. code-block:: python

    In [2]: dis.dis("class A:\n    pass")
      1           0 LOAD_BUILD_CLASS
                  2 LOAD_CONST               0 (<code object A at 0x000002AC0F778F60, file "<dis>", line 1>)
                  4 LOAD_CONST               1 ('A')
                  6 MAKE_FUNCTION            0
                  8 LOAD_CONST               1 ('A')
                 10 CALL_FUNCTION            2
                 12 STORE_NAME               0 (A)
                 14 LOAD_CONST               2 (None)
                 16 RETURN_VALUE
    
    In [3]: dis.dis("def p():\n    return")
      1           0 LOAD_CONST               0 (<code object p at 0x000002AC0F778D20, file "<dis>", line 1>)
                  2 LOAD_CONST               1 ('p')
                  4 MAKE_FUNCTION            0
                  6 STORE_NAME               0 (p)
                  8 LOAD_CONST               2 (None)
                 10 RETURN_VALUE

上面的字节码都是创建类和创建函数的字节码, 而类和函数自己的字节码是没有展示出来而是直接就是一个codeobject了, 因为上面传入的字符串就是定义类/函数的过程而不是函数内部的操作过程

而dis.dis一个函数的时候, 展示的是函数内部的字节码, 比如下面才是展示函数内部的字节码

.. code-block:: python

    In [4]: def p(a, b):
         ...:     c = a + b
         ...:     return c
         ...:
    
    In [5]: dis.dis(p)
      2           0 LOAD_FAST                0 (a)
                  2 LOAD_FAST                1 (b)
                  4 BINARY_ADD
                  6 STORE_FAST               2 (c)
    
      3           8 LOAD_FAST                2 (c)
                 10 RETURN_VALUE


co_xxx
==========

codeobject.co_xxx的变量含义在https://docs.python.org/3/library/inspect.html, 但是要注意一个就是co_names

文档里面说co_names是tuple of names of local variables， 而https://github.com/python/cpython/pull/2743 这里表示co_names是tuple of names of global variables

.. code-block:: python

    In [1]: g=1
    
    In [2]: def p():
       ...:     global g
       ...:     g += 1
       ...:     a = 10
       ...:     res = g + a
       ...:     return res
       ...:
    
    In [3]: x=p.__code__
    
    In [4]: x.co_names
    Out[4]: ('g',)
    
    In [5]: p()
    Out[5]: 12
    
    In [6]: g
    Out[6]: 2
    
    In [7]: def q():
       ...:     a = 100
       ...:     print(g)
       ...:     return a
       ...:
    
    In [8]: a=q.__code__
    
    In [9]: a.co_names
    Out[9]: ('print', 'g')
    
    In [10]: a.co_varnames
    Out[10]: ('a',)
    
    In [11]: x.co_varnames
    Out[11]: ('a', 'res')

例子中显示指明global变量和隐式使用global变量都出现在了co_names中, 但是用户serhiy-storchaka还表示co_names contains not only names of global variables.

用户serhiy-storchaka在pytho bug页面https://bugs.python.org/issue30951提到了

co_names contains not only names of global variables. It contains also local names in the class scope, attribute names, names of imported modules, etc.

.. code-block:: python

    In [1]: import dis
    
    In [2]: class T:
       ...:     data = None
       ...:     def __init__(self, d):
       ...:         self.not_class_data = None
       ...:         self.data = d
       ...:         return
       ...:     def t_method(self):
       ...:         return self.data
       ...:     def use_global(self):
       ...:         print(g)
       ...:         return
       ...:
    
    In [3]: dis.dis(T.t_method)
      8           0 LOAD_FAST                0 (self)
                  2 LOAD_ATTR                0 (data)
                  4 RETURN_VALUE
    
    In [5]: T.t_method.__code__.co_names
    Out[5]: ('data',)
    
    In [6]: T.use_global.__code__.co_names
    Out[6]: ('print', 'g')
    
    In [7]: class T:
       ...:     data = None
       ...:     def __init__(self, d):
       ...:         self.not_class_data = None
       ...:         self.data = d
       ...:         return
       ...:     def t_method(self):
       ...:         return self.data
       ...:     def use_global(self):
       ...:         print(g)
       ...:         return
       ...:     def use_not_class_data(self):
       ...:         print(self.not_class_data)
       ...:         return
       ...:
       ...:
    
    In [8]: T.t_method.__code__.co_names
    Out[8]: ('data',)
    
    In [9]: T.use_global.__code__.co_names
    Out[9]: ('print', 'g')
    
    In [10]: T.use_not_class_data.__code__.co_names
    Out[10]: ('print', 'not_class_data')

由于dis只会去dis codeobject, 而class是没有__code__的, 所以我们可以从类中定义的方法的codeobject中看到co_names包含全局变量, 方法中使用到的类变量(也就是self.xxx)

**我觉得可以理解为所有没有在局部定义的变量名字**

.. code-block:: c

        names = co->co_names;
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
            }
            else {
                v = PyObject_GetItem(locals, name);
                if (v == NULL) {
                    if (!PyErr_ExceptionMatches(PyExc_KeyError))
                        goto error;
                    PyErr_Clear();
                }
            }
            if (v == NULL) {
                v = PyDict_GetItem(f->f_globals, name);
                Py_XINCREF(v);
                if (v == NULL) {
                    if (PyDict_CheckExact(f->f_builtins)) {
                        v = PyDict_GetItem(f->f_builtins, name);
                        if (v == NULL) {
                            format_exc_check_arg(
                                        PyExc_NameError,
                                        NAME_ERROR_MSG, name);
                            goto error;
                        }
                        Py_INCREF(v);
                    }
                    else {
                        v = PyObject_GetItem(f->f_builtins, name);
                        if (v == NULL) {
                            if (PyErr_ExceptionMatches(PyExc_KeyError))
                                format_exc_check_arg(
                                            PyExc_NameError,
                                            NAME_ERROR_MSG, name);
                            goto error;
                        }
                    }
                }
            }
            PUSH(v);
            DISPATCH();
        }


在LOAD_NAME中, name是从names, 而names则是co_names的代称. 而我们从names中得到变量名字之后, 根据LEGB原则去一步步获取名字指向的值

所以co_names存储了所有来自各个作用域的变量名字, 虽然流程上看co_names也可能存储局部变量(不然LOAD_NAME就不会去f_locals找了), 但是根据输出, 就是除了局部变量之外用

到的所有可能的变量名都存储再co_names上了

frameobject
=================

既然我们执行的是codeobject, 为什么还有一个frameobject呢?

codeobject保存的是要执行的字节码机器信息, 但是我们执行的时候, 还需要其他信息, 比如我们需要知道当前执行到了哪个字节码, 而这些执行中的跟踪信息就是存储在frameobject上了

并且解释器执行的时候也是执行frame而不是执行codeobject的, 下面省略了其他frameobject的属性

.. code-block:: c

    typedef struct _frame {
        PyObject_VAR_HEAD
        struct _frame *f_back;      /* previous frame, or NULL 前一个frameobject*/
        PyCodeObject *f_code;       /* code segment 我们要执行的codeobject*/
        PyObject *f_builtins;       /* builtin symbol table (PyDictObject) */
        PyObject *f_globals;        /* global symbol table (PyDictObject) */
        PyObject *f_locals;         /* local symbol table (any mapping) */

        // builtin, globals, locals都是指向codeobject上

        // 最后执行的一个字节码是什么
        int f_lasti;                /* Last instruction if called */

        // 当前执行到了哪一行
        int f_lineno;               /* Current line number */

    } PyFrameObject;


而在真正执行的时候, 大概流程就是

.. code-block:: c

    const _Py_CODEUNIT *next_instr; /*下一个指令, 也就是下一个字节码 */
    int opcode;        /* Current opcode 字节码*/
    int oparg;         /* Current opcode argument, if any 字节码的参数*/
    enum why_code why; /* Reason for block stack unwind 是否出错*/
    const _Py_CODEUNIT *first_instr; /* 第一个字节码 */


    // 一开始first_instr指向字节码的首部, co_code就是一个bytes对象, 这里PyBytes_AS_STRING则返回bytes对象中存储数据的数组
    first_instr = (_Py_CODEUNIT *) PyBytes_AS_STRING(co->co_code);
  

    // 如果一开始最后执行的字节码为-1, 那么显然f_lasti / sizeof这个就是为0, next_instr就是第1个字节码
    // 如果lasti不为-1, 说明我们上一次执行到位置N之后停止了, 可能是gil被抢走了, 可能是我们是生成器, 所以我们的首部就不是第一个字节码而是第N个字节码
    // 那么显然我们必须先移动到lasti, 然后再加1, 这样才是正确的位置
    // 
    next_instr = first_instr;
    if (f->f_lasti >= 0) {
        // 进入这里说明我们上次已经执行到了第N个位置的字节码了
        assert(f->f_lasti % sizeof(_Py_CODEUNIT) == 0);
        next_instr += f->f_lasti / sizeof(_Py_CODEUNIT) + 1;
    }



    // 然后进入获取当前要执行字节码, 然后把next_instr移动一位
    fast_next_opcode:
        f->f_lasti = INSTR_OFFSET();

        /* MEXTOPARG就是这样
        Py_CODEUNIT word = *next_instr; 
        opcode = _Py_OPCODE(word); 
        oparg = _Py_OPARG(word); 
        next_instr++;*/
        NEXTOPARG();



   // 拿到opcdeo和oparg, 进入执行阶段

   switch (opcode) {

       case XXX:
           // do something

           // FAST_DISPATCH就是回到fast_next_opcode, 重新获取字节码, 然后走一遍switch
           FAST_DISPATCH()

   }



