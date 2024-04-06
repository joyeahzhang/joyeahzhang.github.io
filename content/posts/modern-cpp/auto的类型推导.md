+++
title = 'Auto的类型推导'
discription = ""
tags = ["auto" "type deduction"]
categories = ["modern-cpp"]
date = 2024-04-06T11:18:15+08:00
draft = false
slug = ""
+++

# auto的类型推导
在modern-cpp中，建议在声明变量时使用auto而非固定类型（fix types），这强调了理解auto类型推导的重要性。

一个经典的cpp笑话也佐证了这一点:

>你认为modern cpp将会是怎么样的？
>
>```cpp
> auto auto(auto){
>  auto auto{};
>  auto(auto);
>  return auto;    
>}
>```

## 熟悉template的类型推导

在学习auto关键字的类型推导过程前，请仔细回顾《C++模板中的类型推导》，这是因为auto的类型推导和模板类型推导大同小异。

在《C++模板中的类型推导》中我们仔细讲述了三种不同ParamType的处理方式：

> ParamType是非万能引用的引用或指针
>
> ParamType是万能引用
>
> ParamType非引用和指针
>
> 进一步讨论了expr是c-sytle数组和function的推导方式

auto的推导也大致分为这样的几种情况。

## auto的类型推导
### 与template类似的推导
为了便于理解，我们可以认为auto类型推导和模板类型推导有这样的对应关系：

auto就相当于模板中的T，声明符specifier就相当于ParamType:
```cpp
// auto -> T
// secifier即const auto & -> ParamType
// expr即27
const auto & x = 27;
```

如果和函数模板对应起来，那么上述语句的推导过程与以下的语句无异：

```cpp
template <typename T>
void func(const T& param);
func(27);
```

按照模板的类型推导规则，const T&是非万能引用的引用，采用两步走：1. 无视expr的引用；2. 将expr的类型与ParamType进行模式匹配以确定T；

这里expr是int&&，step1无视引用后为int；step2模式匹配后得到ParamType为const int&，得到T为int

回到最初的const auto& x = 27;我们按照一样的处理手段，可以得到specifier是const int &; auto是int。

其余auto的推导情况都有对应的template推导可以参考：
```cpp
auto x = 27;
// 等价于非引用，非指针的ParamType
template<typename T>
void func_for_x(T param);
func_for_x(27);

const auto rx = &x;
// 等价于非引用，非指针的ParamType
template<typename T>
void func_for_rx(const T& param);
func_for_rx(x);

const auto&& urx = 27;
// 等价于非引用，非指针的ParamType
template<typename T>
void func_for_urx(const T&& param);
func_for_urx(27);

// 与template decution相同
// 遵循array-to-pointer decay rule
const char hello[] = "Hello world";
auto hello1 = hello; // auto 是 const char*
auto& hello2 = hello; // auto 是 const char(&)[12]

void someFunc(int, double); 
 // someFunc`s type is void(int, double)
auto func1 = someFunc; // func1's type is void (*)(int, double)
auto& func2 = someFunc; // func2's type is void (&)(int, double)
```

### initializer_list带来的特殊情况
auto类型推导与模板类型推导的不同之处，皆由initializer_list引起。

modern-cpp支持以下四种初始化方式，不同的初始化方式在进行auto推导时使用的expr是不同的：
```cpp
auto x1 = 27; // expr type is int, value is 27
auto x2(27); // 同上
auto x3 = {27}; // expr type is initializer_list<int>, value is {27}
auto x4{27}; // 同上
```

由于初始化列表initializer_list<T>本身是一个模板，因此在使用initializer_list<T>对auto进行推导时，实际是两个推导子过程：
1. 对initializer_list<T>进行模板类型推导以确定T
2. 根据已经确定的initializer_list对auto进行类型推导

这个过程中，第一步经常会由于程序员的疏忽而失败，典型的错误是：
```cpp
auto arr = {1, 2, 3.0};
```
由于initializer_list中的元素不具备相同的类型，所以initializer_list<T>模板类型推导会失败，进而导致auto的类型推导失败。

**值得一提的是，模板不支持对类型为initializer_list的expr进行类型推导**，这是auto与template的一个重大区别！
```cpp
template<typename T>
void func(const T& param);
func({1,2,3}); // 编译错误:expr的类型是initializer_list<int>类型
```

## 使用auto作为返回值和函数参数
C++14以来，auto被允许作为函数返回值，也可以作为lambda function的参数类型。值得注意的是，在这些情况下，auto的类型推导遵循与template类型推导相同的规则。那么也就意味着，不能将initializer_list类型的expr直接作为返回值和参数提供给auto。

```cpp
auto createInitList()
{
 return { 1, 2, 3 }; // error: can't deduce type
}
```

```cpp
std::vector<int> v;
…
auto resetV =[&v](const auto& newValue) { v = newValue; }; // C++14
…
resetV({ 1, 2, 3 }); // error! can't deduce type for { 1, 2, 3 }
```

## TODO
C++14之后auto更加应用广泛，需要补充更新的使用方式...