# Item 6: Explicitly disallow the use of compiler-generated functions you do not want

## 核心思想

有些类天然不应该被拷贝（如代表唯一资源的对象）。但正如 Item 5 所述，如果你不声明 copy constructor 和 copy assignment operator，编译器会自动生成。本条款讲解如何**主动阻止**编译器生成你不想要的函数。

## 问题：想禁止拷贝，但两头堵

```cpp
class HomeForSale { ... };

HomeForSale h1;
HomeForSale h2;
HomeForSale h3(h1);  // 不应该编译通过！
h1 = h2;             // 不应该编译通过！
```

- **不声明**拷贝操作？→ 编译器自动生成，类照样可拷贝
- **声明了**拷贝操作？→ 那类依然支持拷贝

## 解法一：声明为 private + 不提供定义

**关键洞察**：编译器自动生成的函数都是 `public` 的。

1. **自己声明**拷贝操作 → 阻止编译器自动生成
2. **放到 `private`** → 阻止外部代码调用
3. **只声明不定义** → member/friend 函数误调时在链接期报错

```cpp
class HomeForSale {
public:
    // ...
private:
    // ...
    HomeForSale(const HomeForSale&);             // 只声明，不定义
    HomeForSale& operator=(const HomeForSale&);  // 只声明，不定义
};
```

参数名省略不写是惯例——函数永远不会被实现。

**效果**：
- 外部代码拷贝 → **编译期**错误（private 不可访问）
- member/friend 函数拷贝 → **链接期**错误（函数无定义）

此技巧在标准库中广泛使用，如 `ios_base`、`basic_ios`、`sentry` 等。

## 解法二：Uncopyable 基类（将链接期错误提前到编译期）

定义一个专门用于禁止拷贝的基类：

```cpp
class Uncopyable {
protected:
    Uncopyable() {}                          // 允许派生类构造和析构
    ~Uncopyable() {}
private:
    Uncopyable(const Uncopyable&);           // 禁止拷贝
    Uncopyable& operator=(const Uncopyable&);
};
```

派生类通过 private 继承即可：

```cpp
class HomeForSale : private Uncopyable {
    // 不再需要自己声明拷贝操作
};
```

**原理**：编译器为派生类生成拷贝操作时，需要调用基类的对应函数，但基类的拷贝操作是 private 的 → 编译期报错。

**细节**：
- 使用 **private 继承**（不是 public）
- `Uncopyable` 的 destructor 不需要是 virtual
- 无数据成员，可受益于空基类优化（EBO），但可能因多重继承而失效
- Boost 库提供了现成的 `boost::noncopyable`

## Things to Remember

- 要禁止编译器自动生成的功能，将对应的成员函数**声明为 private 且不提供实现**。使用像 `Uncopyable` 这样的基类是一种实现方式。

---

## 补充：现代 C++ 的 `= delete`（C++11）

C++11 引入的 `= delete` 让上述所有技巧都成为了历史。

```cpp
class HomeForSale {
public:
    HomeForSale(const HomeForSale&) = delete;
    HomeForSale& operator=(const HomeForSale&) = delete;
};
```

**优势**：
- 语义明确，一眼就知道是故意禁止
- **任何人**调用都在**编译期**直接报错（不用等到链接期）
- 推荐放在 `public` 区域——这样报错信息更清晰（不会先报 "access private" 再报其他错误）
- 不需要基类的 trick

### `= delete` 的更多用途

`= delete` 不仅能禁止拷贝，还能禁止**任何函数**，比如阻止隐式类型转换：

```cpp
class Widget {
public:
    void process(int x);
    void process(double x) = delete;   // 禁止传 double
    void process(bool x) = delete;     // 禁止传 bool
};

Widget w;
w.process(42);     // ✅
w.process(3.14);   // ❌ 编译错误
w.process(true);   // ❌ 编译错误
```

还能禁止特定的模板实例化：

```cpp
template<typename T>
void processPointer(T* ptr) { ... }

template<>
void processPointer<void>(void*) = delete;    // 禁止 void*

template<>
void processPointer<char>(char*) = delete;    // 禁止 char*（避免误传 C 风格字符串）
```

### 对比总结

| 方面 | C++03 做法 | C++11 `= delete` |
|------|-----------|-------------------|
| 外部调用 | 编译期报错（private） | 编译期报错 |
| member/friend 调用 | 链接期报错 | 编译期报错 |
| 代码量 | 较多（声明 + 可能需要基类） | 一行搞定 |
| 可读性 | 需要了解惯用法 | 意图一目了然 |
| 适用范围 | 仅限特殊成员函数 | 任何函数 |
