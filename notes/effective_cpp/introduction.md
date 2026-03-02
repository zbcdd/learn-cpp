# Introduction 读书笔记

> Effective C++, 3rd Edition — Scott Meyers
> PDF 页码：22-31（书本页码：1-10）

## 本书定位

- **目标读者**：已经掌握 C++ 语法基础，想要写出更好的 C++ 程序的开发者。
- **不是**全面参考手册，也**不是**入门教程。
- 本书是 **55 条具体建议（Items）**，每条相对独立，但彼此有交叉引用。
- 核心价值：别的书教你语言各部分是什么、怎么编译通过；这本书教你**如何组合它们写出好程序**，以及**如何避免编译器不会告诉你的问题**。

## 两大类建议

1. **通用设计策略**：继承 vs 模板、public vs private 继承、组合 vs private 继承、成员函数 vs 非成员函数、值传递 vs 引用传递……
2. **具体语言特性的细节**：赋值运算符返回类型、virtual destructor、`operator new` 内存不足时的行为等。

## 重要原则

- **没有"唯一正确之道"** — 每条 Item 都是 guideline 而非绝对规则。
- **解释（rationale）是最重要的部分** — 只有理解了"为什么"，才能判断某条建议是否适用于你的具体场景。
- 只涉及**标准 C++**，不搞平台相关 hack，可移植性是关键考量。

## 术语表

### Declaration vs Definition

- **Declaration（声明）**：告诉编译器名字和类型，省略细节。
  ```cpp
  extern int x;                      // 对象声明
  std::size_t numDigits(int number); // 函数声明
  class Widget;                      // 类声明
  template<typename T>               // 模板声明
  class GraphNode;
  ```
- **Definition（定义）**：提供声明所省略的细节（分配内存、提供函数体、列出类成员）。
  ```cpp
  int x;                                    // 对象定义
  std::size_t numDigits(int number) { ... } // 函数定义
  class Widget { public: Widget(); ... };   // 类定义
  ```

### Signature（签名）

- 函数的参数类型和返回类型。如 `numDigits` 的签名是 `std::size_t(int)`。
- **注意**：C++ 标准的"签名"定义不包含返回类型，但 Meyers 在本书中将返回类型也视为签名的一部分。
- 标准签名包含成员函数的 `const`/`volatile` 限定符，因此 `const` 与非 `const` 版本是不同的签名，可以重载：
  ```cpp
  class Widget {
      int& operator[](std::size_t index);             // 非 const 版本
      const int& operator[](std::size_t index) const; // const 版本 — 不同签名
  };
  ```

### `std::size_t`

- 一个无符号整数类型的 typedef（别名），具体对应哪个类型由平台和编译器决定（implementation-defined）：64 位系统上通常是 `unsigned long` 或 `unsigned long long`，32 位系统上通常是 `unsigned int`。标准只要求它是无符号的，且足够大以表示任何对象的大小。
- 也是 `vector`/`deque`/`string` 的 `operator[]` 参数类型。
- 因 C 的历史原因，可能在全局作用域、`std` 命名空间或两者都有，取决于 `#include` 的头文件。

### `explicit` 关键字

- 阻止编译器进行**隐式类型转换**，仍允许显式转换：
  ```cpp
  class B {
  public:
      explicit B(int x = 0, bool b = true);
  };

  void doSomething(B bObject);

  doSomething(28);    // 错误！不能从 int 隐式转换到 B
  doSomething(B(28)); // OK，显式构造
  ```
- Meyers 的建议：**除非有充分理由允许隐式转换，否则一律声明为 `explicit`**。

### Default Constructor（默认构造函数）

- 可以**无参调用**的构造函数。两种情况：
  - 没有参数：`A()`
  - 每个参数都有默认值：`explicit B(int x = 0, bool b = true)` — 这也是 default constructor
- `explicit C(int x)` **不是** default constructor（参数无默认值，不能无参调用）

### Copy Constructor vs Copy Assignment Operator

```cpp
class Widget {
public:
    Widget();                                // default constructor
    Widget(const Widget& rhs);              // copy constructor
    Widget& operator=(const Widget& rhs);   // copy assignment operator
};

Widget w1;          // default constructor
Widget w2(w1);      // copy constructor
w1 = w2;            // copy assignment operator
Widget w3 = w2;     // copy constructor！（不是 assignment）
```

**判断规则**：定义新对象 → constructor；对象已存在 → assignment。

- **Pass-by-value 的本质就是调用 copy constructor**。
- 用户自定义类型一般应该用 **pass-by-reference-to-const**（详见 Item 20）。

### 引用与 const：`int&` vs `const int&`

`int&` 是对 `int` 的引用（可读写），`const int&` 是对 `const int` 的引用（只读）：

```cpp
int x = 10;

int& r = x;        // 可读可写
r = 20;             // OK，x 变成 20

const int& cr = x;  // 只读
cr = 30;            // 编译错误：不能通过 const 引用修改
```

关键区别在于**能绑定什么**：

```cpp
void f(int& r);        // 只接受非 const 左值
void g(const int& r);  // 非 const 左值、const 左值、临时对象都接受
```

- **非 const 左值**：有名字、可修改的对象，如 `int x = 10;` 中的 `x`。
- **const 左值**：有名字、不可修改的对象，如 `const int y = 20;` 中的 `y`。
- **临时对象（右值）**：表达式求值产生的无名中间结果，如 `3 + 4`、`std::string("hello")`、字面量 `42`。

```cpp
int x = 10;
const int y = 20;

f(x);       // OK — 非 const 左值
f(y);       // 错误 — const 左值，int& 暗示可能修改，违反 const 约束
f(42);      // 错误 — 临时对象，非 const 引用不能绑定

g(x);       // OK — 非 const 左值
g(y);       // OK — const 左值
g(42);      // OK — 临时对象，const 引用会延长其生命周期
```

这就是为什么 `const T&` 是按引用传递的首选——既避免拷贝，又能接受所有类型的实参，还向调用方承诺不修改。

### Interface（接口）

- C++ 没有 Java/C# 的 `interface` 关键字（Item 31 讨论如何近似实现）。
- Meyers 用 "interface" 泛指：函数签名、类的可访问元素（public/protected/private interface）、模板类型参数的约束。

### Client（使用者）

- 使用你写的代码（接口）的人或代码。
- **设计哲学：写代码时始终想着使用者。**

### Undefined Behavior（未定义行为）

- 某些操作在 C++ 中的行为是字面意义上的"未定义"，运行时什么都可能发生。
  ```cpp
  int *p = 0;
  std::cout << *p;    // 解引用空指针 → UB

  char name[] = "Darla";
  char c = name[10];  // 数组越界 → UB
  ```
- 最可能的结果：程序时而正常、时而崩溃、时而产生错误结果。

## 命名惯例

| 缩写 | 含义 |
|------|------|
| `lhs` / `rhs` | left-hand side / right-hand side（二元运算符参数名） |
| `pw`, `pa` | pointer to Widget, pointer to Airplane |
| `rw`, `ra` | reference to Widget, reference to Airplane |
| `mf` | member function |
| `ctor` / `dtor` | constructor / destructor |
| `Widget` | 通用示例类名，无特殊含义 |

## Threading Considerations

- C++03 及之前的标准**没有线程概念**。Meyers 会在相关 Item 中指出多线程下的注意事项。
- **现代补充**：C++11 已引入 `<thread>`、`<mutex>`、`<atomic>` 等标准线程支持，这段内容有历史局限性。

## TR1 和 Boost

- **TR1**（Technical Report 1）：向标准库添加新功能的规范（hash table、智能指针、正则表达式等），组件在 `std::tr1` 命名空间。
- **Boost**：提供可移植、peer-reviewed 的开源 C++ 库，TR1 大部分功能基于 Boost。
- **现代补充**：TR1 的内容已全部纳入 C++11 标准，直接在 `std` 命名空间使用（如 `std::shared_ptr`、`std::unordered_map`、`std::regex`）。Boost 至今仍然活跃，是很多标准库新功能的试验场。
