bufer协议和memoryview
=======================

1. https://eli.thegreenplace.net/2011/11/28/less-copies-in-python-with-the-buffer-protocol-and-memoryviews

2. https://stackoverflow.com/questions/18655648/what-exactly-is-the-point-of-memoryview-in-python

buffer协议简单来说就是支持把对象的内部数据指针暴露出来, 比如bytes对象是支持buffer协议的, 意味着它有方法获取其数据指针.

暴露出对象指针之后还需要对指针进行管理, 所以使用Py_buffer结构来保存哪个对象暴露的指针是什么


.. code-block:: c

    typedef struct bufferinfo {
        void *buf;            /* 要暴露哪个数据指针, 一般是obj自己的数据指针, 也可以是和obj没关系的指针 */
        PyObject *obj;        /* owned reference Py_buffer关联的python对象*/
        Py_ssize_t len;
        Py_ssize_t itemsize;  /* This is Py_ssize_t so it can be
                                 pointed to by strides in simple case.*/
        int readonly;
        int ndim;
        char *format;
        Py_ssize_t *shape;
        Py_ssize_t *strides;
        Py_ssize_t *suboffsets;
        void *internal;
    } Py_buffer;

同时还有一个_PyManagedBufferObject对象, 一个管理Py_bufer的对象, 其中记录了Py_buffer被引用了多少次, 当引用次数为0的时候才能释放这个指针指向的内存

而buffer协议一般是和memoryview一起使用, 因为Py_buffer是C级别的对象, 要在python中使用就必须使用memoryview来包装成memoryview对象, 这样python层才能操作你暴露出来的指针

memoryview是提供对Py_buffer进行操作的的对象, **好处就是因为操作的指针, 所以可以省去拷贝操作**, 比如obj=obj[1:]这样的分片赋值操作就不需要拷贝了, 在参考2中有memoryview进行分片操作的性能对比

我们要操作bytes对象的内存指针, 先构建一个bytes对象的memoryview对象, 那么在c代码中


.. code-block:: c

    PyObject *
    PyMemoryView_FromObject(PyObject *v)
    {
        _PyManagedBufferObject *mbuf;
    
        if (PyMemoryView_Check(v)) {
            PyMemoryViewObject *mv = (PyMemoryViewObject *)v;
            CHECK_RELEASED(mv);
            return mbuf_add_view(mv->mbuf, &mv->view);
        }
        else if (PyObject_CheckBuffer(v)) {
            PyObject *ret;
            // 1. 这里我们得到Py_buffer的管理对象
            mbuf = (_PyManagedBufferObject *)_PyManagedBuffer_FromObject(v);
            if (mbuf == NULL)
                return NULL;
            // 2. 构建memoryview对象!!!!!!!!!!!!!!
            ret = mbuf_add_view(mbuf, NULL);
            Py_DECREF(mbuf);
            return ret;
        }
    
        PyErr_Format(PyExc_TypeError,
            "memoryview: a bytes-like object is required, not '%.200s'",
            Py_TYPE(v)->tp_name);
        return NULL;
    }


    static PyObject *
    _PyManagedBuffer_FromObject(PyObject *base)
    {
        _PyManagedBufferObject *mbuf;
    
        mbuf = mbuf_alloc();
        if (mbuf == NULL)
            return NULL;
    
        // 这里会调用对象本身的buffer接口
        if (PyObject_GetBuffer(base, &mbuf->master, PyBUF_FULL_RO) < 0) {
            mbuf->master.obj = NULL;
            Py_DECREF(mbuf);
            return NULL;
        }
    
        return (PyObject *)mbuf;
    }

    int
    PyObject_GetBuffer(PyObject *obj, Py_buffer *view, int flags)
    {
        PyBufferProcs *pb = obj->ob_type->tp_as_buffer;
    
        if (pb == NULL || pb->bf_getbuffer == NULL) {
            PyErr_Format(PyExc_TypeError,
                         "a bytes-like object is required, not '%.100s'",
                         Py_TYPE(obj)->tp_name);
            return -1;
        }
        // 这里我们看到如果一个对象支持buffer协议的话, 会调用相关函数
        return (*pb->bf_getbuffer)(obj, view, flags);
    }


所以一个对象支持buffer协议的话, 那么就必须存在tp_as_buffer这个函数组, 这个函数组有bf_getbuffer和bf_releasebuffer这两个函数

.. code-block:: c

    typedef struct {
         getbufferproc bf_getbuffer;
         releasebufferproc bf_releasebuffer;
    } PyBufferProcs;



我们看看bytes对象中的bf_getbuffer


.. code-block:: c

    bytearray_getbuffer(PyByteArrayObject *obj, Py_buffer *view, int flags)
    {
        void *ptr;
        if (view == NULL) {
            PyErr_SetString(PyExc_BufferError,
                "bytearray_getbuffer: view==NULL argument is obsolete");
            return -1;
        }
        ptr = (void *) PyByteArray_AS_STRING(obj);
        /* cannot fail if view != NULL and readonly == 0 */
        // 这里是构建Py_buffer对象的函数
        (void)PyBuffer_FillInfo(view, (PyObject*)obj, ptr, Py_SIZE(obj), 0, flags);
        obj->ob_exports++;
        return 0;
    }


我们看到最终ptr是指向bytes对象中的数据数组, 然后使用PyBuffer_FillInfo把这个ptr给暴露出去

下面会使用一个无拷贝创建numpy对象的例子去看如何使用Py_buffer和memoryview


Cython中无拷贝创建numpy的ndarray
=======================================

通常我们会在c中得到一个数据指针, 比如图片的数据, 希望无拷贝返回这个指向这个指针的numpy.ndarray


.. code-block:: python

    def process_img(ndarray):
        Mat mat = process(ndarray)
        // mat是opencv Mat结构
        res = no_copy_create_ndarray_from_mat(mat)
        return res


首先, numpy.ndarray中入参参数可以传入一个支持python buffer协议的对象

*buffer : object exposing buffer interface, optional Used to fill the array with data.*


这里注意的是python buffer并不是python object, 所以不能直接传入python buffer对象, 我们可以使用memoryview


所以首先创建一个Py_buffer对象, 由于我们的指针没有关联到python对象, 那么我们不能通过object来得到Py_buffer对象, 但是我们可以手动构建Py_buffer对象

.. code-block:: python

    cdef Py_buffer buf_info
    PyBuffer_FillInfo(&buf_info, NULL, data, vlen, 1, PyBUF_FULL_RO)


其中data是数据指针, vlen是data的长度. 函数PyBuffer_FillInfo创建一个Py_buffer对象, 但是不会复制data, 同时第二个参数是关联的python对象, 因为我们没有python对象, 所以传NULL


.. code-block:: c

    int
    PyBuffer_FillInfo(Py_buffer *view, PyObject *obj, void *buf, Py_ssize_t len,
                      int readonly, int flags)
    {
    
        view->obj = obj;
        if (obj)
            Py_INCREF(obj);
        view->buf = buf;
        view->len = len;
        view->readonly = readonly;
        view->itemsize = 1;
        view->format = NULL;
        if ((flags & PyBUF_FORMAT) == PyBUF_FORMAT)
            view->format = "B";
        view->ndim = 1;
        view->shape = NULL;
        if ((flags & PyBUF_ND) == PyBUF_ND)
            view->shape = &(view->len);
        view->strides = NULL;
        if ((flags & PyBUF_STRIDES) == PyBUF_STRIDES)
            view->strides = &(view->itemsize);
        view->suboffsets = NULL;
        view->internal = NULL;
        return 0;
    }

所以只是把data赋值为Py_buffer中的buf字段而已, 然后我们使用memoryview来管理我们的Py_buffer

.. code-block:: python

    Pydata  = PyMemoryView_FromBuffer(&buf_info)

这里Pydata就是PyMemoryView_FromBuffer中新创建的PyMemoryViewObject

.. code-block:: c

    PyObject *
    PyMemoryView_FromBuffer(Py_buffer *info)
    {
        _PyManagedBufferObject *mbuf;
        PyObject *mv;
    
        if (info->buf == NULL) {
            PyErr_SetString(PyExc_ValueError,
                "PyMemoryView_FromBuffer(): info->buf must not be NULL");
            return NULL;
        }
    
        // 这里分配一个ManagedBufferObject
        mbuf = mbuf_alloc();
        if (mbuf == NULL)
            return NULL;
    
        /* info->obj is either NULL or a borrowed reference. This reference
           should not be decremented in PyBuffer_Release(). */
        mbuf->master = *info;
        mbuf->master.obj = NULL;
    
        // 这里生成新的PyMemoryViewObject对象!!!!!
        mv = mbuf_add_view(mbuf, NULL);
        // 这里减少mbuf的引用计数, 所以mbuf的作用只是辅助生成mv
        Py_DECREF(mbuf);
    
        return mv;
    }


    static PyObject *
    mbuf_add_view(_PyManagedBufferObject *mbuf, const Py_buffer *src)
    {
        PyMemoryViewObject *mv;
        Py_buffer *dest;
    
        if (src == NULL)
            src = &mbuf->master;
    
        if (src->ndim > PyBUF_MAX_NDIM) {
            PyErr_SetString(PyExc_ValueError,
                "memoryview: number of dimensions must not exceed "
                Py_STRINGIFY(PyBUF_MAX_NDIM));
            return NULL;
        }
    
        // 这里从内存池中分配出PyMemoryViewObject对象的内存!!!!!
        mv = memory_alloc(src->ndim);
        if (mv == NULL)
            return NULL;
    
        dest = &mv->view;
        init_shared_values(dest, src);
        init_shape_strides(dest, src);
        init_suboffsets(dest, src);
        init_flags(mv);
    
        mv->mbuf = mbuf;
        // 增加mbuf的引用计数, mbuf不会被立即释放!!!!
        Py_INCREF(mbuf);
        mbuf->exports++;
    
        return (PyObject *)mv;
    }


    // 这里生成memoryview对象, 同时加入GC
    static inline PyMemoryViewObject *
    memory_alloc(int ndim)
    {
        PyMemoryViewObject *mv;
    
        mv = (PyMemoryViewObject *)
            PyObject_GC_NewVar(PyMemoryViewObject, &PyMemoryView_Type, 3*ndim);
        if (mv == NULL)
            return NULL;
    
        mv->mbuf = NULL;
        mv->hash = -1;
        mv->flags = 0;
        mv->exports = 0;
        mv->view.ndim = ndim;
        mv->view.shape = mv->ob_array;
        mv->view.strides = mv->ob_array + ndim;
        mv->view.suboffsets = mv->ob_array + 2 * ndim;
        mv->weakreflist = NULL;
    
        // 加入GC, 所以mv也是使用引用计数去管理
        _PyObject_GC_TRACK(mv);
        return mv;
    }


而mbuf和pybuffer的关系则是mbuf保存了pybuffer对应的一些export信息

.. code-block:: c

    typedef struct {
        PyObject_HEAD
        int flags;          /* state flags */
        Py_ssize_t exports; /* number of direct memoryview exports */
        Py_buffer master; /* snapshot buffer obtained from the original exporter */
    } _PyManagedBufferObject;


我们看到每次mv被一个memoryview给包装一次, exports就加1, 所以exports不为0显然就不能释放mv


最后我们把得到的PyMemoryViewObject, 传给numpy.ndarray

.. code-block:: python

    ary = np.ndarray(shape=shape, buffer=Pydata, order='c', dtype=np.uint8)


我们返回出去的是一个memoryview对象, 一旦memoryview对象被引用计数机制释放, 那么它会去释放我们的Py_buffer对象

但是, Py_buffer对象并一定会释放我们的data

.. code-block:: c

    void
    PyBuffer_Release(Py_buffer *view)
    {
        PyObject *obj = view->obj;
        PyBufferProcs *pb;
        // 这里如果obj是NULL, 则不会释放存储在Py_buffer对象上的buf数据
        if (obj == NULL)
            return;
        pb = Py_TYPE(obj)->tp_as_buffer;
        if (pb && pb->bf_releasebuffer)
            pb->bf_releasebuffer(obj, view);
        view->obj = NULL;
        Py_DECREF(obj);
    }


因为我们在创建Py_buffer的时候, 第二个参数也就是obj, 传入的是NULL, 所以即使memoryview对象被释放, 那么我们的data指针也不会被释放

如果我们复用data指针, 那么就必须知道一旦data被覆盖, 那么后续的memoryview可能访问到错误的数据

如果我们不能正确的释放data指针, 那么会造成内存泄漏


