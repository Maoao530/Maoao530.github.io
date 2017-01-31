---
title: python 装饰器decorator
date: 2016-05-17 23:08:59
categories:
- Python基础
tags:
- python
---

2分钟告诉你什么是python装饰器。

<!-- more -->

# 一、什么是装饰器

Python的装饰器的英文名叫`Decorator`，这个和设计模式中的`Decorator Pattern`是两种东西。
Python中的装饰器主要用于对已经有的模块做一些“修饰工作”。比如说，我们经常需要在函数调用前后自动打印日志，又不想要改变原有函数的模块，这个时候，我们便可以写一个打印log的装饰器。

# 二、Hello World

```python
# 定义log装饰器
def log(func):
    def wrapper(*args, **kw):
        print('start %s()' % func.__name__)
        return func(*args, **kw)
        print('end %s()' % func.__name__)
    return wrapper

@log
def foo():
    print('foo function ...')

foo()

```
当运行代码，你会发现有如下输出：
```python
C:\Users\maoao\Desktop\Project>python Test.py
start foo()
foo function ...
end foo()
```
有木有发现很神奇？

# 三、Decorator 的本质

对于Python的这个@注解语法糖来说，当你在用某个@decorator来修饰某个函数func时，如下所示:
```python
@log
def foo():
    print('foo function ...')
```
相当于执行了语句：
```python
foo = log(foo)
```
其实就是把一个函数当参数传到另一个函数中，然后再把decorator这个函数的返回值赋值回了原来的func。

不信我们可以做如下验证：
```python
def fuck(fn):
    print ("fuck %s ! " % fn.__name__.upper())
 
@fuck
def gfw():
    pass
```
还没有执行gfw就输出我们每个人的心声了有木有。

知道这点本质，当你看到有**多个decorator**：
```python
@decorator_one
@decorator_two
def func():
    pass
```
相当于：
```python
func = decorator_one(decorator_two(func))
```
**带参数的decorator**：
```python
@decorator(arg1, arg2)
def func():
    pass
```
相当于：
```python
func = decorator(arg1,arg2)(func)
```
这意味着decorator(arg1, arg2)这个函数需要返回一个“真正的decorator”。

# 四、带参数的装饰器示例

假设我们可以编写一个带参数的装饰器：
```python
def log(text):
    def decorator(func):
        def wrapper(*args, **kw):
            print('%s, start %s()' % (text, func.__name__))
            func(*args, **kw)
            print('%s, end %s()' % (text, func.__name__))
        wrapper.__name__ = func.__name__
        return wrapper
    return decorator

@log('SHUN_TAG')
def foo():
    print('foo function ...')

foo()
print foo.__name__

# 输出：

# C:\Users\maoao\Desktop\Project>python Test.py
# SHUN_TAG, start foo()
# foo function ...
# SHUN_TAG, end foo()
# foo
```
@@log('SHUN_TAG')实际上会执行如下语句：
`foo = log('SHUN_TAG')(foo)`
然后**最终会返回wrapper函数给foo**，另外要记得`wrapper.__name__ = func.__name__`，防止有些函数签名的代码回报错。

也可以用@functools.wraps(func)来代替上述写法：
```python
import functools

def log(text):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kw):
            print('%s, start %s()' % (text, func.__name__))
            func(*args, **kw)
            print('%s, end %s()' % (text, func.__name__))
        return wrapper
    return decorator
```
其实也没有什么复杂的东西。

# 五、class式的 Decorator

最后再来看下decorator的class方式，还是看个示例：
```python
class myDecorator(object):
 
    def __init__(self, fn):
        print "inside myDecorator.__init__()"
        self.fn = fn
 
    def __call__(self):
        self.fn()
        print "inside myDecorator.__call__()"
 
@myDecorator
def aFunction():
    print "inside aFunction()"
 
print "Finished decorating aFunction()"
 
aFunction()
 
# 输出：
# inside myDecorator.__init__()
# Finished decorating aFunction()
# inside aFunction()
# inside myDecorator.__call__()
```
用类的方式声明一个decorator。我们可以看到这个类中有两个成员：
- 一个是__init__()，这个方法是在我们给某个函数decorator时被调用，所以，需要有一个fn的参数，也就是被decorator的函数。
- 一个是__call__()，这个方法是在我们调用被decorator函数时被调用的。
上面输出可以看到整个程序的执行顺序。

# 六、总结

decorator可以增强函数的功能，定义起来虽然有点复杂，但使用起来非常灵活和方便。

References:
- [Python修饰器的函数式编程](http://coolshell.cn/articles/11265.html)
- [廖雪峰的教程](http://www.liaoxuefeng.com/wiki/0014316089557264a6b348958f449949df42a6d3a2e542c000/0014318435599930270c0381a3b44db991cd6d858064ac0000)

