Metaprogramming and Magic Method
#########################################

1. https://python-3-patterns-idioms-test.readthedocs.io/en/latest/Metaprogramming.html
    
2. https://www.geeksforgeeks.org/metaprogramming-metaclasses-python/

3. https://stackabuse.com/python-metaclasses-and-metaprogramming/

4. https://docs.python.org/3.6/reference/datamodel.html

5. https://stackoverflow.com/questions/6760685/creating-a-singleton-in-python 创建一个单例对象metaclass几乎是最合适的方法

meta类就是创建类的类, meta programming就是自定义类的创建过程

.. code-block:: python

    # 参考2的例子
    class MultiBases(type): 
        def __new__(cls, clsname, bases, clsdict): 
            if len(bases)>1: 
                raise TypeError("Inherited multiple base classes!!!") 
              
            return super().__new__(cls, clsname, bases, clsdict) 
      
    class Base(metaclass=MultiBases): 
        pass


以及

.. code-block:: python

    # 参考3
    class HelloMeta(type):
        def hello(cls):
            print("greetings from %s, a HelloMeta type class" % (type(cls())))
    
        # Call the metaclass
        def __call__(self, *args, **kwargs):
            cls = type.__call__(self, *args)
    
            setattr(cls, "hello", self.hello)
    
            return cls

    class TryHello(object, metaclass=HelloMeta):
        def greet(self):
            self.hello()


在创建类的过程中我们看到创建的时候会得到(手动传入metaclass, 或者从bases中选第一个, 或者默认是type)一个meta, 然后调用这个meta去创建类

.. code-block:: c

    PyObject *margs[3] = {name, bases, ns};
    cls = _PyObject_FastCallDict(meta, margs, 3, mkw);

同时调用type就是调用type中的tp_call->tp_new, 而tp_call和tp_new分别对应__call__和__new__, 似乎我们只要定义一个类, 其__call__或者__new__返回一个

类就可以了而不用继承于type


.. code-block:: python

    class A:
        def __call__(*args, **kwargs):
            return type(*args, *kwargs)

    class AA(metaclass=A):
        pass

但是这样是错误的, 因为自定义类中的__call__的第一个参数是self, 意味着A.__call__是一个实例方法而不是类方法, 所以当调用A去创建AA的时候

并不会调用到A.__call__, 但是代码中显然还是调用到tp_call, 这个tp_call依然是type_call, 所以说明自定义类的tp_call依然是type.tp_call

能不能只定义__new__? 这是可以的, 因为type.tp_call会调用到type.tp_new, 而tp_new(__new__)是一个类方法, 我们可以重载的

.. code-block:: python

    In [138]: class A(type):
         ...:     def __new__(cls, *args, **kwargs):
         ...:         print("A.__new__", args, kwargs)
         ...:         return super().__new__(cls, *args, **kwargs)
         ...:
         ...:
    
    In [139]: class AA(metaclass=A):
         ...:     pass
         ...:
    A.__new__ ('AA', (), {'__module__': '__main__', '__qualname__': 'AA'}) {}

    # 上面和下面是等价的

    In [142]: class A:
         ...:     def __new__(cls, *args, **kwargs):
         ...:         print("A.__new__", args, kwargs)
         ...:         return type(*args, **kwargs)
         ...:
         ...:
    
    In [143]: class AA(metaclass=A):
         ...:     pass
         ...:
    A.__new__ ('AA', (), {'__module__': '__main__', '__qualname__': 'AA'}) {}

上面两个例子是等价的, 区别是是否显示继承于type. 是否显示继承自影响到super调用是否正确, 如果A继承于type, 显然super就是type了, super.__new__则是type的tp_new了, 自然是正确的

而如果不继承于type同时还使用super的话, 那么由于所有的对象, 包括类, 都是继承自object(通过mro可知), 那么显然super(A, cls)得到的就是object, 不继承type的例子中直接调用super.__new__就是错了

.. code-block:: python

    In [186]: class A:
         ...:     def __new__(cls, *args, **kwargs):
         ...:         print("A.__new__", args, kwargs)
         ...:         return super(A, cls).__new__(*args, **kwargs)
         ...:
         ...:
         ...:
    
    In [187]:
    
    In [187]: class AA(metaclass=A):
         ...:     pass
         ...:
    A.__new__ ('AA', (), {'__module__': '__main__', '__qualname__': 'AA'}) {}
    ---------------------------------------------------------------------------
    TypeError                                 Traceback (most recent call last)
    <ipython-input-187-734a7cbc9257> in <module>()
    ----> 1 class AA(metaclass=A):
          2     pass
    
    <ipython-input-186-af6cdee87de8> in __new__(cls, *args, **kwargs)
          2     def __new__(cls, *args, **kwargs):
          3         print("A.__new__", args, kwargs)
    ----> 4         return super(A, cls).__new__(*args, **kwargs)
          5
          6
    
    TypeError: object.__new__(X): X is not a type object (str)


这个错误是object.tp_new中跑出来的, 同时object.tp_new还会调用到object.__new__

.. code-block:: c

    static PyObject *
    tp_new_wrapper(PyObject *self, PyObject *args, PyObject *kwds)
    {
        arg0 = PyTuple_GET_ITEM(args, 0);
        if (!PyType_Check(arg0)) {
            PyErr_Format(PyExc_TypeError,
                         "%s.__new__(X): X is not a type object (%s)",
                         type->tp_name,
                         Py_TYPE(arg0)->tp_name);
            return NULL;
    
         // 调用真正的object.__new__!!!!!!!!!!!!!!
        res = type->tp_new(subtype, args, kwds);
        Py_DECREF(args);
        return res;
    }


object.__new__的第一个参数是类型, 是生成实例用的

.. code-block:: python

    In [194]: class Q:
         ...:     pass
         ...:
    
    In [195]: x=object.__new__(Q)
    
    In [196]: x
    Out[196]: <__main__.Q at 0x2a1bcf39550>
    
    In [197]: q=Q()
    
    In [198]: q
    Out[198]: <__main__.Q at 0x2a1bcf39390>

我们看到x就是Q的一个实例, 所以如果不继承自type的话, __new__直接用type就好了, 因为此时super是object而不是type, 如果继承自type的话, 直接super是安全的

但是如果重载了__new__的话会干扰到对象的生成

.. code-block:: python

    In [144]: a=A()
    A.__new__ () {}
    ---------------------------------------------------------------------------
    TypeError                                 Traceback (most recent call last)
    <ipython-input-156-03f587ed9da6> in <module>()
    ----> 1 a=A()
    
    <ipython-input-147-da6251ce7bfa> in __new__(cls, *args, **kwargs)
          2     def __new__(cls, *args, **kwargs):
          3         print("A.__new__", args, kwargs)
    ----> 4         return type(*args, **kwargs)
          5
          6
    
    TypeError: type() takes 1 or 3 arguments


**我们看到创建实例的时候也会调用到__new__, 所以__new__可以返回类也可以返回对象**

**meta programming的关键在于__new__以及__new__的返回**

__new__和__init__
==========================


对应自定义类, __new__是类方法, 其在创建对象的是被调用, 而__init__则是对象被创建完成之后调用的初始化过程, 一个是创建一个是初始化

.. code-block:: python

    In [154]: class P:
         ...:     def __new__(cls, *args, **kwargs):
         ...:         print("P.__new__", args, kwargs)
         ...:         return super().__new__(cls,*args, **kwargs)
         ...:     def __init__(self):
         ...:         print("P.__init__")
         ...:         return
         ...:
    
    In [155]: p=P()
    P.__new__ () {}
    P.__init__

    # 参数会从__new__传入到__init__

    In [157]: p=P("a")
    P.__new__ ('a',) {}
    ---------------------------------------------------------------------------
    TypeError                                 Traceback (most recent call last)
    <ipython-input-157-bedb31e540f4> in <module>()
    ----> 1 p=P("a")
    
    <ipython-input-154-1347c97fe92b> in __new__(cls, *args, **kwargs)
          2     def __new__(cls, *args, **kwargs):
          3         print("P.__new__", args, kwargs)
    ----> 4         return super().__new__(cls,*args, **kwargs)
          5     def __init__(self):
          6         print("P.__init__")
    
    TypeError: object() takes no parameters


我们看到想要创建一个对象, 先调用__new__, 然后再调用__init__, 其参数也是先传入__new__, 再传入__init__

这是因为自定义类中的tp_call依然是type.tp_call, 重载自定义类的__call__无法覆盖tp_call, 因为__call__是实例方法

所以作为meta类的时候在创建类的时候, meta类的__call__不会被调用

而tp_new是可以重载, 也就是类的__new__可以覆盖掉类默认的tp_new, 由于type.tp_call调用的就是tp_new, 那么作为meta类的时候__new__是起作用的

同时类的默认__new__(tp_new, 其在C代码中指向的函数是object_new)是创建实例的调用, 是创建一个实例

所以看起来__new__可以返回类也可以返回实例, 但是我们可以这么看, meta类的__new__返回的依然是实例, 只是其实例是一个类!


__slots__
===============

https://stackoverflow.com/questions/472000/usage-of-slots


__prepare__
===============

https://www.python.org/dev/peps/pep-3115/

__prepare__ returns a dictionary-like object which is used to store the class member definitions during evaluation of the class body. In other words, the class body is evaluated as a function block (just like it is now), except that the local variables dictionary is replaced by the dictionary returned from __prepare__. This dictionary object can be a regular dictionary or a custom mapping type.

__prepare__返回的是预设属性, 当你需要预先设置某些属性的时候就返回一个字典指明预设kv

.. code-block:: python

    In [205]: class A(type):
         ...:     def __prepare__(*args, **kwargs):
         ...:         print("A.__prepare__", args, kwargs)
         ...:         return {"a": 1}
         ...:
         ...:
         ...:
    
    In [206]: class AA(metaclass=A):
         ...:     pass
         ...:
    A.__prepare__ ('AA', ()) {}
    
    In [207]: aa=AA()
    
    In [208]: aa
    Out[208]: <__main__.AA at 0x2a1bd172f28>
    
    In [209]: aa.a
    Out[209]: 1


但是__prepare__预设的属性和类属性重名的话, 会被类属性修改掉的

.. code-block:: python

    In [210]: class A(type):
         ...:     def __prepare__(*args, **kwargs):
         ...:         print("A.__prepare__", args, kwargs)
         ...:         return {"a": 1}
         ...:
         ...:
         ...:
    
    In [211]:
    
    In [211]: class AA(metaclass=A):
         ...:     a = 2
         ...:
         ...:
    A.__prepare__ ('AA', ()) {}
    
    In [212]: a=AA()
    
    In [213]: a.a
    Out[213]: 2



