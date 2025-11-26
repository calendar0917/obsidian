---
title: "C++ 模板及STL技术"
subtitle: "C++ 模板及STL技术"
summary: "记录 C++ 模板及STL技术的学习"
description: "记录 C++ 模板及STL技术的学习"
image: "https://raw.githubusercontent.com/calendar0917/images/master/20251013082109789.png"
date: 2025-10-14
lastmod: 2025-10-16
draft: false
toc:
 enable: true
weight: false
categories: ["笔记"]
tags: ["技术"]
---

## 模板

建立通用的模具，提高复用性

### 函数模板

作用：

- 建立一个通用函数，其函数返回值类型和形参类型可以不具体制定，用一个虚拟的类型来代表。
- 支持自动类型推导和显式指定类型

```c++
//函数模板
template<typenameT>//声明一个模板，告诉编译器后面代码中紧跟着的T不要报错，T是一个通用数据类型
void mySwap(T &a,T &b)
{
    T temp = a;
    a=b;
    b = temp;
}
myswap<int>(a,b)
```

注意：

- 自动类型推导，必须推导出一致的数据类型T，才可以使用
- 模板必须要确定出 T 的数据类型，才可以使用（不能不推导）

### 普通函数和模板函数的区别

- 普通函数调用时可以发生自动类型转换（隐式类型转换）
- 函数模板调用时，如果利用自动类型推导，不会发生隐式类型转换

调用规则：

- 优先调用普通函数
- 可通过空模板参数列表来强制调用函数模板
- 函数模板也可以发生重载
- 如果函数模板可以产生更好的匹配，优先周用函数模板

### 局限性

需要为特定的类型（如数组、自定义对象等）提供具体化模板

```c++
template<>bool myCompare (Person &p1, Person &p2)
{
    if(p1.name == p2.name){
        return true;
    }
    else
    {
        return false;
    }
}
```

### 类模板

```c++
teplate<typename T>
class ...
```

#### 与函数模板的区别

- 没有自动类型推导
- 可以有默认参数类型

#### 成员函数创建时机

在模板调用（可以确定类型）时，才创建

#### 类模板对象作参数

- 指定传入类型
- 参数模板化
  - 对象中的参数变为模板进行传递
- 整个类模板化

#### 类模板与继承

- 类继承的父类是一个类模板时，子类在声明的时候，要指定出父类中T的类型
  - 若不指定，编译器无法给子类分配内存

- 如果想灵活指定出父类中T的类型，子类也需变为类模板

#### 类模板成员的类外实现

```cpp
//构造函数类外实现
template<class T1,class T2>
Person<T1, T2>::Person(T1 name, T2 age)
{
    this->m_Name = name;
    this->m_Age = age;
}

//成员函数类外实现
template<class T1, class T2> 
void Person<T1, T2>::showPerson()
{
    
}
```

#### 类模板分文件编写

问题：

- 模板中成员函数创建时机是在调用阶段，导致分文件编写时链接不到

解决：

- 解决方式1：直接包含.cpp源文件
- 解决方式2：将声明和实现写到同一个文件中，并更改后缀名为.hpp，hpp是约定的名称，并不是强制

#### 类模板与友元

- 类内实现：直接在类内声明友元即可

- 类外实现：需要提前让编译器知道全局函数的存在

## STL

### 初识

Standard Template Library：标准模板库

- 主要组件：容器、算法、迭代器
- 容器和算法通过迭代器连接
- STL 采用模板实现

六大组件：容器、算法、迭代器、仿函数、适配器（配接器）、空间配置器

- 容器：数据结构
  - 序列式：强调值的排序
  - 关联式：如树，没有严格的物理联系
- 算法 Algorithms：实现常用算法
  - 质变：与原来不同
  - 非质变：查找、计数、遍历等
- 迭代器：胶合容器和算法
  - 类似指针，提供访问容器的接口
  - 双向、随机访问
- 仿函数：行为类似函数，可所谓算法的某种策略
- 适配器（配接器）：用于修饰容器、算法、迭代器或接口
- 空间配置器：负责空间的配置和管理

#### vector 使用

```c++
#include <vector>

vector<int> v;
v.pushback()
v.begin(); // 返回迭代器，指向容器中第一个数据
v.end(); // 指向容器最后一个数据的下一个位置
vector<int>::iterator pBegin = v.begin() // 接收
    
//第一种遍历方式:
while (pBegin != pEnd) {
    cout << *pBegin << endl;
    pBegin++;
}

//第二种遍历方式:
for (vector<int>::iterator it = v.begin(); it != v.end(); it++) {
    cout << *it << endl;
}
cout << endl;

//第三种遍历方式:
//使用STL提供标准遍历算法 头文件 algorithm
for_each(v.begin(), v.end(), MyPrint);
```

### String

string 是 C++ 风格的字符串，本质是一个类

- 类内部封装 `char*`，是管理 `char*` 的容器

#### 构造函数

```cpp
string();//创建一个空的字符串例如：stringstr;
string(const chan* s);//使用字符串s初始化
string(const string& str）;//使用一个string对象初始化另一个string对象
string(Int n, char c);//使用n个字符c初始化
```

#### 赋值

```cpp
string& operator=(const char*s);//char*类型字符串赋值给当前的字符串
string& operator=(const string &s）;//把字符串s赋给当前的字符串
string& operator=(char c);//字符赋值给当前的字符串
string& assign(const char *s);//把字符串s赋给当前的字符串
string& assign(const char *s,int n);//把字符串s的前n个字符赋给当前的字符串
string& assign(const string &s);//把字符串s赋给当前字符串
string& assign(int n,char c);//用n个字符c赋给当前字符串
```

#### 拼接

```cpp
string& operator+=(const char*str);//重载+=操作符
string& operator+=(const char c);//重载+=操作符
string& operator+=(const string& str);//重载+=操作符
string& appnd(const char *s);//把字符串s连接到当前字符串结尾
string& append(const char *s,int n);//把字符串s的前n个字符连接到当前字符串结尾
string& append(const string &s);//同operator+=(const string&str)
string&append(conststring&s，intpos，intn);/字符串s中从pos开始的n个字符连接到字符串结尾
```

#### 查找、替换

```cpp
// 查找 str 第一次出现位置，从 pos 开始查找
int find(const string& str, int pos = 0) const;
// 查找 s 第一次出现位置，从 pos 开始查找
int find(const char* s, int pos = 0) const;
// 从 pos 位置查找 s 的前 n 个字符第一次位置
int find(const char* s, int pos, int n) const;
// 查找字符 c 第一次出现位置
int find(const char c, int pos = 0) const;
// 查找 str 最后一次位置，从 pos 开始查找
int rfind(const string& str, int pos = npos) const;
// 查找 s 最后一次出现位置，从 pos 开始查找
int rfind(const char* s, int pos = npos) const;
// 从 pos 查找 s 的前 n 个字符最后一次位置
int rfind(const char* s, int pos, int n) const;
// 查找字符 c 最后一次出现位置
int rfind(const char c, int pos = 0) const;
// 替换从 pos 开始 n 个字符为字符串 str
string& replace(int pos, int n, const string& str);
// 替换从 pos 开始的 n 个字符为字符串 s
string& replace(int pos, int n, const char* s);
```

#### 比较

```cpp
int compare(const string& s)const; //与字符串s比较
int compare(const char *s) const;//与字符串s比较
```

#### 字符存取

```cpp
char& operator[] (int n);//通过[方式取字符
char& at(int n);//通过at方法获取字符
```

#### 插入、删除

```cpp
string& insert(int pos, const char*s);//插入字符串
string& insert(int pos,const string& str);//插入字符串
string& insert(int pos,int n,char c);//在指定位置插入n个字符c
string& erase(int pos,int n = npos);//删除从Pos开始的n个字符
```

#### 截取

```cpp
string substr(int pos=θ,int = npos)const;
```

### vector

据结构和数组非常相似，也称为单端数组

- 数组是静态空间，而 vector 可以动态扩展
- 动态扩展：不是在原空间之后续接新空间，而是找更大的内存空间，然后将原数据拷贝新空间，释放原空间

#### 构造

```cpp
vector<T>v;//采用模板实现类实现，默认构造函数
vector(v.begin(),v.end());//将v[begin(),end()区间中的元素拷贝给本身。
vector(n,elem);//构造函数将n个elem拷贝给本身。
vector(const vector &vec);//拷贝构造函数。
```

#### 赋值

```cpp
vector& operator=(const vector &vec);//重载等号操作符
assign(beg,end);//将[beg，end)区间中的数据拷贝赋值给本身。
assign(n,elem);//将n个elem拷贝赋值给本身。
```

#### 容量

```cpp
empty();//判断容器是否为空
capacity();//容器的容量
size();//返回容器中元素的个数
resize(int num);//重新指定容器的长度为num，若容器变长，则以默认值填充新位置。
//如果容器变短，则末尾超出容器长度的元素被删除。
resize（intnum，elem)；//重新指定容器的长度为num，若容器变长，则以elem值填充新位置。
//如果容器变短，则末尾超出容器长度的元素被删除
```

#### 插入、删除

```cpp
push_back(ele);//尾部插入元素ele
pop_back();//删除最后一个元素
insert(const_iterator pos,ele);//选代器指向位置pos插入元素ele
insert(const_iteratorpos，intcount，ele);//迭代器指向位置pos插入count个元素ele
erase(const_iterator pos);//删除选代器指向的元素
erase(const_iteratorstart，const_iteratorend);//删除迭代器从start到end之间的元素
clear();//删除容器中所有元素
```

#### 存取

```cpp
at(int idx);//返回索引lidx所指的数据
operator[];//返回索引lidx所指的数据
front();//返回容器中第一个数据元素
back();//返回容器中最后一个数据元素
```

#### 呼唤

`swap(vec)`

收缩空间：`vector<int>(v).swap(v)`

- 匿名对象自动被回收

#### 预留空间

`reserve(int len)`

- 分配内存，但内存上数据并未初始化，不可访问
- 用于减少开辟空间次数
