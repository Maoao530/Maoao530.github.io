---
title: 从Python Metaclass说起...
date: 2016-04-12 23:50:27
categories: 
- Python进阶
tags:
- Python
---

在Python中，所有数据类型都可以视为对象，当然也可以自定义对象。自定义的对象数据类型就是面向对象中的类（'Class'）的概念。
类也是对象，可以把类看成是元类（'Metaclass'）创建出来的对象。

<!-- more -->

# 一、理解python中的class


在理解元类之前，你需要先掌握Python中的类。类同样也是一种`对象`。是的，没错，就是对象。只要你使用关键字class，Python解释器在执行的时候就会创建一个对象。

```python
class S(object):
    pass

s = S()
print (s) # <__main__.S object at 0x00F3B390>
print (S) # <class '__main__.S'>
```

通过以上片段我们可以知道，'s'是一个对象实例，而'S'是一个类。而通过type()我们可以查看它的类型：


```python
print (type(s)) # <class '__main__.S'>
print (type(S)) # <class 'type'>
```

可以看到s的类型是`Class S`，而S的类型是`type`，那么type是什么呢？

```python
print(type(S)       # <class 'type'>
print(type(type))   # <class 'type'>
```
为什么class的类型是type？？


# 二、理解type


动态语言和静态语言最大的不同，就是函数和类的定义，不是编译时定义的，而是运行时动态创建的。
事实上，我们说class也是对象，而这个对象是运行时动态创建的，创建class的方法就是使用type()函数。

我们可以打印出**`help(type)`** :
```python
Help on class type in module builtins:

class type(object)
 |  type(object_or_name, bases, dict)
 |  type(object) -> the object's type
 |  type(name, bases, dict) -> a new type
 |
 |  Methods defined here:
 |
 |  __call__(self, /, *args, **kwargs)
 |      Call self as a function.
 |
 |  __delattr__(self, name, /)
 |      Implement delattr(self, name).
 |
 ```

我们主要看`type(name, bases, dict)`这个方法，事实上当我们定义class的时候，就是调用此方法来创建class对象的。

```python
def say_hello(self):
    print("say hello")

name = 'Student'                    # 类名
bases = (object,)                   # 父类集合
attrs = {'say_hello' : say_hello}   # 类属性和方法

Student = type(name, bases, attrs)
s = Student()
s.say_hello()
```
等价于

```python
class Student(object):
    def say_hello(self):
        print("say hello")
s = Student()
s.say_hello()
```

# 三、理解metaclass

metaclass，直译为元类，metaclass允许你动态的控制类的创建行为。换句话说，你可以把类看成是metaclass创建出来的“实例”。  

因此我们可以**`先定义metaclass`，然后用`metaclass创建类`，最后用`类创建实例`**。  

我们先看一个简单的例子，这个metaclass可以给我们自定义的MyList增加一个add方法：

```python
# 1. metaclass是类的模板，所以必须从`type`类型派生：
class ListMetaclass(type):
    def __new__(cls, name, bases, attrs):
        attrs['add'] = lambda self, value: self.append(value)
        return type.__new__(cls, name, bases, attrs)

# 2. 定义了metaclass，接下来我们要定义类，并指定用metaclass来创建：
class MyList(list, metaclass=ListMetaclass):
    pass

```
当我们传入关键字参数metaclass时，它指示Python解释器在创建MyList时，要通过`ListMetaclass.__new__()`来创建，因此我们在`__new__()`函数里面修改类的定义，比如，加上新的方法，然后，返回修改后的定义。
`__new__()`方法接收到的参数依次是：
- 当前准备创建的类的对象
- 类的名字           
- 类继承的父类集合       
- 类的方法集合 
即：
```python
 cls ==> `<class '__main__.ListMetaclass'>`
 name ==> `MyList`
 bases ==> (<class 'list'>,) 
 attrs ==> {'__qualname__': 'MyList', '__module__': '__main__'}
```
当返回后，我们可以测试一下MyList类是否有add()方法：
```python
L = MyList()
L.add(1)
print(L)  

###### 输出 ######
[1]
```
动态修改类行为有什么意义？直接在MyList定义中写上add()方法不是更简单吗？ 正常情况下，确实应该直接写，通过metaclass修改纯属变态。  

但是，总会遇到需要通过metaclass修改类定义的。ORM就是一个典型的例子。ORM全称“Object Relational Mapping”，即对象-关系映射。

如果做过java web的同学应当知道，**`Hibernate`**就是一个对象关系映射的框架。就是把关系数据库的一行映射为一个对象，也就是**一个类对应一个表**，这样，写代码更简单，不用直接操作SQL语句。

下一节，我们会学习如何用python来构建一个ORM框架。  
  



References:
- [StackOverflow](http://stackoverflow.com/questions/100003/what-is-a-metaclass-in-python?answertab=active#tab-top)
- [廖雪峰的教程](http://www.liaoxuefeng.com/wiki/0014316089557264a6b348958f449949df42a6d3a2e542c000/0014319106919344c4ef8b1e04c48778bb45796e0335839000)









