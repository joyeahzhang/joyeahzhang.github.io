+++
title = 'C++模板中的类型推导'
discription = ""
tags = ["template","type deduction"]
categories = ["modern-cpp"]
date = 2024-04-04T11:32:20+08:00
draft = false
slug = ""
+++

# 理解模板中的类型推导

> When users of a complex system are ignorant of how it works, yet happy with what it does, that says a lot about the design of the system. By this measure, template typededuction in C++ is a tremendous success.
> 当用户无需知道一个复杂的系统如何工作，却能够愉快地使用它时，那就证明了这个系统具有很好的设计。从这个角度来看，C++模板推导无疑取得了巨大的成功。

在C++中，一个函数模板的基本形式应该是类似这样的：
```cpp
template<typename T>
RetType Func(ParamType param);
```

这里的`ParamType`似乎有点突兀，实际上它是指给`T`添加上一些常量修饰const、引用后缀、可变修饰volatile等修饰后的参数类型。例如如下示例中`ParamType`就是指`cosnt T&`。
```cpp
template<typename T>
RetType Func(cosnt T& param);
```

**C++函数模板类型推导实际上就是对`ParamType`和`T`进行类型推导**。那么由谁来出发，最终推导得到`ParamType`和`T`呢？这就离不开模板函数所接受到的实际参数`expr`了。
```cpp
auto ret = Func(expr);
```
当表达式`expr`作为参数被传递给模板函数时（函数调用语句），模板类型推导就开始了（模板实例化）。

在明确如上的基本概念后，我们来看C++14及以前所出现的各种函数模板的类型推导。

## case 1:当ParamType是非万能引用的引用或指针时

首先我们来明确什么是万能引用。万能引用是一种能够接收左值引用和右值引用的“看起来无所不能”的引用形式，它总是和模板一起出现，基本形式是：
```cpp
template<typename T>
RetType func(T&& param);
```
其中`T&&`就是万能引用。具体关于万能引用的用法和细节，留在以后再讲，本文只需要说明“万有引用长这样”。本小节讨论的是不同于这样的引用和指针。

当`ParamType`是**非万能引用的指针和引用**，C++的处理方式是简单的，它遵循以下两个步骤：
> 1. 如果`expr`是一个引用，那么忽视其引用
> 2. 将忽视引用的`expr`和`ParamType`进行模式匹配，进而确定`T`

这看起来似乎晦涩，实际却很简单，让我们来实际看几个例子。

```cpp
int a = 1;
const b = 1;
int& c = a;
const int& d = 1;
```
首先看最简单的例子：

```cpp
template<typename T>
void func1(T& param);

// expr: int，不是引用，无需第一步处理
// ParamType: T&，与int进行模式匹配，推导出ParamType为int&，T为int
func1(a);

// expr: const int，不是引用，无需第一步处理
// ParamType: T&，与const int进行模式匹配，推导出ParamType为const int&，T为const int
func1(b);

// expr: int&，是引用，第一步处理，忽视引用
// ParamType: T&，与int进行模式匹配，推导出ParamType为int&，T为int
func1(c);

// expr: const int&，是引用，第一步处理，忽视引用
// ParamType: T&，与cosnt int进行模式匹配，推导出ParamType为const int&，T为const int
func1(d);
```
在这个例子中，我们还能意识到一个重要的特点：当我们将一个`const object`类型的`expr`传递给引用类型的`ParamType`时，无需担心`ParamType`是一个引用，可能导致object被修改。**C++模板推导会保留`expr`的`const`特性**。

进一步感受模式匹配（这个例子将会直观地体现“模式匹配`ParamType`以确定`T`"这个说法）：

```cpp
template<typename T>
void func2(const T& param);

// expr: int，不是引用，无需第一步处理
// ParamType: const T&，与int进行模式匹配，推导出ParamType为const int&，T为int
func2(a);

// expr: const int，不是引用，无需第一步处理
// ParamType: const T&，与const int进行模式匹配，推导出ParamType为const int&，T为int
func2(b);

// expr: int&，是引用，第一步处理，忽视引用
// ParamType: const T&，与int进行模式匹配，推导出ParamType为const int&，T为int
func2(c);

// expr: const int&，是引用，第一步处理，忽视引用
// ParamType: const T&，与cosnt int进行模式匹配，推导出ParamType为const int&，T为int
func2(d);
```
注意`func1(d)`和`func2(d)`最后得到的`T`的区别。

至于`ParamType`是指针类型，情况基本是一样的。
```cpp
template<typename T>
RetType Func(T* param);
```
它遵循类似引用的处理方案：
> 1. 如果`expr`是一个指针，那么忽视其指针性质
> 2. 将忽视引用的`expr`和`ParamType`进行模式匹配，进而确定`T`

```cpp
int a = 1;
cosnt int* b = &a;

template<typename T>
void func3(T* param);

// expr: int，无需处理
// ParamType: T*，与int进行模式匹配，推导出ParamType为int*，T为int
func3(a);

// expr: int*，第一步处理，忽视指针
// ParamType: T*，与int进行模式匹配，推导出ParamType为int*，T为int
func3(&a);
// expr: const int*，第一步处理，忽视指针
// ParamType: T*，与const int进行模式匹配，推导出ParamType为const int*，T为const int
func3(b);
```

## case 2:当ParamType是一个万能引用时
万能引用的基本形式已经在case1中提到了，其基本形式是：
```cpp
template<typename T>
RetType Func(T&& param);
```
对于这种情况的处理准则是：
> 如果`expr`是一个左值，那么`T`和`ParamType`都会被推导为左值引用。这个一个不容质疑的真理，它有两点需要注意：1.这是唯一一种T会被推导成为引用的场景。2.即使`expr`是一个右值引用，`ParamType`也会被推导为左值引用。
> 如果`expr`是一个右值，那么一切就按照case 1来办吧！

请看以下的示例：

```cpp
template<typename T>
void func(T&& param);

int x = 1;
const int cx = x;
const int& crx = x;

// expr: int，左值；T和ParamType都被推导了int&
func(x);
// expr: const int，左值；T和ParamType都被推导了const int&
func(x);
// expr: const int&，左值；T和ParamType都被推导了const int&
func(x);
// expr: int&&，右值；
// 第一步无视引用
// 第二步模式匹配：expr: int，ParamType推导为int&&，T推导为int
func(1);
```

## case 3: ParamType既非引用也非指针

当`ParamType`既非引用也非指针，那我就是在讨论`pass-by-value`的事了。

```cpp
template<typename T>
RetType Func(T param);
```

`pass-by-value`的语义是函数传参时会将形式参数拷贝出一份新的对象作为实际参数传递给函数。这种特性，使得“`ParamType`既非引用也非指针”的类型推导很简单，它遵守如下的方式：

> 1. 如果`expr`是一个引用，那么忽略它的引用（因为实参会拷贝出新的对象，所以不care形参是否是引用）
> 2. 进一步，忽略掉`expr`的`const`，`volatile`等特性（理由同上）

看一个简单的例子：

```cpp
template<typename T>
void func(T&& param);

int x = 1;
const int cx = x;
const int& crx = x;

// expr: int，T和ParamType都被推导了int
func(x);
// expr: const int，T和ParamType都被推导了int
func(x);
// expr: const int&，T和ParamType都被推导了int
func(x);
// expr: int&&，T和ParamType都被推导了int
func(1);
```
emmm，就是这样...

一个有点意思的，考察对指针的理解的例子是：
```cpp
template<typename T>
void f(T param); 
const char* const ptr = // ptr is const pointer to const object
 "Fun with pointers";
f(ptr); // 类型推导时，应该去掉 const char* const中的哪些const呢？
```

## expr是C-style数组时

虽然现在鼓励使用`std::array`，但是我们还是得稍微考虑下`c-style`数组作为模板推导的`expr`的情况，因为它略显特殊。

学习过`C/C++`的同学都知道，`c-style`的数组一般被按照指针来处理，这个特性被称为“数组退化指针（`array-to-pointer decay rule`）”准则。

`array-to-pointer decay rule` 在模板类型推导中也是发挥着“有限的”作用。

1. 当`ParamType`按值传递时，`array-to-pointer`发挥作用
2. 当`ParamType`按引用传递时，`array-to-pointer`失效，数组将以数组的语义（起始地址+长度）传递给函数

举例说明：
```cpp
const char name[] = "J. P. Briggs"; 

template<typename T>
void f1(T param); // 按值传递

void f2(T& param); // 按引用传递

f1(name); //ParamType推导为const char*，T为const char*
f2(name); //ParamType推导为const char(&)[13], T为const char[13]

```

通过以上特性，我们可以通过模板推导写出在编译期确定`c-style`数组长度的代码：

```cpp
template<typename T,std::size_t N>
constexpr std::size_t ArraySize(const T(&)[N]) noexcept{
    return N;
}
```
## expr是函数时

这种情况的处理方式和数组类似，但是实际影响却不如数组。如果ParamType是按值传递，那么推导为函数指针；如果ParamType是按引用传递，那么就推导为函数引用...（实在不知道这两者有多少差异）


## TODO
C++17之后，auto的使用更加广泛了，似乎effective modern cpp中讲的这些不够用了，需要查阅更多资料...