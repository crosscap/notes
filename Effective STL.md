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

copy in, copy out. 这就是STL的工作方式.

考虑到这些复制过程, 如果你向容器中填充对象, 而对象的复制操作又很费时, 那么向容器中填充对象这一简单的操作将会成为程序的性能“瓶颈”.

而且, 如果这些对象的“副本”有特殊的含义, 那么把它们放入容器时将不可避免地会产生错误.

在存在继承关系的情况下, 复制动作会导致剥离 (slicing). "剥离"问题意味着向基类对象的容器中插入派生类对象几乎总是错误的.

使复制动作高效, 正确, 并防止剥离问题发生的一个简单办法是使容器包含指针而不是对象.

不幸的是, 指针的容器也有其自身的一些令人头疼的, 与STL相关的问题. 如果你想避开这些使人头疼的问题, 同时又想避免效率, 正确性和剥离这些问题, 你可能会发现智能指针 (smart pointer) 是一个诱人的选择.

### 第4条: 调用 empty 而不是检查 size() 是否为 0

empty对所有的标准容器都是常数时间操作, 而对一些list实现, size耗费线性时间.

### 第5条: 区间成员函数优先于与之对应的单元素成员函数

原因:

1. 通过使用区间成员函数, 通常可以少写一些代码.
2. 使用区间成员函数通常会得到意图清晰和更加直接的代码.

区间成员函数: 像STL算法一样, 使用两个迭代器参数来确定该成员操作所执行的区间.

注意存在assign这么一个使用极其方便的成员函数.对所有的标准序列容器 (vector、string、deque和list), 它都存在.

|               目标                |               函数             |
| -------------------------------- | ------------------------------ |
|       完全替换一个容器的内容       |        赋值 (assignment)       |
| 把一个容器复制到相同类型的另一个容器 |   operator= 是可选择的赋值函数  |
|         给容器一组全新的值         | assign, operato= 则不能满足要求 |

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

### 第6条: 当心 C++ 编译器最烦人的分析机制

注意 C++ 中的一条普遍规律: 尽可能地解释为函数声明

例如, 下面的代码会被解析为: 第一个参数的名称是dataFile, 它的类型是istream_iterator\<int\>, dataFile两边的括号是多余的, 会被忽略; 第二个参数没有名称, 它的类型是指向不带参数的函数的指针, 该函数返回一个istream_iterator<int>.

```cpp
ifstream dataFile("ints.dat");
list<int> data(istream_iterator<int>(dataFile), istream_iterator<int>());
```

把形式参数的声明用括号括起来是非法的, 但给函数参数加上括号却是合法的, 所以通过增加一对括号, 我们强迫编译器按我们的方式来工作.

```cpp
list<int> data((istream_iterator<int>(dataFile)), istream_iterator<int>());
```

使用 istream_iterator 和区间构造函数时, 注意到这一点是有益的.

更好的方式是在对 data 的声明中避免使用匿名的 istream_iterator 对象, 而是给这些迭代器一个名称, 下面的代码应该总是可以工作的.

```cpp
ifstream dataFile("ints.dat");
istream_iterator<int> dataBegin(dataFile);
istream_iterator<int> dataEnd;
list<int> data(dataBegin, dataEnd);
```

### 第7条: 如果容器中包含了通过 new 操作创建的指针, 切记在容器对象析构前将指针 delete 掉

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

### 第8条: 永远不要创建包含 auto_ptr 的容器

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

#### 删除使判别式返回 true 的每一个对象

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

#### 删除使 badValue 返回 true 的元素同时向一个日志 (log) 文件中写一条信息

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

### 第10条: 了解分配子 (allocator) 的约定和限制

如果希望编写自定义的分配子, 需要注意:

- 你的分配子是一个模板, 模板参数T代表你为它分配内存的对象的类型.
- 提供类型定义pointer和reference, 但是始终让pointer为T*, reference为T&.因为C++标准很明确地指出, 允许库实现者假定每个分配子的指针类型等同于T* , 而分配子的引用类型就是T&.
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

### 第12条: 切勿对 STL 容器的线程安全性有不切实际的依赖

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

## 第2章 vector 和 string

### 第13条: vector 和 string 优先于动态分配的数组

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

### 第14条: 使用 reserve 来避免不必要的重新分配

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

### 第15条: 注意 string 的多样性

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

### 第16条: 了解如何把 vector 和 string 数据传给旧的 API

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

### 第17条: 使用 "swap 技巧" 除去多余的容量

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

### 第20条: 为包含指针的关联容器指定比较类型

`set<string*> ssp;` 是 `set<string*, less<string*>> ssp;` 的简写. 更准确地说, 是 `set<string*, less<string*>, allocator<string*>> ssp;` 的简写.

如果想让string*指针在集合中按字符串的值排序, 那么不能使用默认的比较函数子类less\<string*\>, 必须自己编写比较函数子类

```cpp
class StringPtrLess :
    public binary_function<const string*, const string*, bool>
{
public:
    bool operator()(const string* ps1, const string* ps2) const
    {
        return *ps1 < *ps2;
    }
};
```

使用StringPtrLess作为set\<string*\>的比较函数子类:

```cpp
typedef set<string*, StringPtrLess> StringPtrSet;
StringPtrSet ssp;
```

随后可以按如下方式打印出ssp中的所有字符串:

```cpp
for (StringPtrSet::const_iterator i = ssp.cbegin(); i != ssp.cend(); ++i) {
    cout << **i << endl;
}
```

如果使用算法, 需要写一个函数它在打印string* 指针之前知道怎样才能解除指针引用, 然后与for_each一起使用这个函数

```cpp
void printStringPtr(const string* ps)
{
    cout << *ps << endl;
}
for_each(ssp.cbegin(), ssp.cend(), printStringPtr);
```

或者更进一步, 写一个通用的解除指针引用的函数子类, 然后与transform和ostream_iterator一起使用:

```cpp
struct Dereference
{
    template <typename T>
    const T& operator()(const T* ptr) const
    {
        return *ptr;
    }
};

transform(ssp.cbegin(), ssp.cend(), ostream_iterator<string>(cout, "\n"), Dereference());
```

创建一个函数子类而不是简单地为集合写一个比较函数的原因: set模板的三个参数每个都是一个类型, 而比较函数不是一个类型, 它是一个函数.

大多数情况下这个比较类型只是解除指针的引用并对所指向的对象进行比较. 考虑到这种情况, 你最好手头上为这样的比较函数子准备一个模板.

```cpp
struct DereferenceLess
{
    template <typename PtrType>
    bool operator()(const PtrType* pT1, const PtrType* pT2) const
    {
        return *pT1 < *pT2;
    }
};
```

这样的模板使我们不必编写像StringPtrLess这样的类, 因为我们可以使用DereferenceLess来代替:

```cpp
typedef set<string*, DereferenceLess> StringPtrSet;
```

本条款是关于包含指针的关联容器的, 但它同样也适用于其他一些容器, 这些容器中包含的对象与指针的行为相似, 比如智能指针和迭代器. 对于容器中包含了指向T对象的迭代器或智能指针的情形, DereferenceLess也同样可用作比较类型.

### 第21条: 总是让比较函数在等值情况下返回false

如果你的比较函数在等值情况下返回true, 那么关联容器会错误地判断相等的两个元素不等价, 从而导致容器中出现重复的元素. 对于set和map, 这是不允许的. 对于multiset和multimap, 错误的等价性判断会导致某些函数的行为不符合预期.

注意, 对比较函数<取反将得到>=, 而非期望的>.

记住, 比较函数的返回值表明的是按照该函数定义的排列顺序即一个值是否在另一个之前, 相等的值从来不会有前后顺序关系.

### 第22条: 切勿直接修改 set 或 multiset 中的键

像所有的标准关联容器一样, set 和 multiset 按照一定的顺序来存放自己的元素, 而这些容器的正确行为也是建立在其元素保持有序的基础之上的, 如果你把关联容器中一个元素的值改变了, 新的值可能不在正确的位置上, 这将会打破容器的有序性.

对于 map 和 multimap 如果有程序试图改变这些容器中的键, 它将不能通过编译. 对于一个 map\<K,V\> 或 multimap\<K,V\> 类型的对象, 其中的元素类型是 pair\<const K,V\>. 对于set\<T\>或multiset\<T\>类型的对象, 容器中元素的类型是 T, 而不是const T.

某个实现可能会使 set\<T\>::iterator 的 operator* 返回一个const T& 从而使 set 或 multiset 中的元素不可被修改, 所以试图修改 set 或 multiset 中元素的代码将是不可移植的.

应对方法:

- 如果你不关心可移植性, 而你想改变set或multiset中元素的值, 并且你的STL实现允许你这么做, 则请继续做下去. 只是注意不要改变元素中的键部分, 即元素中能够影响容器有序性的部分.
- 如果你重视可移植性, 就要确保set和multiset中的元素不能被修改, 至少不能未经过强制类型转换（cast）就修改.

#### 怎样正确地修改元素的非键部分, 并且是可移植的做法

例如

```cpp
class Employee { };

struct IDNumberLess
{
    public binary_function<Employee, Employee, bool>
    {
        bool operator()(const Employee& lhs, const Employee& rhs) const
        {
            return lhs.idNumber() < rhs.idNumber();
        }
    };
};

typedef set<Employee, IDNumberLess> EmpIDSet;
EmpIDSet se;
```

对于下面的操作:

```cpp
Employee selectedID;

EmpIDSet::iterator i = se.find(selectedID);
if (i != se.end()) {
    i->setTitle("CEO");
}
```

在某些实现中不可进行, 为了使它能够编译和正确执行如下进行:

```cpp
if (i != se.end()) {
    const_cast<Employee&>(*i).setTitle("CEO");
}
```

大多数强制类型转换都可以避免, 包括我们刚刚考虑过的那个转换。如果你想以一种总是可行而且安全的方式来修改 set, multiset, map 和 multimap 中的元素, 则可以分5个简单步骤来进行:

1. 找到你想修改的容器的元素
2. 为将要被修改的元素做一份副本
3. 修改该副本, 使它具有你期望它在容器中的值
4. 把该元素从容器中删除, 通常是通过调用erase来进行的
5. 把新的值插入到容器中, 如果按照容器的排列顺序, 新元素的位置可能与被删除元素的位置相同或紧邻, 则使用"提示"(hint)形式的insert, 以便把插入的效率从对数时间提高到常数时间, 把你从第1步得来的迭代器作为提示信息, 像下面这样:

```cpp
EmpIDSet::iterator i = se.find(selectedID); // 1
if (i != se.end()) {
    Employee e(*i);                         // 2
    e.setTitle("CEO");                      // 3
    se.erase(i++);                          // 4
    se.insert(i, e);                        // 5
}
```

### 第23条: 考虑用排序的 vector 替代关联容器

对于许多应用, 散列容器可能提供的常数时间查找能力优于 set, multiset, map 和 multimap 的确定的对数时间查找能力.

标准关联容器通常被实现为平衡的二叉查找树. 它对将插入, 删除, 查找混合在一起的应用提供了很好的性能, 但很多应用程序使用其数据结构的方式并不这么混乱, 它们使用其数据结构的过程可以明显地分为3个阶段:

1. 设置阶段: 向创建一个新的数据结构, 并插入大量元素. 在这个阶段, 几乎所有的操作都是插入和删除操作, 很少或几乎没有查找操作
2. 查询阶段: 查询该数据结构以找到特定的信息. 在这个阶段, 查找操作占主导地位, 插入和删除操作很少
3. 重组阶段: 改变该数据结构的内容, 或许是删除所有的当前数据, 再插入新的数据. 在这个阶段行为上与阶段1类似, 完成后又进入阶段2

对以这种方式使用其数据结构的应用程序来说, 排序的vector可能比关联容器提供了更好的性能, 因为只有对排序的容器才能够正确地使用查找算法binary_search、lower_bound和equal_range等.

原因:

- 大小: 关联容器基于节点, 每个节点都至少有指向左右子节点的指针, 一个指向父节点的指针, 而vector只有元素本身以及一些vector本身的开销 (通常是3个机器字，即3个指针或者2个指针加1个int), 这使得一个内存页中可存放的vector数据往往比关联容器多
- 引用的局域性: 如果你的STL实现没有采取措施来提高这些树节点的引用局域性, 这些节点将会散布在你的全部地址空间中, 导致大量的页面错误, 即便使用了可提供聚集特性的自定义内存管理器, 关联容器在页面错误这一点上也会有更多的问题

在排序的vector中存储数据可能比在标准关联容器中存储同样的数据要耗费更少的内存, 而考虑到页面错误的因素, 通过二分搜索法来查找一个排序的vector可能比查找一个标准关联容器要更快一些.

一段使用排序的vector而不是set的代码骨架:

```cpp
vector<Widget> vw;
...
sort(vw.begin(), vw.end());

Widget w;
...
if (binary_search(vw.begin(), vw.end(), w)) {
    // 找到了
}

vector<Widget>::iterator i = lower_bound(vw.begin(), vw.end(), w);
if (i != vw.end() && !(w < *i)) {
    // 找到了
}

pair<vector<Widget>::iterator, vector<Widget>::iterator> range = equal_range(vw.begin(), vw.end(), w);
if (range.first != range.second) {
    // 找到了
}

sort(vw.begin(), vw.end());
```

如果你决定用一个vector来替换map或multimap, 需要注意, 如果你声明了一个map\<K,V\>, 那么map中储存的是pair\<const K,V\>, 但用vector来模仿map或multimap时必须使用pair\<K,V\>.

下面的例子给出了注意事项:

```cpp
typedef pair<string, int> Data;

class DataCompare
{
public:
    bool operator()(const Data& lhs, const Data& rhs) const
    {
        return keyLess(lhs.first, rhs.first);
    }

    bool operator()(const Data& lhs, const Data::first_type& k) const
    {
        return keyLess(lhs.first, k);
    }

    bool operator()(const Data::first_type& k, const Data& rhs) const
    {
        return keyLess(k, rhs.first);
    }

private:
    bool keyLess(const Data::first_type& k1, const Data::first_type& k2) const
    {
        return k1 < k2;
    }
};
```

把排序的vector当作映射表来使用, 其本质上就如同将它用作一个集合一样, 唯一的区别是, 需要用DataCompare对象作为比较函数:

```cpp
vector<Data> vd;
...
sort(vd.begin(), vd.end(), DataCompare());

string s;
...
if (binary_search(vd.begin(), vd.end(), s, DataCompare())) {
    // 找到了
}

vector<Data>::iterator i = lower_bound(vd.begin(), vd.end(), s, DataCompare());
if (i != vd.end() && !(DataCompare()(s, *i))) {
    // 找到了
}

pair<vector<Data>::iterator, vector<Data>::iterator> range = equal_range(vd.begin(), vd.end(), s, DataCompare());
if (range.first != range.second) {
    // 找到了
}

...
sort(vd.begin(), vd.end(), DataCompare());
```

### 第24条: 效率至关重要时, 请在 map::operator[] 与 map::insert 之间谨慎做出选择

假定我们有一个Widget类:

```cpp
class Widget
{
public:
    Widget();
    Widget(double widget);
    Widget& operator=(double widget);
    ...
};
```

map的operator[]函数与众不同, 它的设计目的是为了提供"添加和更新" (add or update) 功能. 也就是说, 表达式 `m[key] = value` 会在m中添加一个键值对, 键为key, 值为value, 如果m中已经有了一个键为key的元素, 那么它的值将被更新为value.

对于如下语句:

```cpp
map<int, Widget> m;
m[1] = 1.5;
```

功能上等同于:

```cpp
typedef map<int, Widget> IntWidgetMap;
pair<IntWidgetMap::iterator, bool> result = m.insert(IntWidgetMap::value_type(1, Widget()));
result.first->second = 1.5;
```

如果"直接使用我们所需要的值构造一个Widget"比"先默认构造一个Widget再赋值"效率更高, 那么, 我们最好把对operator[]的使用 (包括与之相伴的构造和赋值) 换成对insert的直接调用:

```cpp
m.insert(IntWidgetMap::value_type(1, 1.5));
```

它通常会节省3个函数调用: 一个用于创建默认构造的临时Widget对象, 一个用于析构该临时对象, 另一个是调用Widget的赋值操作符.

当作为"添加"操作时, insert比operator[]效率更高, 当我们做"更新"操作时, 即当一个等价的键已经在映射表中时, 形势恰好反过来了.

使用 operator[] 和 insert 各自的代码:

```cpp
m[k] = v;

m.insert(IntWidgetMap::value_type(k, v)).first->second = v;
```

当向映射表中添加元素时, 要优先选用insert; 当更新元素时, 优先选用operator[].

可以实现一个函数使其兼具insert和operator[]的优点:

```cpp
template<typename MapType, typename KeyArgType, typename ValueArgType>
typename MapType::iterator efficientAddOrUpdate(MapType& m, const KeyArgType& k, const ValueArgType& v)
{
    typename MapType::iterator lb = m.lower_bound(k);
    if (lb != m.end() && !(m.key_comp()(k, lb->first))) {
        lb->second = v;
        return lb;
    } else {
        typedef typename MapType::value_type MVT;
        return m.insert(lb, MVT(k, v));
    }
}
```

关于这一实现有趣的一点是: KeyArgType和ValueArgType不必是存储在映射表中的类型, 只要它们能够转换成存储在映射表中的类型就可以了.

### 第25条: 熟悉非标准的散列容器

注意: 在C++11中, 散列容器已经被标准化, 它们的名字与被标准化之前的非标准散列容器不同, 分别是unordered_set, unordered_multiset, unordered_map 和 unordered_multimap (原先是hash_set, hash_multiset, hash_map 和 hash_multimap).

散列容器是关联容器, 需要知道存储在容器中的对象的类型, 用于这种对象的比较函数, 用于这些对象的分配子, 还需要指定一个散列函数, 散列容器的声明大致如下:

```cpp
template<typename T,
         typename HashFunction,
         typename CompareFunction,
         typename Allocator = allocator<T>>
class hash_container;
```

SGI的设计中值得注意的一点是使用了equal_to作为默认的比较函数, 而不是less, 因为散列关联容器和与之对应的标准容器不同, 其元素不是以排序方式存放的.

Dinkumware则把默认的散列函数和比较函数放在一个单独的类似于traits的hash_compare类中.

无论是SGI设计方案还是Dinkumware设计方案, 可以让散列实现来决定所有的策略:

```cpp
hash_set<int> intTable;
```

## 第4章 迭代器

### 第26条: iterator 优先于 const_iterator, reverse_iterator 以及 const_reverse_iterator

STL中的所有标准容器都提供了4种迭代器类型, 对容器类container\<T\>而言, iterator类型的功效相当于T*, 而const_iterator则相当于const T* (或者T const*). reverse_iterator和const_reverse_iterator则是iterator和const_iterator的逆向版本.

为什么应该尽可能使用iterator, 而避免使用const或者reverse型的迭代器:

- 有些版本的insert和erase函数要求使用iterator
- 要想隐式地将一个const_iterator转换成iterator是不可能的, 第27条中讨论的将const_iterator转换成iterator的技术并不普遍适用, 而且效率也不能保证
- reverse_iterator转换而来的iterator在使用之前可能需要相应的调整, 详见第28条

虽然容器类支持4种不同的迭代器类型但其中有一种迭代器 (iterator) 有着特殊的地位.

下面的图片展示了迭代器的转换关系:

![迭代器的转换关系](.%2Fimage%2FEffective%20STL.assets%2F%E8%BF%AD%E4%BB%A3%E5%99%A8%E8%BD%AC%E6%8D%A2.png)

注意: 这个图中没有显示出来的事实是: 通过base()得到的迭代器也许并非你所期待的迭代器, 详见第28条.

如果STL的实现存在一些缺陷 (将比较操作符重载为成员函数), 那么iterator和const_iterator之间的比较操作可能会出现问题.

从const正确性的角度来看, 仅仅为了避免一些可能存在的STL实现缺陷而放弃const_iterator显得有欠公允, 但考虑到在容器类的某些成员函数中指定使用iterator的现状, 得出iterator较之const_iterator更为实用的结论也就不足为奇了, 更何况, 从实践的角度来看, 并不总是值得卷入const_iterator的麻烦中.

PS: 现在情况可能有所改变, 需要查阅些资料来确认.

### 第27条: 使用 distance 和 advance 将容器的 const_iterator 转换成 iterator

强制类型转换的代码将会导致编译错误, 也许某些vector和string容器能通过编译, 因为它们的迭代器是指针, 但即便在这样的STL实现中reverse_iterator和const_reverse_iterator仍然是真正的类, 不能被强制类型转换. 所以即使对于vector和string容器, 将const迭代器强制转换成迭代器也是不可取的.

下面是一种将const_iterator转换成iterator的方法:

```cpp
typedef deque<int> IntDeque;
typedef IntDeque::iterator Iter;
typedef IntDeque::const_iterator ConstIter;

IntDeque d;
ConstIter ci;
...
Iter i(d.begin());
advance(i, distance<ConstIter>(i, ci));
```

PS: 这里介绍的方法对使用了引用计数的string实现可能无效

对于随机访问的迭代器, 此技术花费的时间是常数时间, 对于双向迭代器, 此技术花费的时间是线性时间.

### 第28条: 正确理解由 reverse_iterator 的 base() 员函数所产生的 iterator 的用法

一个数组的begin(), end(), rbegin(), rend(), 以及其中生成的iterator和reverse_iterator之间的关系如下:

![迭代器之间的关系](.%2Fimage%2FEffective%20STL.assets%2F%E8%BF%AD%E4%BB%A3%E5%99%A8%E4%B9%8B%E9%97%B4%E7%9A%84%E5%85%B3%E7%B3%BB.png)

对于插入和删除操作, 按如下进行:

- 如果要在一个reverse_iterator ri指定的位置上插入新元素, 则只需在ri.base()位置处插入元素即可, 对于插入操作而言, ri和ri.base()是等价的, ri.base()是真正与ri对应的iterator.
- 如果要在一个reverse_iterator ri指定的位置上删除一个元素, 则需要在ri.base()前面的位置上执行删除操作, 对于删除操作而言, ri和ri.base()是不等价的, ri.base()不是与ri对应的iterator.

但要注意, C和C++都规定了从函数返回的指针不应该被修改, 如果在你的STL平台上string和vector的iterator是指针的话, 类似`--ri.base()`这样的表达式就无法通过编译, 可以先递增reverse_iterator解决:

```cpp
v.erase((++ri).base());
```

### 第29条: 对于逐个字符的输入请考虑使用 istreambuf_iterator

把一个文本文件的内容复制到一个string对象中，以下的代码看上去是一种合理的解决方案:

```cpp
ifstream inputFile("intrestingData.txt");
inpytFile.unsetf(ios::skipws); // 不跳过空白字符
/** 以上语句可用 如下语句代替
 * inputFile >> noskipws;
*/
string fileData((istream_iterator<char>(inputFile)), istream_iterator<char>());
```

你可能会发现整个复制过程远不及你希望的那般快,  istream_iterator 内部使用的 operator>> 函数实际上执行了格式化的输入, 这意味着你每调用一次 perator>> 操作符, 它都要执行许多附加的操作:

- 一个内部的sentry对象的构造和析构 (sentry是在调用operator>>的过程中进行设置和清理行为的特殊iostream对象)
- 检查那些可能会影响其行为的流标志
- 检查所有可能发生的读取错误
- 果遇到错误的话, 还需要检查输入流的异常屏蔽标志以决定是否抛出相应的异常

使用 istreambuf_iterator 可以避免这些问题, 它直接从流的缓冲区中读取下一个字符:

```cpp
ifstream inputFile("intrestingData.txt");
string fileData((istreambuf_iterator<char>(inputFile)), istreambuf_iterator<char>());
```

对于非格式化的逐个字符输入过程, 你总是应该考虑使用istreambuf_iterator; 同样的, 对于非格式化的逐个字符输出过程, 你总是应该考虑使用ostreambuf_iterator.

## 第5章 算法

### 第30条: 确保目标区间足够大

下面的代码是错误的:

```cpp
int transmorgrify(int x);

vector<int> v;
...
vector<int> results;
transform(v.begin(), v.end(), results.end(), transmorgrify);
```

这种transform调用是错误的, 因为它导致了对无效对象的赋值操作, 要通过调用back_inserter生成一个迭代器来指定目标区间的起始位置:

```cpp
transform(v.begin(), v.end(), back_inseter(results), transmorgrify);
```

back_inserter返回的迭代器将使得push_back被调用, 所以back_inserter可适用于所有提供了push_back方法的容器 (所有的标准序列容器: vector、string、deque和list); front_inserter在内部利用了push_front, 所以front_inserter仅适用于那些提供了push_front成员函数的容器 (如deque和list)

由于front_inserter将通过push_front来加入每个对象, 所以这些对象在results中的顺序将会与在values中的顺序相反, 如果你希望transform把输出结果存放在results的前端, 同时保留它们在values中原有的顺序, 那么你只需按相反顺序遍历values即可

inserter将用于把算法的结果插入到容器中的特定位置上:

```cpp
vector<int> v;

vector<int> results;
...
transform(v.begin(), v.end(), inserter(results, results.begin() + results.size() / 2), transmorgrify);
```

插入操作的目标容器是vector或者string, 则你可以遵从第14条的建议, 预先调用reserve, 从而可以提高插入操作的性能, 在每次执行插入操作的时候, 你仍然需要承受因移动元素而带来的开销, 但这样做至少可以避免因重新分配容器内存而带来的开销:

```cpp
vector<int> v;

vector<int> results;
...
results.reserve(results.size() + v.size());
transform(v.begin(), v.end(), inserter(results, results.begin() + results.size() / 2), transmorgrify);
```

当一个算法需要向vector或者string中加入新元素的时候, 即使已经调用了reserve, 你也必须使用插入型的迭代器, 使用reserve但同时又不使用一个插入型的迭代器将会导致算法内部不确定的行为, 并且破坏容器的数据一致性.

对于希望覆盖容器中已有的元素的情况, 仍然要遵从本条款的建议.

```cpp
vector<int> v;

vector<int> results;
...
if (results.size() < v.size()) {
    results.resize(v.size());
}
transform(v.begin(), v.end(), results.begin(), transmorgrify);
```

使用clear清空results后使用插入型迭代器也可以达到相同的效果:

```cpp
...
results.clear();
results.reserve(v.size());
transform(v.begin(), v.end(), back_inseter(results), transmorgrify);
```

如果所使用的算法需要指定一个目标区间, 那么必须确保目标区间足够大, 或者确保它会随着算法的运行而增大 (使用插入型迭代器).

### 第31条: 了解各种与排序有关的选择

- 如果需要对vector, string, deque或者数组中的元素执行一次完全排序, 那么可以使用sort或者stable_sort
- 如果有一个vector、string、deque或者数组, 并且只需要对等价性最前面的n个元素进行排序, 那么可以使用partial_sort
- 如果有一个vector、string、deque或者数组, 并且需要找到第n个位置上的元素或者需要找到等价性最前面的n个元素但又不必对这n个元素进行排序, 那么可以使用nth_element
- 如果需要将一个标准序列容器中的元素按照是否满足某个特定的条件区分开来, 那么可以使用partition和stable_partition
- 对于list而言:
    - 可以直接调用partition和stable_partition算法
    - 用list::sort来替代sort和stable_sort算法
    - 如果你需要获得partial_sort或nth_element算法的效果, 可以有一些间接的途径来完成这项任务

大多数情况下都可以使用sort来完成排序任务

当进行不完全的排序时, 可以使用partial_sort

```cpp
bool qualityCompare(const Widget& lhs, const Widget& rhs);
...
partial_sort(w.begin(), w.begin() + 20, w.end(), qualityCompare);
```

当仅需要找到最大或最小的n个元素, 不关心它们的顺序时, 使用 nth_element, 有一段相当复杂的对nth_element的描述, 将在后文中列出

```cpp
nth_element(w.begin(), w.begin() + 19, w.end(), qualityCompare);
```

partial_sort和nth_element的调用大致相同, 仅第二个参数的含义不同, partial_sort的第二个参数是一个用于指定排序的区间的迭代器, 区间内的元素将被排序, 根据STL对区间的定义, 它实际上指向了目标区间外的第1个元素, 而nth_element的第二个参数则标识出容器中的某个特定位置, 该位置之前的元素都比该位置上的元素小, 该位置之后的元素都比该位置上的元素大.

对于等价元素的排序, partial_sort和nth_element的行为无法控制, 而sort是不稳定的, 但stable_sort是稳定的.

nth_element除了可以用来找到排名在前的n个元素以外还有其他的用途例如寻找中间值, 在下面的代码中演示:

```cpp
vector<Widgt>::iterator begin(w.begin());
vector<Widgt>::iterator end(w.end());

vector<Widgt>::iterator goalPositon;

// find the middle element
goalPositon = begin + w.size() / 2;
nth_element(begin, goalPositon, end, qualityCompare);

// find the 75% element
goalPositon = begin + w.size() * 3 / 4;
nth_element(begin, goalPositon, end, qualityCompare);
```

对于position, 它把所有满足某个特定条件的元素放在区间的前部, 它是不稳定的, 同时它有一个对应的稳定版本stable_partition.

```cpp
bool hasAcceptableQuality(const Widget& w);
...
vector<Widget>::iterator goodEnd = partition(w.begin(), w.end(), hasAcceptableQuality);
```

sort, stable_sort, partial_sort和nth_element算法都要求随机访问迭代器, 所以这些算法只能被应用于vector, string, deque和数组, 而partition和stable_partition只要求双向迭代器就能完成工作.

list是唯一需要排序却无法使用随机访问排序算法的容器, 所以list特别提供了sort成员函数 (list::sort执行的是稳定排序).

如果需要对list中的对象使用partial_sort或者nth_element算法的话只能通过间接途径来完成:

- 将list中的元素复制到一个提供随机访问迭代器的容器中然后对该容器执行你所期望的算法
- 先创建一个list::iterator的容器, 再对该容器执行相应的算法, 然后通过其中的迭代器访问list的元素
- 利用一个包含迭代器的有序容器中的信息, 通过反复地调用splice成员函数, 将list中的元素调整到期望的目标位置

除此之外, 可以通过使用标准的关联容器来保证容器中的元素始终保持特定的顺序; 也可以使用标准的非STL容器priority_queue, 它总是保持其元素的顺序关系 (对STL容器的定义要求STL容器支持迭代器, 而priority_queue并不支持迭代器, 所以它不能称为STL容器).

依照算法的时间, 空间效率将本条款中讨论过的算法列出如下, 其中消耗资源较少的算法排在前面:

1. partition
2. stable_partition
3. nth_element
4. partial_sort
5. sort
6. stable_sort

对排序算法的选择应该更多地基于你所需要完成的功能，而不是算法的性能

### 第32条: 如果确实需要删除元素, 则需要在 remove 这一类算法之后调用 erase

下面是remove的声明:

```cpp
template<typename ForwardIterator, typename T>
ForwardIterator remove(ForwardIterator first, ForwardIterator last, const T& value);
```

remove需要一对迭代器来指定所要进行操作的元素区间, 不接受容器作为参数, 所以remove并不知道这些元素被存放在哪个容器中. 但是, 从容器中删除元素唯一的办法是调用容器的成员函数, 所以remove不可能从容器中删除元素, 也就是说remove从容器中删除元素, 而容器中的元素数目却不会因此而减少.

**remove不是真正意义上的删除, 因为它做不到.**

remove移动了区间中的元素, 其结果是"不用被删除"的元素被移到了区间的前部并保持原有的顺序, 返回的一个迭代器指向最后一个"不用被删除"的元素之后的位置, 这个返回值相当于该区间新的逻辑结尾.

位于区间后部的元素的值是未定义的, 一般而言会留其旧的值, 所以这里不一定留有被删除的值, 如果不想失去这些值, 那么就应该使用partition而不是remove.

remove的实现大致如下:

```cpp
template<typename ForwardIterator, typename T>
ForwardIterator remove(ForwardIterator first, ForwardIterator last, const T& value)
{
    ForwardIterator result = first;
    while (first != last) {
        if (!(*first == value)) {
            *result = *first;
            ++result;
        }
        ++first;
    }
    return result;
}
```

在内部remove遍历整个区间, 用需要保留的元素的值覆盖掉那些要被删除的元素的值, 这种覆盖是通过对那些需要被覆盖的元素的赋值来完成的, 第33条将要解释的那样, 如果remove所覆盖掉的这些值是指针的话, 那么这可能会存在严重的问题.

如果你真想删除元素, 那么应该在remove之后调用erase:

```cpp
vector<int> v;
...
v.erase(remove(v.begin(), v.end(), value), v.end());
```

由于remove和erase经常组合使用, 所以list的成员函数remove将二者合并, 是STL中唯一一个名为remove并且确实删除了容器中元素的函数.

同时注意, remove并不是唯一一个适用于这种情形的算法, remove_if和unique也需要配合erase删除元素, unique与list的结合也与remove的情形类似.

### 第33条: 对包含指针的容器使用 remove 这一类算法时要特别小心

类的定义和相关vector大致如下:

```cpp
class Widget
{
public:
    ...
    bool isCertified() const;
    ...
};

vector<Widget*> v;
```

当容器中存放的是指向动态分配的对象的指针的时候, 应该避免使用remove和类似的算法 (remove_if和unique), 很多情况下partition算法是个不错的选择.

如果无法避免对这种容器使用remove, 那么一种可以消除该问题的做法是, 在进行erase-remove习惯用法之前, 先把那些指向未被验证过的Widget的指针删除并置成空, 然后清除该容器中所有的空指针.

```cpp
void delAndNullifyUncertified(Widget*& pWidget)
{
    if (!pWidget->isCertified()) {
        delete pWidget;
        pWidget = 0;
    }
}

for_each(v.begin(), v.end(), delAndNullifyUncertified);

v.erase(remove(v.begin(), v.end(), static_cast<Widget*>(0)), v.end());
```

这种做法的前提是你不希望该矢量中保留任何空指针, 如果你希望它保留空指针的话, 你可能只好自己写循环来删除那些满足条件的指针.

如果容器中存放的不是普通指针, 而是具有引用计数功能的智能指针, 那么与remove相关的困难就不再存在了, 你可以直接使用erase-remove习惯用法:

```cpp
typedef shared_ptr<Widget> SPW;
vector<SPW> v;
...
v.push_back(SPW(new Widget));
...
v.erase(remove_if(v.begin(), v.end(), not1(mem_fun(&Widget::isCertified))), v.end());
```

为了使以上的代码能够工作, 编译器必须能够把智能指针类型 (shared_ptr\<Widget\>) 式地转换为对应的内置指针类型 (Widget*).

### 第34条: 了解哪些算法要求使用排序的区间作为参数

要求排序区间或者作用于排序区间更加有效的算法如下:

- binary_search
- lower_bound
- upper_bound
- equal_range
- set_union
- set_intersection
- set_difference
- set_symmetric_difference
- merge
- inplace_merge
- includes
- unique
- unique_copy

#### 要求排序的算法

##### 二分查找算法

binary_search, lower_bound, upper_bound和equal_range 要求排序的区间是因为它们都使用了二分查找算法, 这些算法承诺了对数时间的查找效率.

这些算法并不一定保证对数时间的查找效率, 只有当它们接受了随机访问迭代器的时候,它们才保证有这样的效率.

##### 集合操作算法

set_union, set_intersection, set_difference, set_symmetric_difference 提供了线性时间效率的集合操作.

##### 归并算法

merge和inplace_merge实际上实现了合并和排序的联合操作: 它们读入两个排序的区间, 然后合并成一个新的排序区间, 其中包含了原来两个区间中的所有元素, 它们具有线性时间的性能.

##### includes

includes算法用于判断一个区间是否包含另一个区间, 它要求两个区间都是排序的, 它承诺线性时间的效率.

##### 分析

对于这些算法而言, 他们承诺的时间效率是基于输入区间是排序的这一前提的, 如果输入区间不是排序的, 那么这些算法的时间效率将会变得不可预测.

#### unique和unique_copy

unique和unique_copy与上述讨论过的算法有所不同, 它们即使对于未排序的区间也有很好的行为, C++标准如下描述unique:

> Eliminates all but the first element from every consecutive group of equivalent elements from the range $[first, last)$ and returns a past-the-end iterator for the new logical end of the range.
> (删除 $[first, last)$ 区间中所有连续的等价元素中除了第一个元素之外的所有元素, 并返回一个指向新的逻辑结尾的迭代器)

如果想让unique删除区间中所有重复的元素, 那么所有相同的元素要都是连续存放的, 这正是排序操作所要达到的目标之一. 在实践中, unique通常用于删除一个区间中的所有重复值, 所以总是要确保传给unique的区间是排序的.

此外, unique使用了与remove类似的办法来删除区间中的元素, 而并非真正意义上的删除.

#### 排序的定义

如果你为一个算法提供了一个排序的区间, 而这个算法也带一个比较函数作为参数, 那么一定要保证你传递的比较函数与这个排序区间所用的比较函数有一致的行为.

所有要求排序区间的算法 (本条款中提到的除了unique和unique_copy以外的算法) 均使用等价性来判断两个对象是否"相同", unique和unique_copy在默认情况下使用"相等"判断, 改变这种默认行为只需给这些算法传递一个其他的预定义比较函数作为两个值"相同"的定义即可.

### 第35条: 通过 mismatch 或 lexicographical_compare 实现简单的忽略大小写的字符串比较

用STL实现忽略大小写的字符串比较是否困难取决于所要求的通用性, 例如是否考虑strcmp不支持的国际化的问题.

本节将讨论容易的版本, (因为困难的版本其实与STL关系并不大, 它的问题涉及许多与地域有关的问题，关于这些地域问题请参阅附录A).

当程序员需要忽略大小写的字符串比较功能的时候, 他们往往需要两个不同的调用接口: 一个与strcmp很类似 (将返回一个负数、零或者正数), 另一个与operator\<很类似 (将返回true或者false), 下面将实现这两个接口.

首先需要一种办法来判断两个字符是否相同, 而不去管它们的大小写. 下面的字符比较函数是一个简化了的方案, 它与strcmp的字符串比较方法很相似.

```cpp
int ciCharCompare(char c1, char c2)
{
    int lc1 = tolower(static_cast<unsigned char>(c1));
    int lc2 = tolower(static_cast<unsigned char>(c2));
    if (lc1 < lc2)
        return -1;
    else if (lc1 > lc2)
        return 1;
    else
        return 0;
}
```

static_cast的作用是为了避免在某些平台上char是有符号的而导致的问题, 将它强制转换成一个unsigned char从而确保char有符号时它的值可以用unsigned char来表示, 这也是用int来存储来保存tolower的返回值的原因.

有了ciCharCompare之后很容易写出接口中的第一个函数, 它根据两个字符串之间的关系返回一个负数, 零或者正数, 它建立在mismatch算法的基础上, 因为mismatch将标识出两个区间中第一个对应值不相同的位置, 它要求两个区间都是有效的, 所以需要将短的字符串作为第一个区间传入.

```cpp
int ciStringComparelmpl(const string& s1, const string& s2);

int ciStringCompare(const string& s1, const string& s2)
{
    if (s1.size() <= s2.size()) {
        return ciStringComparelmpl(s1, s2);
    } else {
        return -ciStringComparelmpl(s2, s1);
    }
}

int ciStringComparelmpl(const string& s1, const string& s2)
{
    typedef pair<string::const_iterator, string::const_iterator> PSCI; // pair of string const iterator
    PSCI p = mismatch(s1.begin(), s1.end(), s2.begin(), not2(ptr_fun(ciCharCompare)));
    if (p.first == s1.end()) {
        if (p.second == s2.end()) {
            return 0;
        } else {
            return -1;
        }
    }
    return ciCharCompare(*p.first, *p.second);
}
```

ciStringCompare的第二种实现方法将产生一个很常用的STL判别式 (可以被用作关联容器中的比较函数), 只需要修改ciCharCompare, 使它成为一个具有判别式接口的字符比较函数, 然后把执行字符串比较的工作交给lexicographical_compare:

```cpp
bool ciCharLess(char c1, char c2)
{
    int lc1 = tolower(static_cast<unsigned char>(c1));
    int lc2 = tolower(static_cast<unsigned char>(c2));
    return lc1 < lc2;
}

bool ciStringCompare(const string& s1, const string& s2)
{
    return lexicographical_compare(s1.begin(), s1.end(), s2.begin(), s2.end(), ciCharLess);
}
```

lexicographical_compare是strcmp的一个泛化版本, 可以与任何类型的值的区间一起工作, 它接受一个判别式, 由该判别式来决定两个值是否满足一个用户自定义的准则, 如果第一个区间的值在第二个区间的值之前, 则返回true, 否则返回false.

与此同时, 忽略大小写的字符串比较函数也普遍存在于标准C库的非标准扩展中, 它们的名字一般为strcmp或者strcmpi, 如果你愿意牺牲一点移植性, 并且你知道你的字符串中不会包含内嵌的空字符, 而且你不考虑国际化支持, 那么只需要把两个string转化成const char* 指针, 然后调用strcmp或strcmpi可能是更好的选择.

### 第36条: 理解 copy_if 算法的正确实现

注意: 在 C++11 中, copy_if 算法已经被加入到 STL 中.

STL 中有11个包含 copy 的算法, 但没有 copy_if, 如果想简单地复制区间中满足某个判别式的所有元素, 那就需要自己来实现.

11个copy算法如下:

- copy
- copy_backward
- replace_copy
- replace_copy_if
- remove_copy
- remove_copy_if
- reverse_copy
- unique_copy
- rotate_copy
- partial_sort_copy
- uninitialized_copy

利用remove_copy_if算法可以实现copy_if算法:

```cpp
template<typename InputIterator, typename OutputIterator, typename Predicate>
OutputIterator copy_if(InputIterator begin, InputIterator end, OutputIterator destBegin, Predicate p)
{
    while (begin != end) {
        if (p(*begin)) {
            *destBegin = *begin;
            ++destBegin;
        }
        ++begin;
    }
    return destBegin;
}
```

### 第37条: 使用 accumulate 或者 for_each 进行区间统计

对于常见的一些信息, STL中有专门的算法来完成计算区间任务.

- count: 区间中有多少个元素
- count_if: 区间中满足某个条件的元素有多少个
- max_element: 区间中最大的元素
- min_element: 区间中最小的元素

需要按照某种自定义的方式对区间进行统计处理时, 可以使用accumulate算法, 它和其它3个数值算法 (inner_product, partial_sum, adjacent_difference) 位于numeric头文件中.

accumulate 有两种形式, 第一种形式有两个迭代器和一个初始值, 它返回该初始值加上由迭代器标识的区间中的值的总和, 用于计算和返回的类型与初始值的类型相同, 迭代器则只要求是输入迭代器, 所以 istream_iterator 和 istreambuf_iterator 也是可以的.

第二种形式带一个初始值和一个任意的统计函数, 这使得 accumulate 更加通用.

例如, 考虑如何用accumulate来计算一个容器中字符串的长度总和, 需要下面的统计函数 (有两个参数, 一个是到目前为止区间中的元素的统计值, 另一个是区间的下一个元素, 函数的返回值是新的统计值):

```cpp
string::size_type
stringLengthSum(string::size_type sumSoFar, const string& s)
{
    return sumSoFar + s.size();
}
```

随后这样将 accumulate 和统计函数配合使用:

```cpp
set<string> ss;
...
string::size_type lengthSum = accumulate(ss.begin(), ss.end(), static_cast<string::size_type>(0), stringLengthSum);
```

传给 accumulate 的函数不允许有副作用 (标准要求), 所以这个时候可以使用 for_each 算法, 它的第三个参数是一个函数 (一般是一个函数子类), 对区间中的每个元素都要调用这个函数, 但这个函数只接收一个实参, 执行完毕后会返回它的函数.

先忽略副作用的问题不谈, for_each 和 accumulate 在两个方面有所不同

- 名字 accumulate 暗示着这个算法将会计算出一个区间的统计信息, 而 for_each 听起来就好像是对一个区间的每个元素做一个操作, for_each 来统计一个区间是合法的, 但是不如 accumulate 来得清晰
- accumulate 直接返回我们所要的统计结果, 而for_each却返回一个函数对象, 我们必须从这个函数对象中提取出我们所要的统计信息, 在 C++ 中这意味着我们必须在函数子类中加入一个成员函数以便获得我们想要的统计信息

## 第6章 函数子, 函数子类, 函数及其他

### 第38条: 遵循按值传递的原则来设计函数子类

无论是C还是C++, 都不允许将一个函数作为参数传递给另一个函数, 必须传递函数指针, 函数指针是按值传递的.

尽管可以显式地指明其模板参数的类型强制函数指针按引用传递并返回, 但是STL的使用者几乎从来不会做这样的事情, 并且这样做可能无法通过编译.

由于函数对象往往会按值传递和返回, 所以必须确保你编写的函数对象在经过了传递之后还能正常工作, 这意味着两件事:

- 函数对象必须尽可能地小, 否则复制的开销会非常昂贵
- 函数对象必须是单态的 (即不是多态的), 也就是说不得使用虚函数

但上述要求不能总是得到满足, 一个既允许函数对象可以很大并且/或者保留多态性, 又可以与STL所采用的按值传递函数子的习惯保持一致解决方案是: 将所需的数据和虚函数从函数子类中分离出来, 放到一个新的类中, 然后在函数子类中包含一个指针, 指向这个新类的对象.

如果你希望创建一个包含大量数据并且使用了多态性的函数子类, 那么就应该创建一个小巧的, 单态的类, 其中包含一个指针, 指向另一个实现类, 并且将所有的数据和虚函数都放在实现类中:

```cpp
template<typename T>
class BPFCImpl:
    public unary_function<Widget, void>
{
private:
    Widget pWidget;
    int x;
    ...
    virtual ~BPFCImpl();
    virtual void operator()(const T& val) const;

    friend class BPFC<T>;
};

template<typename T>
class BPFC:         // Big Polymorphic Functor Class
    public unary_function<T, void>
{
private:
    shared_ptr<BPFCImpl<T>> pImpl;

public:
    void operator()(const T& val) const
    {
        pImpl->operator()(val);
    }
};
```

如果函数子类用到了这项技术, 那么它必须以某种合理的方式来支持复制动作. 上述BPFC类必须确保BPFC的复制构造函数正确地处理了它所指向的BPFCImpl对象, 例如使用shared_ptr.

### 第39条: 确保判别式是 "纯函数"

> 判别式 (predicate): 一个返回值为bool类型 (或者可以隐式地转换为bool类型) 的函数
> 纯函数 (pure function): 返回值仅仅取决于其参数的函数, 在 C++ 中纯函数所能访问的数据应该仅局限于参数以及常量
> 判别式类 (predicate class): 一个函数子类, 它的 operator() 是一个判别式

满足上述要求最简单的解决途径就是在判别式类中使用 const 修饰符, 将 operator() 函数声明为 const, 但是这样做还远远不够, 因为函数还是可以访问某些变量, 而一个精心设计的判别式类应该保证其 operator() 函数完全独立于所有这些变量, 也就是说在判别式类中将 operator() 声明为 const 对于判别式的正确行为是必要的但是仍不充分.

### 第40条: 若一个类是函数子, 则应使它可配接

假设有一个包含Widget对象指针的list容器, 另有一个函数可用来判断某个Widget指针所指的对象是否足够 "有趣", 想找到该list中第一个满足isInteresting()条件的Widget指针非常简单:

```cpp
list<Widget*> widgetPtrs;
bool isInteresting(Widget* pw);
...
list<Widget*>::iterator i = find_if(widgetPtrs.begin(), widgetPtrs.end(), isInteresting);
if (i != widgetPtrs.end()) {
    ...
}
```

反之, 如果想找到第一个不满足 isInteresting() 条件的 Widget 指针, 直接在 isInteresting 上作用一个 not1 即使用 `not1(isInteresting)` 是不行的, 必须先将ptr_fun应用在isInteresting上:

```cpp
list<Widget*>::iterator i = find_if(widgetPtrs.begin(), widgetPtrs.end(), not1(ptr_fun(isInteresting)));
```

ptr_fun 完成了一些类型定义的工作, 这些类型定义是not1所必需的, 而 IsInteresting 作为一个基本的函数指针, 它缺少not1所需要的类型定义.

在 STL 中, 4个标准的函数配接器 (not1, not2, bind1st, bind2nd) 都要求一些特殊的类型定义 (argument_type, first_argument_type, second_argument_type, result_type), 提供了这些必要的类型定义的函数对象被称为可配接的 (adaptable) 函数对象, 不同种类的函数子类所需提供的类型定义也不尽相同, 但是除非你要编写自定义的配接器, 否则你并不需要知道有关这些类型定义的细节, 提供这些类型定义最简便的办法是让函数子从特定的基类 (从一个基结构) 继承.

如果函数子类的 operator()只有一个实参, 那么它应该从 std::unary_function 继承; 如果函数子类的 operator() 有两个实参，那么它应该从 std::binary_function 继承.

不过由于 unary_function 和 binary_function 是 STL 提供的模板, 所以不能直接继承它们, 而是继承它们所产生的结构, 这就要求指定某些类型实参:

- 对于unary_function, 指定函数子类 operator() 所带的参数的类型, 以及返回类型
- 对于binary_function, 指定三个类型: operator() 的第一个和第二个参数的类型, 以及返回类型

传递给 unary_function 和 binary_function 的模板参数正是函数子类的 operator() 的参数类型和返回类型, 唯一有点怪异的是, operator() 的返回类型是 unary_function 或 binary_function 的最后一个实参.

传递给 unary_function 或 binary_function 的非指针类型需要去掉 const 和引用 (&) 部分; 而如果 operator() 带有指针参数, 传给 unary_function 或 binary_function 的类型与 operator() 的参数和返回类型完全相同.

```cpp
template<typename T>
class MeetsThreshold:
    public unary_function<Widget, bool>
{
private:
    const T threshold;
public:
    MeetsThreshold(const T&);
    bool operator()(const Widget&) const;
};

struct WidgetNameCompare:
    public binary_function<Widget, Widget, bool>
{
    bool operator()(const Widget&, const Widget&) const;
};

struct PtrWidgetNameCompare:
    public binary_function<Widget*, Widget*, bool>
{
    bool operator()(const Widget*, const Widget*) const;
};
```

如果一个函数子的所有成员都是公有的, 那么通常会将其声明为结构而不是类. 究竟是选择结构还是类来定义函数子纯属个人编码风格, 但是应该注意到, STL中所有的无状态函数子类 (如less\<T\>, plus\<T\>等) 一般都被定义成结构.

可配接的函数对象可以像下面这样使用:

```cpp
list<Widget> widgets;
...
lst<Widget>::reverse_iterator i1 = find_if(widgets.rbegin(), widgets.rend(), not1(MeetsThreshold<int>(42)));
Widget w;
list<Widget*>::iterator i2 = find_if(widgetPtrs.begin(), widgetPtrs.end(), bind2nd(WidgetNameCompare(), w));
```

STL函数对象是C++函数的一种抽象和建模形式, 而每个C++函数只有一组确定的参数类型和一个返回类型. 尽管有时确实需要函数子类具有多种不同的调用形式, 但这样的函数子类是例外, 而不是规则, 通常情况下, 函数子类应该只有一个调用形式, 并且应该从 std::unary_function 或 std::binary_function 继承.

### 第41条: 理解 ptr_fun, mem_fun, mem_fun_ref 的来由

如果有一个函数f和一个对象x, 现在希望在x上调用f, 而我们在x的成员函数之外, 为了执行这个调用C++提供了3种不同的语法:

```cpp
f(x);       // 形式 #1 f为一个非成员函数
x.f();      // 形式 #2 f为一个成员函数, x为一个对象或者对象的引用
p->f();     // 形式 #3 f为一个成员函数, p为一个指向对象的指针
```

STL中一种很普遍的惯例: 函数或者函数对象在被调用的时候, 总是使用非成员函数的语法形式, 也就是形式 #1

所以, 如果 test 函数是 Widget 类的一个非成员函数那么下面的代码是合法的:

```cpp
void test(const Widget&);
vector<Widget> vw;

for_each(vw.begin(), vw.end(), test);   // 形式 #1
```

但是如果 test 是 Widget 类的一个成员函数, 下面的代码就不合法了:

```cpp
class Widget
{
public:
    ...
    void test() const;
};

vector<Widget> vw;
list<Widget*> lpw;

for_each(vw.begin(), vw.end(), &Widget::test);   // 期望形式 #2
for_each(lpw.begin(), lpw.end(), &Widget::test); // 期望形式 #3
```

mem_fun 和 mem_fun_ref 的作用就是将成员函数调用转换成非成员函数调用, 也就是将形式 #2 和 #3 转换成形式 #1, 针对它们所配接的成员函数的原型的不同有几种变化形式, 其中一个如下:

```cpp
template<typename R, typename C>
mem_fun_t<R, C>
mem_fun(R (C::*pmf)());
```

mem_fun带一个指向某个成员函数的指针参数pmf, 并且返回一个mem_fun_t类型的对象;
 mem_fun_t 是一个函数子类, 它拥有该成员函数的指针并提供了operator()函数, 在operator()中调用了通过参数传递进来的对象上的该成员函数.

mem_fun 将形式 #3 转换成形式 #1,并产生一个类型为mem_fun_t的配接器对象;
 而 mem_fun_ref 则将形式 #2 转换成形式 #1, 并产生一个类型为mem_fun_ref_t的配接器对象:

```cpp
for_each(vw.begin(), vw.end(), mem_fun_ref(&Widget::test)); // 形式 #2
for_each(lpw.begin(), lpw.end(), mem_fun(&Widget::test));   // 形式 #3
```

对于 ptr_fun, 它的作用是将一个普通函数转换成一个函数子类, 他将为函数指针引入类型定义, 所以当算法不需要这种类型定义的时候, 可以不使用 ptr_fun, 所以对于 ptr_fun 有两种使用策略: 一种是每次将一个函数传递给一个STL组件的时候总是使用它, STL不会在意而且这样做也不会带来运行时的性能损失; 另一种是只在需要的时候使用它.

mem_fun 和 mem_fun_ref 的情形则截然不同, 每次在将一个成员函数传递给一个 STL 组件的时候你就要使用它们, 它们不仅仅引入了一些类型定义, 还转换调用语法的形式来适应算法.

### 第42条: 确保 less\<T\> 与 operator\< 具有相同的语义

作为一般性的规则, 对std名字空间的组件进行修改是被禁止的, 但是在某些特定的情况下, 有些对std名字空间的修补工作仍然是允许的, 但是要注意, 大多数情况下你应该有比特化std模板更好的选择.

C++ 允许程序员做出一些合理的假设, 假设使用 less 总是等价于使用operator\< 也是合理的, operator\< 不仅仅是 less 的默认实现方式, 它也是程序员期望less所做的事情.

应该尽量避免修改less的行为, 因为这样做很可能会误导其他的程序员, 如果你使用了less, 无论是显式地或是隐式地, 你都需要确保它与 operator\< 具有相同的意义. 如果你希望以一种特殊的方式来排序对象, 那么最好创建一个特殊的函数子类, 它的名字不是 less 就可以, 这样做其实是很简单的.

## 第7章 在程序中使用 STL

### 第43条: 算法调用优先于手写的循环

如果你有一个支持 redraw 的 Widget 类, 当你想重画一个 list 中的所有 Widget 对象的时候, 可以用一个循环来完成, 也可以使用 for_each 算法:

```cpp
class Widget
{
    ...
public:
    void redraw() const;
    ...
};

list<Widget> lw;
...
for(list<Widget>::iterator i = lw.begin(); i != lw.end(); ++i) {
    i->redraw();
}

for_each(lw.begin(), lw.end(), mem_fun_ref(&Widget::redraw));
```

调用算法通常是更好的选择, 它往往优先于任何一个手写循环, 有三个原因:

- **效率**: 算法通常比程序员自己写的循环效率更高
- **正确性**: 自己写循环比使用算法更容易出错
- **可维护性**: 使用算法的代码通常比手写循环的代码更加简洁明了

#### 效率

从效率方面来说, 算法有三个理由比显式循环更好, 其中两个是主要的, 一个是次要的.

次要一点的理由是: 使用算法可以减少冗余的计算, 例如, STL 的实现者尽可能让大多数编译器都能够将循环中的计算提到外面来以避免重复计算.

主要的理由1: 类库实现者可以根据他们对于容器实现的了解程度对遍历过程进行优化

主要的理由2: 除了一些不太重要的算法以外, 其他几乎所有的STL算法都使用了复杂的计算机科学算法

#### 正确性

自己写循环需要关注迭代器的有效性, 以避免迭代器没有被正确地维护好或者使用了无效的迭代器, 使用算法则不需要关心这些问题,  此外, 合理地使用算法也可以避免某些逻辑错误.

#### 可维护性

算法的名称描述了它们的功能, 这使得代码更加容易理解, 也使得代码更加容易维护.

#### 总结

要想表明在一次迭代中该完成什么工作，则使用循环比算法更为清晰; 如果你要做的工作与一个算法所实现的功能很相近, 那么用算法调用更好; 但是如果你的循环很简单, 而若使用算法来实现的话, 却要求混合使用绑定器和配接器或者要求一个单独的函数子类, 那么可能使用手写的循环更好; 如果你在循环中要做的工作很多而且又很复杂, 则最好使用算法调用.

使用了 STL 的精巧的 C++ 程序比不用 STL 的程序所包含的循环要少得多, 这样我们就提高了软件的抽象层次, 而使我们的软件更易于编写, 更易于文档化, 也更易于扩展和维护.
