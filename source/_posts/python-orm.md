---
title: 如何构建一个orm框架？
date: 2016-04-25 23:54:48
categories: 
- Python进阶
tags:
- Python
---

ORM全称“Object Relational Mapping”，即对象-关系映射，就是把关系数据库的一行映射为一个对象，也就是一个类对应一个表，这样，写代码更简单，不用直接操作SQL语句。

<!-- more -->

# 1. 开始之前

开始之前，请先掌握python metaclass的知识，请参考我的文章：[从Python Metaclass说起](https://maoao530.github.io/2016/04/12/python-metaclass/)  

为什么要使用metaclass？因为要编写一个ORM框架，所有的类都只能动态定义，只有使用者才能根据表的结构定义出对应的类来。

掌握了元类的知识后，我们来尝试编写一个ORM框架。首先，假设我们有一个User表，那么我们可能会写出如下代码：
```python
class User(Model):
    # 定义类的属性到列的映射：
    id = IntegerField('id')
    name = StringField('username')
    email = StringField('email')
    password = StringField('password')

# 创建一个实例：
u = User(id=12345, name='Michael', email='test@orm.org', password='my-pwd')
# 保存到数据库：
u.save()
```
怎么样？我们可以不用直接去操作SQL了，看上去是不是非常简单呢？

# 2. 设计Field类

接下来我们要定义Field类，它负责保存数据库表的**字段名**和**字段类型**：

```python
class Field(object):
    def __init__(self, name, column_type):
        self.name = name
        self.column_type = column_type
    def __str__(self):
        return '<%s:%s>' % (self.__class__.__name__, self.name)

class StringField(Field):
    def __init__(self, name):
        super(StringField, self).__init__(name, 'varchar(100)')

class IntegerField(Field):
    def __init__(self, name):
        super(IntegerField, self).__init__(name, 'bigint')
```

# 3. 设计Model类

Model类是数据库表类的基类。

在Model类中，就可以定义各种操作数据库的方法，比如save()，delete()，find()，update等等：

```python
# 创建Model类时，指定通过ModelMetaclass来创建
class Model(dict, metaclass=ModelMetaclass):

    def __init__(self, **kw):
        print("Model __init__ : ", kw )
        super(Model, self).__init__(**kw)

    def __getattr__(self, key):
        print("__getattr__: %s > %s" % (key,self[key]))
        try:
            return self[key]
        except KeyError:
            raise AttributeError(r"'Model' object has no attribute '%s'" % key)

    def __setattr__(self, key, value):
        print("__setattr__ : ", key)
        self[key] = value

    #只是模拟打印出sql语句
    def save(self):
        fields = []
        params = []
        args = []
        for k, v in self.__mappings__.items():
            fields.append(v.name)  # id,username,email,password 字段名
            params.append('?')     # ????
            args.append(getattr(self, k, None))
        sql = 'insert into %s (%s) values (%s)' % (self.__table__, ','.join(fields), ','.join(params))
        print('SQL: %s' % sql)
        print('ARGS: %s' % str(args))
```

# 4. 设计ModelMetaclass元类

最后就是mnetaclass元类的编写了。

使用Model中定义的metaclass的ModelMetaclass来创建User类，也就是说，**metaclass可以隐式地继承到子类**:

```python
class ModelMetaclass(type):
    # 准备创建的类的对象，类名，父类集合，类方法集合
    def __new__(cls, name, bases, attrs):
        print('Found model: %s' % name)
        if name=='Model':
            return type.__new__(cls, name, bases, attrs)
        
        mappings = dict()
        for k, v in attrs.items():
            if isinstance(v, Field):
                print('Found mapping: %s ==> %s' % (k, v))
                mappings[k] = v
        for k in mappings.keys():
            attrs.pop(k)
        attrs['__mappings__'] = mappings    # 保存属性和列的映射关系
        attrs['__table__'] = name           # 假设表名和类名一致
        return type.__new__(cls, name, bases, attrs)
```

至此，核心代码基本上写完了，怎么样？看起来也不是很难。我们来梳理一下：

1. 定义一个class User(Model)类
2. Python解释器通过父类Model的metaclass找到ModelMetaclass类，通过它来创建User
3. ModelMetaclass保存User类的一些信息，比如表名，字段等
4. 当我们调用save()方法时，会去用到第三步骤保存的信息，构造出SQL语句，将数据保存到数据库中

我们编写代码：
```python
u = User(id=12345, name='Michael', email='test@orm.org', password='my-pwd')
u.save()
```
输出：
```python
Found model: User
Found mapping: email ==> <StringField:email>
Found mapping: password ==> <StringField:password>
Found mapping: id ==> <IntegerField:uid>
Found mapping: name ==> <StringField:username>
SQL: insert into User (password,email,username,id) values (?,?,?,?)
ARGS: ['my-pwd', 'test@orm.org', 'Michael', 12345]
```
可以看到，save()方法打印出了SQL语句和参数列表，我们可以根据自己的需求，来将此信息存储到数据库中。


References:
- [廖雪峰的教程](http://www.liaoxuefeng.com/wiki/0014316089557264a6b348958f449949df42a6d3a2e542c000/0014319106919344c4ef8b1e04c48778bb45796e0335839000) 
