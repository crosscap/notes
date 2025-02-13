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

总结:

- C++ 高效编程守则视情况而变化, 这取决于你使用 C++ 的哪一部分

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

总结:

- 对于常量, 最好使用 const 对象或者 enum 而非 #define
- 对于形似函数的宏, 最好使用 inline 函数而非 #define

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

#### 总结

- 将某些东西声明为 const 可帮助编译器侦测出错误用法, const 可被施加于任何作用域内的对象, 函数参数, 函数返回值, 成员函数本体
- 编译器强制实施 bitwise const, 但编写程序时应该遵循 conceptual constness
- 当 const 和 non-const 成员函数有着实质等价的实现时, 令 non-const 版本调用 const 版本可避免代码重复

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

- 为内置类型对象进行手工初始化
- 构造函数最好使用成员初始化列表, 而非赋值操作, 成员初始化次序和声明次序相同
- 为避免 non-local static 对象之初始化次序问题, 使用 local static 对象替代 non-local static 对象

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

- 编译器可以为 class 创建 default 构造函数, copy 构造函数, copy 赋值运算符, 析构函数

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

- 为驳回编译器自动提供的函数, 可以将相应的成员函数声明为 private 并且不予实现, Uncopyable 这样的 base class 也可以达到这个目的

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

- polymorphic base classes 应该声明一个 virtual 析构函数, 如果 class 带有任何 virtual 函数, 它就应该拥有一个 virtual 析构函数
- class 的设计目的如果不是作为 base class 使用, 或者不是为了具备多态性 (polymorphically), 就不应该声明 virtual 析构函数

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
- 析构函数绝对不要抛出异常, 如果一个被析构函数调用的函数可能抛出异常, 析构函数应该捕获任何异常, 然后吞下它们, 或者结束程序
- 如果客户需要对某个操作函数出现的异常做出反应, 那么 class 应该提供一个普通函数 (如 close) 来执行该操作, 而不应该依赖析构函数来执行该操作

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

- 在构造和析构期间不要调用 virtual 函数, 因为这类调用不会下降至 derived class

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

- 令赋值运算符返回一个 reference to `*this`

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

- 确保当对象自我赋值时 operator= 有正确的行为, 其中的技术包括 "证同测试", 经过设计的语句顺序 和 copy-and-swap
- 确定任何函数如果操作一个以上的对象, 而其中多个对象是同一个对象时, 该函数的行为仍然正确

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

- copying 函数应该确保复制对象内的所有成员变量和所有 base class 成分
- 不要尝试以某个 copying 函数调用另一个 copying 函数, 应该将共同机制放在第三个函数中, 由两个 copying 函数共同调用

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

- 为防止资源泄漏, 使用 RAII 对象管理资源
- 两个常被使用的 RAII class 是 unique_ptr 和 shared_ptr, shared_ptr 一般是较佳选择, 它的复制行为较为直观

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

- 复制 RAII 对象必须一并复制它所管理的资源, 所以资源的 copying 行为决定 RAII 对象的 copying 行为
- 普遍而常见的 RAII class *copying* 行为是 抑制 copying, 施行引用计数, 不过其他的行为也可能被实现

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

- APIs 往往要求访问原始资源, 所以 RAII class 需要提供一个 "取得其所管理之原始资源" 的办法
- 对原始资源的访问可能经由显式转换或隐式转换, 前者比较安全, 但后者比较方便

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

- new 时使用 [] 分配内存, delete 时也要使用 []; new 时不使用 [], delete 时也不使用 []

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

- 以独立语句将 newed 对象置入智能指针, 以确保资源被释放, 避免异常抛出导致资源泄漏

## 4 设计与声明

### 18 让接口容易被正确使用, 不易被误用

有如下方法可以让接口容易被正确使用, 不易被误用:

- 导入外覆类型
- 以函数替换对象, 表现某个特定的值 (与 tips 04 相关)
- 使自建 types 与默认 types 行为保持一致
- 返回智能指针代替返回裸指针, 将删除器绑定到 shared_ptr 上防止调用错误的 delete 方法

shared_ptr 相对于裸指针的额外花销是: 2 倍大的内存占用, 使用动态内存记录自身数据和删除器所需数据, 以 virtual 形式调用删除器, 在多线程程序修改引用次数时需要线程同步化, 这些成本并不显著, 但可以大大降低客户犯错误的机会

总结:

- 设计接口易于正确使用, 不易误用, 这是接口设计者的目标
- 促进正确使用包括接口的一致性, 以及与内置类型兼容
- 防止误用包括避免建立新类型, 限制类型上的操作, 束缚对象值, 消除客户的资源管理责任
- shared_ptr 支持定制删除器, 可防范 DLL 问题, 可用来自动解除互斥锁等等

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

- 设计 class 就像设计 type 一样, 在定义新的 type 前, 请考虑本条款覆盖的所有讨论主题

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

- 尽量以 pass-by-reference-to-const 替换 pass-by-value, 前者通常比较高效, 并可避免切割问题
- 以上规则并不适用于内置类型, STL 迭代器和函数对象, 对它们而言 pass-by-value 通常比较适当

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

- 绝不要返回 pointer 或 reference 指向一个 local stack 对象, 或返回 reference 指向一个 heap-allocated 对象, 或返回 reference 指向一个 local static 对象而有可能同时需要多个这样的对象

### 22 将成员变量声明为 private

将成员变量声明为 private 而非 public 或者 protected 的原因:

- 语法一致性: 只有通过函数才能访问成员变量可以使代码更加一致
- 可细微划分访问控制: 更精确地控制成员变量, 保证约束条件成立
- 封装: 通过函数可以改变实现方法, 而不改变接口

总结:

- 将成员变量声明为 private, 以确保语法一致性, 可细微划分访问控制, 允诺约束条件获得保证, 并提供 class 作者以充分的实现弹性
- protected 并不比 public 更具封装性

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

- 宁可拿 non-member, non-friend 函数替换 member 函数, 以提升封装性, 包裹弹性, 机能扩充性

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

- 如果某个函数的所有参数 (包括被 this 指针所指的那个隐喻参数) 都需要类型转换, 那么它必须是个 non-member 函数

### 25 考虑写出一个不抛异常的 swap 函数

swap 函数原本只是 STL 的一部分, 而后成为异常安全性编程的脊柱, 并可用来处理自赋值问题

默认的 swap 函数平平无奇:

```cpp
template<typename T>
void swap(T& a, T& b)
{
    T temp(a);
    a = b;
    b = temp;
}
```

对于 "以指针指向一个对象, 内含真正的数据" 的类, 常见的表现形式是 pimpl 手法 (pointer to implementation):

```cpp
class WidgetImpl {
public:
    ...
private:
    int a, b, c;
    std::vector<double> v;
    ...
};

class Widget {  // pimpl 手法
public:
    Widget(const Widget& rhs);
    Widget& operator=(const Widget& rhs)
    {
        ...
        *pImpl = *(rhs.pImpl);
        ...
        return *this;
    }
    ...
private:
    WidgetImpl* pImpl;
};
```

对于这种类, 只需要交换 pImpl 指针即可, 尽管我们不被允许改变 std 命名空间内的任何东西, 但是可以为标准 templates 制造特化版本, 标准的做法是在类内实现一个 public swap 函数, 然后在类外实现一个非成员的全特化 swap 函数, 以调用类内的 swap 函数:

```cpp
class Widget {
public:
    ...
    void swap(Widget& other)
    {
        using std::swap;
        swap(pImpl, other.pImpl);
    }
    ...
};

namespace std {
    template<>
    void swap<Widget>(Widget& a, Widget& b)
    {
        a.swap(b);
    }
}
```

而当 WidgetImpl 和 Widget 都是模板类时, 上述方法不适用, function template 不允许被偏特化, 但可以进行重载， 即声明一个 non-member swap 函数

```cpp
template<typename T>
class WidgetImpl { ... };
template<typename T>
class Widget { ... };

namespace std {
    // wrong
    template<typename T>
    void swap<Widget<T>>(Widget<T>& a, Widget<T>& b)
    {
        a.swap(b);
    }

    // right but not standard
    template<typename T>
    void swap(Widget<T>& a, Widget<T>& b)
    {
        a.swap(b);
    }
}
```

但是注意标准规定不准在 std 命名空间内添加任何东西, 所以标准的做法是将 swap 函数放在和 class 相同的命名空间中, 例如:

```cpp
namespace WidgetStuff {
    template<typename T>
    class WidgetImpl { ... };
    template<typename T>
    class Widget { ... };

    template<typename T>
    void swap(T& a, T& b)
    {
        a.swap(b);
    }
}
```

此时根据 C++ 的名称查找规则 (argument-dependent lookup, ADL 或者 Koenig lookup) 会找到 WidgetStuff 内的专属 swap 函数

值得注意的是, 对于类而言, 如果希望专属 swap 函数在尽可能多的情况下被调用, 需要同时在该 class 的命名空间内写一个 non-member swap 函数, 以及在 std 命名空间内写一个全特化的 swap 函数, 原因是让下面的函数调用都能找到专属 swap 函数:

```cpp
template<typename T>
void doSomething(T& lhs, T& rhs)
{
    using std::swap;    // make std::swap available
    swap(lhs, rhs);     // call the best version of swap
    // std::swap(lhs, rhs);  // only call swap in std namespace
}
```

首先如果 swap 的缺省实现对你的 class 或者 template class 提供可接受的效率, 则无需做额外的事情; 其次如果效率不足 (总是意味着你的 class 或 template class 使用了某种 pimpl 手法)需要做下面的事情

1. 提供一个 public swap 成员函数, 高效的交换两个对象的内部数据, 注意这个函数不应该抛出异常
2. 在你的 class 或者 template 所在的命名空间内提供一个 non-member swap 函数, 以调用上述 swap 成员函数
3. 如果正在写一个 class 则为这个 class 特化 std::swap, 以调用上述 swap 成员函数

最后, 如果你调用 swap 函数, 请确定包含一个 using 声明, 以便让 std::swap 在你的函数内曝光, 然后不加用任何 namespace 修饰符, 直接使用 swap 函数

总结:

- 当 std::swap 对你的类型效率不高时, 提供一个 swap 成员函数, 并确定这个函数不抛出异常
- 如果你提供一个 member swap 函数, 也提供一个 non-member swap 函数, 以调用 member swap 函数, 对于 class (非 template) 还要特化 std::swap
- 调用 swap 时应针对 std::swap 使用 using 声明式, 然后调用 swap 并且不带任何 namespace 修饰符
- 为 "用户定义类型" 进行 std templates 全特化是好的, 但是不要尝试在 std 内加入全新的东西

## 5 实现

### 26 尽可能延后变量定义式的出现时间

过早出现的变量定义式可能导致构造和析构函数被调用, 从而导致效率降低, 此外, 使用默认构造函数初始化变量可能会导致不必要的初始化开销

尽可能延后不仅仅指的是延后定义, 直到非得使用该变量时, 还指尝试延后到能够给出初始值为止

```cpp
std::string encryptPassword(const std::string& password)
{
    using namespace std;
    // string encrypted; // too early
    if (password.length() < minimumPasswordLength) {
        throw std::logic_error("Password is too short");
    }
    // string encrypted;   // use default constructor
    string encrypted(password); // ok
    ...
    return encrypted;
}
```

对于循环有两种定义方法, 各自的成本如下:

```cpp

// A. 在循环外定义
// 成本: 1 个构造函数调用, 1 个析构函数调用, n 个赋值操作
Widget w;
for (int i = 0; i < n; ++i) {
    w = 和 i 相关的值;
}

// B. 在循环内定义
// 成本: n 个构造函数调用, n 个析构函数调用
for (int i = 0; i < n; ++i) {
    Widget w(和 i 相关的值);
}
```

如果一个赋值操作的成本低于一个构造函数加上一个析构函数的成本, A 的成本更低, 否则 B 的成本更低, 同时 A 造成名称的作用域比 B 更大, 有时会影响代码的可理解性和易维护性, 所以除非:

1. 知道赋值成本比构造加析构成本低
2. 正在处理代码中效率高敏感 (performance-sensitive) 的部分

否则都应该选择 B

总结:

- 尽可能延后变量定义式的出现时间, 这样做可增加程序清晰度并改善程序效率

### 27 尽量少做转型动作

常见的转型动作有:

```cpp
// old style cast
(T) expression; // C 风格转型
T (expression);  // 函数风格转型

// new style cast (C++-style cast)
const_cast<T>(expression);
dynamic_cast<T>(expression);
reinterpret_cast<T>(expression);
static_cast<T>(expression);
```

新式转型各自的作用是:

- const_cast: 用来移除对象的 const 性质, 新式转型中唯一能够移除 const 性质的转型
- dynamic_cast: 用来执行安全向下转型 (downcast), 也就是决定某对象是否归属于继承体系中的某个特定类型, 唯一无法由旧式转型执行的转型, 也是唯一可能耗费重大运行成本的转型
- reinterpret_cast: 用来执行低级转型, 实际动作可能取决于编译器
- static_cast: 强迫隐式转换, 但是无法将 const 转为 non-const

新式转型的优点是:

- 更容易被识别
- 转型动作更加专门化

唯一需要旧式转型的情况是, 调用一个 explicit 构造函数将一个对象传递给一个函数, 但是这种情况下最好还是使用 C++ 风格的强制类型转换

```cpp
class Widget {
public:
    explicit Widget(int size);
    ...
};

void f(const Widget& w);

f(Widget(10));              // function style
f(static_cast<Widget>(10)); // C++ style
```
任何一个类型转换都会导致编译器生成额外的代码, 例如:

```cpp
int x = 10;
double d = static_cast<double>(x);

class Base { ... };
class Derived : public Base { ... };
Derived d;
Base* pb = &d;
```

上述代码中, 为了使 Base 指针指向 Derived 对象可能会在 Derived 对象的基础上加上一个偏移量, 这导致额外代码的产生, 也表明了应该避免做出 "对象在 C++ 中如何布局" 的假设, 更不应该以此为基础进行转型

对 `this` 执行转型动作后产生的对象并非原对象, 而是一个新的临时副本对象, 例如:

```cpp
class Window {
public:
    virtual void onResize() { ... }
};

class SpecialWindow : public Window {
public:
    virtual void onResize() {
        // static_cast<Window>(*this).onResize();  // wrong 如果对数据进行修改, 会修改的是临时对象
        Window::onResize();
        ...
    }
    void blink();
};
```

由于 dynamic_cast 的执行成本较高, 而使用 dynamic_cast 往往是因为想在认定为 derived class 对象上执行 derived class 的操作, 但是只有指向 "base" 的 pointer 或者 reference, 有两个办法可以避免这个问题:

#### 使用类型安全容器

使用容器并在其中储存直接指向 derived class 对象的指针, 但这样做无法在同一个容器内储存不同类型的对象

```cpp
// unefficient
typedef std::vector<std::shared_ptr<Window>> VPW;
VPW winPtrs;
...
for (VPW::iterator it = winPtrs.begin(); it != winPtrs.end(); ++it) {
    if (SpecialWindow* psw = dynamic_cast<SpecialWindow*>(it->get())) {
        psw->blink();
    }
}

// efficient
typedef std::vector<std::shared_ptr<SpecialWindow>> VPSW;
VPSW winPtrs;
...
for (VPSW::iterator it = winPtrs.begin(); it != winPtrs.end(); ++it) {
    (*it)->blink();
}
```

#### 使用 virtual 函数

通过 base 接口处理所有可能的 derived class 对象, 也就是提供 virtual 函数

```cpp
class Window {
public:
    virtual void blink() {}
};

class SpecialWindow : public Window {
public:
    virtual void blink() { ... }
};

...
for (VPW::iterator it = winPtrs.begin(); it != winPtrs.end(); ++it) {
    (*it)->blink();
}
```

上述两种方法不是万能的, 如果 dynamic_cast 是必要的, 则应该使用, 但是应该尽量避免使用连串的 dynamic_cast, 这样导致代码运行缓慢, 且基础不稳:

```cpp
for (VPW::iterator it = winPtrs.begin(); it != winPtrs.end(); ++it) {
    if (SpecialWindow* psw = dynamic_cast<SpecialWindow1*>(it->get())) {
        ...
    } else if (SpecialWindow2* psw = dynamic_cast<SpecialWindow2*>(it->get())) {
        ...
    } else if (SpecialWindow3* psw = dynamic_cast<SpecialWindow3*>(it->get())) {
        ...
    }
}
```

优良的 C++ 代码很少使用转型, 但是完全避免也不现实, 应该尽可能隔离转型, 通常是把它隐藏在某个函数内, 从而使用函数接口保护代码

#### 总结

- 尽量避免转型, 特别是在注重效率的代码中避免 dynamic_cast, 如果转型是必要的试着进行替代设计
- 如果转型是必要的, 试着将它隐藏在某个函数背后
- 使用 C++-style cast 而不是 old-style cast

### 28 避免返回 handles 指向对象内部成分

- 成员变量的封装性最多只等于 "返回其 reference" 的函数的访问级别, 函数不当地返回 reference 会导致封装性下降
- 如果 const 函数传出一个 reference, 如果这个数据与对象自身有关联又被存储于对象之外, 那么函数的调用者可以修改这笔数据, 这是 bitwise constness 的一个附带效果

以上的情况对于指针或者迭代器同样适用, 这些被统称为 handles (号码牌, 用于取得某个对象)

将返回的 handles 加上 const 可解决上述的问题, 但是仍然可能产生 dangling handles (空悬的号码牌) 问题, 最常见的情况是函数返回值

总结:

- 避免返回 handles (包括 reference, 指针, 迭代器) 指向函数内部, 遵守该条款可增加封装性, 帮助 const 成员函数的行为一致, 并将发生 dangling handles 的可能性降至最低

### 29 为 "异常安全" 而努力是值得的

当异常被抛出时, 带有异常安全性的函数会:

- 不泄漏任何资源
- 不允许数据损坏

可以通过对象管理资源的方式解决资源泄漏的问题

异常安全函数提供下面三个保证之一:

- 基本承诺: 如果异常被抛出, 程序内的任何事物仍然保持在有效状态, 但是不保证程序的现实状态
- 强烈保证: 如果异常被抛出, 程序状态不改变, 就像这个函数从未被调用一样
- 不抛掷 (no throw) 保证: 承诺绝不抛出异常

使用智能指针往往可以帮助提供强烈保证, copy and swap 也可以实现: 为打算修改的对象做出副本, 然后在副本身上进行修改, 最后通过不抛出异常的 swap 进行置换, 为了使用不抛出异常的 swap 往往把数据放入另一个类 (实现对象) 中, 然后在源对象在设置一个指针指向实现对象, 这被称为 pimpl idiom 手法

但是要注意, 当进行 copy and swap 的函数内存在其他的函数, 则此函数的异常安全性与这些函数相关, 即使这些函数也提供强烈保证也可能导致安全性下降, 此外, copy and swap 会导致效率下降, 所以在强烈保证无法完成之时, 需要考虑提供基本保证, 而如果使用的函数无法提供任何异常安全性保证, 那么使用此函数的代码也无法提供安全性保证, 这将导致整个系统无法提供异常安全性保证

总结:

- 异常安全函数即使发生异常也不会泄漏资源或者允许任何数据结构损坏, 它有三种可能的保证: 基本, 强烈, 不抛异常
- 强烈保证往往可以通过 copy and swap 实现, 但是强烈保证并非对所有函数都可实现或具备显示意义
- 函数提供的异常安全性通常最高只等于所调用之各个函数的异常安全保证的最弱者

### 30 透彻了解 inline 的里里外外

inline 不仅可以免除函数调用的开销, 让编译器对插入的代码进行优化, 但是 inline 会导致代码体积增大, 从而增加缓存未命中的可能性, 从而导致性能下降. 不过, 如果使用 inline 后生成的代码甚至比函数调用的开销还要小, 那么这个时候 inline 就是一个很好的选择.

记住, inline 是一种向编译器提供的申请, 不是强制性的要求, 申请的方式有隐喻的和显式的两种, 喻的方式是将函数定义放在类定义中 (包括成员函数和 friend 函数), 显式的方式是在函数定义前加上 inline 关键字:

```cpp
// 隐喻的
class A {
public:
    void f() { /* ... */ }          // 隐喻的成员函数 inline
    friend void g() { /* ... */ }   // 隐喻的 friend 函数 inline
};

// 显式的
inline void h() { /* ... */ }       // 显式的 inline 函数
```

inline 函数通常一定置于头文件中, 因为大多数建制环境在编译时进行 inlining, 这要求编译器知道函数的定义, 只有少数编译器支持链接时 inlining 甚至是运行时 inlining (基于 .NET CLI 的托管环境), 而 Template 函数与之类似, 所以往往也要将 Template 函数的定义放在头文件中, 但是要注意 inline 和 Template 并不是一回事, 不要误认为 inline 函数必须声明为 Template 函数.

再次强调, inline 是一种向编译器提供的申请, 不是强制性的要求, 编译器可以选择忽略这个申请, 比如当函数过于复杂时, 编译器可能会选择不进行 inlining; 编译器会拒绝 virtual 函数的 inline 申请, 因为 virtual 函数的调用是动态绑定的, 总之看似 inline 的函数是否会被 inline 取决于建制环境, 打开相应的警告选项可以查看编译器是否进行了 inline.

即使函数可以被 inline, 编译器可能还是要生成函数的实体, 用于函数指针等操作, 这意味着通过函数指针调用 inline 函数时可能不会进行 inlining

构造函数和析构函数往往不适合 inline, 因为看上去简单的构造函数和析构函数可能会自动由编译器生成复杂的代码

一旦 inline 函数发生了变化, 所有调用这个函数的地方都要重新编译, 而 non-inline 只需要重新链接, 对于动态连接甚至不需要任何修改

大部分调试器不支持 inline 函数的调试

所以决定某函数是否 inline 时e可以先不把任何函数声明为 inline 或者只将那些十分短小的函数声明为 inline, 最后通过分析将合适的函数声明为 inline 达到优化的目的 (优化过程参考 28 法则).

总结:

- 将大多数 inline 函数限制在小型, 被频繁调用的函数上, 这可使日后的调试和二进制升级更容易, 也可使潜在的代码膨胀问题最小化, 程序速度提升机会最大化
- 不要因为 finction templates 出现在头文件就将它们声明为 inline

### 31 将文件间的编译依存关系降至最低

编译依存关系是指一个文件的修改导致其他文件的重新编译, 为了降低编译依存关系, 可以使用前置声明 (forward declaration), 但在 C++ 中仍存在一些问题, 所以需要将对象实现隐藏于指针之后, 这被称为 pimpl idiom (pointer to implementation), 这种 class 内指针往往称为 pImpl:

```cpp
#include <string>
#include <memory>

class PersonImpl;  // forward declaration
class Date;
class Address;

class Person {
public:
    Person(const std::string& name, const Date& birthday, const Address& addr);
    std::string name() const;
    std::string birthDate() const;
    std::string address() const;
    ...
private:
    std::shared_ptr<PersonImpl> pImpl;
};
```

这个分离的关键在于以声明的依存性替换定义的依存性

- 如果可以使用指针或引用, 就不要使用对象
- 如果能够, 尽量以 class 声明式替换 class 定义式: 即使声明的函数返回某 class 或者以传值的方式接收某 class, 也可以只包含 class 的声明式
- 为声明式和定义式提供不同的头文件

```cpp
// class Date;
#include "datefwd.h"
Date today();
void clearAppointments(Date d);
```

使用 pimpl idiom 的 class 被称为 Handle classes, 它们可以将所有函数转交给相应的实现类完成实际的工作

```cpp
#include "Person.h"
#include "PersonImpl.h"

Person::Person(const std::string& name, const Date& birthday, const Address& addr)
    : pImpl(new PersonImpl(name, birthday, addr))
{}

Person::~Person()
{
    delete pImpl;
}

std::string Person::name() const
{
    return pImpl->name();
}
```

另一种制作 Handle classes 的方法是令 Handle classes 成为一种特殊的 abstract base classes (抽象基类), 被称为 Interface classes

Interface class 的使用者为了使用这个 class 可以使用 factory (工厂) 函数 (又称 virtual 构造函数) 来创建该类的 derived classes, 它们返回指针 (或者智能指针) 指向被动态分配取得的对象, 这样的函数往往被声明为 static 的.

```cpp
class Person {
public:
    virtual ~Person();
    virtual std::string name() const = 0;
    virtual std::string birthDate() const = 0;
    virtual std::string address() const = 0;
    ...
    static std::shared_ptr<Person> create(const std::string& name, const Date& birthday, const Address& addr);
};

class RealPerson : public Person {
    ...
};

std::shared_ptr<Person> Person::create(const std::string& name, const Date& birthday, const Address& addr)
{
    return std::shared_ptr<Person>(new RealPerson(name, birthday, addr));
}

// usage
std::shared_ptr<Person> pp(Person::create(name, birthday, addr));
```

更现实的 create 会创建不同类型的 derived classes

Interface class 的最常见的机制有: 1. 从 Interface class 继承接口规格然后实现出接口所覆盖的函数; 2. 多重继承

Handle classes 和 Interface classes 解除了接口和实现之间的依赖关系, 但是这种解耦会导致一些开销:

- Handle classes 会导致额外的间接寻址
- Handle classes 的 implementation pointer 的内存开销和初始化开销
- Interface classes 中函数调用时因 virtual 函数而导致的开销和 vptr 的内存开销
- 二者依赖 inline 函数

要在代码效率和文件间依赖关系之间取得平衡

总结:

- 支持编译依存性最小化的一般构想是: 相依于声明式而非定义式, 基于此构想的具体手段包括 Handle classes 和 Interface classes
- 程序库头文件应该以完全且仅有声明式 (full and declaration-only forms) 的形式存在, 不管是否涉及 templates

## 6 继承与面向对象设计

### 32 确定你的 public 继承塑模出 is-a 关系

public 继承意味着 is-a 关系, 也就是子类是基类的一种类型, 如果让 class D 以 public 方式继承 class B, 那么每一个 D 类型都是一个 B 类型, B 比 D 表达更一般化的概念, 而 D 比 B 表达更特殊化的概念, 所以任何接收 B 类型的地方 (包括 pointer-to-B 和 reference-to-B) 都可以接收 D 类型, 但注意上述论述只适用于 public 继承

注意在 C++ 中的继承代表了能施于 Base class 的所有事物都能施于 Derived class, 这可能与现实世界的 is-a 关系存在差别, 对于 Derived class 无法施行的函数有三种处理方式:

1. 使函数抛出运行期异常
2. 不为 Derived class 提供该函数
3. 重新设计继承体系

is-a 并非存在于 class 之间唯一的关系, 还有 has-a (持有) 关系以及 is-implemented-in-terms-of (根据某物实现出) 关系, 于条款 38 和 39 讨论

总结:

- public 继承意味着 is-a 关系, 适用于 Base class 的所有事物都能施于 Derived class, 因为每一个 Derived class 对象也是一个 Base class 对象

### 33 避免遮掩继承而来的名称

本题材事实上和作用域有关, 如下面的代码:

```cpp
int x;
void someFunc()
{
    double x;
    ...
}
```

在 someFunc 中, x 会遮掩全局的 x, 作用域如下图所示:

![scope](./image/Effective%20C++.assets/scope.png)

引入继承后的代码如下:

```cpp
class Base {
    int x;
public:
    virtual void mf1() = 0;
    virtual void mf2();
    void mf3();
};

class Derived : public Base {
public:
    virtual void mf1();
    void mf4();
};
```

作用域如下图所示:

![class_scope](./image/Effective%20C++.assets/class_scope.png)

C++ 查找名称的顺序是:

1. 查找 local 作用域
2. 查找外围作用域, 也就是 Derived class 的作用域
3. 继续向外围移动, 也就是沿着继承体系向上移动, 直到找到 Base class 的作用域
4. 查找对应的命名空间
5. 查找全局作用域

加入重载后的代码如下:

```cpp
class Base {
    int x;
public:
    virtual void mf1() = 0;
    virtual void mf1(int);
    virtual void mf2();
    void mf3();
    void mf3(double);
};

class Derived : public Base {
public:
    virtual void mf1();
    void mf3();
    void mf4();
};
```

作用域如下图所示:

![class_scope2](./image/Effective%20C++.assets/class_scope2.png)

在这里所有 mf1 和 mf3 都会遮掩 Base class 的同名函数, 也就是 Base Class 内同名函数不会被继承, 为了继承 Base class 内的同名函数, 可以使用 using 声明式, 这意味着如果继承 base class 并加上重载函数, 而你又希望重新定义或覆写一部分, 那么必须为每一个被遮掩的函数加上 using 声明式, 由于 base class 内是 public 的, 所以 derived class 内需要在 public 区域内加上 using 声明式:

```cpp
class Derived : public Base {
public:
    using Base::mf1;
    using Base::mf3;
    virtual void mf1();
    void mf3();
    void mf4();
};
```

尽管从定义上讲不允许不继承 public 函数, 但是不继承 private 函数是可以的, 这时如果希望继承某一 private 函数而不继承其重载函数可以使用转交函数

```cpp
class Derived : private Base {
public:
    virtual void mf1(int) { Base::mf1(); }
};
```

总结:

- derived class 内的名称会遮掩 base class 内的名称, 这和 public 继承的期望不符
- 为了让 base class 内的名称在 derived class 内可见, 可以使用 using 声明式或转交函数

### 34 区分接口继承和实现继承

一个 base class 将影响它的继承类:

- 成员函数的接口总是会继承
- 声明一个 pure virtual 函数的目的是为了让 derived class 只继承函数接口
- 声明一个简朴的 (非纯) impure virtual 函数的目的是为了让 derived class 继承该函数接口和缺省实现
- 声明一个 non-virtual 函数的目的是为了让 derived class 继承该函数的接口和强制性实现, 这代表函数的不变性凌驾于特异性

为了避免 derived class 缺省继承 impure virtual 函数可以将 virtual 函数改为 pure virtual 函数同时在 Base class 中提供缺省实现, 为了继承该实现 derived class 可以使用一个 inline 函数调用实现

```cpp
class Airport { ... };

class Airplane {
public:
    virtual void fly(const Airport& destination) = 0;
    virtual void error(const string& message);
    int objectID() const;
    ...
protected:
    void defaultFly(const Airport& destination);
};

void Airplane::defaultFly(const Airport& destination)
{
    ...
}

class ModelA : public Airplane {
public:
    // 继承接口和缺省实现
    virtual void fly(const Airport& destination) { Airplane::defaultFly(destination); }
};

class ModelC : public Airplane {
public:
    // 继承接口
    virtual void fly(const Airport& destination);
};
```

也可以借助 "pure virtual 函数可以拥有自己的实现" 解决上述问题的同时避免污染命名空间, 不过会导致无法让两个函数使用不同的保护级别

```cpp
class Airplane {
public:
    virtual void fly(const Airport& destination) = 0;
    ...
};

void Airplane::fly(const Airport& destination)
{
    ...
}

class ModelA : public Airplane {
public:
    // 继承接口和缺省实现
    virtual void fly(const Airport& destination) { Airplane::fly(destination); }
};
```

总结:

- 区分接口继承和实现继承, 在 public 继承下, derived class 总是继承 base class 的接口
- pure virtual 函数只具体指定接口继承
- 简朴的 (非纯) impure virtual 函数具体指定接口继承和缺省实现继承
- non-virtual 函数具体指定接口继承和强制性实现继承

### 35 考虑 virtual 函数以外的其他选择

#### 使用 virtual 函数

```cpp
class GameCharacter {
public:
    virtual int healthValue() const;
    ...
};
```

#### 籍由 Non-Virtual Interface (NVI) 手法实现 *Template Method* 模式

这一思想流派认为 virtual 函数应该几乎总是 private, 所以应该将 healthValue 保留为 public non-virtual 函数, 并调用一个 private virtual 函数

```cpp
class GameCharacter {
public:
    int healthValue() const
    {
        ...
        int result = doHealthValue();
        ...
        return result;
    }
    ...
private:
    virtual int doHealthValue() const;
};
```

NVI 手法的一个优点是可以在 public 函数中加入一些额外的操作, 这样 class 的设计者保证了 virtual 函数 "何时被调用", 而继承类则实现 "如何完成函数"

#### 藉由 Function Pointers 实现 *Strategy* 模式

这一设计主张计算函数和类型无关, 所以可以要求每个类型的构造函数接受一个函数指针, 这样就可以在构造函数中指定函数

```cpp
class GameCharacter;

int defaultHealthCalc(const GameCharacter& gc);

class GameCharacter {
public:
    typedef int (*HealthCalcFunc)(const GameCharacter&);
    explicit GameCharacter(HealthCalcFunc hcf = defaultHealthCalc) : healthCalcFunc(hcf) {}
    int healthValue() const { return healthCalcFunc(*this); }
    ...
private:
    HealthCalcFunc healthCalcFunc;
};
```

这种 ***Strategy*** 模式可以带来如下优点:

- 同一类型下的不同实体可以有不同的实现函数
- 实现函数可以在运行期内变更

但是注意, 这种情况下的实现函数无法访问 non-public 成员, 唯一能解决问题的办法是弱化 class 的封装如声明为 friend 或者提供所需 non-public 成分的访问函数

#### 藉由 std::function 实现 *Strategy* 模式

注: C++11 中引入了 `std::function`, 用于替代 `tr1::function`

使用 `std::function` 代替函数指针, 使得构造函数可以接受任何可调用对象, 例如函数指针, 函数对象, lambda 表达式, 经过 `std::bind` 处理的成员函数等, 只要与 `std::function` 的签名式 (target signature) 兼容即可

```cpp
#include <functional>

class GameCharacter {
public:
    typedef std::function<int (const GameCharacter&)> HealthCalcFunc;
    explicit GameCharacter(HealthCalcFunc hcf = defaultHealthCalc) : healthCalcFunc(hcf) {}
    int healthValue() const { return healthCalcFunc(*this); }
    ...
private:
    HealthCalcFunc healthCalcFunc;
};
```

函数指针, 函数对象, 成员函数各自的使用方法如下:

```cpp
short calcHealth(const GameCharacter& gc); // returns short

struct HealthCalculator {
    int operator()(const GameCharacter& gc) const;
};

class GameLevel {
public:
    float health(const GameCharacter& gc) const;
};

GameCharacter g1(calcHealth);
GameCharacter g2(HealthCalculator());
GameLevel level;
GameCharacter g3(std::bind(&GameLevel::health, level, std::placeholders::_1));
```

#### 古典的 *Strategy* 模式

在设计模式中传统的 *Strategy* 模式是将实现函数做成一个分离的继承体系中的 virtual 成员函数, 结果如下:

![strategy](./image/Effective%20C++.assets/strategy.png)

```cpp
class GameCharacter;

class HealthCalcFunc {
public:
    virtual int calc(const GameCharacter& gc) const;
    virtual ~HealthCalcFunc();
};

HealthCalcFunc defaultHealthCalc;

class GameCharacter {
public:
    explicit GameCharacter(HealthCalcFunc* phcf = &defaultHealthCalc) : pHealthCalc(phcf) {}
    int healthValue() const { return pHealthCalc->calc(*this); }
    ...
private:
    HealthCalcFunc* pHealthCalc;
};
```

#### 摘要

- 使用 non-virtual interface (NVI) 手法是实现 *Template Method* 模式的一种特殊情况, 它以 non-virtual public 函数调用包裹较低访问性 (private 或 protected) 的 virtual 函数
- 将 virtual 函数替换为函数指针成员变量是 *Strategy* 模式的一种分解表现形式
- 以 std::function 成员变量替换 virtual 函数从而允许任何可调用事物 (callable entities) 搭配兼容于需求的签名式也是 *Strategy* 模式的一种分解表现形式
- 将 virtual 函数替换为另一个继承体系中的 virtual 函数是 *Strategy* 模式的一种传统表现形式

#### 总结

- virtual 函数的替代方案包括 NVI 手法及 *Strategy* 设计模式的多种形式, NVI 手法自身是一个特殊形式的 *Template Method* 模式
- 将功能从成员函数移到 class 外部函数带来的一个缺点是非成员函数无法访问 class 的 non-public 成员
- std::function 对象的行为就像一般函数指针, 这样的对象可以容纳与给定的目标签名式兼容的任何可调用物

### 36 绝不重新定义继承而来的 non-virtual 函数

由于 non-virtual 函数是静态绑定的, 而 virtual 函数是动态绑定的, 所以下面的代码中通过不同的指针 (或引用) 调用同一个对象的同一个函数会产生不同的结果, 而不管是条款 32 中的 is-a 关系还是条款 34 中的不变性凌驾于特异性的讨论都认为这种情况不该发生:

```cpp
class Base {
public:
    void mf() { ... }
    ...
};

// 重新定义 non-virtual 函数
class Derived : public Base {
public:
    void mf() { ... }
    ...
};

Derived x;

Base* pb = &x;
pb->mf(); // 调用 Base::mf

Derived* pd = &x;
pd->mf(); // 调用 Derived::mf
```

总结:

- 绝不重新定义继承而来的 non-virtual 函数

### 37 绝不重新定义继承而来的缺省参数值

由于仅可以重新定义 virtual 函数所以本章等价为 "绝不重新定义继承而来的 virtual 函数的缺省参数值", 原因在于 virtual 函数的缺省参数值是静态绑定的, 而 virtual 函数是动态绑定的

```cpp
class Shape {
public:
    enum ShapeColor { Red, Green, Blue };
    virtual void draw(ShapeColor color = Red) const = 0;
    ...
};

class Rectangle : public Shape {
public:
    virtual void draw(ShapeColor color = Green) const;
    ...
};

Rectangle r;
Shape* ps = &r; // 静态类型为 Shape*, 动态类型为 Rectangle*
ps->draw(); // 调用 Rectangle::draw, 但是参数值为 Red
```

利用 NVI 手法可以避免这个问题, 通过将缺省参数值放入 non-virtual 函数中, 然后调用 virtual 函数, 由于 non-virtual 函数不能被重新定义, 所以 virtual 函数的缺省参数值也就不会被重新定义

```cpp
class Shape {
public:
    enum ShapeColor { Red, Green, Blue };
    void draw(ShapeColor color = Red) const { doDraw(color); }
    ...
private:
    virtual void doDraw(ShapeColor color) const = 0;
};

class Rectangle : public Shape {
public:
    virtual void doDraw(ShapeColor color) const;
    ...
};
```

总结:

- 绝不重新定义继承而来的缺省参数值, 因为缺省参数值是静态绑定的, 而 virtual 函数是动态绑定的

### 38 通过复合塑模出 has-a 或 "根据某物实现出" 关系

当某种类型的对象内含它种类型的对象时, 这种关系被称为复合 (composition), 例如:

复合还有一些同义词, 包括 layering (分层), containment (包含), aggregation (聚合) 和 embedding (内嵌)

复合意味着 has-a 关系或者 is-implemented-in-terms-of (根据某物实现出) 关系, 当复合发生于应用域 (application domain) 的对象之间时, 通常是 has-a 关系, 当复合发生于实现域 (implementation domain) 的对象之间时, 通常是 is-implemented-in-terms-of 关系

```cpp
class Address { ... };
class PhoneNumber { ... };
class Person {  // 复合, has-a 关系
public:
    ...
private:
    std::string name;
    Address addr;
    PhoneNumber phone;
};
```

当利用 list 实现 set 时表现出 is-implemented-in-terms-of 关系而非 is-a 和 has-a 关系:

```cpp
template <typename T>
class Set {
public:
    bool member(const T& item) const;
    void insert(const T& item);
    void remove(const T& item);
    std::size_t size() const;
    ...
private:
    std::list<T> rep;   // 复合, is-implemented-in-terms-of 关系
};
```

总结:

- 复合的意义和 public 继承完全不同
- 在应用域中, 复合表示 has-a 关系, 在实现域中, 复合表示 is-implemented-in-terms-of 关系

### 39 明智而审慎地使用 private 继承

private 继承的行为特点:

- dierived class 不会被自动转换为 base class
- 从 private 继承的所有成员在 derived class 中都会变为 private

private 继承意味着 is-implemented-in-terms-of (根据某物实现出) 关系, 也就是只继承实现而不继承接口, 而这与复合有相同的含义, 但建议尽量使用复合而非 private 继承, 只有当需要访问 protected 成员和/或需要重新定义一个或多个 virtual 的时候或者在很激进的情况下节省空间才使用 private 继承, 原因如下:

- privete 复合可以阻止 derived class 修改接口
- 降低编译依存性
- 复合更加容易理解

但是要注意, 当需要重新定义 virtual 函数时, 复合也可以实现, 只是代码稍显复杂

很激进的情况下节省空间指的是处理的 class 不带任何数据时 (没有 non-static 成员变量, 没有 virtual 函数, 没有 virtual base classes), 这时使用复合由于 C++ 要求独立 (非附属) 对象的大小一定不为 0, 所以使用单一 private 继承可以节省空间, 这称为 EBO (Empty Base Optimization; 空白基类最优化), 但这种情况很少见

总结:

- private 继承意味着 is-implemented-in-terms-of (根据某物实现出) 关系, 它通常比复合的级别低, 但是当 derived class 需要访问 protected 成员或者重新定义继承的 virtual 函数时, private 继承是合适的选择
- 和复合相比, private 继承可以形成 empty base class 最优化, 这对某些开发者而言可能很重要

### 40 明智而审慎地使用多重继承

多重继承 (multiple inheritance; MI) 在 C++ 社群内的看法不一, 本节将介绍两方面的观点

首先要注意的是, MI 会导致较多的歧义 (ambiguity) 机会, 例如:

```cpp
class BorrowableItem {
public:
    void checkOut() { ... }
    ...
};

class ElectronicGadget {
private:
    bool checkOut() { ... }
    ...
};

class MP3Player :
    public BorrowableItem,
    public ElectronicGadget
{
    ...
};

MP3Player mp;
mp.checkOut();                  // 歧义
mp.BorrowableItem::checkOut();  // 明确
```

多重继承时要注意不要让继承的 base classes 有相同的更高级的 base class, 这样会导致 "钻石型多重继承问题":

```cpp
class File { ... };
class InputFile : public File { ... };
class OutputFile : public File { ... };
class IOFile :
    public InputFile,
    public OutputFile { ... };
```

继承图如下:

```mermaid
classDiagram
    File <|-- InputFile
    File <|-- OutputFile
    InputFile  <|-- IOFile
    OutputFile <|-- IOFile
```

这样会导致问题: IOFile 应该有 File 的 2 个成员变量还是 1 个成员变量? C++ 对两个情况都有支持, 而进行复制是缺省做法 (即有 2 个成员变量), 而如果希望各个 base classes 共享更高级的 base class, 那么在这个带数据的 base class 要成为 virtual base class

```cpp
class File { ... };
class InputFile : virtual public File { ... };
class OutputFile : virtual public File { ... };
class IOFile :
    public InputFile,
    public OutputFile { ... };
```

继承图如下:

```mermaid
classDiagram
    File <|.. InputFile
    File <|.. OutputFile
    InputFile  <|-- IOFile
    OutputFile <|-- IOFile
```

从正确性的角度来看 public 继承总应该是 virtual 的, 但是 virtual 继承会导致 class 的体积增大, 访问成员变量的运行速度变慢, 初始化的过程也更加复杂, 所以在使用 virtual 继承时要遵循以下建议:

- 除非绝对必要, 否则不要使用 virtual 继承
- 必须使用 virtual 继承时避免在其中放置数据

总结:

- 多重继承比单一继承更复杂, 他可能导致歧义性和对 virtual 继承的需求
- virtual 继承会增加大小, 速度, 初始化 (和赋值) 复杂度等成本, 所以 virtual base class 最好不要包含数据
- 多重继承的一个合理用途是 public 继承接口而 private 继承实现的结合

## 7 模板与泛型编程

### 41 了解隐式接口和编译期多态

总结:

- classes 和 templates 都支持接口 (interface) 和多态 (polymorphism)
- 对 classes 而言, 接口是显式 (explicit) 的, 以函数签名为中心, 多态则是通过 virtual 函数发生于运行期 (runtime)
- 对 templates 而言, 接口是隐式 (implicit) 的, 以有效表达式 (valid expressions) 为中心, 多态则是通过 template 实例化和函数重载解析 (function overloading resolution) 发生于编译期 (compile-time)

### 42 了解 typename 的双重意义

在 template 声明中 class 和 typename 完全相同, 但是 typename 还有其他用法:

```cpp
// same
template <typename T> class Widget;
template <class T> class Widget;
```

typename 内出现的名称如果相依于某个 template 参数, 称之为从属名称 (dependent names), 如果从属名称在 class 内呈嵌套状则称之为嵌套从属名称 (nested dependent names), 而如果嵌套从属名称指的是某一类型则称之为嵌套从属类型名称 (nested dependent type names), 而不依靠任何 template 参数的名称称之为非从属名称 (non-dependent names)

嵌套从属名称可能导致解析困难, 所以默认情况下编译器会认为嵌套从属名称是一个对象, 所以对于嵌套从属类型名称需要使用 typename 关键字, 只有后面提到的例外除外:

```cpp
template <typename C>
void print2nd(const C& container)
{
    if (container.size() >= 2) {
        // C::const_iterator iter(container.begin());       // 错误
        typename C::const_iterator iter(container.begin()); // 正确
        ++iter;
        int value = *iter;
        ...
    }
}
```

需要注意的是不是嵌套从属类型名称的情况下不需要使用 typename 关键字, typename 也不可以出现在 base classes list (基类列表) 内的嵌套从属类型名称前, 也不可以在 member initialization list (成员初始化列表) 内作为 base class 的修饰符

```cpp
template <typename T>
class Derived : public Base<T>::Nested {            // don't need typename
public:
    explicit Derived(int x) : Base<T>::Nested(x) {  // don't need typename
        typename Base<T>::Nested temp;              // need typename
    }
};
```

总结:

- 声明 template 参数时, 前缀关键字 class 和 typename 可互换
- 使用关键字 typename 标识嵌套从属类型名称, 但是不要在 base classes list 或者 member initialization list 以它作为 base class 的修饰符

### 43 学习处理模板化基类内的名称

由于全特化的存在, 编译器往往拒绝在 templatized base class (模板化基类) 内查找继承而来的名称, 有三种办法令这种行为失效:

- 在 base class 函数调用动作之前加上 this->
- 使用 using 声明式
- 明白指出被调用的函数位于 base class 内, 但这样会导致 virtual 函数的多态性失效

总结:

- 可在 derived class templates 内通过 this-> 指涉 base class templates 内的成员名称, 或者藉由一个明白写出的 "base class 资格修饰符" 完成 (using 声明式或者指出函数位于 base class 内)

### 44 将与参数无关的代码抽离 templates

为了避免代码膨胀 (code bloat), 需要进行共性与变性分析 (commonality and variability analysis), 也就是将存在共性的部分使用函数, 继承, 组合等方法抽离出来, 留下变性的部分, 但 template 的共性和变性和 non-template 相比不明显, 所以需要进行更加细致的分析

```cpp
template <typename T, std::size_t N>
class SquareMatrix {
public:
    ...
    void invert();
};
```

这个例子中不同的 N 将导致代码膨胀, 所以可以将 invert 函数抽离出来; 为了让 SquareMatrixBase::invert 知道操作什么数据可以让 SquareMatrixBase 存储一个指向数据的指针或者使用函数参数传递数据指针; derived class 则可以选择存储数据的方式

```cpp
template <typename T>
class SquareMatrixBase {
protected:
    SquareMatrixBase(std::size_t n, T* pMem) : size(n), pData(pMem) { }
    void setDataPtr(T* ptr) { pData = ptr; }
    ...
    void invert(std::size_t matrixSize);
private:
    std::size_t size;
    T* pData;   // 指向数据的指针
};

template <typename T, std::size_t N>
class SquareMatrix : private SquareMatrixBase<T> {
public:
    SquareMatrix() : SquareMatrixBase<T>(N, 0), data(new T[N * N]) { this->setDataPtr(data.get()); }
    ...
    void invert() { this->invert(N); }
private:
    boost::scoped_array<T> data;
};
```

使用 this 调用 invert 函数是因为在 templatized base class (模板化基类) 内查找继承而来的名称时编译器往往拒绝, 详见条款 43; 使用 private 继承是因为 base class 只是为了帮助 derived class 实现, 是 is-implemented-in-terms-of (根据某物实现出) 关系, 详见条款 39

但是要注意以上改变和原版比互有优劣, 原版代码可能通过常量广传等方法达到最优化而共享版本可能降低执行文件大小从而降低 working set [^working_set] 大小, 从而强化高速缓存内的引用集中化, 二者哪个更优取决于具体情况

[^working_set]: 在虚内存环境内执行的进程所使用的内存页的集合

对象的大小是另外需要关注的问题, 这与封装性相关

本条款只和 non-type template parameters (非类型模板参数) 有关, 但 type template parameters (类型模板参数) 也可能导致代码膨胀, 如 int 和 long 也许有相同的二进制描述却可能生成不同的实例化代码, 强型指针 (strongly typed pointers 即 T\*)也可能导致代码膨胀, 所以应该让它们调用无类型指针 (untyped pointers 即 void\*) 的函数完成实际工作

总结:

- Templates 生成多个 class 和多个函数, 所以任何 templates 代码都不应该与某个造成膨胀的 templates 参数产生相依关系
- 因非类型模板参数导致的代码膨胀可以消除, 做法是通过用函数参数或 class 成员变量替换 template 参数
- 因类型模板参数导致的代码膨胀可以降低 做法是让带有完全相同二进制表述的具现类型共享实现代码

### 45 运用成员函数模板接受所有兼容类型

如果不具体写出, 那么即使是具有继承关系的类通过模板产生的具现体之间也不能互相转换

#### Templates 和泛型编程 (Generic Programming)

通过模板构造函数可以实现可以实现任意类型的转换, 这种模板是所谓 member function templates (成员函数模板), 而这类构造函数被称为泛化 (generalized) copy 构造函数

为防止模板产生不期望的行为, 可以在模板中使用某些成员函数约束转换行为

```cpp
template <typename T>
class SmartPtr {
public:
    template <typename U>
    SmartPtr(const SmartPtr<U>& other) : heldPtr(other.get()) { ... }
    T* get() const { return heldPtr; }
    ...
private:
    T* heldPtr;
};
```

member function templates (成员函数模板) 除了构造函数还可以用于支持赋值操作

需要注意的是在 class 内声明泛化 copy 构造函数 (是 member template) 并不会阻止编译器生成类自己的 copy 构造函数 (non-template), 赋值操作也是如此, 所以如果希望额外控制类的 copy 构造函数和赋值操作, 需要自己声明

总结:

- 请使用 member function templates (成员函数模板) 生成 "可接受所有兼容类型" 的函数
- 如果声明 member templates 用于泛化 copy 构造函数或赋值操作, 还是需要声明正常的 copy 构造函数和赋值操作

### 46 需要类型转换时请为模板定义非成员函数

使用模板扩展条款 24 中的 Rational 类, 但由于 template 实参推导过程中不将通过构造函数发生的隐式转换考虑在内, 通过 templates class 内的 firend 函数可以指涉某个特定函数可以缓和上述的挑战, 但是需要注意的是此时无法将定义和实现分离, 否则无法通过链接

注意 friend 函数中使用 template 名称作为 "template 和其参数" 的简写, 这在 class template 内部是合法的

```cpp
template <typename T>
class Rational {
public:
    Rational(const T& numerator = 0, const T& denominator = 1); // Tips 20 说明了为什么这里使用 pass-by-reference
    const T numerator() const;      // Tips 28 说明了为什么返回值使用 pass-by-value
    const T denominator() const;    // Tips 03 说明为社么这里使用 const
    ...

    friend const Rational operator*(const Rational& lhs, const Rational& rhs)
    {
        return Rational(lhs.numerator() * rhs.numerator(), lhs.denominator() * rhs.denominator());
    }
};
```

如果不希望 firend 函数成为 inline 的, 可以让它调用一个外部辅助函数, 这样的话定义了 Rational 的头文件可能是这样的:

```cpp
template <typename T> class Rational;

template <typename T>
const Rational<T> doMultiply(const Rational<T>& lhs, const Rational<T>& rhs); // help function

template <typename T>
class Rational {
public:
    ...
    friend const Rational operator*(const Rational& lhs, const Rational& rhs)
    {
        return doMultiply(lhs, rhs);
    }
};

template <typename T>
const Rational<T> doMultiply(const Rational<T>& lhs, const Rational<T>& rhs)
{
    return Rational<T>(lhs.numerator() * rhs.numerator(), lhs.denominator() * rhs.denominator());
}
```

由于许多编译器要求把所有的 template 定义式写在头文件内, 所以上面把 doMultiply 定义在头文件内是合理的

总结:

- 当编写的 class template 提供的与此 class 相关的函数需要进行类型转换时, 请将这些函数定义为 "class template 内部的 friend 函数"

### 47 请使用 traits classes 表现类型信息

为了让编译期间可以取得某些类型信息需要使用 traits 技术, 他要求对内置类型和用户自定义类型都能正常工作, 所以标准技术是把它放进一个 template 及其一个或多个特化版本中, 它往往是一个 struct, 但这个 template 被称为 traits class

实现 traits class 的方法是:

1. 确认若干希望将来可取得的类型相关信息
2. 为该信息选择一个名称
3. 提供一个 template 和一组特化版本, 内含希望支持的类型信息

使用 traits class 的方法是:

1. 建立一组重载函数或函数模板, 彼此之间的差异仅在于各自的 traits class 参数, 令每个函数的实现码和对应的 traits class 特化版本相互关联
2. 建立一个控制函数或函数模板, 用于调用上述的重载函数或函数模板, 以及传递适当的 traits class 所提供的信息

std 中的 traits class 有:

- std::iterator_traits
- std::char_traits
- std::numeric_limits
- std::allocator_traits
- std::type_traits (C++11, 用于判断类型特性, 原位于 TR1)

总结:
- traits classes 使得类型相关信息在编译期间可用, 它们以 templates 和 template 特化完成实现
- 整合重载技术后, traits classes 有可能在编译期对类型进行 if...else 测试

### 48 认识 template 元编程

Template metaprogramming (TMP, 模板元编程) 是编写 template-based C++  程序并执行于编译期的过程, 有如下优点:

- 让某些事情更容易
- 将某些工作从运行期转移到编译期, 从而在编译期间发现错误, 也可以使生成的程序更加高效, 不过会导致编译时间变长

Tips 47 中的 traits classes 是 TMP 的一个例子, Boost 中也有针对 TMP 的库, 例如 Boost's MPL

TMP 中没有真正的循环构建所以需要使用递归, TMP 使用的是 "递归模板具现化" (recursive template instantiation), 下面是计算阶乘的例子:

```cpp
template <unsigned n>
struct Factorial {
    enum { value = n * Factorial<n - 1>::value };
};

template <>
struct Factorial<0> {
    enum { value = 1 };
};
```

TMP 的作用:

- 确保度量单位正确
- 优化矩阵运算
- 生成用户客制之设计模式 (design pattern) 实现品

TMP 或许永远不会成为主流, 但是对某些程序员如程序库开发人员是重要工具

总结:

- 模板元编程 (TMP) 可将工作从运行期转移到编译期, 因而得以实现早期错误侦测和更高的执行效率
- TMP 可被用来生成 "基于政策选择组合" 的客户定制代码, 也可以用来避免生成对某些特殊类型不适合的代码

## 8 定制 new 和 delete

### 49 了解 new-handler 的行为

当 operator new 无法满足内存需求时会抛出 std::bad_alloc 异常 (以前是返回 null 指针), 在抛出异常前会调用 new-handler 函数, 这是个用户通过调用 std::set_new_handler 注册的函数, 该函数的原型如下, 它将待注册的函数作为参数传入, 并返回被调用前使用的 new-handler 函数

```cpp
namespace std {
    typedef void (*new_handler)();
    new_handler set_new_handler(new_handler p) throw(); // 空白 throw() 表示不抛出异常
}
```

当 operator new 无法满足内存需求时会不断调用 new-handler 函数, 直到找到足够的内存, 所以 new-handler 函数要考虑下面的事情:

- 让更多内存可用
- 安装另一个 new-handler
- 卸除 new-handler
- 抛出 std::bad_alloc (或其派生类) 异常
- 不返回, 通常调用 abort 或 exit

通过为 class 定义 operator new 和 set_new_handler 可以实现类专属的 new-handler 并且需要一个 static 成员变量来保存当前的 new-handler, set_new_handler 将获得的指针储存并返回此前储存的指针, operator new 则需要做下面的事情:

1. 调用标准 set_new_handler, 告知 Widget 的错误处理函数, 这会将 Widget 的 new-handler 注册为 global new-handler
2. 调用 global operator new, 以尝试满足 Widget 的内存需求, 如果分配失败则会调用刚刚安装的 new-handler, 如果 global operator new 最终无法分配内存, 则会抛出 std::bad_alloc 异常, 这时 Widget 的 operator new 需要恢复原本的 new-handler 并传播异常, 为防止资源泄漏可以将 global new-handler 视为资源并运用资源管理对象
3. 如果 global operator new 成功分配内存, 则 Widget 的 operator new 返回一个指针指向分配所得, Widget 的析构函数会管理 global new-handler 将它被安装的 new-handler 恢复

```cpp
// Widget.h
class Widget {
public:
    ...
    static std::new_handler set_new_handler(std::new_handler p) throw();
    static void* operator new(std::size_t size) throw(std::bad_alloc);

private:
    static std::new_handler currentHandler;
};

// Widget.cpp

std::new_handler Widget::currentHandler = 0;

std::new_handler Widget::set_new_handler(std::new_handler p) throw()
{
    std::new_handler oldHandler = currentHandler;
    currentHandler = p;
    return oldHandler;
}

void* Widget::operator new(std::size_t size) throw(std::bad_alloc)
{
    NewHandlerHolder h(std::set_new_handler(currentHandler));
    return ::operator new(size);
}

// NewHandlerHolder.h
class NewHandlerHolder {
public:
    explicit NewHandlerHolder(std::new_handler nh) : handler(nh) {}
    ~NewHandlerHolder() { std::set_new_handler(handler); }

private:
    std::new_handler handler;
    NewHandlerHolder(const NewHandlerHolder&);              // 防止 copying
    NewHandlerHolder& operator=(const NewHandlerHolder&);   // 防止 assignment
};
```

将设定 class 专属之 new-handler 的能力抽离为一个 mixin 风格的 base class 并转化为 templates 可以让其他类继承接口并保证每个实体使用不同的 currentHandler

```cpp
template <typename T>
class NewHandlerSupport {
public:
    static std::new_handler set_new_handler(std::new_handler p) throw();
    static void* operator new(std::size_t size) throw(std::bad_alloc);
    ...

private:
    static std::new_handler currentHandler;
};

template <typename T>
std::new_handler NewHandlerSupport<T>::set_new_handler(std::new_handler p) throw()
{
    std::new_handler oldHandler = currentHandler;
    currentHandler = p;
    return oldHandler;
}

template <typename T>
void* NewHandlerSupport<T>::operator new(std::size_t size) throw(std::bad_alloc)
{
    NewHandlerHolder h(std::set_new_handler(currentHandler));
    return ::operator new(size);
}

// 将 currentHandler 初始化为 null
template <typename T>
std::new_handler NewHandlerSupport<T>::currentHandler = 0;

// Widget.h
class Widget : public NewHandlerSupport<Widget> {
public:
    ...
};
```

Widget 的这种继承方法被称为 curiously recurring template pattern (CRTP, 奇异递归模板模式), 可以把它理解为 Do it for me, 这种方法也很容易导致多重继承, 相关内容详见条款 40

为兼容此前返回 null 指针的 operator new, 可以使用 nothrow 形式

```cpp
Widget* pw = new (std::nothrow) Widget;
if (pw == 0) {
    ...
}
```

但是注意, 无法保证 Widget 的构造函数不抛出异常, 所以使用 nothrow new 只能保证 operator new 不抛掷异常, 所以没有必要使用 nothrow new

总结:

- set_new_handler 允许客户指定一个函数, 当 operator new 无法分配内存时会调用这个函数
- nothrow new 是一个颇为局限的工具, 它适用于内存分配, 后继的构造函数调用可能会抛出异常

### 50 了解 new 和 delete 的合理替换时机

替换 operator new 和 operator delete 的原因有:

- 检测运用上的错误
- 强化效能
  - 增加分配和归还的速度
  - 降低缺省内存管理器带来的额外空间开销
  - 弥补缺省内存管理器中的非最佳齐位 (suboptimal alignment)
  - 将相关对象成簇集中
  - 获得非传统的行为
- 收集使用上的统计资料

由于各种各样的技术细节问题 (如内存对齐等), 一般不建议替换 operator new 和 operator delete, 可行的替代是查看编译器是否可切换不同的状态满足需求, 购买某些商业产品并重新链接也是一个选择, 开放源码领域中的内存管理器如 Boost.Pool 也可以考虑

总结:

- 有许多理由需要自定的 operator new 和 operator delete, 包括改善效能, 对 heap 运用错误进行调试, 收集 heap 使用信息

### 51 编写 new 和 delete 时需固守常规

operator new 的正常行为包括:

- 返回正确的值: 如果分配成功则返回指向分配内存的指针, 如果分配失败则按照 Tips 49 中的方法处理, 并抛出一个 std::bad_alloc 异常
- 内存不足时调用 new-handler: new-handler 是 nullptr 时直接抛出异常, 否则调用 new-handler 完成处理工作
- 对付零内存需求的准备: 即使是要求 0 bytes 的内存也要返回一个合法的指针, 一般是把 0 bytes 当作 1 byte 处理
- 避免掩盖正常形式的 operator new

operator new 的伪码 (pseudocode) 如下:

```cpp
void* operator new(std::size_t size) throw(std::bad_alloc)
{
    using namespace std;
    if (size == 0)  // 处理 0 bytes 的内存需求
        size = 1;
    while (true) {
        if (void* mem = malloc(size))
            return mem;
        new_handler globalHandler = set_new_handler(0);
        set_new_handler(globalHandler);     // 获得当前 new-handler
        if (globalHandler)
            (*globalHandler)();             // 调用 new-handler
        else
            throw bad_alloc();              // 如果没有 new-handler 则抛出异常
    }
}
```

注意其中有一个 set_new_handler(0) 的调用, 这是为了获得当前的 new-handler, 而且有无穷循环, new-handler 必须如 Tips 49 中所述那样处理来打破循环

需要注意的是 operator new 会被 derived class 继承, 而 operator new 往往是为某大小特制的所以它很可能不适用于 derived class, 正确做法是增加一个内存申请量判断, 由于 C++ 要求非附属 (独立式) 对象的大小不为 0, 所以不必担心 `sizeof(Base) == 0` 的情况

```cpp
class Base {
public:
    static void* operator new(std::size_t size) throw(std::bad_alloc);
    ...
};

class Derived : public Base { ... };

void* Base::operator new(std::size_t size) throw(std::bad_alloc)
{
    if (size != sizeof(Base))
        return ::operator new(size);
    ...
}
```

如果打算控制 class 专属的 array 内存分配行为, 需要重载 operator new[], 也就是 array new, 这时唯一能做的是分配一块未加工内存 (raw memory)

对于 operator delete 只需要保证 C++ 的要求: 删除 null 指针永远安全, 下面是 operator delete 的伪码:

```cpp
void operator delete(void* rawMemory) throw()
{
    if (rawMemory == 0)
        return;
    free(rawMemory);
}
```

member 版本也很简单, 只需要多加一个动作检查删除数量, 当 operator new 转交给 ::operator new 时, operator delete 也应该转交给 ::operator delete

```cpp
class Base {
public:
    static void operator delete(void* rawMemory, std::size_t size) throw();
};

void Base::operator delete(void* rawMemory, std::size_t size) throw() {
    if (rawMemory == 0)
        return;
    if (size != sizeof(Base)) {
        ::operator delete(rawMemory);
        return;
    }
    free(rawMemory);
}
```

如果基类的析构函数不是 virtual 的那么传递给 operator delete 的 size 可能是错误的

总结:

- operator new 内应该包含一个无穷循环, 并在其中尝试分配内存, 如果分配失败则调用 new-handler, 也应该有能力处理 0 bytes 的内存需求, Class 的专属版本还应该处理 "比正确大小更大的 (错误) 申请"
- operator delete 内应该在收到 null 指针时不做任何事, Class 的专属版本还应该处理 "比正确大小更大的 (错误) 申请"

### 52 写了 placement new 也要写 placement delete

对于 `Widget* pw = new Widget;` 需要调用两个函数: operator new 和 Widget 的构造函数, 而如果第二个函数抛出异常, 第一个函数得到的内存必须被释放, 这一任务在 C++ 运行期系统身上, 它调用 operator new 对应的 operator delete, 对于下面的正常签名式 (signature) 的 operator new 和 operator delete 这不成问题:

```cpp
// 正常 operator new
void* operator new(std::size_t size) throw(std::bad_alloc);

// 正常 operator delete
void operator delete(void* rawMemory) throw();                      // global 作用域正常的签名式
void operator delete(void* rawMemory, std::size_t size) throw();    // class 作用域典型的签名式
```

而如果 operator new 接受的参数除了 size 之外还有其他参数便称为 placement new, 特别有用的一个版本是接受一个指针指向对象该被构造之处, placement new 大多数情况下专指这种版本

```cpp
void* operator new(std::size_t size, void* pMemory) throw();
```

这个版本的 operator new 已被纳入 C++ 标准程序库, 位于 `<new>` 头文件内

在上述发生异常的场景下, 运行期系统寻找参数个数和类型都匹配的 operator delete, 与 operator new 相似, 接受额外参数的 operator delete 称为 placement delete, 如果找不到对应的 placement delete, 运行期系统会什么都不做, 所以需要提供一个对应的 placement delete, 而调用 `delete pw;` 时会调用 Widget 的析构函数, 然后调用正常的 operator delete 而非对应的 placement delete

注意成员函数的名称会掩盖外围作用域的同名函数, 需要使用 Tips 33 中的方法处理这个问题, 这里要注意的是缺省情况下 C++ 在 global 中提供下面的 operator new, 如果除了自定义的 operator new 还需要使用它们, 在使用 Tips 33 中的方法外还要建立 class 专属的对应 operator delete, 这些 operator delete 可以调用 global 版本的 operator delete 来提供平常的行为

```cpp
void* operator new(std::size_t) throw(std::bad_alloc);          // normal
void* operator new(std::size_t, void*) throw();                 // placement
void* operator new(std::size_t, const std::nothrow_t&) throw(); // nothrow
```

我们可以建立一个 base class 内含所有正常形式的 operator new 和 operator delete

```cpp
class StandardNewDeleteForms {
public:
    // normal
    static void* operator new(std::size_t size) throw(std::bad_alloc)
    {
        return ::operator new(size);
    }

    static void operator delete(void* pMemory) throw()
    {
        ::operator delete(pMemory);
    }

    // placement
    static void* operator new(std::size_t size, void* ptr) throw()
    {
        return ::operator new(size, ptr);
    }

    static void operator delete(void* pMemory, void* ptr) throw()
    {
        ::operator delete(pMemory, ptr);
    }

    // nothrow
    static void* operator new(std::size_t size, const std::nothrow_t& nt) throw()
    {
        return ::operator new(size, nt);
    }

    static void operator delete(void* pMemory, const std::nothrow_t& nt) throw()
    {
        ::operator delete(pMemory, nt);
    }
};

// use StandardNewDeleteForms
class Widget : public StandardNewDeleteForms {
public:
    using StandardNewDeleteForms::operator new;
    using StandardNewDeleteForms::operator delete;

    // 自定义的 operator new 和 operator delete
    static void* operator new(std::size_t size, std::ostream& logStream) throw(std::bad_alloc)
    {
        ...
    }

    static void operator delete(void* pMemory, std::ostream& logStream) throw()
    {
        ...
    }
    ...
};
```

总结:

- 当你写了一个 placement new 时, 请确定你也写了对应的 placement delete, 否则可能发生资源泄漏
- 当你声明 placement new 和 placement delete 时, 请确定不要无意识地遮掩它们的正常版本

## 9 杂项讨论

### 53 不要轻易忽视编译器的警告

总结:

- 严肃对待编译器的警告信息, 努力在编译器最严苛的警告级别下争取无任何警告的编译
- 不要过度依赖编译器的警告信息, 因为不同编译器的警告信息不同

### 54 让自己熟悉包括 TR1 在内的标准程序库

注: TR1 是 "Technical Report 1", C++ 标准委员会在此描述一些新的标准库组件, 这些组件大部分在 C++11 中被纳入标准库

C++98 中加入的标准库组件:

- STL (Standard Template Library, 标准模板库): 覆盖容器 (container), 迭代器 (iterator), 算法 (algorithm), 函数对象 (function object), 容器适配器 (container adapter), 函数对象适配器 (function object adapter)
- Iostreams: 覆盖用户自定义缓冲功能, 国际化IO, 以及预先定义好的对象如 cin, cout, cerr, clog
- 国际化支持: 覆盖多区域能力, 如 wchar_t 和 wstring
- 数值处理
- 异常阶层体系
- C89 标准程序库

本书中 TR1 中加入的标准库组件 (C++11 中已经纳入标准库):

- 智能指针 std::shared_ptr, std::weak_ptr, std::unique_ptr
- std::function
- std::bind

其他的 TR1 新特性 (与 Template 无关):

- hash table
- 正则表达式
- Tuples
- std::array
- std::mem_fn
- std::reference_wrapper
- 随机数生成工具
- 数学特殊函数 (未纳入 C++11 标准库, 在 C++17 中纳入)
- C99 兼容扩充

其他的 TR1 新特性 (与 Template 有关):

- Type traits
- std::result_of

文中建议通过 Boost 获得 TR1-like 的实现, 但使用 C++11 的话, 可以直接使用 C++11 标准库

总结:

- C++ 标准程序库的主要机能由 STL, Iostreams, locales 组成, 并包含 C99 标准程序库
- TR1 (C++11) 添加了智能指针, 一般化函数指针, hash table, 正则表达式, Tuples 等新特性
- TR1 自身只是一份规范, 为获得来源可以使用 Boost, 但是 C++11 中已经将 TR1 的大部分内容纳入标准库

### 55 让自己熟悉 Boost

Boost 是一个由 C++ 标准委员会成员发起的一个开源社区, 旨在提供 C++ 标准库的扩展, 以及对 C++ 标准库的实现, 包括下面的内容:

- 字符串与文本处理
- 容器
- 函数对象和高级编程
- 泛型编程
- 模板元编程
- 数字和数值
- 正确性与测试
- 数据结构
- 语言间的支持
- 内存
- 杂项

总结:

- Boost 是一个开源社区, 也是一个网站, 致力于免费, 源码开放, 同僚复审的 C++ 程序库开发, 在 C++ 标准化进程中发挥了重要作用
- Boost 提供了许多 C++ 标准库的扩展, 以及对 C++ 标准库的实现
