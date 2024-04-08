+++
title = '学会使用decltype'
discription = ""
tags = ["decltype"]
categories = ["modern-cpp"]
date = 2024-04-08T16:36:53+08:00
draft = false
slug = ""
+++

# 学会使用decltype

> Given a name or an expression, decltype tells you the name’s or the expression’s type.
>
> 把一个变量名或者表达式交给decltype，它会将对应的类型告诉你
> 
> Typically, what it tells you is exactly what you’d predict. 
>
> 一般来说，decltype会准确地给出一个你所期待的结果。
> 
> Occasionally however, it provides results that leave you scratching your head and turning to reference works or online Q&A sites for revelation.
>
> 偶尔，它也会给你一个让你抓破脑袋的结果，让你不得不去参考书或者在线问答网站上寻求一个合理的解释。

对于常规的decltype的用法，本文只给出一些示例，因为它们表现地如此自然，无需过多解释：
```cpp
// decltype(i) is const int
const int i = 0; 

// decltype(w) is const Widget&
// decltype(f) is bool(const Widget&)
bool f(const Widget& w); 

struct Point {
 // decltype(Point::x) is int
 // decltype(Point::y) is int
 int x, y; 
}; 

// decltype(w) is Widget
Widget w; 

// decltype(f(w)) is bool
if (f(w)){ 
    //do something... 
}

// simplified version of std::vector
template<typename T> 
class vector {
public:
 //…
 T& operator[](std::size_t index);
 //…
};

// decltype(v) is vector<int>
vector<int> v; 
//…
// decltype(v[0]) is int&
if (v[0] == 0) { 
    //do something…
} 
```

## C++11使用decltype声明返回值类型
在C++中有一些场合，返回值类型直接或间接依赖于函数参数的类型。

### case 1
试看一个简单的例子，我有一个模板类channel:
```cpp
template<typename T>
class channel;
```
我为channel重载输入运算符<<，为了保证channel能够支持连续输出的语法：
```cpp
channel<int> ch;
ch<<1<<2<<3;
```
我需要将<<的放回值定义为channel<int>的引用。
有时，我们的函数签名很复杂，例如：
```cpp
template<typename Type>
channel<typename std::decay(Type)::type>& 
    operator<<(channel<typename std::decay(Type)::type>ch, Type&& in);
```
这样的签名就十分的冗长，为了简便，我们可以使用decltype进行精简：
```cpp
// C++11 version
template<typename Type>
auto& operator<<(channel<typename std::decay(Type)::type>ch, Type&& in) -> decltype(ch);

// C++14 version
template<typename Type>
auto& operator<<(channel<typename std::decay(Type)::type>ch, Type&& in);
```
如上是一个最简单的使用decltype+auto精简函数签名的方式。

### case 2
让我来看一个稍微复杂点的例子。

当我们想要编写一个模板函数，它接收一个顺序型容器container，和一个index参数，在验证用户权限后，返回container中下标为index的元素的引用。（类似[]运算符的行为）

因为container有很多种，它的元素类型可能有很多种，所以在C++11之前，这种函数并不好书写，最大的问题在于返回值类型不好表示。不过在C++11这个问题迎刃而解：
```cpp
// example code 1
//C++11版本，不够完善
template<typename Container, typename Index>
auto AuthAndAcess(Container& c, Index i) -> decltype(c[i]){
    AuthenticateUser();
    return c[i];
}
```
以上的函数通过decltype+尾返回值手段实现使用函数参数类型来表示返回值。这里auto只是一个占位符，表示函数采用尾返回值，decltype(c[i])可以成功将返回值类型描述为c[i]相同的类型，即T&。

在C++14中，精简了这种表达，程序员可以在语法上省略掉尾返回值，实现的**略有差别**的效果。
```cpp
// example code 2
//C++14版本，不够完善
template<typename Container, typename Index>
auto AuthAndAcess(Container& c, Index i){
    AuthenticateUser();
    return c[i];
}
```
结合“auto的类型推导”一文的内容，auto作为返回值时采用template的类型推导准则。在这个例子中返回值推导类似于如下的语句
```cpp
auto no-named-obj = c[i];
```
虽然c[i]是一个引用，但是由于auto是“非引用非指针”的情况，因此在类型推导时会无视c[i]的引用、const等属性，最终auto被推导为int。

那么这段代码在发生如下的调用时，就会报错：
```cpp
AuthAndAcess(c,i) = 10;
```
因为我们尝试将一个值赋予给rvalue。

为了解决C++版本遇到的问题，我们给出如下的修订：
```cpp
// example code 3
//C++14版本，不够完善
template<typename Container, typename Index>
auto& AuthAndAcess(Container& c, Index i){
    AuthenticateUser();
    return c[i];
}
```
按照类型推导原则，auto&能够基本满足我们的需求。但是这里又有另一个问题（虽然不常见）：如果container实际返回的是一个object而非object&，按照类型推导原则，auto&会得到一个object&，这看起来又在另一个角度触发了example code 2的错误...

让我们在看另一种处理方式：
```cpp
// example code 4
//C++14版本，不够完善
template<typename Container, typename Index>
decltype(auto) AuthAndAcess(Container& c, Index i){
    AuthenticateUser();
    return c[i];
}
```
虽然这个版本的语法看似怪异，为auto进行decltype？但是实际上是可以理解的：auto表示返回值类型需要被推导，decltype表示类型推导采用decltype的方式而非template方式（auto作为返回值时默认采用template方式进行类型推导）

> auto specifies that the type is to be deduced, and decltype says that decltype rules should be used during the deduction.

这样一来，无论c[i]返回T&还是T，我们的代码都能正确地推导返回值类型了。

decltype(auto)不仅可以用于函数返回值，它同样可以应用于变量初始化，只要你想要初始化过程中使用decltype的类型推导准则：
```cpp
Widget w;
const Widget& cw = w;

// auto type deduction: myWidget1's type is Widget
auto myWidget1 = cw; 

// decltype type deduction:myWidget2's type is const Widget&
decltype(auto) myWidget2 = cw; 
```

直到example code 4我仍然说AuthAndAccess是可以继续完善的，其实这似乎有点吹毛求疵，但是却仍然值得讨论。

请问，如果我想把一个rvalue的container交给AuthAndAccess，代码能够正常运行吗？
```cpp
auto item = AuthAndAccess(CreateContainer(), 5);
```

显然，左值引用无法接受一个rvalue，为了达到我们的目的，我们有两条路可以走：
1. 保留原有版本的AuthAndAccess，并且重载一个接收rvalue的版本。
2. 使用万能引用和完美转发同时处理lvalue和rvalue
   
我相信你和我一样，一定会选择第二种方式！

让我们来看最终版本的代码：
```cpp
// example code 5
// C++14 version
template<typename Container, typename Index>
decltype(auto) AuthAndAccess(Container&& c, Index i){
    AuthenticateUser();
    return std::forward<Container>(c)[i];
}

// C++11 version
template<typename Container, typename Index>
auto AuthAndAccess(Container&& c, Index i) ->decltype(c[i]){
    AuthenticateUser();
    return std::forward<Container>(c)[i];
}
```
值得一提的是，在这个版本中我们不再关注Container的具体类型，同等地我们也不知道Container的Index是什么对象。我们按照直觉对Index采用了pass-by-value，这可能导致一些额外的拷贝，但是当我们学习STL，我们会发现vector等容器的[]操作符在重载时也是这样写的，我们可以认为这种直觉是正当的...

**Todo**：

其实这里我还有个不懂的地方：如果Container接收一个rvalue作为参数，那么rvalue应该是即将被摧毁的，我们返回rvalue中的元素的reference，这不应该很怪吗？

## decltype区别对待name和expr
>Applying decltype to a name yields the declared type for that name.
>
>将decltype应用与变量名，它总是产生变量名对应的具体类型
>
>For lvalue expressions more complicated than names, however, decltype ensures that the type reported is always an lvalue reference. 
>
>对于一个比变量名更加复杂的左值表达式，decltype总是返回一个左值引用

```cpp
auto x = 1;
decltype(auto) y = x; // y is int
decltype(auto) z = (x); // x is int&
```

## 需要记住的事
>decltype almost always yields the type of a variable or expression without any modifications.
>
>For lvalue expressions of type T other than names, decltype always reports a type of T&.
>
>C++14 supports decltype(auto), which, like auto, deduces a type from its initializer, but it performs the type deduction using the decltype rules.