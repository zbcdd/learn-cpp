# Item 2: Prefer consts, enums, and inlines to #defines

## 核心思想

**"宁用编译器，不用预处理器"（prefer the compiler to the preprocessor）。**

`#define` 在预处理阶段做文本替换，编译器根本看不到宏名。这导致三个问题：调试困难（符号表中没有宏名）、错误信息难以理解（只显示替换后的字面值）、可能导致目标代码膨胀（同一字面值出现多份副本）。应当用 `const`、`enum`、`inline` 来替代 `#define` 的各种用途。

## 用 const 替代 #define 定义常量

### 基本用法

```cpp
// 不好：预处理器替换，编译器看不到 ASPECT_RATIO
#define ASPECT_RATIO 1.653

// 好：编译器可见，进入符号表，最多一份副本
const double AspectRatio = 1.653;
```

### 特殊情况一：常量指针

常量定义通常放在头文件中，指针本身也要声明为 `const`，需要写两个 `const`：

```cpp
const char * const authorName = "Scott Meyers";
```

更好的做法是使用 `std::string`：

```cpp
const std::string authorName("Scott Meyers");
```

### 特殊情况二：类专属常量

要让常量的作用域限定在类内部，声明为 `static const` 成员：

```cpp
class GamePlayer {
private:
    static const int NumTurns = 5;   // 声明（整型可类内初始化）
    int scores[NumTurns];
};
```

- 这是**声明**而非定义。对于 `static` 整型常量（`int`、`char`、`bool`），只要不取地址，光声明就够用
- 如果需要取地址或编译器要求定义，在实现文件中补充定义（不再给初始值）：

```cpp
const int GamePlayer::NumTurns;   // 定义，放在 .cpp 文件中
```

- 对于非整型（如 `double`），必须在类外定义并赋值：

```cpp
// 头文件
class CostEstimate {
private:
    static const double FudgeFactor;
};

// 实现文件
const double CostEstimate::FudgeFactor = 1.35;
```

### `#define` 没有作用域和封装性

`#define` 一旦定义就在整个编译单元剩余部分有效，无法限定在类内，也不存在 "private `#define`"。而 `const` 数据成员可以享受完整的封装保护。

## The Enum Hack

当编译器不支持类内初始化整型常量，又需要在编译期使用该值时（如声明数组大小），可以用 enum hack：

```cpp
class GamePlayer {
private:
    enum { NumTurns = 5 };     // NumTurns 是 5 的符号名
    int scores[NumTurns];      // OK
};
```

Enum hack 值得了解的理由：

1. **行为更接近 `#define`**：不能取 `enum` 的地址（`const` 可以），如果你想禁止别人获取常量的指针/引用，`enum` 能表达这种约束
2. **不会导致不必要的内存分配**：和 `#define` 一样，`enum` 不会分配存储空间
3. **实用主义**：大量现有代码使用 enum hack，它是模板元编程的基础技巧

## 用 inline 模板函数替代类函数宏

### 类函数宏的问题

```cpp
#define CALL_WITH_MAX(a, b) f((a) > (b) ? (a) : (b))
```

即使给所有参数加了括号，仍然会出现不可预测的行为：

```cpp
int a = 5, b = 0;
CALL_WITH_MAX(++a, b);       // a 被递增两次！
CALL_WITH_MAX(++a, b+10);   // a 只被递增一次
```

原因是宏做文本替换，`++a` 在展开后出现多次，副作用次数取决于三元表达式的求值路径。

### 解决方案：inline 模板函数

```cpp
template<typename T>
inline void callWithMax(const T& a, const T& b)
{
    f(a > b ? a : b);
}
```

优势：
- 参数只求值一次，行为可预测
- 类型安全
- 遵守作用域和访问控制（可以是类的 `private` 成员函数）

## 预处理器并未完全淘汰

有了 `const`、`enum`、`inline` 后，对 `#define` 的需求大大减少，但预处理器仍有用武之地：
- `#include` 仍然不可或缺
- `#ifdef` / `#ifndef` 在条件编译中仍然重要

## 现代 C++ 补充

### `const` vs `constexpr` 辨析

这两个关键字容易混淆，核心区别在于**约束的时机不同**：

- **`const`**：表示"初始化之后不可修改"，但值**可以在运行时确定**
- **`constexpr`**：表示"值**必须在编译期就能确定**"，隐含 `const`

```cpp
int getSize() { return 42; }

const int a = getSize();       // OK：运行时初始化，之后不可改
constexpr int b = getSize();   // 错误：getSize() 不是编译期可求值的
constexpr int c = 42;          // OK：42 编译期就能确定
```

**用于变量时**，`constexpr` 比 `const` 更严格——它强制要求编译期求值，而 `const` 只是"不可修改"：

```cpp
const int x = 10;             // 编译期常量（恰好是字面值）
const int y = someFunc();     // 运行时常量（只是不可修改）
constexpr int z = 10;         // 保证是编译期常量
```

**用于函数时**，`constexpr` 函数是"两栖"的——参数是编译期常量时在编译期求值，否则退化为运行时调用：

```cpp
constexpr int square(int x) { return x * x; }

constexpr int val = square(5);  // 编译期计算，val = 25
int n = 5;
int val2 = square(n);           // 运行时调用，也合法
```

**用于类的静态成员时**（与本 Item 直接相关）：

```cpp
class GamePlayer {
    static const int NumTurns = 5;          // C++03 风格
    static constexpr int MaxLevel = 10;     // C++11 起，更明确表达"编译期常量"
    static constexpr double Rate = 0.75;    // const 在 C++03 中做不到非整型类内初始化
};
// C++17 起，constexpr 静态成员隐含 inline，不再需要类外定义
```

简明对比：

| 特性 | `const` | `constexpr` |
|------|---------|-------------|
| 含义 | 不可修改 | 编译期可求值 |
| 值的确定时机 | 编译期或运行时均可 | 必须编译期 |
| 隐含关系 | — | 隐含 `const` |
| 能修饰函数 | 仅成员函数（表示不修改 `*this`） | 普通函数和成员函数均可 |
| 非整型类静态成员类内初始化 | C++03 不行 | C++11 起可以 |
| 需要类外定义 | 可能需要 | C++17 起不需要（隐含 `inline`） |

一句话总结：**`const` 是对程序员的承诺："我不会改它"；`constexpr` 是对编译器的承诺："你在编译时就能算出它的值"。** 能用 `constexpr` 的场景优先用 `constexpr`。

### 其他现代 C++ 补充

- **`constexpr`（C++11 起）**：可以替代 `const` 和 enum hack，在类内直接定义各种类型的静态常量：
  ```cpp
  class GamePlayer {
  private:
      static constexpr int NumTurns = 5;       // 整型
  };

  class CostEstimate {
  private:
      static constexpr double FudgeFactor = 1.35;  // 非整型也行
  };
  ```
  `constexpr` 隐含 `const`，在 C++17 中 `constexpr` 静态成员变量隐含 `inline`，彻底解决了需要类外定义的问题。

- **`constexpr` 函数（C++11 起）**：如果函数的目的是编译期计算，`constexpr` 函数比 `inline` 模板函数更合适。

- **现代编译器的内联优化**已非常智能，即使不写 `inline` 关键字，编译器也会自主判断是否内联。`inline` 在现代 C++ 中更多用于解决 ODR（One Definition Rule）问题，而非作为优化提示。

- **C++20 `modules`**：`import` 机制有望逐步替代 `#include`，进一步减少对预处理器的依赖。

## Things to Remember

- **对于简单常量，优先使用 `const` 对象或 `enum`，而非 `#define`。**
- **对于类函数宏（function-like macros），优先使用 `inline` 函数，而非 `#define`。**
