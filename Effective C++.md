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

### 04 确保对象使用前已被初始化

对于 C part of C++ 初始化可能导致运行期成本时, 不保证发生初始化, 对于 non-C part of C++ 最好的办法是永远在对象被使用前进行初始化

对于无任何成员的内置类型需要手动完成, 对于内置类型以外的东西, 初始化由构造函数完成, 规则是: 确保每一个构造函数都将对象的每一个成员初始化, 同时注意区分成员初始化和赋值操作

```cpp
class PhoneNumber
{
    ...
};
class ABEntry
{
public:
    ABEntry(const std::string& name, const std::string& address, const std::list<PhoneNumber>& phones);

private:
    std::string theName;
    std::string theAddress;
    std::list<PhoneNumber> thePhones;
    int numTimesConsulted;
};

ABEntry::ABEntry(const std::string& name, const std::string& address, const std::list<PhoneNumber>& phones)
{
    theName = name;         // 赋值操作
    theAddress = address;   // 赋值操作
    thePhones = phones;     // 赋值操作
    numTimesConsulted = 0;  // 赋值操作
}
```

由于 C++ 规定对象的初始化操作发生在构造函数体执行之前, 所以使用成员初始化列表 (member initialization list) 是更好的选择, 这样做结果相同但通常效率更高

```cpp
ABEntry::ABEntry(const std::string& name, const std::string& address, const std::list<PhoneNumber>& phones)
    : theName(name), theAddress(address), thePhones(phones), numTimesConsulted(0)
{
    // 成员初始化
}
```

内置类型也可以使用成员初始化列表, 并且当使用 default 构造函数时, 也可以使用成员初始化列表, 只要指定 nothing 作为参数即可

如果成员是 const 或者 reference 类型, 则必须使用成员初始化列表

C++ 的成员初始化次序是固定的: 先完成基类的初始化, 随后按照声明次序完成成员的初始化, 即使在成员初始化列表中的次序和声明次序不同, 也是按照声明次序初始化

non-local static 对象指的是 global 对象和 namespace 作用域内的对象, 以及 class 内和 file 作用域内的 static 对象, 函数内的 static 对象属于 local static 对象

编译单元指的是产出单一目标文件的源文件, 基本上指的是单一源码文件加上其包含的头文件

为解决不同编译单元中定义之 non-local static 对象的初始化次序问题, 可以将每个 non-local static 对象搬到一个独立函数中, 该函数返回一个 reference 指向该对象, 然后用户调用该函数, 这是 Singleton 模式的一种实现

总结:
> 为内置类型对象进行手工初始化
> 构造函数最好使用成员初始化列表, 而非赋值操作, 成员初始化次序和声明次序相同
> 为避免 non-local static 对象之初始化次序问题, 使用 local static 对象替代 non-local static 对象

## 2 构造, 析构, 赋值运算

### 05 了解 C++ 默默编写并调用哪些函数

C++ 会自动声明一个 copy 构造函数, 一个 copy 赋值运算符, 一个析构函数, 如果没有声明任何构造函数, 则 C++ 会自动声明一个默认构造函数, 所有这些函数都是 public 和 inline 的

所以下面两段代码是类似的:

```cpp
class Empty {};

class Empty {
public:
    Empty() {}
    Empty(const Empty& rhs) {}
    ~Empty() {}
    Empty& operator=(const Empty& rhs) {}
};
```

注意, 编译器提供的析构函数是 non-virtual 的, 除非基类的析构函数是 virtual 的

当类中有 reference 或者 const 成员时, 编译器不会自动生成 copy 赋值运算符, 当 base 类的 copy 赋值运算符是 private 时, 编译器不会自动生成 derived 类的 copy 赋值运算符

总结:

> 编译器可以为 class 创建 default 构造函数, copy 构造函数, copy 赋值运算符, 析构函数

### 06 若不想使用编译器自动生成的函数, 就该明确拒绝

如果不希望编译器自动生成的函数被使用, 可以将其声明为 private 并且不予实现, 用户如果使用这些函数则会得到编译期错误, 而 member 函数和 friend 函数会导致链接期错误

通过设计一个 base class, 并将其 copy 构造函数和 copy 赋值运算符声明为 private, 可以防止 derived class 生成这两个函数, 从而将错误移至编译期

```cpp
class Uncopyable {
protected:
    Uncopyable() {}
    ~Uncopyable() {}
private:
    Uncopyable(const Uncopyable&);
    Uncopyable& operator=(const Uncopyable&);
};
```

boost::noncopyable 是一个类似的类

总结:

> 为驳回编译器自动提供的函数, 可以将相应的成员函数声明为 private 并且不予实现, Uncopyable 这样的 base class 也可以达到这个目的

补充: 至少在 C++11 之后, 可以使用 delete 关键字来达到相同的目的

```cpp
class Uncopyable {
public:
    Uncopyable() = default;
    ~Uncopyable() = default;
    Uncopyable(const Uncopyable&) = delete;
    Uncopyable& operator=(const Uncopyable&) = delete;
};
```

### 07 为多态基类声明 virtual 析构函数

给 base class 声明一个 virtual 析构函数后, 此后删除指向 derived class 对象的 base class 指针时, 会调用 derived class 的析构函数, 如果 base class 的析构函数不是 virtual 的, 则不会调用 derived class 的析构函数, 会导致资源泄漏

如果一个 class 带有任何 virtual 函数, 它就应该拥有一个 virtual 析构函数, 而 base class 没有 virtual 函数时, 往往代表它不会被当作 base class 使用, 所以不需要 virtual 析构函数, 否则会因为生成一个 virtual table 而增加空间开销, 还导致无法同其他语言进行交互

不要继承一个不带 virtual 析构函数的 class, 否则用指向 derived class 对象的 base class 指针删除对象时, 可能导致 derived class 的析构函数不被调用, 从而导致资源泄漏

注: C++11 之后, 可以使用 final 关键字来阻止继承, final 既可以用于类, 也可以用于虚函数

将析构函数声明为纯虚函数可以用于实现抽象基类

```cpp
class AWOV {
public:
    virtual ~AWOV() = 0;
};
AWOV::~AWOV() {}    // 必须提供定义
```

给 base class 声明一个 virtual 析构函数是为了 polymorphic base classes (带多态性质的基类), 即为了用 base class 接口处理 derived class 对象, 否则不需要 virtual 析构函数, 比如不被用于 base class 的 class 和提供某些特定功能的 class 如 06 的 Uncopyable

总结:

> polymorphic base classes 应该声明一个 virtual 析构函数, 如果 class 带有任何 virtual 函数, 它就应该拥有一个 virtual 析构函数
> class 的设计目的如果不是作为 base class 使用, 或者不是为了具备多态性 (polymorphically), 就不应该声明 virtual 析构函数

### 08 别让异常逃离析构函数

为了处理析构函数中的异常有两种方法:

如果析构函数内部抛出异常, 可以调用 std::abort 来终止程序

```cpp
class DBConnection {
public:
    ...
    ~DBConnection()
    {
        try {
            db.close();  // 关闭数据库连接
        } catch (...) {
            std::abort();  // 终止程序
        }
    }
};
```

如果析构函数内部抛出异常, 在内部进行捕获处理, 即吞下异常, 尽管这样会压制 "某些任务失败" 的信息

```cpp
class DBConnection {
public:
    ...
    ~DBConnection() noexcept(false)  // noexcept(false) 表示析构函数可能抛出异常
    {
        try {
            db.close();  // 关闭数据库连接
        } catch (...) {
            // 如果 db.close() 抛出异常, 则不要让异常逃离析构函数
            ...
        }
    }
};
```

还有一种方法是重新设计接口, 使其用户有机会对可能出现的问题做出反应, 比如提供一个 close 函数, 使用户可以在析构函数之前调用 close 函数

```cpp
class DBConn {
public:
    ...
    void close()
    {
        db.close();
        closed = true;
    }
    ~DBConn()
    {
        if (!closed) {
            try {
                db.close();  // 关闭数据库连接
            } catch (...) {
                ...
            }
        }
    }
private:
    DBConnection db;
    bool closed;
};
```

总结:
> 析构函数绝对不要抛出异常, 如果一个被析构函数调用的函数可能抛出异常, 析构函数应该捕获任何异常, 然后吞下它们, 或者结束程序
> 如果客户需要对某个操作函数出现的异常做出反应, 那么 class 应该提供一个普通函数 (如 close) 来执行该操作, 而不应该依赖析构函数来执行该操作

### 09 绝不在构造和析构过程中调用 virtual 函数

如果在构造函数或者析构函数中调用 virtual 函数, 会导致对象的类型是 base class 而非 derived class , 从而调用的是 base class 的 virtual 函数, 而非 derived class 的 virtual 函数

C++ 中会先调用 base class 的构造函数, 再调用 derived class 的构造函数, 而析构函数的调用顺序相反, 这样在 base class 的构造函数和析构函数执行时, derived class 的成员变量还未初始化或已经析构, 所以在此时对象的类型是 base class 而非 derived class, 此时的运行期类型信息也是 base class 而非 derived class

由于无法使用 virtual 函数从 base class 向下调用, 在构造期间需要令 derived class 将必要的信息向上传递至 base class 的构造函数

```cpp
class Transaction
{
public:
    explicit Transaction(const std::string& logInfo);
    // virtual void logTransaction() const = 0;
    void logTransaction(const std::string& logInfo) const;  // 非 virtual 函数
    ...
};

Transaction::Transaction(const std::string& logInfo)
{
    ...
    logTransaction(logInfo);
}

class BuyTransaction : public Transaction
{
public:
    BuyTransaction(parameters);
    // virtual void logTransaction() const; // 现在不需要此函数
    ...
private:
    static std::string createLogString(parameters);
};

BuyTransaction::BuyTransaction(parameters)
    : Transaction(createLogString(parameters))
{
    ...
}
```

注意 createLogString, 此函数也相对于将 parameters 传递给 base class 的构造函数更加简洁, 也更具可读性, 它是 static 的, 这样就不会意外指向 derived class 的成员变量

总结:

> 在构造和析构期间不要调用 virtual 函数, 因为这类调用不会下降至 derived class

### 10 令 operator= 返回一个 reference to `*this`

为了支持连锁赋值, 赋值运算符应该返回一个 reference 指向操作符的左侧实参, 也就是 `*this`, 对于其他赋值运算符也应该如此

```cpp
class Widget {
public:
    ...
    Widget& operator=(const Widget& rhs)
    {
        ...
        return *this;
    }
    ...
    Widget& operator+=(const Widget& rhs)
    {
        ...
        return *this;
    }
    ...
    Widget& operator=(int rhs)
    {
        ...
        return *this;
    }
    ...
};
```

总结:
> 令赋值运算符返回一个 reference to `*this`

### 11 在 operator= 中处理"自我赋值"

如果运用 Tips 13 和 14, 运用对象管理资源, 则可以确定所谓的 "资源管理对象" 在 copy 时有正确的举措, 但如果是尝试自行管理资源, 则需要处理 "自我赋值" 问题, 否则可能遇到 "在停止使用资源前删除资源" 的问题

```cpp
class Bitmap { ... };
class Widget {
public:
    ...
    Widget& operator=(const Widget& rhs)
    {
        // 会导致资源泄漏的代码
        delete pb;
        pb = new Bitmap(*rhs.pb);
        return *this;
    }
    ...
private:
    Bitmap* pb;
};
```

可以最开始加上 `if (this == rhs) return *this;` 可以部分解决问题, 但是不具备异常安全性, 但解决了异常安全性的同时往往可以解决 "自我赋值" 问题

```cpp
class Widget {
public:
    ...
    Widget& operator=(const Widget& rhs)
    {
        Bitmap* pOrig = pb;
        pb = new Bitmap(*rhs.pb);
        delete pOrig;
        return *this;
    }
    ...
};
```

为关注效率问题, 可以把 "证同测试" 放在赋值操作的最开始, 但由于自我赋值的概率不高, 且证同测试会引入诸如控制流和代码大小等开销, 所以不建议这样做

使用 copy and swap 技术也可以解决 "自我赋值" 问题

```cpp
class Widget {
public:
    ...
    Widget& operator=(const Widget& rhs)
    {
        Widget temp(rhs);
        swap(temp);
        return *this;
    }
    ...
    void swap(Widget& other)
    {
        using std::swap;
        swap(pb, other.pb);
    }
    ...
};
```

由于某 class 的 copy assignment 操作符可能被声明为以 by value 的方式接受参数, 并且以 by value 方式传递东西会造成一份副本, 所以也可以写成如下形式

```cpp
class Widget {
public:
    ...
    Widget& operator=(Widget rhs)
    {
        swap(rhs);
        return *this;
    }
    ...
};
```

这样做有些牺牲清晰性, 但可能使编译器产生更高效的代码

总结:
> 确保当对象自我赋值时 operator= 有正确的行为, 其中的技术包括 "证同测试", 经过设计的语句顺序 和 copy-and-swap
> 确定任何函数如果操作一个以上的对象, 而其中多个对象是同一个对象时, 该函数的行为仍然正确

### 12 复制对象时勿忘其每一个成分

copy 构造函数, copy 赋值运算符统称为 copying 函数

如果选择自己编写 copying 函数, 那么如果 class 内有些成员没有被复制, 编译器也不会给出警告, 在非标准形式的 operator= (如 operator+=) 中也要注意这个问题

为 dirved class 编写 copying 函数时, 也要调用 base class 的 copying 函数

```cpp
class Base {
public:
    Base(const Base& rhs)
        : mem1(rhs.mem1), mem2(rhs.mem2)
    {}
    Base& operator=(const Base& rhs)
    {
        mem1 = rhs.mem1;
        mem2 = rhs.mem2;
        return *this;
    }
private:
    int mem1;
    int mem2;
};

class Derived : public Base {
public:
    Derived(const Derived& rhs)
        : Base(rhs), mem3(rhs.mem3), mem4(rhs.mem4)
    {}
    Derived& operator=(const Derived& rhs)
    {
        Base::operator=(rhs);
        mem3 = rhs.mem3;
        mem4 = rhs.mem4;
        return *this;
    }
private:
    int mem3;
    int mem4;
};
```

所以说复制每一个成分指的是, 当编写一个 copying 函数时:

1. 复制所有 local 成员变量
2. 调用所有 base class 的 适当的 copying 函数

尽管 copy 构造函数和 copy 赋值运算符的实现往往相似, 但不能让它们其中的一个调用另一个来避免重复, 合理的做法是将共同的功能提取出来, 放在一个 private 函数 (往往命名为 init) 中, 然后两个 copying 函数都调用这个函数

总结:

> copying 函数应该确保复制对象内的所有成员变量和所有 base class 成分
> 不要尝试以某个 copying 函数调用另一个 copying 函数, 应该将共同机制放在第三个函数中, 由两个 copying 函数共同调用
