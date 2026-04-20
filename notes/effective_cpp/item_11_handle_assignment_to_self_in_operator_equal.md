# Item 11: 在 `operator=` 中处理自我赋值

## 核心思想

对象可能被赋值给自己（self-assignment），例如 `w = w`。这看起来很蠢，但在别名（aliasing）的情况下很容易发生。如果 `operator=` 不处理自我赋值，可能导致**使用已删除的资源**。更重要的是，追求**异常安全**通常能自动获得自我赋值安全。

## 自我赋值什么时候会发生？

显式的 `w = w` 很少见，但别名场景很常见：

```cpp
a[i] = a[j];     // 如果 i == j，就是自我赋值
*px = *py;        // 如果 px 和 py 指向同一对象，就是自我赋值
```

甚至不需要是同一类型——继承体系中，基类引用/指针可以指向派生类对象：

```cpp
class Base { ... };
class Derived : public Base { ... };

void doSomething(const Base& rb, Derived* pd);
// rb 和 *pd 可能是同一个对象
```

## 问题演示：不安全的 `operator=`

```cpp
class Bitmap { ... };

class Widget {
private:
    Bitmap* pb;  // 指向堆上分配的对象
};

// ❌ 不安全的实现
Widget& Widget::operator=(const Widget& rhs)
{
    delete pb;                  // 删除当前 bitmap
    pb = new Bitmap(*rhs.pb);  // 使用 rhs 的 bitmap 的副本
    return *this;
}
```

**问题**：如果 `*this` 和 `rhs` 是同一个对象，`delete pb` 同时也删除了 `rhs.pb`。接下来 `new Bitmap(*rhs.pb)` 就在访问一个已删除的对象——未定义行为！

## 三种解决方案

### 方案一：身份检测（identity test）

```cpp
Widget& Widget::operator=(const Widget& rhs)
{
    if (this == &rhs) return *this;  // 自我赋值检测

    delete pb;
    pb = new Bitmap(*rhs.pb);
    return *this;
}
```

**优点**：简单直观，自我赋值时直接返回。

**缺点**：仍然不是**异常安全**的！如果 `new Bitmap` 抛出异常，`pb` 已经被 delete 了，Widget 会持有一个悬空指针（dangling pointer）。

### 方案二：精心安排语句顺序（推荐）

```cpp
Widget& Widget::operator=(const Widget& rhs)
{
    Bitmap* pOrig = pb;             // 记住原来的 pb
    pb = new Bitmap(*rhs.pb);       // 让 pb 指向 rhs.pb 的副本
    delete pOrig;                   // 删除原来的 bitmap
    return *this;
}
```

**关键技巧**：先 `new` 再 `delete`。

- **异常安全**：如果 `new Bitmap` 抛异常，`pb` 没有被修改，Widget 保持原样。
- **自我赋值安全**：即使 `*this == rhs`，我们先复制了 bitmap，再删除旧的，不会出问题。
- 不需要身份检测（虽然可以加上作为优化，但通常不值得，因为自我赋值很少发生，而 `if` 分支会影响指令预取、缓存和流水线）。

### 方案三：Copy and Swap 惯用法

```cpp
class Widget {
    void swap(Widget& rhs);  // 交换 *this 和 rhs 的数据
};

// 写法 A：显式拷贝
Widget& Widget::operator=(const Widget& rhs)
{
    Widget temp(rhs);   // 拷贝 rhs
    swap(temp);          // 交换 *this 与副本
    return *this;
}

// 写法 B：利用按值传参自动拷贝
Widget& Widget::operator=(Widget rhs)   // 注意：按值传递！
{
    swap(rhs);           // 交换 *this 与副本
    return *this;
}
```

**原理**：

1. 先创建 `rhs` 的一个副本
2. 将 `*this` 的数据与副本交换
3. 副本（现在持有旧数据）在析构时自动释放旧资源

**写法 B 的巧妙之处**：把拷贝操作从函数体移到了参数构造阶段，编译器有时能生成更高效的代码。但 Scott Meyers 本人认为这种写法牺牲了清晰性。

> **补充**：Copy and Swap 是 C++ 中非常经典的惯用法，在 Item 29（异常安全代码）中有更深入的讨论。它的核心优势是将资源管理的复杂性封装在 swap 和析构函数中，`operator=` 本身变得极其简单。在现代 C++（C++11 以后），如果类正确实现了移动语义，按值传参的写法还能自动处理移动赋值的场景。

## 三种方案对比

| 方案 | 自我赋值安全 | 异常安全 | 效率 | 复杂度 |
|------|:---:|:---:|------|------|
| 身份检测 | ✅ | ❌ | 自我赋值时最快 | 低 |
| 语句重排 | ✅ | ✅ | 总有一次拷贝 | 中 |
| Copy and Swap | ✅ | ✅ | 总有一次拷贝 | 低（需要实现 swap） |

> **补充**：实际项目中，如果你使用了智能指针（如 `std::shared_ptr`、`std::unique_ptr`）来管理资源（遵循 Item 13 和 14 的建议），编译器自动生成的 `operator=` 通常就是自我赋值安全的，你不需要手动处理这些问题。手动管理原始指针时才需要特别注意。

## Things to Remember

- **确保 `operator=` 在对象自我赋值时行为正确**。技术手段包括：比较源和目标对象的地址、精心安排语句顺序、以及 copy-and-swap。
- **确保任何操作多个对象的函数，在多个对象实际上是同一个对象时也能正确运行**。
