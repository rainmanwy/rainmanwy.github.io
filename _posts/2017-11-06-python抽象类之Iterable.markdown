---
layout: post
title: python抽象类之Iterable
date: 2017-11-06
description: 前一篇《Python的抽象类学习-1》中，了解了python的抽象类的metaclass ABCMeta。抽象类可以通过抽象方法装饰器来定义自己的抽象方法。下面通过分析Iterable，Iterator进一步了解python的抽象类。 # Add post description (optional)
img:  # Add image post (optional)
tags: [python]
---
前一篇《Python的抽象类学习-1》中，了解了python的抽象类的metaclass ABCMeta。抽象类可以通过抽象方法装饰器来定义自己的抽象方法。下面通过分析Iterable，Iterator进一步了解python的抽象类。

## 通过register方式创建虚拟的子类

从ABCMeta的文档中，可以看到通过register方法，可以把不相关的类注册成子类。
> You can also register
unrelated concrete classes (even built-in classes) and unrelated
ABCs as 'virtual subclasses' -- these and their descendants will
be considered subclasses of the registering ABC by the built-in
issubclass() function, but the registering ABC won't show up in
their MRO (Method Resolution Order) nor will method
implementations defined by the registering ABC be callable (not
even via super()).

从文档可以看到：
* 被注册的类不用继承抽象类，但会被当做抽象类的子类
* 被注册类不用实现抽象类的抽象方法
* 抽象类不会出现在被注册类的MRO中
* 被注册类的子类也会被当做抽象类的子类

这个可以通过代码很方便的得到验证：
```
class MyAbstractClass(metaclass=ABCMeta):
    
    @abstractmethod
    def my_method(self):
        print('MyAbstractClass my_method')


class MyImplement(object):
    pass

class MyChildImplement(MyImplement):
    pass

MyAbstractClass.register(MyImplement)


inst = MyImplement()
print(isinstance(inst, MyAbstractClass))
print(issubclass(MyImplement, MyAbstractClass))
print(MyImplement.mro())
print(MyImplement.__bases__)

child = MyChildImplement()
print(isinstance(child, MyAbstractClass))
print(issubclass(MyChildImplement, MyAbstractClass))
print(MyChildImplement.mro())
print(MyChildImplement.__bases__)

输出：
True                     
True
[<class '__main__.MyImplement'>, <class 'object'>]
(<class 'object'>,)
True
True
[<class '__main__.MyChildImplement'>, <class '__main__.MyImplement'>, <class 'object'>]
(<class '__main__.MyImplement'>,)
```
其实register只是会建立起虚拟子类的关系，但并不限制子类一定要实现抽象父类的方法，也不会真正改变子类本身的继承关系。

## 虚拟子类的用途

register的用途是什么？ 此处基于python2.7，查看Iterable和str的关系，也许能更好的认识到它的用途。
Iterable的实现，基于python2.7（_abcoll.py）：
```
class Iterable:
    __metaclass__ = ABCMeta

    @abstractmethod
    def __iter__(self):
        while False:
            yield None

    ...

Iterable.register(str)
```
如果要继承Iterable，则需要实现__iter__方法。python2.7的str是由C实现的，切并未实现__iter__方法：
```
>>> myStr = 'hello'
>>> dir(myStr)
['__add__', '__class__', '__contains__', '__delattr__', '__doc__', '__eq__', 
'__format__', '__ge__', '__getattribute__', '__getitem__', '__getnewargs__', 
'__getslice__', '__gt__', '__hash__', '__init__', '__le__', '__len__', 
'__lt__', '__mod__', '__mul__', '__ne__', '__new__', '__reduce__', 
'__reduce_ex__', '__repr__', '__rmod__', '__rmul__', '__setattr__', 
'__sizeof__', '__str__', '__subclasshook__', '_formatter_field_name_split', 
'_formatter_parser', 'capitalize', 'center', 'count', 'decode', 'encode', 
'endswith', 'expandtabs', 'find', 'format', 'index', 'isalnum', 'isalpha', 
'isdigit', 'islower', 'isspace', 'istitle', 'isupper', 'join', 'ljust', 
'lower', 'lstrip', 'partition', 'replace', 'rfind', 'rindex', 'rjust', 
'rpartition', 'rsplit', 'rstrip', 'split', 'splitlines', 'startswith', 
'strip', 'swapcase', 'title', 'translate', 'upper', 'zfill']
```
但是，如果我们通过isinstance，issubclass查看，会发现str又确实是Iterable的子类：
```
>>> isinstance(myStr, Iterable)
True
>>> issubclass(str, Iterable)
True
>>>
```
原因就在于str被注册为Iterable的子类。

## 虚拟子类的实现原理

register的实现：
```
   def register(cls, subclass):
        """Register a virtual subclass of an ABC.

        Returns the subclass, to allow usage as a class decorator.
        """
        if not isinstance(subclass, type):
            raise TypeError("Can only register classes")
        if issubclass(subclass, cls):
            return subclass  # Already a subclass
        # Subtle: test for cycles *after* testing for "already a subclass";
        # this means we allow X.register(X) and interpret it as a no-op.
        if issubclass(cls, subclass):
            # This would create a cycle, which is bad for the algorithm below
            raise RuntimeError("Refusing to create an inheritance cycle")
        cls._abc_registry.add(subclass)
        ABCMeta._abc_invalidation_counter += 1  # Invalidate negative cache
        return subclass
```
register的实现很简单，主要就是“cls._abc_registry.add(subclass)”，往_abc_registry中添加了要作为虚拟子类的类。如果查看_abc_registry被使用的地方，可以看到ABCMeta重载了isinstance和issubclass方法，并在issubclass中使用了_abc_registry
```
   def __instancecheck__(cls, instance):
        """Override for isinstance(instance, cls)."""
        # Inline the cache checking
        subclass = instance.__class__
        ...
        subtype = type(instance)
        if subtype is subclass:
            ...
            # Fall back to the subclass check.
            return cls.__subclasscheck__(subclass)
        return any(cls.__subclasscheck__(c) for c in {subclass, subtype})

    def __subclasscheck__(cls, subclass):
        """Override for issubclass(subclass, cls)."""
        ...
        # Check the subclass hook
        ok = cls.__subclasshook__(subclass)
        ...
        # Check if it's a subclass of a registered class (recursive)
        for rcls in cls._abc_registry:
            if issubclass(subclass, rcls):
                cls._abc_cache.add(subclass)
                return True
        ...
```
从实现可以知道，issubclass方法传入的子类会与_abc_registry中注册的虚拟子类比较。

如果此子类在_abc_registry中
或者是_abc_regsitry中某个虚拟子类的子类
则返回True。因此str注册为Iterable的虚拟子类后，isinstance和issubclass会返回True。

## 实现Iterable
我们实现一个空类，未实现任何方法：
```
from collections import Iterable

class MyIter(object):
    pass

inst = MyIter()
print(isinstance(inst, Iterable))
print(issubclass(MyIter, Iterable))

输出：
False
False
```
这个结果是我们所期望的。MyIter未实现__iter__抽象方法，所以并不认为是Iterable的子类。如果实现了__iter__呢
```
class MyIter(object):
    
    def __iter__(self):
        pass

inst = MyIter()
print(isinstance(inst, Iterable))
print(issubclass(MyIter, Iterable))

输出：
True
True
```
当__iter__被实现后，MyIter被认为是Iterable的子类。但此处有一个问题：MyIter并未继承Iterable抽象类，也未通过register方法注册为Iterable的虚拟子类，为什么还是会被当做Iterable的子类呢？
如查看Iterable的实现，可以看到方法__subclasshook__：
```
   @classmethod
    def __subclasshook__(cls, C):
        if cls is Iterable:
            return _check_methods(C, "__iter__")
        return NotImplemented
```
_check_methods的实现：
```
def _check_methods(C, *methods):
    mro = C.__mro__
    for method in methods:
        for B in mro:
            if method in B.__dict__:
                if B.__dict__[method] is None:
                    return NotImplemented
                break
        else:
            return NotImplemented
    return True
```
Iterable的__subclasshook__查看了类C的mro中，是否有谁实现了__Iter__方法，如有则返回True。而__subclasshook__会由ABCMeta的__subclasscheck__调用（请查看前面的__subclasscheck__的实现代码）。

## 结论
* 通过注册虚拟子类的方法，可以把本来不相关的类，注册为抽象类的子类。注册本身并不会改变子类的类层次结构、初始化等，也对是否实现抽象类的抽象方法没有限制。
* 注册虚拟子类，影响的是isinstance和issubclass方法。这些方法在ABCMeta中被重载
* Iterable重载了ABCMeta中的__subclasshook__，可以影响isinstance，issubclass方法。实现类只要实现了Iterable的抽象方法，isinstance、issubclass就会返回实现类是Iterable的子类，而不需要实现类显式的去继承Iterable.