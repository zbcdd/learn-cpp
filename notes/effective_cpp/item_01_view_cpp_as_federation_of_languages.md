# Item 1: View C++ as a federation of languages

## 核心思想

不要把 C++ 看作一门单一的语言，而是看作一个**由多个相关子语言组成的联邦**（a federation of related languages）。在每个子语言内部，规则是简单、直接、易记的；但当你从一个子语言切换到另一个子语言时，适用的"最佳实践"会发生变化。

很多人觉得 C++ "规则混乱、到处是例外"，其实不是 C++ 自相矛盾，而是不同子语言之间适用不同的规则。一旦意识到这一点，很多看似矛盾的规则就变得合理了。

## C++ 的四个子语言

### 1. C

C++ 的底层基础。块（blocks）、语句、预处理器、内置数据类型、数组、指针等都来自 C。当你使用这部分特性时，适用的规则是 C 的规则：**没有模板、没有异常、没有重载**。

### 2. Object-Oriented C++

即当年"C with Classes"所涵盖的内容：类（构造函数、析构函数）、封装、继承、多态、虚函数（动态绑定）等。经典的面向对象设计原则在这个子语言中最为适用。

### 3. Template C++

泛型编程部分，也是大多数程序员最不熟悉的部分。模板的影响遍及整个 C++，很多编程规则都有"模板专属条款"。模板甚至催生了一个全新的编程范式——**模板元编程（TMP, Template Metaprogramming）**。

### 4. The STL

STL 是一个模板库，但它是一个非常特殊的模板库。它在**容器（containers）、迭代器（iterators）、算法（algorithms）、函数对象（function objects）** 方面有自己的一套约定。当你使用 STL 时，需要遵循 STL 的惯例。

## 经典例子：参数传递策略随子语言而变

| 子语言 | 推荐的参数传递方式 | 原因 |
|--------|-------------------|------|
| **C** 部分 | pass-by-value | 内置类型拷贝开销小，更高效 |
| **Object-Oriented C++** | pass-by-reference-to-const | 用户定义的构造/析构函数使拷贝开销大 |
| **Template C++** | pass-by-reference-to-const | 你甚至不知道传入的对象是什么类型 |
| **STL** | pass-by-value | 迭代器和函数对象本质上模仿 C 指针，按值传递是惯例 |

## Things to Remember

- C++ 高效编程的规则因你所使用的 C++ 子语言部分而异。

---

## 【补充】现代 C++ 视角

本书写于 C++03 时代（第三版 2005 年出版），以下是现代 C++ 的相关补充：

### 移动语义对参数传递的影响

C++11 引入了**移动语义（move semantics）**，在一定程度上改变了参数传递的考量。对于函数内部**需要持有一份副本**的场景，**pass-by-value + move** 有时比 pass-by-reference-to-const 更简洁。

#### 方案对比：setter 函数

**方案一：两个重载（性能最优）**

```cpp
class Widget {
    std::string name_;
public:
    void setName(const std::string& n) { name_ = n; }         // 左值：拷贝
    void setName(std::string&& n)      { name_ = std::move(n); } // 右值：移动
};
```

性能最优，但每多一个参数就要多写一倍的重载，N 个参数需要 2^N 个重载。

**方案二：pass-by-value + move（代码更简洁）**

```cpp
class Widget {
    std::string name_;
public:
    void setName(std::string n) {   // 按值传入
        name_ = std::move(n);       // 然后移动
    }
};
```

只需要一个函数，自动适配两种调用场景：

- **传入左值**时：拷贝构造 `n`，然后 move → 共计 **1 次拷贝 + 1 次移动**
- **传入右值**时：移动构造 `n`，然后 move → 共计 **2 次移动**

相比两个重载版，每种情况多了一次移动操作。但对于 `std::string` 这类类型，移动开销非常小（本质上就是几个指针/整数的拷贝），通常可以忽略不计。

#### 为什么 pass-by-value 版会多一次移动？

关键在于**引用绑定不是移动**。两个重载版中，参数是引用类型（`const std::string&` 或 `std::string&&`），引用只是原对象的别名，绑定过程**零开销**，不会触发任何构造函数。函数内部直接对原对象执行一次赋值操作即可。

```cpp
// 两个重载版的执行过程：
w.setName(s);              // n 绑定到 s（零开销）→ name_ = n（1 次拷贝赋值）
w.setName(std::string("hi")); // n 绑定到临时对象（零开销）→ name_ = std::move(n)（1 次移动赋值）
```

而 pass-by-value 版中，参数 `n` 是一个独立的值对象，调用时必须**先构造出 `n`**（拷贝或移动），然后函数内部再把 `n` 移动给 `name_`。这个"构造 `n`"的步骤就是多出来的那一次操作。

```cpp
// pass-by-value 版的执行过程：
w.setName(s);              // 拷贝构造 n（1 次拷贝）→ name_ = std::move(n)（1 次移动赋值）
w.setName(std::string("hi")); // 移动构造 n（1 次移动）→ name_ = std::move(n)（1 次移动赋值）
```

一句话总结：**引用版参数直接"指向"原对象，一步到位赋值；值版参数必须先构造出一个副本，再从副本移动过去，所以多了一步。**

**适用前提**：函数内部本来就需要持有一份副本（如存入成员变量）。如果函数只是读取参数而不需要拷贝，`const&` 仍然是最优选择。

### 按值传参时编译器如何选择拷贝/移动

按值传参时，编译器根据实参的**值类别（value category）** 自动选择调用拷贝构造还是移动构造：

- **左值**（lvalue）：有名字、可以取地址 → 调用**拷贝构造函数**
- **右值**（rvalue）：临时对象或 `std::move()` 的结果 → 调用**移动构造函数**

```cpp
std::string s = "hello";

w.setName(s);                  // s 是左值 → 拷贝构造 n
w.setName(std::string("hi"));  // 临时对象是右值 → 移动构造 n
w.setName(std::move(s));       // std::move 把左值转为右值 → 移动构造 n
```

这不是 pass-by-value 的特殊行为，而是 C++11 以来所有按值传参的通用规则。

### `const T&` 为什么是"万能接收器"却不够完美

`const std::string&` 可以绑定到三种情况：

1. **非 const 左值**：`std::string s = "hi"; setName(s);`
2. **const 左值**：`const std::string s = "hi"; setName(s);`
3. **右值**：`setName(std::string("hi"));` — C++ 允许 const 引用绑定到临时对象

这也是为什么在 C++11 之前，`const T&` 被当作"万能接收器"来用。但问题在于：**无论传入的是什么，函数内部都只能执行拷贝赋值**，因为你无法从一个 const 引用移动。所以即使调用者传入了一个右值（本可以被移动），也浪费了一次不必要的拷贝。这正是 C++11 引入右值引用重载的动机。

### 其他补充

- C++11/14/17 引入了 **lambda 表达式**，在很大程度上替代了 STL 中传统的函数对象（function objects），但 STL 基于迭代器和算法组合的设计哲学至今依然适用。
- 现代 C++ 可以说还有第五个"子语言"：**constexpr 编程 / 编译期计算**，它与 TMP 有相似之处但语法更友好、更强大（C++20 的 `consteval`、`constexpr` 容器等）。
