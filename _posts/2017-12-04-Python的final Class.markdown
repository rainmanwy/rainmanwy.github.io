---
layout: post
title: python的final class
date: 2017-12-04
description: 本文介绍了python中的final class，以及如何实现自己的final class # Add post description (optional)
img:  # Add image post (optional)
tags: [python]
---
本文介绍了python中的final class，以及如何实现自己的final class。在java中，当一个类前面加上final后，表面这个类不希望被继承。python中并没有final这样的关键字，那python中是否也有final class呢？我们怎么实现自己的final class呢？

## Python自带的final class

我们可以试运行一下下边的代码
```
g = iter('ab')
print(type(g))

class MyIter(type(g)):
    pass

i = MyIter()
```
这段代码中，希望继承str_iterator，实现自己的迭代器。运行后的结果：
```
输出：
<class 'str_iterator'>
Traceback (most recent call last):
  File "/home/rainman/test.py", line 9, in <module>
    class MyIter(type(g)):
TypeError: type 'str_iterator' is not an acceptable base type
```
看来str_iterator不运行被继承。这个错误试怎么发生的呢，python是怎么判断这个类不能被继承呢。

### python类定义的tp_flags字段

查看python的C实现，可以找到两处会抛出这个异常的地方：
```
typeobject.c：

static PyTypeObject *
best_base(PyObject *bases)
{
    ...

    for (i = 0; i < n; i++) {
        ...

        if (!PyType_HasFeature(base_i, Py_TPFLAGS_BASETYPE)) {
            PyErr_Format(PyExc_TypeError,
                         "type '%.100s' is not an acceptable base type",
                         base_i->tp_name);
            return NULL;
        }

        ...
    }
    assert (base != NULL);

    return base;
}

object.h:

#define PyType_HasFeature(t,f)  (((t)->tp_flags & (f)) != 0)

/* Set if the type allows subclassing */
#define Py_TPFLAGS_BASETYPE (1UL << 10)

```
重点在”PyType_HasFeature"，它会查看类型的tp_flags字段，是否设置了“Py_TPFLAGS_BASETYPE”。而“Py_TPFLAGS_BASETYPE”正是用来标志一个类是否能够被继承。

### str_iterator类定义

查看str_iterator的类定义，确实没有设置标志“Py_TPFLAGS_BASETYPE”
```
PyTypeObject PyUnicodeIter_Type = {
    PyVarObject_HEAD_INIT(&PyType_Type, 0)
    "str_iterator",         /* tp_name */
    sizeof(unicodeiterobject),      /* tp_basicsize */

    ...

    Py_TPFLAGS_DEFAULT | Py_TPFLAGS_HAVE_GC,/* tp_flags */

    ...
};
```

## 实现自己的final class

如果是通过C语言来实现，当然可以参照python内部的final class。但如果纯python代码的方式来实现呢？ 我们知道python的metaclass可以控制生成类，参照python的abstract class，我们应该也可以实现类似的功能。例如：
```
class FinalMeta(type):

    def __new__(mcls, name, bases, dict):
        for base in bases:
            if isinstance(base, FinalMeta):
                raise TypeError("type '{0}' is not an acceptable base type".format(base.__name__))
        cls = super().__new__(mcls, name, bases, dict)
        return cls

class Parent(dict, metaclass=FinalMeta):
    pass

class Child(Parent):
    pass

输出：
Traceback (most recent call last):
  File "/home/rainman/test.py", line 18, in <module>
    class Child(Parent):
  File "/home/rainman/test.py", line 11, in __new__
    raise TypeError("type '{0}' is not an acceptable base type".format(base.__name__))
TypeError: type 'Parent' is not an acceptable base type
```
可以定义个叫FinalMeta的metaclass，在__new__方法生成类时，判断类的基类中，是否有类型为FinalMeta的类。即通过isintance判断类的类型是否是FinalMeta。
