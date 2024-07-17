# Effective STL

## 本书术语

### 容器分类

- 标准序列容器
    - vector
    - string
    - deque
    - list
- 标准关联容器
    - set
    - multiset
    - map
    - multimap

### 迭代器

- 针对输入输出流的迭代器
    - 输入迭代器 (只读迭代器, 每个被遍历到的位置上只能被读取一次)
    - 输出迭代器 (只写迭代器, 每个被遍历到的位置上只能被写入一次)
- 前向迭代器 (读写, 只能向前移动, 所有STL都提供比此更强大的迭代器)
- 双向迭代器 (读写, 可向前向后移动)
- 随机访问迭代器 (读写, 并提供了“迭代器算术”)

### 其他

所有重载了函数调用操作符 (即operator())的类都是一个函数子类 (functor class).这些类的对象可以被当作函数来调用.

绑定器: STL中的函数子类, 用于将一个函数和一个或多个参数绑定在一起.包括`bind1st`和`bind2nd`.

### 时间复杂度

- $O(1)$: 常数时间
- $O(logN)$: 对数时间
- $O(N)$: 线性时间

## 第1章 容器

### 第1条: 慎重选择容器类型

#### C++容器简要回顾

- **标准STL序列容器** : vector、string、deque和list.
- **标准STL关联容器** : set、multiset、map和multimap.
- **非标准序列容器** : slist (单向链表)和rope (“重型”string).
- **非标准关联容器** : hash_set、hash_multiset、hash_map和hash_multimap .
- **vector\<char\>作为string的替代** .
- **vector作为标准关联容器的替代** .
- **几种标准的非STL容器** : 包括数组、bitset、valarray、stack、queue和priority_queue.

#### STL容器的一种分类方法

- 连续内存容器 (基于数组的容器)
    - vector
    - string
    - deque
    - rope
- 基于节点的容器
    - list
    - slist
    - 标准的关联容器 (平衡树)
    - 非标准的关联容器 (哈希表)

#### 选择容器的一些原则

你是否需要在容器的任意位置插入新元素？如果需要, 就选择序列容器

你是否关心容器中的元素是排序的？

你选择的容器必须是标准C++的一部分吗？

你需要哪种类型的迭代器？

当发生元素的插入或删除操作时, 避免移动容器中原来的元素是否很重要？

容器中数据的布局是否需要和C兼容？

元素的查找速度是否是关键的考虑因素？

如果容器内部使用了引用计数技术 (reference counting), 你是否介意？

对插入和删除操作, 你需要事务语义 (transactional semantics)吗？

你需要使迭代器、指针和引用变为无效的次数最少吗？

如果序列容器的迭代器是随机访问类型, 而且只要没有删除操作发生, 且插入操作只发生在容器的末尾, 则指向数据的指针和引用就不会变为无效, 这样的容器是否对你有帮助？

### 第2条: 不要试图编写独立于容器类型的代码

这些限制的根源在于, 对不同类型的序列容器, 使迭代器、指针和引用无效 (invalidate)的规则是不同的.

考虑到有时候不可避免地要从一种容器类型转到另一种, 你可以使用常规的方式来实现这种转变: 使用封装 (encapsulation)技术.最简单的方式是通过对容器类型和其迭代器类型使用类型定义 (typedef).

所以, 不要这么写:

```cpp
class Widget { };

vector<Widget> vw;
Widget bestWidget;

vector<Widget>::iterator i = find(vw.begin(), vw.end(), bestWidget);
```

而应该这么写:

```cpp
class Widget { };
typedef vector<Widget> WidgetContainer;
typedef WidgetContainer::iterator WCIterator;

widgetContainer cw;
widget bestWidget;

WCIterator i = find(cw.begin(), cw.end(), bestWidget);
```

这样就使得改变容器类型要容易得多, 尤其当这种改变仅仅是增加一个自定义的分配子时, 就显得更为方便.

```cpp
class Widget { };
template <typename T> SpecialAllocator { };
typedef vector<Widget, SpecialAllocator<Widget>> WidgetContainer;
typedef WidgetContainer::iterator WCIterator;

widgetContainer cw;
widget bestWidget;

WCIterator i = find(cw.begin(), cw.end(), bestWidget);
```

类型定义只不过是其他类型的别名, 所以它带来的封装纯粹是词法 (lexical)上的.

要想减少在替换容器类型时所需要修改的代码, 你可以把容器隐藏到一个类中, 并尽量减少那些通过类接口 (而使外部)可见的、与容器相关的信息.

### 第3条: 确保容器中的对象副本正确而高效

copy in, copy out.这就是STL的工作方式.

考虑到这些复制过程, 如果你向容器中填充对象, 而对象的复制操作又很费时, 那么向容器中填充对象这一简单的操作将会成为程序的性能“瓶颈”.

而且, 如果这些对象的“副本”有特殊的含义, 那么把它们放入容器时将不可避免地会产生错误.

在存在继承关系的情况下, 复制动作会导致剥离 (slicing).“剥离”问题意味着向基类对象的容器中插入派生类对象几乎总是错误的.

使复制动作高效、正确, 并防止剥离问题发生的一个简单办法是使容器包含指针而不是对象.

不幸的是, 指针的容器也有其自身的一些令人头疼的、与STL相关的问题.如果你想避开这些使人头疼的问题, 同时又想避免效率、正确性和剥离这些问题, 你可能会发现智能指针 (smart pointer)是一个诱人的选择.

### 第4条: 调用empty而不是检查size()是否为0

empty对所有的标准容器都是常数时间操作, 而对一些list实现, size耗费线性时间.

### 第5条: 区间成员函数优先于与之对应的单元素成员函数

原因:

1. 通过使用区间成员函数, 通常可以少写一些代码.
2. 使用区间成员函数通常会得到意图清晰和更加直接的代码.

区间成员函数: 像STL算法一样, 使用两个迭代器参数来确定该成员操作所执行的区间.

注意存在assign这么一个使用极其方便的成员函数.对所有的标准序列容器 (vector、string、deque和list), 它都存在.

| 目标 | 函数 |
| --- | --- |
|完全替换一个容器的内容 | 赋值 (assignment)|
|把一个容器复制到相同类型的另一个容器 | operator= 是可选择的赋值函数 |
|给容器一组全新的值 | assign, operato= 则不能满足要求 |

几乎所有通过利用插入迭代器 (insert iterator)的方式 (即利用inserter、back_inserter或front_inserter)来限定目标区间的copy调用, 其实都可以 (也应该)被替换为对区间成员函数的调用.它更加直截了当地说明了所发生的事情.

从效率方面, 当处理标准序列容器时, 为了取得同样的结果, 使用单元素的成员函数比使用区间成员函数需要更多地调用内存分配子, 更频繁地复制对象, 而且／或者做冗余的操作.

向 v 的前端插入元素, 单元素版本的 insert 相对区间版本总共在三个方面影响了效率

1. 单元素版本有不必要的函数调用
2. 单元素版本把 v 中已有的元素频繁地移动到插入后它们所处的位置, 区间版本不需要 (仅输入迭代器除外)
3. 单元素版本导致内存分配问题

上述分析对 vector 和 string 有效.1, 2 对deque有效, 1 对list有效, 且 list 可以节省指针的复制

支持区间的成员函数

- 区间创建
- 区间插入
- 区间删除
- 区间赋值

### 第6条: 当心C++编译器最烦人的分析机制

注意 C++ 中的一条普遍规律: 尽可能地解释为函数声明

把形式参数的声明用括号括起来是非法的, 但给函数参数加上括号却是合法的, 所以通过增加一对括号, 我们强迫编译器按我们的方式来工作.

使用 istream_iterator 和区间构造函数时, 注意到这一点是有益的.

更好的方式是在对data的声明中避免使用匿名的 istream_iterator 对象, 而是给这些迭代器一个名称.

### 第7条: 如果容器中包含了通过new操作创建的指针, 切记在容器对象析构前将指针delete掉

可以像下面的代码一样delete容器中的指针:

```cpp
for (vector<Widget*>::iterator i = v.begin(); i != v.end(); ++i)
{
    delete *i;
}
```

for循环delete每一个指针存在的问题

1. 新的for循环做的事情和for_each相同, 但不如使用for_each看起来那么清楚
2. 不是异常安全的

为了把类似for_each的循环变成真的使用for_each, 你需要把delete变成一个函数对象,例如DeleteObject类.

```cpp
template <typename T> class DeleteObject
    : public unary_function<T, void>
{
public:
    void operator()(const T* ptr) const
    {
        delete ptr;
    }
};
```

从而提供如下的写法:

```cpp
for_each(v.begin(), v.end(), DeleteObject<Widget>());
```

通过让编译器推断出传给DeleteObject::operator()的指针的类型, 我们可以消除和虚析构函数相关的问题.我们所要做的只是把模板化从DeleteObject移到它的operator()中:

```cpp
struct DeleteObject
{
    template <typename T> void operator()(const T* ptr) const
    {
        delete ptr;
    }
};
```

这种类型推断的缺点是我们舍弃了使DeleteObject可配接 (adaptable)的能力.考虑到DeleteObject的设计初衷 (用于for_each), 很难想象这是一个问题.

新的for_each的调用方式:

```cpp
for_each(v.begin(), v.end(), DeleteObject());
```

但它仍然不是异常安全的.可以用多种方式来解决这一问题, 但最简单的方式可能是用智能指针容器代替指针容器.

使用Boost的shared_ptr (shared_ptr目前已经是C++11的标准库的一部分), 本条款最初的例子可改写为:

```cpp
typedef std::shared_ptr<Widget> SharedWidgetPtr;
vector<SharedWidgetPtr> v;
for (int i = 0; i < n; ++i) {
    v.push_back(SharedWidgetPtr(new Widget));
}
```

你所要记住的是: STL容器很智能, 但没有智能到知道是否该删除自己所包含的指针的程度.当你使用指针的容器, 而其中的指针应该被删除时, 为了避免资源泄漏, 你必须或者用引用计数形式的智能指针对象 (比如shared_ptr)代替指针, 或者当容器被析构时手工删除其中的每个指针.

### 第8条: 永远不要创建包含auto_ptr的容器

注意！在 C++11 中, auto_ptr 已经被弃用, 应该使用 unique_ptr.

auto_ptr的容器 (简称COAP)是被禁止的.试图使用它们的代码不会被编译通过.

使用这样的代码的缺点:

1. COAP是不可移植的.
2. COAP在进行复制操作时会导致所有权的转移.这导致排序、查找、删除等操作变得不可靠.

### 第9条: 慎重选择删除元素的方法

#### 删除某个特定值的所有元素

如果你有一个连续内存的容器, 那么最好的办法是使用erase-remove习惯用法:

```cpp
c.erase(remove(c.begin(), c.end(), value), c.end());
```

对list, 这一办法同样适用.但list的成员函数remove更加有效:

```cpp
c.remove(value);
```

当c是标准关联容器 (例如set、multiset、map或multimap)时, 使用任何名为remove的操作都是完全错误的.正确方法是调用erase:

```cpp
c.erase(value);
```

这样做不仅是正确的, 而且是高效的, 只需要对数时间开销,而且, 关联容器的erase成员函数还有另外一个优点, 即它是基于等价 (equivalence)而不是相等 (equality)的.

#### 删除使判别式返回true的每一个对象

序列容器 (vector、string、deque和list), 我们把每个对remove的调用换成调用remove_if就可以了:

```cpp
c.erase(remove_if(c.begin(), c.end(), pred), c.end());
c.remove_if(pred);
```

标准关联容器解决这一问题有两种办法.

##### 复制

利用remove_copy_if把我们需要的值复制到一个新容器中, 然后把原来容器的内容和新容器的内容相互交换

```cpp
AssocContainer newContainer;
remove_copy_if(c.begin(), c.end(), inserter(newContainer, newContainer.end()), pred);
c.swap(newContainer);
```

这种办法的缺点是需要复制所有不被删除的元素, 而我们可能并不希望付出这么多的复制代价.

##### 逐个删除

因为关联容器没有提供类似remove_if的成员函数, 所以, 我们必须写一个循环来遍历c中的元素, 并在遍历过程中删除元素.但要注意迭代器的有效性.

```cpp
for (AssocContainer::iterator i = c.begin(); i != c.end(); ) {
    if (pred(*i)) {
        c.erase(i++);
    } else {
        ++i;
    }
}
```

#### 删除使badValue返回true的元素同时向一个日志 (log)文件中写一条信息

对于关联容器仅需要对刚才的循环做简单的修改:

```cpp
for (AssocContainer::iterator i = c.begin(); i != c.end(); ) {
    if (pred(*i)) {
        logFile << "Erasing value " << *i << endl;
        c.erase(i++);
    } else {
        ++i;
    }
}
```

对于序列容器, 不能再使用erase-remove习惯用法, 且无法使用上述的循环, 要利用erase的返回值.

```cpp
for (SeqContainer::iterator i = c.begin(); i != c.end(); ) {
    if (pred(*i)) {
        logFile << "Erasing value " << *i << endl;
        i = c.erase(i);
    } else {
        ++i;
    }
}
```

就遍历和删除来说, 可以把list当作vector/string/deque来对待, 也可以把它当作关联容器来对待.两种方式对list都适用, 一般的惯例是对list采取和vector、string和deque相同的方式.

#### 总结

总结本条款中所讲的, 我们有以下结论.

- 要删除容器中有特定值的所有对象:
    - 如果容器是vector、string或deque, 则使用erase-remove习惯用法.
    - 如果容器是list, 则使用list::remove.
    - 如果容器是一个标准关联容器, 则使用它的erase成员函数.
- 要删除容器中满足特定判别式 (条件)的所有对象:
    - 如果容器是vector、string或deque, 则使用erase-remove_if习惯用法.
    - 如果容器是list, 则使用list::remove_if.
    - 如果容器是一个标准关联容器, 则使用remove_copy_if和swap, 或者写一个循环来遍历容器中的元素, 记住当把迭代器传给erase时, 要对它进行后缀递增.
- 要在循环内部做某些 (除了删除对象之外的)操作:
    - 如果容器是一个标准序列容器, 则写一个循环来遍历容器中的元素, 记住每次调用erase时, 要用它的返回值更新迭代器.
    - 如果容器是一个标准关联容器, 则写一个循环来遍历容器中的元素, 记住当把迭代器传给erase时, 要对迭代器做后缀递增.

### 第10条: 了解分配子 (allocator)的约定和限制

如果希望编写自定义的分配子, 需要注意:

- 你的分配子是一个模板, 模板参数T代表你为它分配内存的对象的类型.
- 提供类型定义pointer和reference, 但是始终让pointer为T* , reference为T&.因为C++标准很明确地指出, 允许库实现者假定每个分配子的指针类型等同于T* , 而分配子的引用类型就是T&.
- 千万别让你的分配子拥有随对象而不同的状态 (per-object state).通常, 分配子不应该有非静态的数据成员.
- 记住, 传给分配子的allocate成员函数的是那些要求内存的对象的个数, 而不是所需的字节数.同时要记住, 这些函数返回T* 指针 (通过pointer类型定义), 即使尚未有T对象被构造出来.
- 一定要提供嵌套的rebind模板, 因为标准容器依赖该模板.

rebind模板的完整代码如下:

```cpp
template<typename T>
class allocator
{
public:
    // rebind 模板
    template<typename U>
    struct rebind {
        typedef allocator<U> other;
    };
};
```

编写自己的分配子时, 建议不要自己从头编写所有的代码.而是在其他人编写的分配子的基础上进行修改或者从标准库中继承.

最终的模板内容大致如下:

```cpp
template<typename T>
class allocator
{
public:
    typedef T* pointer;
    typedef const T* const_pointer;
    typedef T& reference;
    typedef const T& const_reference;
    typedef T value_type;
    typedef size_t size_type;

    // rebind 模板
    template<typename U>
    struct rebind {
        typedef allocator<U> other;
    };
};
```

### 第11条: 理解自定义分配子的合理用法

#### 为什么要自定义分配子

- 经过测试发现STL默认的内存管理器 (即allocator\<T\>)太慢, 或者浪费内存, 或者导致了太多的内存碎片
- allocator<T>是线程安全的, 你的程序不需要
- 某些容器中的对象通常是一起使用的, 想把它们放在一个特殊堆中的相邻位置上, 以便尽可能地做到引用局部化
- 建立一个与共享内存相对应的特殊的堆, 然后在这块内存中存放一个或多个容器

#### 共享内存

有一些特殊过程, 它们采用malloc和free内存模型来管理一个位于共享内存的堆:

```cpp
void* mallocShared(size_t bytes) {
    return malloc(bytes);
}

void freeShared(void* ptr) {
    free(ptr);
}
```

把STL容器的内容放到这块共享内存中去

```cpp
template<typename T>
class SharedMemoryAllocator
{
public:
    pointer allocate(size_type n)
    {
        return static_cast<pointer>(mallocShared(n * sizeof(T)));
    }

    void deallocate(pointer p, size_type n)
    {
        freeShared(p);
    }
};
```

可以通过以下方式来使用这个分配子:

```cpp
typedef vector<double, SharedMemoryAllocator<double>> SharedDoubleVec;

SharedDoubleVec v;
```

此时, v 中的元素将被放到共享内存中.但 v 本身并不在共享内存中.如果希望 v 也在共享内存中, 需要像下面这样做:

```cpp
void *pVecMemory = mallocShared(sizeof(SharedDoubleVec));
SharedDoubleVec *pv = new(pVecMemory) SharedDoubleVec;
...
pv->~SharedDoubleVec();
freeShared(pVecMemory);
```

除非你有充分的理由, 否则, 建议避免这种手工的“分配／构造／析构／释放” (allocate/construct/destroy/deallocate)四步曲.

#### 元素聚集

假设有两个堆, 分别为类Heap1和类Heap2.

```cpp
class Heap1
{
public:
    static void* allocate(size_t bytes, const void* memoryBlockToBeNear = 0);
    static void deallocate(void* ptr);
};

class Heap2 { }; // 与Heap1类似
```

分配子可以使用像Heap1和Heap2这样的类来完成实际的内存分配和释放操作.

```cpp
template<typename T, typename Heap>
class SpecificHeapAllocator
{
public:
    pointer allocate(size_type numObjects, const void* locallyHint = 0)
    {
        return static_cast<pointer>(Heap::allocate(n * sizeof(T), locallyHint));
    }

    void deallocate(pointer p, size_type n)
    {
        Heap::deallocate(p);
    }
};
```

使用 SpecificHeapAllocator 进行元素聚集:

```cpp
vector<int, SpecificHeapAllocator<int, Heap1>> v;
set<int, SpecificHeapAllocator<int, Heap1>> s;

list<int, SpecificHeapAllocator<int, Heap2>> l;
map<int, string, less<int>, SpecificHeapAllocator<pair<const int, string>, Heap2>> m;
```

在这个例子中, 很重要的一点是, Heap1和Heap2都是类型而不是对象.否则它们将不会是等价的分配子, 违反分配子的等价性限制.

### 第12条: 切勿对STL容器的线程安全性有不切实际的依赖

对STL的线程安全性的第一个期望应该是, 它会随不同实现而异.

对一个STL实现最多只能期望:

- **多个线程读是安全的**.多个线程可以同时读同一个容器的内容, 并且保证是正确的.自然地, 在读的过程中, 不能对容器有任何写入操作.
- **多个线程对不同的容器做写入操作是安全的**.多个线程可以同时对不同的容器做写入操作.

库无法实现完全的容器线程安全性, 必须手工做同步控制, 例如:

```cpp
vector<int> v;
...
getMutexFor(v);
vector<int>::iterator i = find(v.begin(), v.end(), 42);
if (i != v.end()) {
    *i = 0;
}
releaseMutexFor(v);
```

另一个更为面向对象的方案是创建一个Lock类, 它在构造函数中获得一个互斥体, 例如:

```cpp
template<typename Container>
class Lock
{
public:
    Lock(const Container& container) : c(container)
    {
        getMutexFor(c);
    }
    ~Lock()
    {
        releaseMutexFor(c);
    }

private:
    const Container& c;
};
```

使用类 (如Lock)来管理资源的生存期的思想通常被称为“获得资源时即初始化”, 其使用方法如下:

```cpp
vector<int> v;
...
Lock<vector<int>> lock(v);
vector<int>::iterator i = find(v.begin(), v.end(), 42);
if (i != v.end()) {
    *i = 0;
}
```

这样的使用方法会带来如下好处:

- 如果我们忘了为Lock创建新的代码块, 则互斥体仍然会被释放, 只不过会晚一些
- 基于Lock的方案在有异常发生时也是强壮的

## 第2章 vector和string

### 第13条: vector和string优先于动态分配的数组

用new来动态分配内存需要额外进行如下工作

- 确保以后会有人用delete来删除所分配的内存
- 确保使用了正确的delete形式
- 确保只delete了一次

vector和string消除了上述的负担, 因为它们自己管理内存.

vector和string是功能完全的STL序列容器, 所以, 凡是适合于序列容器的STL算法都可以使用.

将中的数据传递给期望接受数组的API也是很容易的.

只能想到在一种情况下, 用动态分配的数组取代vector和string是合理的, 而且这种情形只对string适用.许多string实现在背后使用了引用计数技术, 这种策略可以提高很多应用程序的效率.但如果在多线程环境中使用引用计数的string, 由避免内存分配和字符复制所节省下来的时间还比不上花在背后同步控制上的时间.

所使用的string是以引用计数方式来实现的, 而你又运行在多线程环境中, 并认为string的引用计数实现会影响效率, 至少有三种可行的选择

- 检查你的库实现, 看看是否有可能禁止引用计数
- 寻找或开发另一个不使用引用计数的string实现
- 考虑使用vector\<char\>而不是string

### 第14条: 使用reserve来避免不必要的重新分配

对于vector和string, 增长过程是这样来实现的:

1. 分配一块大小为当前容量的某个倍数的新内存.在大多数实现中, vector和string的容量每次以2的倍数增长.
2. 把容器的所有元素从旧的内存复制到新的内存中.
3. 析构掉旧内存中的对象.
4. 释放旧内存.

需要避免不必要的重新分配的原因:

- 这些分配、释放、复制和析构步骤非常耗时
- 每当这些步骤发生时, vector或string中所有的指针、迭代器和引用都将变得无效

注意如下4个类似的函数:

- size()告诉你该容器中有多少个元素.
- capacity()告诉你该容器利用已经分配的内存可以容纳多少个元素.
- resize(Container::size_type n)强迫容器改变到包含n个元素的状态.在调用resize之后, size将返回n.
- reserve(Container::size_type n)强迫容器把它的容量变为至少是n, 前提是n不小于当前的大小.

使用reserve以避免不必要的重新分配的方法:

1. 若能确切知道或大致预计容器中最终会有多少元素, 则此时可使用reserve简单地预留适当大小的空间
2. 先预留足够大的空间, 然后, 当把所有数据都加入以后, 再去除多余的容量.

### 第15条: 注意string的多样性

几乎每个string实现都包含如下信息:

- 字符串的大小 (size)
- 字符串的容量 (capacity)
- 字符串的值 (value)

还可能包含:

- 分配子的一份副本
- 对值的引用计数 (建立在引用计数基础上实现的)

不同的string实现的主要区别在于:

- string的值可能会被引用计数, 也可能不会
- string对象大小的范围可以是一个 char* 指针的大小的1倍到7倍
- 创建一个新的字符串值可能需要零次、一次或两次动态分配内存
- string对象可能共享, 也可能不共享其大小和容量信息
- string可能支持, 也可能不支持针对单个对象的分配子
- 不同的实现对字符内存的最小分配单位有不同的策略

### 第16条: 了解如何把vector和string数据传给旧的API

如果有一个vector v, 而且需要得到一个指向v中数据的指针, 从而可把v中的数据作为数组来对待, 那么只需使用&v[0]. 对于string s, 对应的形式是s.c_str().

为防止传给旧API的指针为空, 应当按如下方式处理:

```cpp
void doSomething(const int* pInts, size_t numInts);

vector<int> v;

if (!v.empty()) {
    doSomething(&v[0], v.size());
}
```

不要使用v.begin()来代替&v[0], 因为begin的返回值是一个迭代器, 不是指针; 当你需要一个指向vector中的数据的指针时, 你永远不应该使用begin.

以上方式对于vector是适用的, 但对于string却是不可靠的, 因为:

- string中的数据不一定存储在连续的内存中
- string的内部表示不一定是以空字符结尾的

对于string, 应当使用c_str()来得到一个指向其数据的指针:

```cpp
void doSomething(const char* pString);

string s;

doSomething(s.c_str());
```

即使字符串长度为0, c_str()也会返回一个指向空字符的指针. 对字符串内部有空字符的情况也是可以的, 但doSomething会把内部的第一个空字符当作结尾的空字符.

vector或string的数据被传递给一个要读取, 而不是改写这些数据的API. 对于string, 这也是唯一所能做的. 对于vector, C API改变了v中元素值的话, 通常是没有问题的, 但被调用的例程不能试图改变vector中元素的个数. 同时要注意, 有些vector对它们的数据有额外的限制, 如果把vector传递给一个将改变该vector中数据的API, 必须保证这些额外的限制还能被满足。

如果你想用来自C API中的元素初始化一个vector, 可以向API传入该vector中元素的存储区域:

```cpp
size_t fillArray(double* pArray, size_t arraySize);
vector<double> vd(maxNumDoubles);
vd.resize(fillArray(&vd[0], vd.size()));
```

这一技术只对vector有效, 因为只有vector才保证和数组有同样的内存布局. 如果你想用来自C API中的数据初始化一个string, 只要让API把数据放到一个vector\<char\>中, 然后把数据从该vector复制到相应字符串中:

```cpp
size_t fillString(char* pArray, size_t arraySize);
vector<char> vc(maxNumChars);
size_t charWritten = fillArray(&vc[0], vc.size());
string s(vc.begin(), vc.begin() + charWritten);
```

先让C API把数据写入到一个vector中, 然后把数据复制到期望最终写入的STL容器中, 这一思想总是可行的

### 第17条: 使用"swap技巧"除去多余的容量

从vector中去除多余的容量的方法:

```cpp
class Widget { };
vector<Widget> v;
...
vector<Widget>(v).swap(v);
```

同样的技巧对string也适用

```cpp
string s;
...
string(s).swap(s);
```

在做swap的时候, 不仅两个容器的内容被交换, 同时它们的迭代器、指针和引用也将被交换 (string除外).

### 第18条: 避免使用vector\<bool\>

一个STL容器, vector\<bool\>只有两点不对

1. 它不是一个容器
2. 它不存储bool值

vector\<bool\>是一个假的容器, 它并不真的储存bool, 相反, 为了节省空间, 它储存的是bool的紧凑表示, vector\<bool\> 无法通过如下代码, 导致其不属于STL容器:

```cpp
vector<bool> vb;
bool* pb = &vb[0];
```

需要vector\<bool\>时, 应该使用deque\<bool\>或者bitset\<N\>来代替.

## 第3章 关联容器

### 第19条: 理解相等 (equality) 和等价 (equivalence) 的区别

find对"相同"的定义是相等, 是以operator==为基础的. set::insert对"相同"的定义是等价, 是以operator<为基础的.

在实际操作中, 相等的概念是基于operator==的. 等价关系是以"在已排序的区间中对象值的相对顺序"为基础的. set\<Widget\>的默认比较函数是less\<Widget\> , 而在默认情况下less\<Widget\>只是简单地调用了针对Widget的operator\< . 如果两个值中的任何一个 (按照一定的排序准则) 都不在另一个的前面, 那么这两个值 (按照这一准则) 就是等价的. 在一般情形下, 一个关联容器的比较函数并不是operator<, 甚至也不是less, 它是用户定义的判别式 (predicate) . 每个标准关联容器都通过key_comp成员函数使排序判别式可被外部使用.

满足以下条件的两个对象被认为是等价的:

```cpp
!c.key_comp()(a, b) && !c.key_comp()(b, a)

// 当c的比较函数是less时, 等价于
!(a < b) && !(b < a)
```
