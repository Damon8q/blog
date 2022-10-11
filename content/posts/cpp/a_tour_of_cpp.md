---
title: "C++语言导学"
date: 2022-10-11T15:00:00+08:00
lastmod: 2022-11-11T15:00:00+08:00
author: nange
draft: false
description: "C++语言导学，作为C++入门"

categories: ["programming"]
tags: ["C++"]
---

## Hello World

```c++
#include <iostream>

int main()
{
    std::cout << "Hello world!"
              << " nange"
              << " ni hao.\n";
    return 0;
}
```

`#include <iostream>` 是告诉编译器，在`iostream`库里引入标准IO输入输出设备，后续的`std::cout`就来自于此库里。

一个c++程序要启动，需要一个`main`函数，返回非0，则表示异常退出。

`<<`符号是“put to”的含义，它把第二个参数写入第一个里面，上面就是把后面一串字符串连接起来，写入标准字符输出设备里面。

`std::`表示命名空间，也就是说`cout`位于标准库命名空间`std::`内部，也可以通过引入命名空间的形式，省去`std::`，如：

```c++
#include <iostream>

using namespace std;

int main()
{
    cout << "Hello world!"
         << " nange"
         << " ni hao.\n";
    return 0;
}
```



## 基础部分

### 函数 Functions

```c++
Elem∗ next_elem(); // no argument; return a pointer to Elem (an Elem*)
void exit(int); // int argument; return nothing
double sqrt(double); // double argument; return a double

double sqrt(double d); // return the square root of d

void print(int); // takes an integer argument
void print(double); // takes a floating-point argument
void print(string); // takes a string argument
void user()
{
	print(42); // calls print(int)
	print(9.65); // calls print(double)
	print("Barcelona"); // calls print(str ing)
}
```

C++函数支持重载。

### 类型、变量、算数运算符

```c++
int inch; // 变量声明
```

一些基本类型举例：

```c++
bool 	// 布尔值，可取true和false
char	// 字符，如'a', 'z', '9'
int 	// 整数，如-273，42，1066
double 	// 双精度浮点数，如-273.16，3.14
unsigned // 非负整数，如0，1，999
```

![image-20221011164602253](/images/image-20221011164602253.png)

一个`char`变量的实际大小为机器上存放一个字符所需空间（通常是一个8位的字节），其他类型都是`char`的整数倍。类型的大小依赖于机器，可使用`sizeof`运算符获得，例如：`sizeof(char)`等于1，`sizeof(int)`通常是4。

变量的算数运算和其他语言相同，再此不再说明。

变量初始化：

```c++
double d1 = 2.3;
double d2 {2.3};
double d3 = {2.3}; // = 不是必须的

int i1 = 7.8 // i1 变成了7
int i2 {7.8} // 报错，浮点数向整数转换
```

使用等号初始化赋值时，如果类型不匹配，会发生隐式转换（为了兼容C语言的代价）。使用`{}`赋值，则不会有此问题。

使用`auto`让编译器自己推断类型：

```c++
auto b = true;   // bool
auto ch = 'x';	 // char
auto i = 123;	 // int
auto d = 1.2;	 // double
auto bb {true};	 // bool
```

### 作用域和生命周期

```c++
vector<int> vec;	// vec是全局的（一个全局整型向量）

struct Record {
    string name;	// name 是 Record的一个成员
    // ...
};

void fct(int arg)	// fct 是全局的，arg是局部的
{
    string motoo {"Who dares wins"};	// motoo是局部的
    auto p = new Record{"Hume"}; 		// p 指向一个未命名的Record
}
```

对象在作用域末尾被销毁。对于名字空间对象，销毁在整个程序的末尾。对于成员来说，销毁点依赖于它所属对象的销毁点。用`new`创建的对象一直“存活”到`delete`销毁它为止。

### 常量

C++支持两种不变性概念：

* `const`：主要用于接口说明，使得在用指针和引用将数据传递给函数时不用担心数据会被改变。编译器会检查`const`做出的承诺。`const`的值可在运行时计算。
* `constexpr`：在编译期求值。必须由编译器计算。

例如：

```c++
constexpr int dmv = 17; 		// 命名常量
int var = 17;			  	   // 不是常量
const double sqv = sqrt(var); 	// 命名常量，可在运行时求值

double sum(const vector<double>&);	// sum不会改变参数的值

vector<double> v {1.2, 3.4, 4.5};	// v不是常量
const double s1 = sum(v);			// 正确：sum(v)在运行时求值
constexpr double s2 = sum(v); 		// 错误：sum(v)不是常量表达式
```

### 指针、数组和引用

```c++
char v[6]; 		// 含有6个字符的数组

char* p;		// 指向字符的指针

char* p = &v[3];	// p 指向v的第4个元素，&即取地址符
char x = *p;		// *p是p所指的对象

```















## 参考资料

* [**A Tour of C++ (2nd Edition)**](https://github.com/Kikou1998/textbook/blob/master/A%20Tour%20of%20C%2B%2B%20(2nd%20Edition)%20(C%2B%2B%20In-Depth%20Series).pdf)

