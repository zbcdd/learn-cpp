# Item 5: Know what functions C++ silently writes and calls

## 核心思想

如果你不自己声明，编译器会自动为你的类生成一组特殊成员函数。理解编译器在什么情况下生成、生成的函数做什么、以及什么时候会拒绝生成，是正确设计 C++ 类的基础。

## 编译器自动生成的函数

当你写一个空类时：

```cpp
class Empty {};
```

编译器实际上会为你生成以下四个函数（均为 `public inline`）：

```cpp
class Empty {
public:
    Empty() { ... }                            // default constructor
    Empty(const Empty& rhs) { ... }            // copy constructor
    ~Empty() { ... }                           // destructor
    Empty& operator=(const Empty& rhs) { ... } // copy assignment operator
};
```

**生成条件：**
- 这些函数只在**被需要时**才生成（但触发门槛很低）
- 如果你声明了**任何一个构造函数**，编译器就**不会**再生成 default constructor
- copy constructor 和 copy assignment operator 只在你没声明时才生成

## 生成的函数做什么？

### Default constructor 和 destructor

- 充当"幕后代码"的容器——编译器在其中插入对**基类**和**非静态数据成员**的构造/析构调用
- 自动生成的 destructor 是 **non-virtual** 的，除非继承自声明了 virtual destructor 的基类

### Copy constructor 和 copy assignment operator

**逐成员拷贝**（memberwise copy）：

```cpp
template<typename T>
class NamedObject {
public:
    NamedObject(const char* name, const T& value);
    NamedObject(const std::string& name, const T& value);
private:
    std::string nameValue;
    T objectValue;
};

NamedObject<int> no1("Smallest Prime Number", 2);
NamedObject<int> no2(no1);  // 编译器生成的 copy constructor
```

- `nameValue`（`std::string`）→ 调用 `std::string` 的 copy constructor（深拷贝）
- `objectValue`（`int`）→ 逐位拷贝

注意：因为已声明了带参构造函数，编译器**不会**生成 default constructor。

## 编译器拒绝生成 `operator=` 的情况

编译器生成 copy assignment operator 的前提是：生成的代码必须**合法**且**合理**。以下三种情况编译器会拒绝生成：

### 1. 类含有引用成员

```cpp
template<typename T>
class NamedObject {
public:
    NamedObject(std::string& name, const T& value);
private:
    std::string& nameValue;   // 引用成员
    const T objectValue;       // const 成员
};

std::string newDog("Persephone");
std::string oldDog("Satch");
NamedObject<int> p(newDog, 2);
NamedObject<int> s(oldDog, 36);
p = s;  // ❌ 编译错误！
```

C++ 中引用一旦绑定就**不能重新绑定**，编译器面临两难：
- 让引用指向新对象？→ C++ 不允许
- 修改引用指向的原对象？→ 会影响不相关的代码，不合理

### 2. 类含有 const 成员

const 对象不能被修改，赋值操作无意义。

### 3. 基类的 `operator=` 为 private

派生类的赋值需要调用基类的赋值操作，但无权访问 private 成员函数。

## Things to Remember

- **编译器可能会隐式生成**类的 default constructor、copy constructor、copy assignment operator 和 destructor

---

## 补充：现代 C++ 扩展

### C++11 的移动语义

C++11 起，编译器还可能自动生成两个额外的特殊成员函数：

- **move constructor** `T(T&&)`
- **move assignment operator** `T& operator=(T&&)`

但生成条件更严格：只要你声明了 copy 操作、move 操作或 destructor 中的任何一个，编译器就可能不再自动生成 move 操作。

### Rule of Three / Five / Zero

| 时代 | 法则 | 涉及的函数 |
|------|------|-----------|
| C++03 | **Rule of Three** | destructor、copy constructor、copy assignment operator |
| C++11+ | **Rule of Five** | 上面三个 + move constructor、move assignment operator |

**Rule of Five**：如果你的类需要自定义五个特殊成员函数中的任何一个，通常应该把五个都显式定义（或标记为 `= default` / `= delete`）。因为需要自定义其中一个，通常意味着类管理了某种资源，其余几个的默认行为大概率也不正确。

**Rule of Zero**：如果类不直接管理资源（而是通过 `std::unique_ptr`、`std::string` 等 RAII 类型间接管理），那就一个都不要写，全部让编译器自动生成。

**决策流程：**

```
需要管理资源吗？
  ├─ 否 → Rule of Zero，什么都不写
  └─ 是 → 能用 RAII 包装类（smart pointer 等）吗？
            ├─ 能 → 用它们，回到 Rule of Zero
            └─ 不能 → Rule of Five，五个全写
```

在实际项目中，绝大多数类都应该是 Rule of Zero。只有最底层直接接触裸资源的"包装类"才需要 Rule of Five——而这些包装类，标准库已经帮你写好了（`unique_ptr`、`shared_ptr`、`vector`、`string` 等）。

### 智能指针的五个特殊函数

#### `std::unique_ptr`（独占语义）

| 函数 | 行为 |
|------|------|
| destructor | 调用 deleter 释放资源 |
| copy constructor | `= delete`，禁止拷贝 |
| copy assignment | `= delete`，禁止拷贝 |
| move constructor | 转移所有权，源指针置空 |
| move assignment | 释放旧资源，转移所有权，源指针置空 |

```cpp
auto p1 = std::make_unique<int>(42);
// auto p2 = p1;              // ❌ 编译错误
auto p2 = std::move(p1);     // ✅ p1 变 nullptr
```

包含 `unique_ptr` 成员的类：拷贝被隐式 delete，移动自动生成——这正是 Rule of Zero 的体现。

#### `std::shared_ptr`（共享语义）

| 函数 | 行为 |
|------|------|
| destructor | 引用计数 -1，减到 0 时释放资源 |
| copy constructor | 引用计数 +1，共享所有权 |
| copy assignment | 旧资源引用计数 -1，新资源 +1 |
| move constructor | 转移所有权，不改变引用计数（更高效） |
| move assignment | 旧资源引用计数 -1，转移新资源所有权 |

```cpp
auto p1 = std::make_shared<int>(42);  // 引用计数 = 1
auto p2 = p1;                         // 拷贝，引用计数 = 2
auto p3 = std::move(p1);              // 移动，引用计数仍 = 2，p1 变 nullptr
```

**选择原则**：大多数情况优先用 `unique_ptr`，语义更清晰、开销更小（没有引用计数的原子操作）。只有确实需要共享所有权时才用 `shared_ptr`。
