# Item 10: Have assignment operators return a reference to `*this`

## 核心思想

赋值运算符（包括 `=`、`+=`、`-=` 等所有赋值类运算符）应当返回一个指向 `*this` 的引用，以支持链式赋值，并与内置类型及标准库类型的行为保持一致。

## 详细说明

### 链式赋值与右结合性

C++ 的赋值操作支持链式书写，并且是**右结合**的：

```cpp
int x, y, z;
x = y = z = 15;        // 链式赋值

// 等价于：
x = (y = (z = 15));    // 右结合：先赋 z，再赋 y，最后赋 x
```

要让这种链式赋值成立，每次赋值操作必须返回左侧对象的引用，作为下一次赋值的右值。

### 自定义类的赋值运算符写法

对自定义类型，遵循同样的惯例——返回 `*this` 的引用：

```cpp
class Widget {
public:
    Widget& operator=(const Widget& rhs)
    {
        // ... 执行赋值逻辑
        return *this;   // 返回左侧对象的引用
    }
};
```

### 适用于所有赋值类运算符

这个惯例不仅限于 copy assignment operator，还包括复合赋值运算符以及参数类型非常规的赋值运算符：

```cpp
class Widget {
public:
    Widget& operator+=(const Widget& rhs)   // +=, -=, *= 等同理
    {
        // ...
        return *this;
    }

    Widget& operator=(int rhs)   // 即便参数类型不常规，也应如此
    {
        // ...
        return *this;
    }
};
```

### 惯例而非强制

不遵循这个惯例的代码能通过编译，但所有内置类型和标准库类型（`string`、`vector`、`complex`、`shared_ptr` 等）都遵循了这一惯例。除非有充分理由，否则应当遵守。

## 补充：现代 C++ 中的情况

> 以下为补充内容，非原书内容。

在 C++11 引入**移动语义**后，move assignment operator 同样应遵循此惯例：

```cpp
class Widget {
public:
    Widget& operator=(Widget&& rhs) noexcept   // move assignment 也返回 *this 引用
    {
        // ... 执行移动逻辑
        return *this;
    }
};
```

## Things to Remember

- **让赋值运算符返回一个指向 `*this` 的引用。** 这适用于所有赋值类运算符（`=`、`+=`、`-=`、`*=` 等），使你的类型与内置类型行为一致，支持链式赋值。
