Python Class查询属性
######################


1. http://m.blog.chinaunix.net/uid-23100982-id-3411776.html

2. https://groups.google.com/forum/?fromgroups=#!topic/python-cn/oZpnaafcoeo

3. http://www.srikanthtechnologies.com/blog/python/mro.aspx

4. https://en.wikipedia.org/wiki/C3_linearization

访问属性缓存
===============

原型链继承, 查找的时候是通过mro顺序查找, 但是有全局缓存, 缓存的key是type.tp_version_tag和hash(method_name)的计算, 参考1, 2

在python3.6下, hash("get_x")和hash("get_y")是值差不是1, 所以参考1, 2中的例子在python3.6不生效


mro算法
=====================

mro的构建是在创建类的时候调用PyType_Ready -> mro_internal -> mro_implementation -> tail_contains计算出来的, 其使用的算法称为C3算法, 在参考4的wiki中有流程






