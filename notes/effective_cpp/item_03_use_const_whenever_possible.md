# Item 3: Use `const` whenever possible

## 核心思想

`const` 是一种**语义约束**，告诉编译器和其他程序员"这个对象不应该被修改"。编译器会帮你强制执行这个约束。原则是：**只要你不打算修改某个值，就用 `const` 声明它**。

---

## 1. `const` 与指针

`const` 出现在 `*` 的左边还是右边，决定了约束的对象不同：

```cpp
char greeting[] = "Hello";

char *p = greeting;              // 指针非 const，数据非 const
const char *p = greeting;        // 指针非 const，数据 const（不能通过 p 修改数据）
char * const p = greeting;       // 指针 const，数据非 const（p 不能指向别处）
const char * const p = greeting; // 指针 const，数据 const（都不能改）
```

**判断技巧**：`const` 在 `*` 左边 → 数据 const；`const` 在 `*` 右边 → 指针 const。也可以**从右往左读**声明。

当 `const` 修饰指向的数据时，以下两种写法等价：

```cpp
void f1(const Widget *pw);   // const 在类型名前
void f2(Widget const *pw);   // const 在类型名后，* 左侧
```

## 2. `const` 与 STL 迭代器

迭代器本质类似指针，因此 `const` 的语义也类似：

- `const std::vector<int>::iterator` ≈ `T* const`：迭代器不能移动，但能修改指向的值
- `std::vector<int>::const_iterator` ≈ `const T*`：迭代器能移动，但不能修改指向的值

```cpp
std::vector<int> vec;

const std::vector<int>::iterator iter = vec.begin();
*iter = 10;   // OK
++iter;       // 错误！iter 本身是 const

std::vector<int>::const_iterator cIter = vec.begin();
*cIter = 10;  // 错误！指向的数据是 const
++cIter;      // OK
```

> **现代 C++ 补充**：C++11 引入了 `cbegin()` / `cend()`，可以直接获得 `const_iterator`：`auto cIter = vec.cbegin();`

## 3. `const` 用于函数返回值

让函数返回 `const` 值可以防止调用者犯错：

```cpp
class Rational { ... };
const Rational operator*(const Rational& lhs, const Rational& rhs);
```

如果不返回 `const`，调用者可能写出 `(a * b) = c` 这样的代码，或者因为手误把 `==` 写成 `=`：

```cpp
if (a * b = c) ...  // 本意是 a * b == c
```

内置类型不允许对运算结果赋值，自定义类型也应该保持一致。

> **现代 C++ 补充**：C++11 的移动语义下，返回 `const` 值会阻止移动构造/移动赋值（`const` 右值无法被 move），性能代价大于防错收益，因此**现代 C++ 中通常不再推荐返回 `const` 值对象**。

### 返回值 `const` 与其他 `const` 的区别

`const` 参数、`const` 成员函数（修饰 `*this`）、`const` 对象，这三者的作用模式相同——**约束"接收方"不能修改某个已存在的对象**，都是在说"这个东西已经存在了，你不许改它"。

返回值的 `const` 则不同——它约束的是一个**刚被创造出来的临时对象**，作用方向是反过来的：**限制调用者拿到结果后能做什么**。

| | 约束谁 | 约束的时机 | 承诺方向 |
|---|---|---|---|
| `const` 参数 | 函数**内部** | 函数执行期间 | "我承诺不改你的东西" |
| `const` 成员函数 | 函数**内部**（`*this`） | 函数执行期间 | "我承诺不改你的东西" |
| `const` 对象 | 该对象的**所有使用者** | 对象的整个生命期 | "我承诺不改你的东西" |
| `const` 返回值 | 函数的**调用者** | 调用者拿到结果之后 | "我给你的东西，你不许改" |

## 4. `const` 成员函数

### 为什么重要

1. **接口清晰**：让使用者一眼区分哪些成员函数会修改对象、哪些不会
2. **支持 `const` 对象操作**：按 reference-to-const 传递对象是提升性能的基本手段（见 Item 20），如果类没有 `const` 成员函数，`const` 引用就几乎无法使用

### `const` 重载

仅 `const` 性不同的成员函数可以构成重载。编译器根据调用对象的 `const` 性自动选择版本：

```cpp
class TextBlock {
public:
    const char& operator[](std::size_t position) const   // const 对象调用
    { return text[position]; }

    char& operator[](std::size_t position)               // 非 const 对象调用
    { return text[position]; }

private:
    std::string text;
};

TextBlock tb("Hello");
const TextBlock ctb("World");

tb[0];    // 调用非 const 版本，返回 char&，可读可写
ctb[0];   // 调用 const 版本，返回 const char&，只读
```

注意：非 `const` 版本必须返回**引用**（`char&`），返回值类型（`char`）会导致赋值无法编译，且修改的只是副本。

## 5. Bitwise constness vs. Logical constness

### Bitwise constness（物理常量性）

- C++ 语言采用的定义：`const` 成员函数不修改对象的任何非 static 数据成员
- 编译器容易检测：看有没有对数据成员赋值
- **漏洞**：如果对象持有指针，函数没改指针本身（通过 bitwise 检查），但可以通过指针修改指向的数据

```cpp
class CTextBlock {
public:
    char& operator[](std::size_t position) const   // bitwise const，但语义上不 const
    { return pText[position]; }
private:
    char *pText;
};

const CTextBlock cctb("Hello");
char *pc = &cctb[0];
*pc = 'J';   // cctb 变成了 "Jello"！const 对象被修改了
```

### Logical constness（逻辑常量性）

- `const` 成员函数可以修改对象内部的某些 bit，但前提是**调用者察觉不到**
- 典型场景：缓存、延迟计算

### `mutable` 关键字

`mutable` 让非 static 数据成员可以在 `const` 成员函数中被修改，弥合 bitwise 和 logical constness 的差距：

```cpp
class CTextBlock {
public:
    std::size_t length() const;
private:
    char *pText;
    mutable std::size_t textLength;     // 即使在 const 成员函数中也能修改
    mutable bool lengthIsValid;
};

std::size_t CTextBlock::length() const
{
    if (!lengthIsValid) {
        textLength = std::strlen(pText);   // OK，mutable 成员
        lengthIsValid = true;               // OK
    }
    return textLength;
}
```

## 6. 避免 `const` / 非 `const` 成员函数的代码重复

当两个版本的逻辑完全相同时，让**非 `const` 版本调用 `const` 版本**：

```cpp
class TextBlock {
public:
    const char& operator[](std::size_t position) const
    {
        // ... 边界检查、日志、数据校验等
        return text[position];
    }

    char& operator[](std::size_t position)
    {
        return const_cast<char&>(                        // 3. 去掉返回值的 const
            static_cast<const TextBlock&>(*this)         // 1. 给 *this 加上 const
            [position]                                   // 2. 调用 const 版本
        );
    }
};
```

**为什么安全**：能调用非 `const` 版本的一定持有非 `const` 对象，去掉返回值的 `const` 不会修改真正的 `const` 对象。

**绝不能反过来**（const 版本调用非 const 版本）：`const` 函数承诺不修改对象，但非 `const` 函数没有这个承诺，会有违反 `const` 约束的风险。

### 从"承诺"的角度理解 `const` 转型的方向性

`const` 本质上是一个**承诺**："我不会修改这个对象"。基于承诺的逻辑：

- **非 `const` → `const`（加承诺）：安全，随时可以做**。原本没承诺不改，现在主动承诺不改——承诺更严格了，不会违反任何已有约束。所以 `static_cast<const T&>` 是安全的，非 `const` 对象可以自由调用 `const` 成员函数。

- **`const` → 非 `const`（撤销承诺）：危险，需要 `const_cast` 强制转换**。已经承诺过不改了，现在想反悔——如果原始对象真的是 `const`（编译器可能把它放在只读内存），撤销承诺后修改就是未定义行为。所以 `const` 对象不能调用非 `const` 成员函数，编译器从语法层面就禁止了。

这也解释了为什么"非 `const` 版本调用 `const` 版本"是唯一正确的方向：

| 方向 | 承诺分析 | 结论 |
|---|---|---|
| 非 const 调用 const | 调用者没做过承诺 → 临时加上承诺调用 const 版本 → 拿到结果后撤销承诺（`const_cast`）。因为对象本身不是 const，撤销承诺不违反任何约束 | **安全** |
| const 调用非 const | 调用者已承诺不改 → 强行撤销承诺（`const_cast`）去调用可能修改对象的函数。如果对象本身是 const，就是违背承诺 | **危险，未定义行为** |

> **现代 C++ 补充**：C++23 引入了 deducing this（显式对象参数），可以用一个函数模板同时覆盖 `const` 和非 `const` 版本，彻底消除双重转型的写法。C++23 之前，书中的模式仍然是标准做法。

---

## Things to Remember

- 声明 `const` 帮助编译器检测使用错误。`const` 可以用在任何作用域的对象上、函数参数和返回类型上、以及成员函数上。
- 编译器强制的是 **bitwise constness**，但你编程时应该追求 **logical constness**。
- 当 `const` 和非 `const` 成员函数的实现基本相同时，让**非 `const` 版本调用 `const` 版本**可以消除代码重复。
