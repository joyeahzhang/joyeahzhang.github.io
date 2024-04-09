+++
title = '乐意去使用auto'
discription = ""
tags = ["auto"]
categories = ["modern-cpp"]
date = 2024-04-09T10:27:32+08:00
draft = false
slug = ""
+++

# 乐意去使用auto

在C++11引入auto之前，我们常常面临着C++的一些难题，比如：

## case 1: 迫使你去初始化变量
```cpp
int x;
```
糟糕，忘记初始化变量了，x的值是未定义的...
还好，auto不会有这样的问题：
```cpp
auto x = 1;
```
在使用auto声明变量时，我们必须给出变量的初始值，以进行auto类型推导...

## case 2: 使你声明复杂类型变量更加简单
```cpp
template<typename Iterator>
void dwim(Iterator begin, Iterator end){
    while (begin != end) {
        typename std::iterator_traits<It>::value_type value = *begin;
        // move the iterator... 
    }
}
```
在不使用auto的情况下，如果我们想要对迭代器解引用，这往往需要一串冗杂的代码...
还好我们有auto:
```cpp
template<typename Iterator>
void dwim(Iterator begin, Iterator end){
    while (begin != end) {
        auto value = *begin;
        // move the iterator... 
    }
}
```

## case 3：让你能够声明只有编译器知道类型的变量
如果我们不使用auto，对于像closure这样只有编译期才知道的类型，我们无法直接声明对应类型的变量...还好我们有auto:
```cpp
auto func = [](const std::unique<Widget>& w1, 
                const std::unique<Widget>& w2)
                { return *w1 < *w2; }
```
在C++14之后，这个例子还可以再精简：
```cpp
// 这里为什么使用auto& 而非 auto 呢？
// 请参考auto类型推导一文...
auto func = [](const auto& w1, const auto& w2)
    { return *w1 < *w2 ; }
```
你或许想说，对于这种场景，我使用std::function同样能够声明一个和lambda类似的可调用对象...但是之后的学习中，我们会建议你尽量使用“auto + lambda” 而非 “std::function + std::bind”

> In other words, the std::func
tion approach is generally bigger and slower than the auto approach, and it may
yield out-of-memory exceptions, too. 
>
> -- reference to Effective Modern C++ Item5

## case 4: 帮助你避免不起眼但致命的隐式转换
如果不使用auto，我们可能使用如下的代码：
```cpp
std::vector<int> vec;
unsigned size = vec.size()
```
晃眼一看，似乎完全合理的代码。实际上，这个过程中发生了隐式类型转换，并且可能带来问题。`vec.size()`实际返回的是`std::vector<int>::size_type`，而非unsigned(虽然都是一种无符号数)。

`std::vector<int>::size_type`在32位机器上为32位，在64位机器上为64位，而`unsigned`始终是32位。那么在64位机器的一些极端情况下，使用`unsigned`接收`std::vector<int>::size_type`类型的值可能导致数据截断...还好我们有auto...
```cpp
auto size = vec.size()
```
让编译器帮你解决你可能不知道`std::vector<int>::size_type`存在的这个问题...

在这方面还有一个典型的例子，让我们看看如下的代码：
```cpp
std::unordered_map<std::string, int> map_;
for(const std::pair<string,int>& p : map_){
    // do something with p...
}
```
请问，这段代码中执行过程中，有什么问题吗？

**有很大问题！**

因为map_的元素实际上是`std::pair<const std::string,int>`而非`std::pair<string,int>`，当我们的for-循环指示将`std::pair<const std::string,int>`赋值给`std::pair<std::string,int>&`类型变量p时，程序只能拷贝一份全新的非const对象赋值给p。

这样带来了两方面的问题：
1. 我们本意是使用引用减少拷贝，但实际上还是发生了拷贝
2. 如果我们程序中想要访问p的地址，实际会访问到未命名的临时对象的地址

还好我们有auto...

```cpp
std::unordered_map<std::string, int> map_;
for(const auto& p : map_){
    // do something with p...
}
```


到此为止，我们至少发现了auto具有的四个适用场景，emmm 确实十分有用...（好用，爱用，多用...）

## auto带来的问题

谈了那么多auto的好处，那么auto有什么劣势呢？

我们在《学会使用decltype》、《C++模板中的类型推导》、《auto的类型推导》中提到了一些：

1. auto作为返回值时，按照template的标准进行类型推导
2. decltype(auto)在面对name和lvalue expr时返回不一致的类型
3. 还有些不记得了...之后补一下

## auto带来的可读性问题...
个人认为auto不带来可读性问题，当你拥有一个良好配置的C++编辑、编译环境时，编辑器中就会帮你完成类型推导，并展示，不影响可读性...

另外，对于程序而言，规范的注释、丰富的文档、合理的变量命名手段才是最重要的可读性保证...

auto对于代码重构方面带来的帮助，也是显示初始化变量所不能比拟的...

## TODO
1. 去学习C++17以后得更加丰富的auto使用方法...
2. 补充auto不适用、或者需要谨慎使用的情况...
