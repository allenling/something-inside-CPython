UNPACK_SEQUENCE和交换元素
################################

https://zhuanlan.zhihu.com/p/159668720

a, b, c = c, b, a这样的操作的时候, Python什么时候使用交换字节码什么时候使用解包字节码呢


1. 当右侧有变量的时候, 总是使用交换操作, 如果右侧都是常量, 那么使用UNPACK_SEQUENCE

   .. code-block:: python

       In [25]: dis.dis("a, b, c = 2,3,a")
         1           0 LOAD_CONST               0 (2)
                     2 LOAD_CONST               1 (3)
                     4 LOAD_NAME                0 (a)
                     6 ROT_THREE
                     8 ROT_TWO
                    10 STORE_NAME               0 (a)
                    12 STORE_NAME               1 (b)
                    14 STORE_NAME               2 (c)
                    16 LOAD_CONST               2 (None)
                    18 RETURN_VALUE

2. 当元素比较多, 就使用UNPACK_SEQUENCE, 否则使用交换操作. 比如3.8开始才启用ROT_FORTH(虽然3.6中有这个字节码), 那么a, b, c, d = d, c, b, a会使用UNPACK_SEQUENCE

   如果交换的变量少于等于3个, 是否使用交换操作, 取决于情况1


UNPACK_SEQUENCE
===================

.. code-block:: python

    In [19]: dis.dis("a,b,c,d=d,c,b,a")
      1           0 LOAD_NAME                0 (d)
                  2 LOAD_NAME                1 (c)
                  4 LOAD_NAME                2 (b)
                  6 LOAD_NAME                3 (a)
                  8 BUILD_TUPLE              4
                 10 UNPACK_SEQUENCE          4
                 12 STORE_NAME               3 (a)
                 14 STORE_NAME               2 (b)
                 16 STORE_NAME               1 (c)
                 18 STORE_NAME               0 (d)
                 20 LOAD_CONST               0 (None)
                 22 RETURN_VALUE

.. code-block:: c

        TARGET(UNPACK_SEQUENCE) {
            PyObject *seq = POP(), *item, **items;
            if (PyTuple_CheckExact(seq) &&
                PyTuple_GET_SIZE(seq) == oparg) {
                items = ((PyTupleObject *)seq)->ob_item;
                while (oparg--) {
                    item = items[oparg];
                    Py_INCREF(item);
                    PUSH(item);
                }
            } else if (PyList_CheckExact(seq) &&
                       PyList_GET_SIZE(seq) == oparg) {
                items = ((PyListObject *)seq)->ob_item;
                while (oparg--) {
                    item = items[oparg];
                    Py_INCREF(item);
                    PUSH(item);
                }
            } else if (unpack_iterable(seq, oparg, -1,
                                       stack_pointer + oparg)) {
                STACKADJ(oparg);
            } else {
                /* unpack_iterable() raised an exception */
                Py_DECREF(seq);
                goto error;
            }
            Py_DECREF(seq);
            DISPATCH();
        }

UNPACK_SEQUENCE的操作很简单, 拿到tuple/list/iterable对象, 然后反向入栈, 然后依次出栈去赋值, 比如例子中就是tuple={vd, vc, vb, va}, 注意看字节码LOAD_NAME是由d开始

然后从后开始进行入栈, 此时栈顺序为(栈顶)vd, vc, vb, va, 然后vd出栈, 使a指向vd, 之后以此类推


