---
layout:     post
title:      malloc&frasl;free与new&frasl;delete的区别
subtitle:   阅读《C++内存管理技术内幕》笔记
date:       2018-12-19
author:     JeeShao
header-img: img/tag-bg.jpg
catalog: true
tags:
    - C++
---

# `malloc/free`与`new/delete`的区别
我们知道在**C++**中，我们经常需要对内存进行手动分配和回收，而且**C++**是兼容**C**的，我们可以在**C++**中编写**C**的程序。而`malloc/free`是**C**中已有的函数，我们为什么还要用`new/delete`呢？

### 主要区别
`malloc/free`是**C++/C**语言的标准库函数，`new/delete`是**C++**的**运算符**。它们都可用于申请动态内存和释放内存。

对于非内部数据类型的对象而言，只用`malloc/free`无法满足动态对象的要求。对象在创建的同时要自动执行构造函数，对象在消亡之前要自动执行析构函数。由于`malloc/free`是库函数而不是运算符，不在编译器控制权限之内，不能够把执行构造函数和析构函数的任务强加于`malloc/free`。

因此**C++**语言需要一个能完成动态内存分配和初始化工作的运算符`new`，以及一个能够完成清理和释放内存工作的运算符`delete`。**`new/delete`不是库函数**。下面通过示例演示一下它们如何实现对象的动态内存管理：
```C++
class Obj
{
	public:
		Obj(void){cout<<"Initialization"<<endl;}
		~Obj(void){cout<<"Destroy"<<endl;}
		void Initialize(void){cout<<"Initialalization"<<endl;}
		void Destroy(void){cout<<"Destroy"<<endl;}
};

void UseMallocFree(void)
{
	Obj *a = (Obj *)malloc(sizeof(Obj));//申请动态内存
	a->Initialize();//初始化
	a->Destroy();//清理
	free(a);//释放动态内存
}

void UseNewDelete(void)
{
	Obj *a = new Obj();//申请动态内存并初始化
	delete a;//清理并释放内存
}
```
类**Obj**的函数**Initialize**模拟了构造函数的功能，函数**Destroy**模拟了析构函数的功能。函数**UseMallocFree**中，由于`malloc/free`不能执行构造函数和析构函数，必须调用成员函数**Initialize**和**Destroy**来完成初始化和清理工作。函数**UseNewDelete**则简单得多。

既然用`free/delete`的功能完全覆盖了`malloc/free`，为什么不淘汰`free/delete`呢？这是因为**C++**程序中经常会用到**C**程序，而**C**程序只能用`malloc/free`管理动态内存。

**PS:**如果用`free`释放`new`创建的对象，那么该对象因无法执行析构函数而可能导致程序出错；如果用`delete`释放`malloc`申请的动态内存，结果也会导致程序出错，且你这样做程序的可读性将变得很差。所以，**`new/delete`和`malloc/free`必须配对使用，生死不离。**

### malloc/free的使用要点
**函数malloc的原型如下：**
`void * malloc(size_t size);`
用malloc申请一块长度为length的整数类型内存：
`int *p = (int *) malloc(sizeof(int) * length);`
我们应当把注意力集中在两个要素上：“类型转换”和"“sizeof”。

malloc返回值得类型是void \*，所以在调用malloc时要显式地进行类型转换，将void \*转换成所需要的指针类型。

malloc函数本身并不识别要申请的内存是什么类型，它只关心内存的总字节数。我们通常记不住int,float等数据类型的变量的确切字节数，比如 int类型在16位系统下是2字节，在32位系统下是4字节，因此我们常在malloc中使用**sizeof**运算符。

**函数free的原型如下：**
`void free(void * memblock);`
为什么free函数不像malloc函数那样复杂呢？这是因为指针*p*的类型以及它所指的内存的容量事先都知道，`free(p)`能正确地释放内存。如果*p*是NULL指针，那么free对*p*无论操作多少次都不会出问题。而如果*p*不是NULL指针，那么对free对*p*连续操作两次就会导致程序运行错误。

### new/delete的使用要点
运算符**new**用起来比函数malloc简单得多，如：
```C++
int *p1 = (int *)malloc(sizeof(int) * length);
int *p2 = new int[length];
```
这是因为new内置了sizeof、类型转换和类型安全检查功能。**对于非内部数据类型的对象而言，new在创建动态对象的同时完成了初始化工作。**如果对象有多个构造函数，那么new的语句也可以有多种形式，例如：
```C++
class Obj
{
	public:
		Obj(void);//无参构造函数
		Obj(int x);//有参构造函数
		…
}

void Test(void)
{
	Obj *a = new Obj();//调用无参构造函数
	Obj *b = new Obj(1);//调用有参构造函数
	…
	delete a;
	delete b;
}
```
如果用new创建对象数组，那么只能使用对象的无参构造函数。比如：
```
Obj *objects = new Obj[100];//创建100个动态Obj对象
```
而不能写成：
```
Obj *objects = new Obj[100](1);
```
最后，在使用**delete**释放对象**数组**时，记住不要丢了符号`[]`。如：
```
delete []objects;//正确
delete objects;//错误
```