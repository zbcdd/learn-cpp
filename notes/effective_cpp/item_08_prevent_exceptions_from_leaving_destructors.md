# Item 8: Prevent exceptions from leaving destructors

## 核心思想

Destructor 绝不应该让异常逃出去。如果 destructor 中需要执行可能失败的操作，应该在 destructor 内部捕获并处理异常；同时，应该给客户端提供一个非 destructor 的函数来执行该操作，让客户端有机会自行处理错误。

## 为什么 destructor 不能抛异常？

### 两个同时活跃的异常 → 灾难

```cpp
class Widget {
public:
    ~Widget() { ... }  // 假设可能抛异常
};

void doSomething() {
    std::vector<Widget> v;  // 假设有 10 个 Widget
    // ...
}  // 析构 v，需要析构所有 Widget
```

如果第一个 Widget 的 destructor 抛了异常，剩下 9 个还得析构（否则资源泄漏）。如果第二个也抛了异常 → **两个同时活跃的异常** → C++ 无法处理 → **程序终止或未定义行为**。

这个问题不限于容器和数组，**任何场景下** destructor 抛异常都可能导致程序终止或 UB。

## 实际场景：destructor 里的操作确实可能失败

```cpp
class DBConnection {
public:
    static DBConnection create();
    void close();  // 关闭连接，可能抛异常
};

class DBConn {
public:
    ~DBConn() {
        db.close();  // ⚠️ 如果 close() 抛异常，异常逃出 destructor
    }
private:
    DBConnection db;
};
```

## 三种解决方案

### 方案一：终止程序（abort）

```cpp
DBConn::~DBConn() {
    try { db.close(); }
    catch (...) {
        // 记录日志
        std::abort();
    }
}
```

**适用场景**：close 失败意味着程序无法继续正确运行。主动 abort 是一种可控的终止，比让异常逃出 destructor 导致 UB 要好。

### 方案二：吞掉异常（swallow）

```cpp
DBConn::~DBConn() {
    try { db.close(); }
    catch (...) {
        // 记录日志
        // 什么都不做
    }
}
```

**适用场景**：close 失败不致命，程序可以继续运行。吞掉异常通常是坏主意（压制了重要信息），但有时比程序崩溃或 UB 要好。

### 方案三（推荐）：双重保障——客户端主动 close + destructor 兜底

```cpp
class DBConn {
public:
    void close() {          // 提供给客户端的显式 close
        db.close();
        closed = true;
    }

    ~DBConn() {
        if (!closed) {      // 客户端没有主动 close → destructor 兜底
            try {
                db.close();
            }
            catch (...) {
                // 记录日志
                // terminate 或 swallow
            }
        }
    }

private:
    DBConnection db;
    bool closed;
};
```

**设计思路——两层防线**：

| 层级 | 谁负责 | 能处理异常吗？ |
|------|--------|---------------|
| **第一层**：客户端主动调用 `close()` | 客户端 | ✅ 可以 try-catch，自由处理 |
| **第二层**：destructor 兜底 | DBConn | ❌ 只能 terminate 或 swallow |

这不是把负担甩给客户端，而是**赋予客户端处理错误的权利**。因为 destructor 里无法让异常传播出去，所以唯一能让客户端处理错误的方式，就是在 destructor 之外提供一个可能抛异常的函数。

如果客户端选择不调 `close()`，依赖 destructor 兜底，那么 destructor 选择 swallow 或 terminate 时客户端没有资格抱怨——你有处理的机会，你选择了放弃。

## Things to Remember

- **Destructor 绝不应该抛出异常。** 如果 destructor 中调用的函数可能抛异常，destructor 应该捕获所有异常，然后吞掉它们或终止程序。
- **如果客户端需要对某个操作的异常做出反应，类应该提供一个普通的（非 destructor 的）函数来执行该操作。**

---

## 补充：现代 C++ 相关

### C++11 destructor 默认 `noexcept`

C++11 起，destructor 默认是 `noexcept` 的（即使你不写）。如果 destructor 中抛出了异常：

```cpp
class Foo {
public:
    ~Foo() {  // 隐式 noexcept
        throw std::runtime_error("boom");  // 违反 noexcept 承诺
    }
};
```

运行时会直接调用 `std::terminate()` → 程序终止。连 UB 的机会都没有。

`noexcept` 是**编译期声明 + 运行时保证**：告诉编译器"不会抛异常"，如果违反，`std::terminate()` 被调用，且**不做栈展开**（局部对象的 destructor 可能不会执行）。

除非显式声明 `noexcept(false)`（非常不推荐）：

```cpp
~Foo() noexcept(false) { ... }  // 允许抛异常，几乎不应该这么做
```

### 现代 C++ 的最佳实践

本条款的"双重保障"设计模式在现代 C++ 中依然完全适用：

```cpp
class DBConn {
public:
    void close() {  // 非 noexcept，客户端可以 catch
        db.close();
        closed = true;
    }

    ~DBConn() noexcept {  // C++11 默认 noexcept，写出来更清晰
        if (!closed) {
            try { db.close(); }
            catch (...) {
                // 记录日志，不让异常逃出
            }
        }
    }

private:
    DBConnection db;
    bool closed = false;  // C++11 类内初始化
};
```
