Small Memory malloc and free
#############################################

1. https://docs.python.org/3/c-api/memory.html

2. http://www.wklken.me/posts/2015/08/29/python-source-memory-1.html

Objects/obmalloc.c

CPython中对于小内存对象会使用缓存的策略加速而不是直接从操作系统中分配, 缓存包括free list和小对象缓存池

free list和小对象缓存池的关系则是free list是某些类型对象自己的缓存列表

而小对象缓存池则是分配/释放策略, 所有分配/释放都交给统一的接口/模块, 这里称为内存管理接口, 那么如何分配和释放则由接口来决定的, 比如释放的时候可以释放到系统也可以先缓存着

在CPython中, 内存管理接口在分配内存的时候, 对于大小小于某个阈值的对象都是从缓存池中分配出来, free list中的对象分配和释放都是经过内存管理接口的

pool和arena的结构和分配逻辑请看参考2, 作者写得很清楚, 这里就不重复地写了


free list
============

free list的作用是对象复用, 比如对象a被释放的时候可以先不释放到操作系统而是把这个失效的对象给先缓存起来, 缓存的方式就是把所有释放的对象使用链表链起来

作为备份使用, 这个链表就称为free list. 当下次要分配同样类型的对象的时候, 优先从free list中拿出备份的内存, 这样就省去了内存分配这样一个开销

CPython中内置的对象很多都预分配了自己的free list来应对小对象的分配(也就是说每个对象的free list的结构和处理都可能不一样)

比如对于整数对象, 所有处于[-5, 257)区间的整数都会指向同一个对象

.. code-block:: python

    In [1]: x=1
    
    In [2]: a=1
    
    In [3]: id(x), id(a)
    Out[3]: (1532610224, 1532610224)
    
    In [4]: x == a
    Out[4]: True
    
    In [5]: x is a
    Out[5]: True

在创建整数的时候

.. code-block:: c

    PyObject *
    PyLong_FromLong(long ival)
    {
        PyLongObject *v;
        unsigned long abs_ival;
        unsigned long t;  /* unsigned so >> doesn't propagate sign bit */
        int ndigits = 0;
        int sign;
    
        // 校验是否是小整数
        CHECK_SMALL_INT(ival);
    
        // 省略
    
    }

    // 如果是小整数那么直接返回小整数对象
    #define CHECK_SMALL_INT(ival) \
        do if (-NSMALLNEGINTS <= ival && ival < NSMALLPOSINTS) { \
            return get_small_int((sdigit)ival); \
        } while(0)

    // get_small_int则是从free list中拿到整数
    static PyObject *
    get_small_int(sdigit ival)
    {
        PyObject *v;
        // 整数一定是在[-5, 257)的范围
        assert(-NSMALLNEGINTS <= ival && ival < NSMALLPOSINTS);
        // 从小整数(free list)中拿到缓存值然后返回
        v = (PyObject *)&small_ints[ival + NSMALLNEGINTS];
        Py_INCREF(v);
        return v;
    }


而float对象也有自己的free list, 其free list为单链表, free list的个数最大为100个, 当某个float对象释放的时候, 优先加入到free list中

如果free list上的对象个数大于100, 那么直接调用内存管理接口取释放


.. code-block:: c

    // 分配float的时候先判断free list中有没有备份的对象
    PyObject *
    PyFloat_FromDouble(double fval)
    {
        PyFloatObject *op = free_list;
        if (op != NULL) {
            // 从free list中拿到一个对象
            free_list = (PyFloatObject *) Py_TYPE(op);
            numfree--;
        } else {
            op = (PyFloatObject*) PyObject_MALLOC(sizeof(PyFloatObject));
            if (!op)
                return PyErr_NoMemory();
        }
        /* Inline PyObject_New */
        (void)PyObject_INIT(op, &PyFloat_Type);
        op->ob_fval = fval;
        return (PyObject *) op;
    }

    // 释放的时候优先加入到free list中
    static void
    float_dealloc(PyFloatObject *op)
    {
        if (PyFloat_CheckExact(op)) {
            // 如果free list已经满了, 就直接释放掉了
            if (numfree >= PyFloat_MAXFREELIST)  {
                PyObject_FREE(op);
                return;
            }
            // 否则加入到free list中缓存起来
            numfree++;
            // 加入free list的操作是把op加入到free list的头部, op.ob_type = free_list, free_list=op
            Py_TYPE(op) = (struct _typeobject *)free_list;
            free_list = op;
        }
        else
            Py_TYPE(op)->tp_free((PyObject *)op);
    }

.. code-block:: python

    '''
    注意的是float的free list是直接用ob_type直接链接起来的...
    free list: float_object
                        ob_type -> float_object
                                           ob_type -> ...
    
    '''

我们看到float和int(或者说long)两者free list处理上由区别, tuple, list等对象也有自己的free list实现, 比如tuple的free list是一个长度为20的数组

下表为i表示所有的大小为i的tuple都链接在一起, 每个size的tuple最多缓存2000个

.. code-block:: python

    '''
    0, 1, 2, 3, 4这些是数组下标, 也表示指定的tuple的元素个数, 比如分配一个元素个数为1(size=1)的tuple的时候, 从free_list_array[1]拿到链表头, 然后从其中拿走一个

    每个size的tuple最多2000个, size=0的tuple比较特殊, 全局一个
    
    free_list_array: 0, 1, 2, 3, 4, .... 20
                     t  t
                        t
                        t
    '''

.. code-block:: python

    In [10]: x=()
    
    In [11]: a=()
    
    In [12]: x==a
    Out[12]: True
    
    In [13]: x is a
    Out[13]: True
    
    In [14]: id(x), id(a)
    Out[14]: (2161143447624, 2161143447624)

小对象
=========

在CPython中, 小对象是从缓存池中分配/释放的, 这个小对象的小定义为512字节, 大于这个大小的对象直接从操作系统分配

.. code-block:: c

    /*
    * Request in bytes     Size of allocated block      Size class idx
    * ----------------------------------------------------------------
    *        1-8                     8                       0
    *        9-16                   16                       1
    *       17-24                   24                       2
    *       25-32                   32                       3
    *       33-40                   40                       4
    *       41-48                   48                       5
    *       49-56                   56                       6
    *       57-64                   64                       7
    *       65-72                   72                       8
    *        ...                   ...                     ...
    *      497-504                 504                      62
    *      505-512                 512                      63
    */


上表表示如果分配的对象大小s, 有1<=s<=8, 那么分配一个大小为8字节的数据块, 以此类推, 这样内存数据就使用数据块的方式对齐了



usedpools
==============

如果我们要分配某个大小的block的话, 比如我们每次要分配8字节的内存的时候, 不可能每次只分配8字节, 很低效, 比较通用的方法就是每次8字节都从某一个比较大的内存中分配出来

比如我们向分配一个大小为16的block, 那么我们可以从某个大内存中分出16字节, 这样这个大内存就可以分配很多个16字节了, 这个大内存称为pool, 每个pool的大小和操作系统有关

.. code-block:: c

    #define SYSTEM_PAGE_SIZE        (4 * 1024)
    #define POOL_SIZE               SYSTEM_PAGE_SIZE        /* must be 2^N */

POOL_SIZE就是一个pool的大小, 为系统page size, 一般为4KB

那么如何找到这个pool呢? pool的管理最直接的一种方式就是使用数组, 也就是所有的pool都存储在一个数组中, 这样如果我们想要分配某个

大小的block的时候, 从pool数组中拿到指定pool, 然后分配就好了. 最粗略的样子就是

.. code-block:: python

    '''
    
    pool_array: [pool0, pool1, pool2, ...]

    
    由于pool0不止有一个, 比如4KB的大小用完了, 我们需要分配另外一个4KB, 所以最简单的, 某个size下的所有的pool都使用双链表链连接起来
    
    pool_array: [pool00, pool10, pool20, ...]
                 pool01  pool11  pool21

    pool00, pool01, pool02, ..., pool0x 都是每次分配8字节的pool, 彼此双链表连接起来
    
    '''

很显然, 一个4K的pool可以分配4*1024 / 8 = 512个大小为8的block, 可以分配4*1024 / 512 = 8个大小为512的block.

pool在代码定义


.. code-block:: c

    /* Pool for small blocks. */
    struct pool_header {
        union { block *_padding;
                uint count; } ref;          /* number of allocated blocks    */
        block *freeblock;                   /* pool's free list head         */
        struct pool_header *nextpool;       /* next pool of this size class  */
        struct pool_header *prevpool;       /* previous pool       ""        */
        uint arenaindex;                    /* index into arenas of base adr */
        uint szidx;                         /* block size class index        */
        uint nextoffset;                    /* bytes to virgin block         */
        uint maxnextoffset;                 /* largest valid nextoffset      */
    };
    
    typedef struct pool_header *poolp;

这里比较重要的是nextpool, prevpool和, 这两个指针说明了pool是使用双链表的结构链接在一起的

从上一节可知一共有64个不同大小的数据块, 显然每个大小的数据块都是存储在pool中的, 也就是每个pool分配的时候都是分配出一个固定大小的block的

那么显然了一共有64个大小不同的pool同时每个大小的pool都是双链表链接, 所以就需要有64*2=128个链表头, 我们期望是这样


.. code-block:: python

    '''
    
    pool_array: [0_head, 0_tail, 1_head, 1_tail, ..., 63_head, 63_tail]
                       ->               ->                   ->
                       <-               <-                   <-

    '''

pool array一共有64*2=128个元素, 每个元素都是pool_header, 这样我们想要分配某个大小的block的时候, 直接从pool array拿就好了, 显然

我们需要分配大小为8的block, 其pool的header存储在pool_array[0]处, 而大小为1的block的pool的header是存储在pool_array[2], 所以大小为

x的block其pool的header是在2*x处. 在CPython中, 上图中的pool_array定义为


.. code-block:: c

    #define PTA(x)  ((poolp )((uint8_t *)&(usedpools[2*(x)]) - 2*sizeof(block *)))
    #define PT(x)   PTA(x), PTA(x)
    
    Static poolp usedpools[2 * ((NB_SMALL_SIZE_CLASSES + 7) / 8) * 8] = {
        PT(0), PT(1), PT(2), PT(3), PT(4), PT(5), PT(6), PT(7)
    #if NB_SMALL_SIZE_CLASSES > 8
        , PT(8), PT(9), PT(10), PT(11), PT(12), PT(13), PT(14), PT(15)
    #if NB_SMALL_SIZE_CLASSES > 16
        , PT(16), PT(17), PT(18), PT(19), PT(20), PT(21), PT(22), PT(23)
    #if NB_SMALL_SIZE_CLASSES > 24
        , PT(24), PT(25), PT(26), PT(27), PT(28), PT(29), PT(30), PT(31)
    #if NB_SMALL_SIZE_CLASSES > 32
        , PT(32), PT(33), PT(34), PT(35), PT(36), PT(37), PT(38), PT(39)
    #if NB_SMALL_SIZE_CLASSES > 40
        , PT(40), PT(41), PT(42), PT(43), PT(44), PT(45), PT(46), PT(47)
    #if NB_SMALL_SIZE_CLASSES > 48
        , PT(48), PT(49), PT(50), PT(51), PT(52), PT(53), PT(54), PT(55)
    #if NB_SMALL_SIZE_CLASSES > 56
        , PT(56), PT(57), PT(58), PT(59), PT(60), PT(61), PT(62), PT(63)
    #if NB_SMALL_SIZE_CLASSES > 64
    #error "NB_SMALL_SIZE_CLASSES should be less than 64"
    #endif /* NB_SMALL_SIZE_CLASSES > 64 */
    #endif /* NB_SMALL_SIZE_CLASSES > 56 */
    #endif /* NB_SMALL_SIZE_CLASSES > 48 */
    #endif /* NB_SMALL_SIZE_CLASSES > 40 */
    #endif /* NB_SMALL_SIZE_CLASSES > 32 */
    #endif /* NB_SMALL_SIZE_CLASSES > 24 */
    #endif /* NB_SMALL_SIZE_CLASSES > 16 */
    #endif /* NB_SMALL_SIZE_CLASSES >  8 */
    };


每个PT都是PTA, PTA的形式, 也就是有2个元素, 所以usedpools一共就有64*2=128个元素, 而PTA中为什么要减去2*sizeof(block*)? 根据代码中的注释有

这是因为usedpools目的只是为了保存双链表而不是真正的poolp结构, 所以我们使用poolp的地址减去2*sizeof(block*)就得到poolp中的nextpool和prevpool这两个指针

这样PTA(0), PTA(0)就指向的地址就是同一个, 也就是prevpool和nextpool指向同一个, 这样表示表该双链表是空的. 

其实大家都没看懂为什么: *It's unclear why the usedpools setup is so convoluted* , 但是理解usedpools中就是双链表的next和prev指针就好了



分配逻辑
===============

查看参考2



