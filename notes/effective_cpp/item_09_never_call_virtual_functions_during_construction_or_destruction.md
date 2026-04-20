# Item 9: Never call virtual functions during construction or destruction

## 核心思想

在构造和析构期间，virtual 函数不会按多态方式工作——它们不会下降到派生类。在基类构造函数中调用 virtual 函数，调用的永远是基类版本，而不是派生类版本。这是 C++ 与 Java/C# 的一个关键差异。

## 问题：构造期间调 virtual 函数

### 错误示例

```cpp
class Transaction {
public:
    Transaction() {
        // ...
        logTransaction();  // 💥 调用 virtual 函数！
    }
    virtual void logTransaction() const = 0;
};

class BuyTransaction : public Transaction {
public:
    virtual void logTransaction() const {
        // 记录"买入"日志
    }
};

BuyTransaction b;
// Transaction() 执行时调用 logTransaction()
// → 调用的是 Transaction::logTransaction（基类版本），不是 BuyTransaction 的！
```

### 为什么？两层原因

**第一层——安全性**：基类构造函数执行时，派生类的数据成员还没有初始化。如果 virtual 函数下降到派生类，派生类的实现几乎一定会访问自己的数据成员——但它们还是未初始化的垃圾值 → 未定义行为。

**第二层——更根本的**：在基类构造期间，对象的**类型就是基类类型**。不仅 virtual 函数如此，`typeid` 和 `dynamic_cast` 也把对象视为基类类型。对象直到派生类构造函数开始执行，才"变成"派生类对象。

### 析构期间同理

```
构造：Base 构造 → [对象是 Base 类型] → Derived 构造 → [对象是 Derived 类型]
析构：Derived 析构 → [对象变回 Base 类型] → Base 析构 → [对象销毁]
```

一旦派生类 destructor 执行完，派生类数据成员变成未定义值，C++ 将对象视为基类对象。

### 两种后果

| `logTransaction` 类型 | 后果 |
|----------------------|------|
| **pure virtual** | 运行时检测到调用纯虚函数 → 程序终止 |
| **普通 virtual**（有基类实现） | 静悄悄地调了基类版本 → 错误行为，没有任何警告，极难调试 |

## 更隐蔽的形式：间接调用

把 virtual 函数调用藏在非 virtual 的辅助函数里，更难发现：

```cpp
class Transaction {
public:
    Transaction() { init(); }           // 看起来无辜
    virtual void logTransaction() const = 0;

private:
    void init() {
        // ...
        logTransaction();               // 💥 间接调用了 virtual！
    }
};
```

编译器可能不会对此发出警告。如果 `logTransaction` 是普通 virtual（非纯虚），程序能编译、链接、运行——但行为是错的。

**原则**：构造函数和析构函数调用的所有函数链，都不应该包含 virtual 函数调用。

## 正确方案：把信息"向上传"而不是"向下调"

```cpp
class Transaction {
public:
    explicit Transaction(const std::string& logInfo) {
        // ...
        logTransaction(logInfo);  // ✅ 非 virtual 调用
    }
    void logTransaction(const std::string& logInfo) const;  // 非 virtual
};

class BuyTransaction : public Transaction {
public:
    BuyTransaction(parameters)
        : Transaction(createLogString(parameters))  // 向上传递信息
    { }

private:
    static std::string createLogString(parameters) {
        return "Buy: ...";
    }
};
```

### 设计要点

1. **`logTransaction` 改为非 virtual**——接收日志信息作为参数，不依赖多态
2. **派生类通过初始化列表**把信息传给基类构造函数
3. **`createLogString` 是 `static` 函数**——不能访问任何非静态成员变量，从编译器层面杜绝访问未初始化数据的风险

### 思路对比

| | 错误做法 | 正确做法 |
|---|---|---|
| 信息流方向 | 基类**向下问**派生类 | 派生类**向上传**给基类 |
| 机制 | virtual 函数调用 | 构造函数参数传递 |
| 问题 | 派生类还没构造好，问不到 | 信息在构造前就准备好了 |

## Things to Remember

- **不要在构造或析构期间调用 virtual 函数**，因为这样的调用永远不会下降到比当前执行的构造/析构函数所在的类更深的派生类。

---

## 补充：与 Java/C# 的对比

在 Java 中，构造函数里调用的虚方法**会**下降到派生类——即使派生类成员还没初始化：

```java
class Base {
    Base() {
        doSomething();  // 调用的是 Derived.doSomething()！
    }
    void doSomething() { }
}

class Derived extends Base {
    private int value = 42;

    @Override
    void doSomething() {
        System.out.println(value);  // 输出 0，不是 42！成员还没初始化
    }
}
```

C++ 选择了更安全的设计：在基类构造期间，对象类型就是基类类型，virtual 函数不下降。Java 的设计允许下降，但代价是可能访问到未初始化的成员——这是 Java 中常见的 bug 来源。
