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

```cpp
class FileSystem
{
public:
    std::size_t numDisks() const;
};

FileSystem& tfs()
{
    static FileSystem fs;
    return fs;
}

void test()
{
    std::size_t disks = tfs().numDisks(); // tfs 改为 tfs()
}
```

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

## 3 资源管理

### 13 以对象管理资源

把资源放进对象内, 依赖 C++ 的析构函数确保资源被释放

注: 原书中介绍了 auto_ptr, 但是 auto_ptr 已经被 C++11 弃用, 用 unique_ptr 替代, 后文中的 auto_ptr 都替换为 unique_ptr

```cpp
void f()
{
    std::unique_ptr<Investment> pInv(createInvestment());
    ...
}  // pInv 被销毁, 资源被释放
```

这个例子示范了 "以对象管理资源" 的两个关键想法:

- 获得资源后立刻放进管理对象内, 也被称为 "资源取得时机便是初始化时机" (Resource Acquisition Is Initialization, RAII)
- 管理对象运用析构函数确保资源被释放, 尽管产生异常时事情有些棘手, 但条款 08 已经能够解决这个问题

unique_ptr 确保只有一个指针指向资源, 如果对 unique_ptr 进行复制, 则会将资源的所有权转移

注: 原书中介绍了 tr1::shared_ptr, 但是 shared_ptr 已经被 C++11 引入, 故后文中的 shared_ptr 都可代表 std::shared_ptr

如果资源要求进行正常的复制, 则需要使用 shared_ptr, 它使用了引用计数技术, 缺点是无法检查环状引用

```cpp
void f()
{
    std::shared_ptr<Investment> pInv(createInvestment());
    ...
}  // pInv 被销毁, 资源被释放
```

由于 unique_ptr 和 shared_ptr 都使用 delete 来释放资源, 而非 delete[], 所以不能将 unique_ptr 和 shared_ptr 用于数组, 因为 vector 和 string 可以替代数组, 如果一定要使用, 可以使用自定义删除器, 或者使用 Boost 的 shared_array 和 scoped_array

如果预制式的 unique_ptr 和 shared_ptr 不能满足需求, 可以制作自己的资源管理类, 但是要注意一些细节, 详见条款 14 和 15

如果工厂函数直接返回一个未加工的指针, 则需要使用者注意调用 delete 或者将指针放入 unique_ptr 和 shared_ptr 改进接口可以防止上述情况, 详见条款 18

总结:

> 为防止资源泄漏, 使用 RAII 对象管理资源
> 两个常被使用的 RAII class 是 unique_ptr 和 shared_ptr, shared_ptr 一般是较佳选择, 它的复制行为较为直观

### 14 在资源管理类中小心 copying 行为

例如, 使用类管理互斥锁时会产生如下代码:

```cpp
class Lock {
public:
    explicit Lock(Mutex* pm)
        : mutexPtr(pm)
    {
        lock(mutexPtr);
    }
    ~Lock()
    {
        unlock(mutexPtr);
    }
private:
    Mutex* mutexPtr;
};
```

但如果对象被复制则会产生不期望的结果, 为解决 "当一个 RAII 对象被复制时会发生什么事" 的问题, 有两种方法:

- **禁止复制**: 对于 Lock 类, 禁止复制是合理的, 因为很少可以拥有 synchronization primitives 的复制, 条款 06 中介绍了如何禁止复制
- **对底部资源引用计数**: 对于 shared_ptr, 采用引用计数技术, 只要内含一个 shared_ptr 成员变量, RAII class 就可以实现引用计数行为, 如果资源被释放时不希望将其删除而是其他动作, 则可以指定删除器

```cpp
class Lock {
public:
    explicit Lock(Mutex* pm)
        : mutexPtr(pm, unlock)
    {
        lock(mutexPtr.get());
    }
private:
    std::shared_ptr<Mutex> mutexPtr;
};
```

- **复制底部资源**: 如果一份资源可以拥有多个副本, 这种情况下复制资源管理对象时, 还应该复制其所包覆的资源, 也就是 "深度复制"
- **转移底部资源的所有权**: 如果在某些罕见情况下希望确保只有一个 RAII 对象拥有未加工资源, 即使被复制也是如此, 此时资源的所有权会从被复制的对象转移到目标的对象, 这时可以使用 unique_ptr

Copying 函数有可能被编译器自动生成, 如果默认行为不合要求则需要自行编写

总结:

> 复制 RAII 对象必须一并复制它所管理的资源, 所以资源的 copying 行为决定 RAII 对象的 copying 行为
> 普遍而常见的 RAII class *copying* 行为是 抑制 copying, 施行引用计数, 不过其他的行为也可能被实现

### 15 在资源管理类中提供对原始资源的访问

为访问某些 API 需要提供对原始资源的访问方法, 这时需要提供一个函数将 RAII class 对象转换为其所内含的原始资源, 有两个办法: 显式转换和隐式转换

shared_ptr 和 unique_ptr 都提供了 get 函数, 用于执行显式转换, 也就是将返回智能指针内部的原始指针, 它们也重载了指针取值操作符 (operator-> 和 operator*), 它们允许隐式转换至底部原始指针

```cpp
class Investment {
public:
    bool isTaxFree() const;
    ...
};

Investment* createInvestment();

std::shared_ptr<Investment> pi1(createInvestment());

bool taxable = !(pi1->isTaxFree());
...
std::unique_ptr<Investment> pi2(createInvestment());

bool taxable = !((*pi2).isTaxFree());
```

提供隐式转换方法可以增加资源管理类的可用性

```cpp
FontHandle getFont();               // C API
void releaseFont(FontHandle fh);    // C API

class Font {
public:
    explicit Font(FontHandle fh)
        : f(fh)
    {}
    ~Font()
    {
        releaseFont(f);
    }
    FontHandle get() const          // 显式转换
    {
        return f;
    }
    operator FontHandle() const     // 隐式转换
    {
        return f;
    }
};

Font f(getFont());
int newFontSize;
...
changeFontSize(f.get(), newFontSize);   // 显式转换
changeFontSize(f, newFontSize);         // 隐式转换
```

但是隐式转换会增加错误发生的机会, 例如这个隐式转换可能在需要 Font 时创建一个 FontHandle

```cpp
Font f1(getFont());
...
FontHandle f2 = f1;  // 隐式转换
```

这会导致 f2 直接指向 f1 底层的资源, 一般会引发各种问题

总结:

> APIs 往往要求访问原始资源, 所以 RAII class 需要提供一个 "取得其所管理之原始资源" 的办法
> 对原始资源的访问可能经由显式转换或隐式转换, 前者比较安全, 但后者比较方便

### 16 成对使用 new 和 delete 时要采取相同形式

调用 new 之后, 有两件事发生: 1. 内存被分配, 2. 针对此内存一个或多个构造函数被调用; 调用 delete 之后, 也有两件事发生: 1. 针对此内存一个或多个析构函数被调用, 2. 内存被释放

对一个指针调用 delete 时, 只有通过 [] 告知才可知道指针指向一个数组, 否则会被认定为指向一个对象

如果调用 new 时使用 [] 来分配内存, 则调用 delete 时也要使用 [], 否则会导致未定义行为; 如果没有使用 [], 则调用 delete 时也不要使用 []

如果 class 中含有一个指针指向动态分配的内存, 并提供多个构造函数时, 必须使用相同形式的 new 将指针成员初始化

对于 typedef 的使用者, 以 new 创建某种类型的对象时需要明白使用哪种 delete

```cpp
typedef std::string AddressLines[4];
AddressLines* pal = new AddressLines;
...
// delete pal;  // 未定义行为
delete[] pal;  // 正确
```

为避免此类错误, 尽量不要对数组形式做 typedef 动作, 使用 vector, string 等代替

总结:

> new 时使用 [] 分配内存, delete 时也要使用 []; new 时不使用 [], delete 时也不使用 []

### 17 以独立语句将 newed 对象置入智能指针

假定有如下代码:

```cpp
int priority();
void processWidget(std::shared_ptr<Widget> pw, int priority);
```

如果直接调用:

```cpp
processWidget(new Widget, priority());
```

会导致编译错误, 因为 shared_ptr 的构造函数是 explicit 的, 不能隐式转换, 如下方式可以通过编译, 但可能导致资源泄漏

```cpp
processWidget(std::shared_ptr<Widget>(new Widget), priority());
```

编译器执行 processWidget 之前先做如下三件事:

- 调用 priority
- 执行 new Widget
- 调用 shared_ptr 构造函数

但是顺序是不固定的, 只能确定 new Widget 在 shared_ptr 构造函数前完成, 如果调用 priority 发生在第二个, 并且发生异常, new Widget 分配的内存将无法释放, 从而导致资源泄漏

将语句分离可以避免这个问题

```cpp
std::shared_ptr<Widget> pw(new Widget);
processWidget(pw, priority());
```

总结:

> 以独立语句将 newed 对象置入智能指针, 以确保资源被释放, 避免异常抛出导致资源泄漏

## 4 设计与声明

### 18 让接口容易被正确使用, 不易被误用

有如下方法可以让接口容易被正确使用, 不易被误用:

- 导入外覆类型
- 以函数替换对象, 表现某个特定的值 (与 tips 04 相关)
- 使自建 types 与默认 types 行为保持一致
- 返回智能指针代替返回裸指针, 将删除器绑定到 shared_ptr 上防止调用错误的 delete 方法

shared_ptr 相对于裸指针的额外花销是: 2 倍大的内存占用, 使用动态内存记录自身数据和删除器所需数据, 以 virtual 形式调用删除器, 在多线程程序修改引用次数时需要线程同步化, 这些成本并不显著, 但可以大大降低客户犯错误的机会

总结:

> 设计接口易于正确使用, 不易误用, 这是接口设计者的目标
> 促进正确使用包括接口的一致性, 以及与内置类型兼容
> 防止误用包括避免建立新类型, 限制类型上的操作, 束缚对象值, 消除客户的资源管理责任
> shared_ptr 支持定制删除器, 可防范 DLL 问题, 可用来自动解除互斥锁等等

### 19 设计 class 犹如设计 type

好的 type 有自然的语法, 直观的语义, 以及一个或多个高效实现品

影响 class 设计的因素有:

- 新的 type 的对象应该如何被创建和销毁
- 对象的初始化和对象的赋值应该有什么样的区别
- 新 type 的对象如何被传递给其他函数
- 什么是新 type 的合法值
- 新的 type 需要配合某个继承图系吗
- 新的 type 需要什么样的转换
- 新的 type 需要什么样的操作符和函数
- 什么样的标准函数需要被驳回
- 谁该取用新的 type 的成员
- 什么是新的 type 的 "未声明接口"
- 新的 type 有多一般化
- 真的需要新的 type 吗

总结:

> 设计 class 就像设计 type 一样, 在定义新的 type 前, 请考虑本条款覆盖的所有讨论主题

### 20 宁以 pass-by-reference-to-const 替换 pass-by-value

```cpp
bool validateStudent(Student s);

bool validateStudent(const Student& s);
```

用 pass-by-reference-to-const 替换 pass-by-value 可以避免多次的 copy 构造函数和析构函数调用, 为避免 s 被改变, const 是有必要的

用 by reference 传递参数还可以避免对象切割问题, 避免 derived class 对象被切割为 base class 对象

```cpp
class Window {
public:
    std::string name() const;
    virtual void display() const;
};

class WindowWithScrollBars : public Window {
public:
    virtual void display() const;
};

void printNameAndDisplay(Window w)  // 错误, 会导致 WindowWithScrollBars 对象传入后使用 base class 的 display
{
    std::cout << w.name();
    w.display();
}

void printNameAndDisplay(const Window& w)  // 正确
{
    std::cout << w.name();
    w.display();
}
```

由于 reference 往往由指针实现, 对于内置类型 pass-by-value 的效率往往更高, STL中的迭代器和函数对象也是这样, 但即便是小型的自定义 types, pass-by-reference 也往往是更好的选择

总结:

> 尽量以 pass-by-reference-to-const 替换 pass-by-value, 前者通常比较高效, 并可避免切割问题
> 以上规则并不适用于内置类型, STL 迭代器和函数对象, 对它们而言 pass-by-value 通常比较适当

### 21 必须返回对象时, 别妄想返回 reference

任何函数如果返回一个 reference 或指针指向某个 local 对象都会导致未定义行为

如果在函数内进行堆内存分配, 并返回一个 reference 或指针指向这个对象, 则需要在函数外释放, 这极易导致内存泄漏

在函数内部创建一个 static 对象, 并返回一个 reference 或指针指向这个对象, 会导致线程安全问题, 并且导致函数不可重入

```cpp
class Rational {
public:
    Rational(int numerator = 0, int denominator = 1)
        : n(numerator), d(denominator)
    {}
    ...

private:
    int n, d;

    friend const Rational operator*(const Rational& lhs, const Rational& rhs);
};

// 1. 返回一个 reference
const Rational& operator*(const Rational& lhs, const Rational& rhs) // wrong
{
    static Rational result;
    result = Rational(lhs.n * rhs.n, lhs.d * rhs.d);
    return result;
}

// 2. 返回一个堆内存分配的对象
const Rational& operator*(const Rational& lhs, const Rational& rhs) // wrong
{
    Rational* result = new Rational(lhs.n * rhs.n, lhs.d * rhs.d);
    return *result;
}

// 3. 返回一个 static 对象
const Rational& operator*(const Rational& lhs, const Rational& rhs) // wrong
{
    static Rational result;
    result = Rational(lhs.n * rhs.n, lhs.d * rhs.d);
    return result;
}

// 会导致下面的表达式总为 true
if ((r1 * r2) == (r3 * r4)) { ... }
// 等价于 if (operator==(operator*(r1, r2), operator*(r3, r4))) { ... }
```

一个必须返回新对象的函数的正确写法是: 简单地返回一个对象, 放弃为了效率而返回 reference

```cpp
const Rational operator*(const Rational& lhs, const Rational& rhs)
{
    return Rational(lhs.n * rhs.n, lhs.d * rhs.d);
}
```

总结:

> 绝不要返回 pointer 或 reference 指向一个 local stack 对象, 或返回 reference 指向一个 heap-allocated 对象, 或返回 reference 指向一个 local static 对象而有可能同时需要多个这样的对象

### 22 将成员变量声明为 private

将成员变量声明为 private 而非 public 或者 protected 的原因:

- 语法一致性: 只有通过函数才能访问成员变量可以使代码更加一致
- 可细微划分访问控制: 更精确地控制成员变量, 保证约束条件成立
- 封装: 通过函数可以改变实现方法, 而不改变接口

总结:

> 将成员变量声明为 private, 以确保语法一致性, 可细微划分访问控制, 允诺约束条件获得保证, 并提供 class 作者以充分的实现弹性
> protected 并不比 public 更具封装性

### 23 宁以 non-member, non-friend 替换 member 函数

尽管面向对象编程的基本理念是数据和操作被封装在 class 内, 但并不是所有操作都要被封装在 class 内, 某些情况下, member 函数会导致封装性降低

对于对象内的数据, 越少的代码可以看到数据, 封装性就越好, 所以 non-member, non-friend 不增加可以访问对象内 private 数据的函数的数量, 从而有助于提升封装性

可以将此 non-member, non-friend 函数放在另一个工具类中, 更自然的做法是放在和 class 相同的 namespace 中, 这样子可以将不同的函数放在不同的头文件中, 以减少编译依赖

```cpp
// webbrowser.h
namespace WebBrowserStuff {
    class WebBrowser {
    public:
        void clearCache();
        void clearHistory();
        void removeCookies();
    };
}

// webbrowserbookmarks.h
namespace WebBrowserStuff {
    void clearBookmarks();
    ...
}

// webbrowsercookies.h
namespace WebBrowserStuff {
    void clearCookies(WebBrowser& wb);
    ...
}
```

总结:

> 宁可拿 non-member, non-friend 函数替换 member 函数, 以提升封装性, 包裹弹性, 机能扩充性

### 24 若所有参数皆需类型转换, 请为此采用 non-member 函数

由于只有参数被列于参数列中可以进行隐式类型转换, 而 "被调用之成员函数所隶属的对象" (即 this 对象) 无法进行隐式类型转换, 所以如果所有参数都需要类型转换且使用成员函数, 则导致调用时对参数的顺序有所限制

```cpp
class Rational {
public:
    Rational(int numerator = 0, int denominator = 1);
    const Rational operator*(const Rational& rhs) const;
    ...
};

Rational oneEighth(1, 8);
Rational oneHalf(1, 2);
Rational result;

result = oneHalf * oneEighth;  // right
result = oneEighth * oneHalf;  // right
result = oneEighth * 2;        // right
result = 2 * oneEighth;        // wrong
```

如果将 operator* 改为 non-member 函数, 则可以避免这个问题, 但是要记住, 如果可以避免成为 friend 函数, 就不要成为 friend 函数

```cpp
class Rational {
public:
    Rational(int numerator = 0, int denominator = 1);
    ...
};

const Rational operator*(const Rational& lhs, const Rational& rhs);
```

但是如果离开 Object-Oriented C++ 而进入 Template C++ 的领域, 即让 Rational 成为一个 template class, 本例不一定适用, 详见条款 46

总结:

> 如果某个函数的所有参数 (包括被 this 指针所指的那个隐喻参数) 都需要类型转换, 那么它必须是个 non-member 函数
