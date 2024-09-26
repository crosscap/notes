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
