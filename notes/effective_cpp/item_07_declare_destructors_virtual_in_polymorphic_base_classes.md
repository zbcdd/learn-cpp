# Item 7: Declare destructors virtual in polymorphic base classes

## 核心思想

通过基类指针删除派生类对象时，如果基类的 destructor 不是 virtual 的，行为是**未定义的**（undefined behavior）。多态基类必须声明 virtual destructor；反过来，不打算作为多态基类的类，不应该声明 virtual destructor。

## 问题：non-virtual destructor + 基类指针 delete = UB

### 典型场景

```cpp
class TimeKeeper {
public:
    TimeKeeper();
    ~TimeKeeper();        // ⚠️ non-virtual!
    // ...
};

class AtomicClock : public TimeKeeper { ... };
class WaterClock  : public TimeKeeper { ... };
class WristWatch  : public TimeKeeper { ... };

// 工厂函数
TimeKeeper* getTimeKeeper();

TimeKeeper* ptk = getTimeKeeper();  // 返回派生类对象
// ...
delete ptk;  // 💥 UB！通过基类指针删除，基类 destructor 非 virtual
```

### 为什么是 UB？

因为 non-virtual destructor 是**静态绑定**的——编译器只看指针类型 `TimeKeeper*`，直接调用 `~TimeKeeper()`。派生类的 destructor 根本不会被调用，导致：

- 派生类部分不被析构 → **资源泄漏**
- 对象处于"半销毁"状态 → **数据结构损坏**
- `operator delete` 收到的 size 可能不对 → **堆结构损坏**
- 多重继承时指针值可能不对 → 更严重的问题

## 解决方案：virtual destructor

```cpp
class TimeKeeper {
public:
    TimeKeeper();
    virtual ~TimeKeeper();  // ✅ virtual
    // ...
};

TimeKeeper* ptk = getTimeKeeper();
// ...
delete ptk;  // ✅ 动态绑定 → 先调 ~AtomicClock()，再调 ~TimeKeeper()
```

### 经验法则

> **含有任何 virtual 函数的类，几乎一定应该有 virtual destructor。**

有 virtual 函数 → 设计为多态使用 → 会通过基类指针操作 → 可能通过基类指针 delete → 需要 virtual destructor。

## 反面：不该加 virtual destructor 的情况

### Virtual 的代价：vptr

virtual 函数的实现需要每个对象额外携带一个 **vptr（virtual table pointer）**：

```
Point 对象（无 virtual）         Point 对象（有 virtual）
┌─────────┐                    ┌──────────┐
│  x (4B) │                    │  vptr(8B)│  ← 多了这个
│  y (4B) │                    │  x  (4B) │
└─────────┘                    │  y  (4B) │
  共 8 字节                     └──────────┘
                                 共 16 字节（64位架构）
```

对于 `Point` 这样的小对象：
- 体积增加 50%–100%
- 无法放进 64 位寄存器
- 与 C 语言结构体不再兼容

> **一律加 virtual 和一律不加 virtual 都是错的。**

### 不要继承没有 virtual destructor 的类

`std::string`、所有 STL 容器（`vector`、`list`、`set`、`unordered_map` 等）都没有 virtual destructor，**不应该被继承**：

```cpp
class SpecialString : public std::string { ... };  // ❌ 坏主意！

SpecialString* pss = new SpecialString("Impending Doom");
std::string* ps = pss;
delete ps;  // 💥 UB！
```

## 三类基类的区分

| 类型 | 例子 | 需要 virtual destructor？ |
|------|------|--------------------------|
| **多态基类**：通过基类接口操作派生类对象 | `TimeKeeper` | ✅ 需要 |
| **非多态基类**：是基类，但不通过基类指针操作 | `Uncopyable`（Item 6）、`input_iterator_tag` | ❌ 不需要 |
| **根本不是基类** | `std::string`、STL 容器 | ❌ 不需要 |

## 技巧：Pure virtual destructor

想让类成为抽象类但没有自然的 pure virtual 函数时，可以用 pure virtual destructor：

```cpp
class AWOV {  // "Abstract Without Virtuals"
public:
    virtual ~AWOV() = 0;  // pure virtual → 抽象类
};

AWOV::~AWOV() {}  // ⚠️ 必须提供定义！
```

**为什么必须有定义？** 析构顺序是派生类 → 基类，编译器会自动在派生类的 destructor 末尾插入对基类 destructor 的调用。如果基类 destructor 没有定义，链接器报错。

这是 pure virtual destructor 与普通 pure virtual 函数的区别：普通 pure virtual 函数派生类 override 后，基类版本不会被自动调用，所以可以没有定义；但 destructor 的析构链是强制的，基类版本**一定会被调用**。

## Things to Remember

- **多态基类应该声明 virtual destructor。** 如果一个类含有任何 virtual 函数，它就应该有 virtual destructor。
- **不是设计为基类的类，或不是设计为多态使用的类，不应该声明 virtual destructor。**

---

## 补充：现代 C++ 相关

### `final` 关键字（C++11）

Meyers 在书中感叹 C++ 没有类似 Java `final` / C# `sealed` 的机制来阻止继承。C++11 已经解决了这个问题：

```cpp
class String final {
    // 禁止任何类继承 String
};

class Derived : public String {};  // ❌ 编译错误
```

`final` 也可以用在虚函数上，阻止进一步 override：

```cpp
class Base {
public:
    virtual void foo();
};

class Middle : public Base {
public:
    void foo() final;  // 到此为止，不允许再 override
};

class Bottom : public Middle {
public:
    void foo() override;  // ❌ 编译错误
};
```

### `= 0` vs `= delete` 辨析

这两个语法容易混淆，但语义完全不同：

| | `= 0`（纯虚） | `= delete`（删除） |
|---|---|---|
| 含义 | 派生类**必须 override** | 函数**不存在**，禁止调用 |
| 函数是否存在 | 存在，可以提供定义 | 彻底删除 |
| 适用范围 | 只能用于 virtual 成员函数 | 任何函数 |
| 对类的影响 | 类变成抽象类 | 不影响 |

对 destructor 用 `= delete` 会让对象无法正常销毁，基本没有实用价值：

```cpp
class Bad {
public:
    ~Bad() = delete;
};
// Bad b;        // ❌ 无法创建栈对象
// delete new Bad;  // ❌ 无法 delete
```

### `override` 关键字（C++11）

虽然本条款没有直接提到，但现代 C++ 中重写虚函数时应始终加 `override`，帮助编译器检查你是否真的覆盖了基类的函数：

```cpp
class Derived : public TimeKeeper {
public:
    ~Derived() override;  // 明确表示覆盖基类虚析构函数
};
```
