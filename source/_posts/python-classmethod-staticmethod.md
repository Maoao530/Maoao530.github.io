---
title: python中的@classmethod和@staticmethod
date: 2016-05-17 23:58:59
categories:
- Python基础
tags:
- Python
---

花两分钟了解什么是实例方法，类方法，静态方法。

<!-- more -->

# 1. 定义

python中类方法和静态方法是用装饰器`@classmethod`和`@staticmethod`来定义的。

[点我学习什么是装饰器](https://maoao530.github.io/2016/05/17/python-decorator/)

我们先来看一个简单的实例：

```python
class A(object):
    def foo(self,x):
        print ("executing foo(%s,%s)"%(self,x))

    @classmethod
    def class_foo(cls,x):
        print ("executing class_foo(%s,%s)"%(cls,x))

    @staticmethod
    def static_foo(x):
        print ("executing static_foo(%s)"%x)

```

我们定义了一个`class A`，类A里面分别定义了普通方法foo，`@classmethod`修饰的类方法class_foo，还有`@staticmethod`修饰的静态方法static_foo，那么他们之间有什么区别呢？
我们不妨来验证一下：

```python
a=A()
print(a.foo)
print(a.class_foo)
print(a.static_foo)

# 输出结果
# <bound method A.foo of <__main__.A object at 0x0121B950>>
# <bound method A.class_foo of <class '__main__.A'>>
# <function A.static_foo at 0x01222078>

```
我们从输出结果可以看到：
- foo是绑定在实例a上的，参数self便是实例a本身
- class_foo是绑定在class A上的，参数cls指向class A本身
- static_foo不绑定在a或者A上，所以没有额外的参数

# 2. 使用

那么使用上有什么区别呢？
```python
a = A()
a.foo(1)
a.class_foo(1)
a.static_foo(1)

# A.foo(2) #error
A.class_foo(2)
A.static_foo(2)

# 输出结果
# executing foo(<__main__.A object at 0x0121B950>,1)
# executing class_foo(<class '__main__.A'>,1)
# executing static_foo(1)
# executing class_foo(<class '__main__.A'>,2)
# executing static_foo(2)

```
从上面的代码我们可以看到，使用上的区别：
- foo是绑定在实例a上的，只能通过实例去调用
- class_foo绑定在类A上，可以通过实例a或者类A去调用
- static_foo不绑定任何参数，也可以通过实例a或者类A去调用

# 3. 总结

类和实例都是对象，所以它们可以有方法：

- 实例的方法就叫实例方法。
- 类的方法就叫类方法。
- 静态方法就是写在类里的普通方法,必须用类来调用，比如说有一些跟类有关系的功能，但是运行的时候又不需要实例和类参与的函数，这个时候就可以用@staticmethod，因为如果写一堆全局函数，可能后续会变得难以维护。


References：
- [装饰器@staticmethod和@classmethod有什么区别?](https://taizilongxu.gitbooks.io/stackoverflow-about-python/content/14/README.html)
- [[翻译]PYTHON中STATICMETHOD和CLASSMETHOD的差异](http://www.wklken.me/posts/2013/12/22/difference-between-staticmethod-and-classmethod-in-python.html)
