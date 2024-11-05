# Effective C++

## 0. 导读

## 1 让自己习惯 C++

### 01 视 C++ 为一个语言联邦

今天的 C++ 已经是个多重范型编程语言, 它支持如下的形式:

- 过程形式 (procedural)
- 面向对象形式 (object-oriented)
- 函数形式 (functional)
- 泛型形式 (generic)
- 元编程形式 (metaprogramming)

最简单的方法是将 C++ 视为一个由相关次语言 (sublanguage) 组成的联邦而非单一语言, 联邦内的守则往往通俗易懂, 而当你从一个次语言移往另一个次语言, 守则可能改变, 为理解 C++ 需要认识如下 4 个次语言:

- C: C++ 是以 C 为基础的, 当使用 C part of C++ 时, 高效编程守则映射出 C 的一些缺陷: 没有模板, 没有异常处理, 没有重载等
- Object-Oriented C++: 这部分代表了 C with Class, 这一部分的高效编程守则是面向对象设计的古典守则
- Template C++: 这是 C++ 的泛型编程部分 (generic programming), 这部分的高效编程守则是崭新的模板元编程 (template metaprogramming, TMP)的守则, 但是 TMP 相关规则很少和 C++ 主流编程相互影响
- STL: 它是一个 template 程序库, 但是它的设计和使用方式和 C++ 的模板编程有很大不同, STL 有自己的高效编程守则

从传值方法可以看到这 4 个次语言的不同:

- C: 传值 (pass-by-value)
- Object-Oriented C++: 传引用 (pass-by-reference-to-const)
- Template C++: 传引用 (pass-by-reference-to-const)
- STL: 传值 (pass-by-value)

要记住的是:

> C++ 高效编程守则视情况而变化, 这取决于你使用 C++ 的哪一部分

### 02 尽量以 const, enum, inline 替换 #define

换句话说, 尽量以编译器替换预处理器, 以便获得更好的错误信息, 以及更好的调试支持, 并可能生成更小的目标代码

有两种值得提出的特殊情况:

对于指向常量的常量指针, 需要向下面这样定义:

```cpp
const char * const authorName = "Scott Meyers";
```

const 出现了两次, 此外, string 对象比 char*-based 字符串更好

```cpp
const std::string authorName("Scott Meyers");
```

对于 class 专属常量, 一般情况下它们是 static的

```cpp
class GamePlayer {
private:
    static const int NumTurns = 5;  // 常量声明式
    int scores[NumTurns];           // 使用常量
};
```

这里的 NumTurns 是一个声明式, 而非定义式, 只要不取其地址就可以声明并使用而不需要定义, 但是若需要取其地址, 则需要定义

```cpp
const int GamePlayer::NumTurns; // 定义 NumTurns
```

这个式子需要放进实现分件而非头文件中

这种 class 专属常量是 #define 无法实现的, 因为 #define 无法提供封装性

对旧式的编译器, 无法对 static 成员变量进行初始化, 需要如下处理:

```cpp
class CostEstimate {
private:
    static const double FudgeFactor;            // 声明式, 位于头文件内
};
const double CostEstimate::FudgeFactor = 1.35;  // 定义式, 位于实现文件内
```

更特殊的情况需要使用 enum hack 补偿方法

```cpp
class GamePlayer {
private:
    enum { NumTurns = 5 };  // NumTurns 是 5 的一个符号名称
    int scores[NumTurns];   // 使用 NumTurns
};
```

学习 enum hack 有如下两个原因:

1. enum hack 和 #define 行为上比较相似, 可能这是你想要的
2. enum hack 作为一种技术, 在许多代码中都有使用, 你需要理解它

对于使用 #define 来实现宏的代码, 可以使用 template inline 函数来替代

```cpp
#define CALL_WITH_MAX(a, b) f((a) > (b) ? (a) : (b))

template<typename T>
inline void callWithMax(const T& a, const T& b) {
    f(a > b ? a : b);
}
```

要记住的是:

> 对于常量, 最好使用 const 对象或者 enum 而非 #define
> 对于形似函数的宏, 最好使用 inline 函数而非 #define

### 03 尽量使用 const

const 可用于如下场景:

- 在 class 外部修饰 global 或者 namespace 作用域的常量
- 修饰文件, 函数, block 作用域的被声明为 static 的对象
- 在 class 内部修饰 static 或者 non-static 成员变量
- 对于指针, const 可以修饰指针本身, 也可以修饰指针所指物, 甚至两者都可以修饰

如果 const 出现在 \* 左侧, 则表示指针所指物是常量, 如果 const 出现在 \* 右侧, 则表示指针本身是常量, 如果两者都出现, 则表示指针本身和指针所指物都是常量, 修饰所指物时下面两种写法是等价的:

```cpp
const char * authorName = "Scott Meyers";
char const * authorName = "Scott Meyers";
```

STL 迭代器是指针的 generalization, 所以迭代器就像一个 T\* 指针, 声明迭代器为 const 就像声明指针为 const (即 T\* const), 而非声明指针所指物为 const (即 const T\*), 如果希望迭代器所指物为 const, 需要使用 const_iterator

```cpp
std::vector<int> vec;
...
const std::vector<int>::iterator iter = vec.begin();  // iter 的内容不可变
std::vector<int>::const_iterator cIter = vec.begin(); // cIter 所指物不可变
```

const 可以和函数的返回值, 参数, 函数自身 (需要是成员函数) 产生关系

const 用于返回值和参数可以避免某些错误, 作用和修饰常量类似

#### const 成员函数

const 作用于成员函数是为了该成员函数可作用于 const 对象

两个成员函数如果只是 const 和非 const 的区别则可以重载

```cpp
class TextBlock {
public:
    ...
    const char& operator[](std::size_t position) const
    {
        return text[position];
    }
    char& operator[](std::size_t position)
    {
        return text[position];
    }
private:
    std::string text;
};
```

对 const 成员函数的理解有两种, bitwise const 和 logical const, C++ 采用 bitwise const

const 的对象不允许改变对象内的任何成员变量 (static 成员变量除外), 当有需要使 const 对象的某些值可以被修改时, 可以使用 mutable 修饰

```cpp
class TextBlock {
public:
    ...
private:
    std::string text;
    mutable std::size_t textLength;
    mutable bool lengthIsValid;
    std::size_t computeTextLength() const
    {
        if (!lengthIsValid) {
            textLength = text.size();
            lengthIsValid = true;
        }
        return textLength;
    }
};
```

#### 在 const 和 non-const 成员函数中避免重复

在 non-const 成员函数中调用 const 成员函数并进行转型是避免代码重复的一种方法

```cpp
class TextBlock {
public:
    ...
    const char& operator[](std::size_t position) const
    {
        ... // 边界检查, 志记数据访问, 检验数据有效性等
        return text[position];
    }
    char& operator[](std::size_t position)
    {
        return const_cast<char&>(                           // 将 op[] 返回值的 const 属性移除
            static_cast<const TextBlock&>(*this)[position]  // 为 this 添加 const 属性, 调用 const 成员函数
            // static_cast<const TextBlock&>(*this).operator[](position)  函数调用形式
        );
    }
...
};
```

总结:

> 将某些东西声明为 const 可帮助编译器侦测出错误用法, const 可被施加于任何作用域内的对象, 函数参数, 函数返回值, 成员函数本体
> 编译器强制实施 bitwise const, 但编写程序时应该遵循 conceptual constness
> 当 const 和 non-const 成员函数有着实质等价的实现时, 令 non-const 版本调用 const 版本可避免代码重复
