---
title: 引用计数 Android智能指针
date: 2016-12-10 11:48:56
categories:
- Android进阶
tags:
- Android智能指针
---


引用计数机制

以前学cocos写游戏的时候有接触过这个概念。

引用计数是计算机编程语言中的一种内存管理技术，是指将资源（可以是对象、内存或磁盘空间等等）的被引用次数保存起来，当被引用次数变为零时就将其释放的过程。使用引用计数技术可以实现自动资源管理的目的。

<!-- more -->

# 一、什么是引用计数

简单来讲，当我们创建一个对象的实例并在堆上申请内存时，对象的引用计数就为1，在其他对象中需要持有这个对象时，就需要把该对象的引用计数加1，需要释放一个对象时，就将该对象的引用计数减1，直至对象的引用计数为0，对象的内存会被立刻释放。

# 二、什么是智能指针？

C语言、C++语言没有自动内存回收机制，关于内存的操作的安全性依赖于程序员的自觉。程序员每次new出来的内存块都需要自己使用delete进行释放，流程复杂可能会导致忘记释放内存而造成内存泄漏。而智能指针也致力于解决这种问题，使程序员专注于指针的使用而把内存管理交给智能指针。

# 三、使用引用计数来实现智能指针

了解了引用计数，我们可以使用它来写我们的智能指针类了。

# 3.1 基础对象类

首先，我们来定义一个基础对象类Student类，这个是我们实际使用的对象，我们为Student类创建如下接口：

```cpp
class Student                                       
{
public:
    Student(){ cout<<"Student()"; }
    ~Student(){ cout<<"~Student()";}
};
```

# 3.2 辅助管理类

在创建`智能指针类`之前，我们先创建一个辅助管理类。这个类的所有成员皆为私有类型，因为它不被普通用户所使用。为了只为智能指针使用，还需要把智能指针类声明为辅助类的友元。
这个辅助类含有两个数据成员：`引用计数count`与`基础对象指针`。也即辅助类用以封装使用计数与基础对象指针。

```cpp
template <typename T>
class U_Ptr                                  
{
private:
    
    friend class SmartPtr;                      //友元类能直接操作U_Ptr类成员
    U_Ptr(T *ptr) :p(ptr), count(1) { }         //初始化1
    ~U_Ptr() { delete p; }                      //虚析构函数
    
    int count;                      // 引用计数
    T *p;                           // 实际的对象                                        
};
```

# 3.3 智能指针类

设计一个智能指针类SmartPtr类，我们这里只关注rp指针和构造函数、析构函数。
rp是基础管理类，SmartPtr类通过rp来间接增加或者减少引用计数count，当引用计数为0，则delete rp，而rp的析构函数，会去释放真正的对象。

```cpp
template <typename T>
class SmartPtr
{
public:
    SmartPtr(T *ptr) :rp(new U_Ptr(ptr)) { }      //构造函数
    SmartPtr(const SmartPtr &sp) :rp(sp.rp) { ++rp->count; } //复制构造函数
    SmartPtr& operator=(const SmartPtr& rhs) {               //赋值函数
        ++rhs.rp->count;    
        if (--rp->count == 0)    
            delete rp;
        rp = rhs.rp;
        return *this;
    }
    
    ~SmartPtr() {                           //析构函数（虚函数）       
        if (--rp->count == 0)   
            delete rp;
        else 
        cout << "还有" << rp->count << "个指针指向基础对象" << endl;
    }
    
private:
        U_Ptr *rp;  
};
```

# 四、使用示例

```cpp
int main()
{
    //定义一个基础对象类指针
    Student *pS = new Student();

    //定义三个智能指针类对象，对象都指向基础类对象pa
    //使用花括号控制三个指针指针的生命期，观察计数的变化

    {
        SmartPtr<Student> sptr1(pS);//此时计数count=1
        {
            SmartPtr<Student> sptr2(sptr1); //调用复制构造函数，此时计数为count=2
            {
                SmartPtr<Student> sptr3=sptr1; //调用赋值操作符，此时计数为conut=3
            }
            //此时count=2
        }
        //此时count=1；
    }
    //此时count=0；pS对象被delete掉

    system("pause");
    return 0;
}
```

引用计数实现的方式多种多样，上面是一种比较简单的参考实现。

# 五、Android智能指针

原始的引用计数无法解决循环引用问题。什么是循环引用？举一个简单例子：

> 系统中有两个对象A和B，在对象A的内部引用了对象B，而在对象B的内部也引用了对象A。当两个对象A和B都不再使用时，垃圾收集系统会发现无法回收这两个对象的所占据的内存的，因为系统一次只能收集一个对象，而无论系统决定要收回对象A还是要收回对象B时，都会发现这个对象被其它的对象所引用，因而就都回收不了，这样就造成了内存泄漏。

这样，就要采取另外的一种引用计数技术了，即对象的引用计数同时存在强引用和弱引用两种计数。比如Android的智能指针。

## 5.1 强指针和弱指针

Android中定义了两种智能指针类型，一种是强指针sp（strong pointer），一种是弱指针（weak pointer）。其实成为强引用和弱引用更合适一些。强指针与一般意义的智能指针概念相同，通过引用计数来记录有多少使用者在使用一个对象，如果所有使用者都放弃了对该对象的引用，则该对象将被自动销毁。

弱指针也指向一个对象，但是弱指针仅仅记录该对象的地址，不能通过弱指针来访问该对象，也就是说不能通过弱智真来调用对象的成员函数或访问对象的成员变量。要想访问弱指针所指向的对象，需首先将弱指针升级为强指针（通过wp类所提供的promote()方法）。弱指针所指向的对象是有可能在其它地方被销毁的，如果对象已经被销毁，wp的promote()方法将返回空指针，这样就能避免出现地址访问错的情况。

每一个可以被智能指针引用的对象都同时被附加了另外一个 weakref_impl类型的对象，这个对象中负责记录对象的强指针引用计数和弱指针引用计数。这个对象是Android智能指针的实现内部使用的，智能指针的使用者看不到这个对象。弱指针操作的就是这个对象，只有当强引用计数和弱引用计数都为0时，这个对象才会被销毁。

## 5.2 使用Android智能指针

假如我有一个类MyClass要使用智能指针，那么需要满足两个条件：
- （1） 这个类是RefBase的子类或间接子类；
- （2） 这个类必须定义`虚`构造函数 :  `~MyClass(){}`

### 强指针
```cpp
sp< MyClass> p_obj = new MyClass(); 
p_obj->func()
```

### 弱指针
```cpp
wp< MyClass> wp_obj = new MyClass();  
p_obj = wp_obj.promote();    // 用.而不是->  
p_obj->func();
```

相关源码：
- [http://androidxref.com/4.4_r1/xref/system/core/include/utils/RefBase.h](http://androidxref.com/4.4_r1/xref/system/core/include/utils/RefBase.h)
- [http://androidxref.com/4.4_r1/xref/system/core/include/utils/StrongPointer.h](http://androidxref.com/4.4_r1/xref/system/core/include/utils/StrongPointer.h)

如果需要了解Android智能指针的实现，可以参考老罗的一篇文章：
[http://blog.csdn.net/luoshengyang/article/details/6786239](http://blog.csdn.net/luoshengyang/article/details/6786239)
