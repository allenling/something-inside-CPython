Python Class
#######################

https://docs.python.org/3/tutorial/classes.html

https://docs.python.org/3/reference/datamodel.html

类即类型, 类型即类, 同时类也是对象

在CPython实现中, 每个类型/类都是PyTypeObject类型的结构体, 比如dict这个类型在CPython中是PyDict_Type的结构, 其定义为

.. code-block:: c

    PyTypeObject PyDict_Type = {
        // 省略了属性
    }

所以定义一个类/类型就是定义/创建一个PyTypeObject类型的结构体, 从名字上来看就知道PyTypeObject称为"TypeObject", 所以定义一个类型就是定义一个类型对象

同时定义类, 比如class A: pass, 也是定义/创建了一个PyTypeObject类型的结构体, 所以类也是类型对象, 也是一个类型



PyObject和PyTypeObject
=============================


CPython中所有的对象概念上都是PyObject, 之所以是概念上是因为这些对象的类型都"不算是"PyObject, 而是每个对象的前n个字节都是PyObject, 这样每个对象都是可以通过C类型转换转成通用的PyObject

比如dict这个类型在C中称为PyDict_Type, 其并不是PyObject类型, 而是PyTypeObject类型, 但是PyTypeObject和PyDict_Type的头部结构都是PyObject

这样所有的对象都能抽象为PyObject, C函数的传参就不需要指定是哪种python对象类型, 比如要指定PyDict_Type或者PyList_Type等等, 而是可以指定为通用的PyObject类型

在具体实现的时候可以通过类型转换转为指定的python对象类型, 比如x为dict类型, 那么x["a"]就是调用PyDict_GetItem函数

.. code-block:: c

    PyObject *
    PyDict_GetItem(PyObject *op, PyObject *key)
    {
        Py_hash_t hash;
        Py_ssize_t ix;
        // PyObject*转为PyDictObject*
        PyDictObject *mp = (PyDictObject *)op;
    
    }


我们看到该函数的传参应该是一个dict类型, 但是定义的是可以使用通用的PyObject类型, 然后在函数内部进行类型转换

既然有对象, 那么对象就应该由类型, 因为PyObject只是一个数据头而不是变量类型, 那么PyObject中自然包含了对象的类型

.. code-block:: c

    typedef struct _object {
        _PyObject_HEAD_EXTRA
        Py_ssize_t ob_refcnt;
        struct _typeobject *ob_type;
    } PyObject;

其中ob_type则存储了该对象的类型, 比如PyDict_Type.ob_type则是PyType_Type, 也就是dict的类型是type, PyType_Type.ob_type则是自己, 也就是PyType_Type, 也就是type的类型也是type

而PyTypeObject则是一个类型/类的定义模板, 其中定义了类型/类由哪些内置函数, 当我们要定义/新建一个类或者类型的时候, 就是定义/创建一个新的PyTypeObject类型的变量

.. code-block:: c

    typedef struct _typeobject {
        PyObject_VAR_HEAD
        const char *tp_name; /* For printing, in format "<module>.<name>" */ // 类的名字!!!

        hashfunc tp_hash;  // 类的__hash__
        ternaryfunc tp_call;  // 类的调用方法, 比如class A: pass; A()
        reprfunc tp_str;    // __str__方法, str(类)会调用
        reprfunc tp_repr;    // __repr__方法, repr(类)会调用
        // 还省略了很多很多....
    
    } PyTypeObject;


从PyTypeObject的声明中我们可以看到类内置可以有__repr__, __str__等等这些魔术方法

比如dict这个类型在定义的时候就定义了自己的类型的__repr__和__str__方法

.. code-block:: c

    PyTypeObject PyDict_Type = {
        PyVarObject_HEAD_INIT(&PyType_Type, 0)
        "dict", // 名字
        PyObject_HashNotImplemented, // tp_hash, dict不能hash
        0,                           // tp_call为0, 说明dict不能调用
        (reprfunc)dict_repr,         // tp_repre, repre(dict)
        // 还省略了很多很多....
    }

可以看到PyDict_Type是PyTypeObject类型的结构体, 当调用repr(dict)的时候, 会调用到dict_repr这个函数, 同时其类型是PyType_Type, 也就是type

type和PyType_Type
=====================

有一个比较特殊的类型就是type, type在CPython中也是一个类型, 虽然看起来是一个函数

.. code-block:: python

    In [1]: type(type)
    Out[1]: type
    
    In [2]: type(dict)
    Out[2]: type
    
    In [3]: dict.__class__
    Out[3]: type

    In [4]: class A:
       ...:     pass
       ...:
    
    In [5]: A.__class__
    Out[5]: type
    
    In [6]: type(A)
    Out[6]: type

我们看到dict的类型是type, 而type的类型也是type, 同时自定义类型A的类型也是type, 所以type是一切类型的基本类型, 同时type的类型也是自己

在CPython中, type定义为PyTypeObject类型, 其定义为

.. code-block:: c

    PyTypeObject PyType_Type = {
        // 这里设置了PyType_Type为ob_type
        PyVarObject_HEAD_INIT(&PyType_Type, 0)
        "type",                                     // tp_name
        (ternaryfunc)type_call,                     // tp_call
        type_new,                                   // tp_new
    };


比较重要的方法是tp_new和tp_call, 这两个函数涉及到类的创建

object和PyBaseObject_Type
============================

python中所有的类都继承于object, object也是继承自自己, object也是一种类型/类, 其定义为

.. code-block:: c

    PyTypeObject PyBaseObject_Type = {
        PyVarObject_HEAD_INIT(&PyType_Type, 0)
        "object",                                   // tp_name
        sizeof(PyObject),                           // tp_basicsize
        PyDoc_STR("object()\n--\n\nThe most base type"),  // tp_doc
    };


在python中可以查看类的mro来查看继承关系

.. code-block:: python

    In [1]: class A:
       ...:     pass
       ...:
    
    In [2]: A.mro()
    Out[2]: [__main__.A, object]
    
    In [3]: object.mro()
    Out[3]: [object]

而MRO中的object一般是在创建类的时候, 构建类的mro的时候默认添加到最后的

class
==============

我们一般使用class来自定义一个类, 字节码为

.. code-block:: python

    In [1]: dis.dis("class A:\n    pass")
      1           0 LOAD_BUILD_CLASS
                  2 LOAD_CONST               0 (<code object A at 0x00000178354DD1E0, file "<dis>", line 1>)
                  4 LOAD_CONST               1 ('A')
                  6 MAKE_FUNCTION            0
                  8 LOAD_CONST               1 ('A')
                 10 CALL_FUNCTION            2
                 12 STORE_NAME               0 (A)
                 14 LOAD_CONST               2 (None)
                 16 RETURN_VALUE


而LOAD_BUILD_CLASS则是加载内置的创建class的函数

.. code-block:: c

        TARGET(LOAD_BUILD_CLASS) {
            _Py_IDENTIFIER(__build_class__);

            PyObject *bc;
            if (PyDict_CheckExact(f->f_builtins)) {
                // 加载__build_class__函数
                bc = _PyDict_GetItemId(f->f_builtins, &PyId___build_class__);
                // 省略

            }
            else {
                // 省略
            }
        }

在CPython中, 内置的__build_class__函数是使用class来创建类的入口, CALL_FUNCTION这个字节码就是调用__build_class__这个C函数了

而字节码中的codeobject则是创建类中的方法和属性定义, 也就是

.. code-block:: python

    class P:
        data = 1
        def get_data(self):
            return self.data

其中codeobject就是定义变量名data, 然后使其指向1, 以及定义函数名get_data, 其指向函数对象的函数体就是执行返回self.data, 执行该字节码就是创建data和get_data这个两个属性

在__build_class__中的大概流程就是先找到meta(根据说明对象来创建类, meta可以是类也可以不是类, 这里只考虑meta为类的情况), 如果没有传入meta, 那么meta为bases的第一个, 否则meta为type

.. code-block:: c

        // 如果传入的metat class为空, 那么meta class就默认为type
        if (meta == NULL) {
            /* if there are no bases, use type: */
            if (PyTuple_GET_SIZE(bases) == 0) {
                meta = (PyObject *) (&PyType_Type);
            }
            /* else get the type of the first base 否则meta class为bases的第一个*/
            else {
                PyObject *base0 = PyTuple_GET_ITEM(bases, 0);
                meta = (PyObject *) (base0->ob_type);
            }
            Py_INCREF(meta);
            isclass = 1;  /* meta is really a class */
        }
 

然后meta和bases中选择合适的class作为最终的meta

.. code-block:: c

    if (isclass) {
        /* meta is really a class, so check for a more derived
           metaclass, or possible metaclass conflicts: */
        winner = (PyObject *)_PyType_CalculateMetaclass((PyTypeObject *)meta,
                                                        bases);
        if (winner == NULL) {
            Py_DECREF(meta);
            Py_XDECREF(mkw);
            Py_DECREF(bases);
            return NULL;
        }
        if (winner != meta) {
            Py_DECREF(meta);
            meta = winner;
            Py_INCREF(meta);
        }
    }

然后调用meata的__prepare__, 根据__prepare__的返回生成ns. ns是保存类的属性/方法字典, type.__prepare__为{}, 所以ns一般也是{}

.. code-block:: c

    prep = _PyObject_GetAttrId(meta, &PyId___prepare__);

然后创建类的属性和方法, 保存到ns中, 然后将name, bases和ns传入meta, 一般是meta为type, 调用meata来最终创建类

.. code-block:: c

    // 执行创建类属性和方法的字节码, 保存到ns中
    cell = PyEval_EvalCodeEx(PyFunction_GET_CODE(func), PyFunction_GET_GLOBALS(func), ns,
                             NULL, 0, NULL, 0, NULL, 0, NULL,
                             PyFunction_GET_CLOSURE(func));
    if (cell != NULL) {
        PyObject *margs[3] = {name, bases, ns};
        // 调用meta这个可调用对象去创建类, meta一般为type
        cls = _PyObject_FastCallDict(meta, margs, 3, mkw);
        // 省略
    }

**在_PyObject_FastCallDict这个语句中会调用meta这个可调用对象去创建类, 在python层就是__call__方法, 在C层面就是tp_call方法**

由于meta一般是type, _PyObject_FastCallDict(meta, margs, 3, mkw)就相当于调用type(name, bases, ns), 所以class语句最终还是调用了type函数

type
==========

type既可以创建类, 也可以输出对象的类型

.. code-block:: python

    In [1]: type("a")
    Out[1]: str
    
    In [2]: A=type("A", (), {})
    
    In [3]: A
    Out[3]: __main__.A

当你向type传入1个参数的时候, 其返回对象的类型, 如果你传入3个参数的时候, 创建一个新类型

但是CPython是c实现的, 显然不能由函数重载, 但是type是python定义的函数, 自然在调用的时候可以去判断参数个数了, 根据参数个数去调用不同的过程

调用type的时候, 就调用到 type_call, 在type_call中

.. code-block:: c

    static PyObject *
    type_call(PyTypeObject *type, PyObject *args, PyObject *kwds)
    {
        PyObject *obj;
    
        if (type->tp_new == NULL) {
            PyErr_Format(PyExc_TypeError,
                         "cannot create '%.100s' instances",
                         type->tp_name);
            return NULL;
        }
    
    
        obj = type->tp_new(type, args, kwds);
    
    }

这个函数的第一个参数就是PyType_Type, 也就是type这个类/类型/对象, 最终会调用到tp_new, 在tp_new中将会判断参数个数

.. code-block:: c

    static PyObject *
    type_new(PyTypeObject *metatype, PyObject *args, PyObject *kwds){
    
        if (metatype == &PyType_Type) {
            const Py_ssize_t nargs = PyTuple_GET_SIZE(args);
            const Py_ssize_t nkwds = kwds == NULL ? 0 : PyDict_Size(kwds);
    
            if (nargs == 1 && nkwds == 0) {
                PyObject *x = PyTuple_GET_ITEM(args, 0);
                Py_INCREF(Py_TYPE(x));
                return (PyObject *) Py_TYPE(x);
            }
    
            /* SF bug 475327 -- if that didn't trigger, we need 3
               arguments. but PyArg_ParseTupleAndKeywords below may give
               a msg saying type() needs exactly 3. */
            // 参数要么是1个要么是3个!!!!!!
            if (nargs != 3) {
                PyErr_SetString(PyExc_TypeError,
                                "type() takes 1 or 3 arguments");
                return NULL;
            }
        }
        // 后续是创建类的过程
    }

我们看到type的tp_new将会判断如果传入的metatype是type的话, 那么将会根据入参个数nargs来判断走那个逻辑

如果nargs ==1并且没有位置参数的话, 说明只有一个顺序参数, 那么返回传入参数的类型, 也就是Py_TYPE(x), 否则将会创建一个类


tp_new
============

tp_new中根据传入的name, bases和ns来创建一个类

首先解析参数

.. code-block:: c

    /* Check arguments: (name, bases, dict) */
    if (!PyArg_ParseTuple(args, "UO!O!:type.__new__", &name, &PyTuple_Type,
                          &bases, &PyDict_Type, &orig_dict))
        return NULL;

这里name被解析为第一个参数, 而传入的ns被解析为orig_dict, 接着再次计算meta 如果winner不等于metatype, 那么调用winner的tp_new

.. code-block:: c

    /* Determine the proper metatype to deal with this: */
    winner = _PyType_CalculateMetaclass(metatype, bases);
    if (winner == NULL) {
        return NULL;
    }

    if (winner != metatype) {
        if (winner->tp_new != type_new) /* Pass it to the winner */
            return winner->tp_new(winner, args, kwds);
        metatype = winner;
    }


然后计算bases, 如果传入的bases为空, 那么默认加入object作为最基础的继承类, 其中PyBaseObject_Type就是内建的object

.. code-block:: c

    /* Adjust for empty tuple bases */
    nbases = PyTuple_GET_SIZE(bases);
    if (nbases == 0) {
        // 没有继承类, 那么至少继承自object!!!!!!!!!!!!!!!!!!!!
        // 这里构建一个大小为1的tuple, 其唯一元素就是object
        bases = PyTuple_Pack(1, &PyBaseObject_Type);
        if (bases == NULL)
            goto error;
        nbases = 1;
    }
    else
        Py_INCREF(bases);

    /* Calculate best base, and check that all bases are type objects */
    base = best_base(bases);
    if (base == NULL) {
        goto error;
    }


拷贝一份ns/orig_dict到dict中

.. code-block:: c

    dict = PyDict_Copy(orig_dict);

这个dict是就是存储类属性的字典, 比如

.. code-block:: python

    type("A", (), {"data": 1})
    
    class A:
        data=1

上面两种方式都使得存储类属性的字典初始化为{"data": 1}, 相当于为类A预定义了属性data=1

调用type.tp_alloc, 为一个新的PyTypeObject分配内存, 其大小和类定义中的__slots__属性相关, 分配好内存之后保存到变量type, 然后设置type的名字为class定义的名字, 然后把ns/orig_dict/dict设置到type->tp_dict中

.. code-block:: c

    type = (PyTypeObject *)metatype->tp_alloc(metatype, nslots);

    type->tp_name = PyUnicode_AsUTF8AndSize(name, &name_size); // 设置类名字
    type->tp_dict = dict;


tp_dict则是class.__dict__, 存储的类本身定义的属性和方法, 而对于实例, instance.__dict__存储的是属于实例自己的属性和方法

.. code-block:: python

    In [9]: class A:
       ...:     name = "A"
       ...:
       ...:
    
    In [10]: a=A()
    
    In [11]: a.__dict__
    Out[11]: {}
    
    In [12]: A.__dict__
    Out[12]:
    mappingproxy({'__dict__': <attribute '__dict__' of 'A' objects>,
                  '__doc__': None,
                  '__module__': '__main__',
                  '__weakref__': <attribute '__weakref__' of 'A' objects>,
                  'name': 'A'})
    
    In [13]: a.data="a"
    
    In [14]: a.__dict__
    Out[14]: {'data': 'a'}

可以看到A.__dict__和a.__dict__的区别.

之后调用PyType_Ready来赋值所有的tp_xxx属性, 比如其中会调用mro_internal函数去设置类的mro属性(属性查找顺序)

.. code-block:: c

    int
    PyType_Ready(PyTypeObject *type){
        /* Calculate method resolution order */
        if (mro_internal(type, NULL) < 0)
            goto error;
    }

同时我们看到一般类的MRO最后一个将会是object

设置好内置的tp_xxx属性之后, 就返回type给调用者. 这样, 一个类就这么"简单地"创建出来了.

**同时我们看到创建类的时候会使用一个meta class来创建, 这个meta class要么是bases中的某一个(经过计算), 要么是type, 调用这个meta可调用对象(也就是t该对象的p_call方法)去去创建类**

**传入的bases会组成mro, mro是在查找属性/方法的时候一级一级查找的顺序, 所以多重继承的查找也是依赖于MRO顺序**



