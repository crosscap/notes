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
    - 输入迭代器（只读迭代器，每个被遍历到的位置上只能被读取一次）
    - 输出迭代器（只写迭代器，每个被遍历到的位置上只能被写入一次）
- 前向迭代器（读写，只能向前移动，所有STL都提供比此更强大的迭代器）
- 双向迭代器（读写，可向前向后移动）
- 随机访问迭代器（读写，并提供了“迭代器算术”）

### 其他

所有重载了函数调用操作符（即operator()）的类都是一个函数子类（functor class）。这些类的对象可以被当作函数来调用。

绑定器：STL中的函数子类，用于将一个函数和一个或多个参数绑定在一起。包括`bind1st`和`bind2nd`。

### 时间复杂度

- O(1)：常数时间
- O(logN)：对数时间
- O(N)：线性时间

## 第1章　容器

### 第1条：慎重选择容器类型

#### C++容器简要回顾

- **标准STL序列容器** ：vector、string、deque和list。
- **标准STL关联容器** ：set、multiset、map和multimap。
- **非标准序列容器** ：slist（单向链表）和rope（“重型”string）。
- **非标准关联容器** ：hash_set、hash_multiset、hash_map和hash_multimap 。
- **vector\<char\>作为string的替代** 。
- **vector作为标准关联容器的替代** 。
- **几种标准的非STL容器** ：包括数组、bitset、valarray、stack、queue和priority_queue。

#### STL容器的一种分类方法

- 连续内存容器（基于数组的容器）
    - vector
    - string
    - deque
    - rope
- 基于节点的容器
    - list
    - slist
    - 标准的关联容器（平衡树）
    - 非标准的关联容器（哈希表）

#### 选择容器的一些原则

你是否需要在容器的任意位置插入新元素？如果需要，就选择序列容器

你是否关心容器中的元素是排序的？

你选择的容器必须是标准C++的一部分吗？

你需要哪种类型的迭代器？

当发生元素的插入或删除操作时，避免移动容器中原来的元素是否很重要？

容器中数据的布局是否需要和C兼容？

元素的查找速度是否是关键的考虑因素？

如果容器内部使用了引用计数技术（reference counting），你是否介意？

对插入和删除操作，你需要事务语义（transactional semantics）吗？

你需要使迭代器、指针和引用变为无效的次数最少吗？

如果序列容器的迭代器是随机访问类型，而且只要没有删除操作发生，且插入操作只发生在容器的末尾，则指向数据的指针和引用就不会变为无效，这样的容器是否对你有帮助？

### 第2条：不要试图编写独立于容器类型的代码

这些限制的根源在于，对不同类型的序列容器，使迭代器、指针和引用无效（invalidate）的规则是不同的。

考虑到有时候不可避免地要从一种容器类型转到另一种，你可以使用常规的方式来实现这种转变：使用封装（encapsulation）技术。最简单的方式是通过对容器类型和其迭代器类型使用类型定义（typedef）。

所以，不要这么写：

```cpp
class Widget { };

vector<Widget> vw;
Widget bestWidget;

vector<Widget>::iterator i = find(vw.begin(), vw.end(), bestWidget);
```

而应该这么写：

```cpp
class Widget { };
typedef vector<Widget> WidgetContainer;
typedef WidgetContainer::iterator WCIterator;

widgetContainer cw;
widget bestWidget;

WCIterator i = find(cw.begin(), cw.end(), bestWidget);
```

这样就使得改变容器类型要容易得多，尤其当这种改变仅仅是增加一个自定义的分配子时，就显得更为方便。

```cpp
class Widget { };
template <typename T> SpecialAllocator { };
typedef vector<Widget, SpecialAllocator<Widget>> WidgetContainer;
typedef WidgetContainer::iterator WCIterator;

widgetContainer cw;
widget bestWidget;

WCIterator i = find(cw.begin(), cw.end(), bestWidget);
```

类型定义只不过是其他类型的别名，所以它带来的封装纯粹是词法（lexical）上的。

要想减少在替换容器类型时所需要修改的代码，你可以把容器隐藏到一个类中，并尽量减少那些通过类接口（而使外部）可见的、与容器相关的信息。
