# Item 4: Make sure that objects are initialized before they're used

## 核心思想

C++ 中对象的初始化行为并不一致——内置类型在某些上下文中会被自动初始化（如静态存储期），在另一些上下文中则不会（如自动存储期）。读取未初始化的值是**未定义行为**。为了避免这类问题，应当养成三个习惯：手动初始化内置类型、使用成员初始化列表、用局部静态对象替代非局部静态对象。

## 1. 手动初始化内置类型

C++ 不保证所有内置类型都会被自动初始化。经验法则：如果处于"C++ 中 C 的部分"（参见 Item 1），且初始化会产生运行时开销，就不保证初始化。例如 C 风格数组不保证内容初始化，但 `vector` 保证。

最佳实践：**永远在使用前手动初始化**。

```cpp
int x = 0;                              // 手动初始化 int
const char * text = "A C-style string"; // 手动初始化指针
double d;
std::cin >> d;                          // 通过输入流"初始化"
```

## 2. 使用成员初始化列表（Member Initialization List）

### 赋值 ≠ 初始化

在构造函数**体内**对成员使用 `=` 是**赋值**，不是初始化。因为 C++ 规定数据成员在构造函数体执行之前就已经被初始化（用户自定义类型调用默认构造函数，内置类型则未初始化）。

```cpp
// 反面例子：这些都是赋值，不是初始化
ABEntry::ABEntry(const std::string& name, const std::string& address,
                 const std::list<PhoneNumber>& phones)
{
    theName = name;           // 赋值
    theAddress = address;     // 赋值
    thePhones = phones;       // 赋值
    numTimesConsulted = 0;    // 赋值
}
```

### 正确做法：使用初始化列表

```cpp
ABEntry::ABEntry(const std::string& name, const std::string& address,
                 const std::list<PhoneNumber>& phones)
: theName(name),
  theAddress(address),        // 真正的初始化：直接调用拷贝构造函数
  thePhones(phones),
  numTimesConsulted(0)
{}                            // 构造函数体为空
```

### 为什么更高效？

- 赋值版本：默认构造 → 拷贝赋值（两步，默认构造的工作浪费了）
- 初始化列表版本：直接拷贝构造（一步到位）

对于内置类型（如 `int`），两者性能无差别，但为了**一致性**，建议也用初始化列表。

### 必须使用初始化列表的情况

- **`const` 成员**：只能初始化，不能赋值
- **引用成员**：只能初始化，不能赋值

```cpp
class Foo {
    const int x;
    int& ref;
public:
    Foo(int val, int& r) : x(val), ref(r) {}  // 必须用初始化列表
};
```

### 初始化顺序

成员的初始化顺序**始终按照类中声明的顺序**，与初始化列表的书写顺序无关。基类先于派生类初始化。

**建议：初始化列表的书写顺序应与成员声明顺序一致**，避免产生依赖未初始化成员的隐晦 bug。

### 多构造函数时的策略

当类有多个构造函数且成员很多时，每个构造函数都写一长串初始化列表会造成重复。Meyers 建议可以把对内置类型的"伪初始化"（赋值）提取到一个私有共用函数中。但总体而言，真正的初始化列表优于赋值。

> **现代 C++ 补充（C++11）**：类内成员初始化器和委托构造函数很好地解决了这个问题：
> ```cpp
> class ABEntry {
>     std::string theName;
>     int numTimesConsulted = 0;     // 类内默认值，作为兜底
> public:
>     ABEntry() : ABEntry("", 0) {}  // 委托构造函数
>     ABEntry(const std::string& name, int times)
>     : theName(name), numTimesConsulted(times) {}
> };
> ```
> 注意：如果同一个成员既有类内默认值，又出现在构造函数初始化列表中，**以初始化列表为准**，类内默认值被忽略。

## 3. 跨翻译单元的非局部静态对象初始化顺序问题

### 概念定义

- **静态对象（static object）**：从构造到程序结束一直存在的对象，包括全局对象、命名空间作用域对象、类/函数/文件作用域中声明为 `static` 的对象
- **局部静态对象（local static object）**：函数内的 `static` 对象
- **非局部静态对象（non-local static object）**：除局部静态以外的所有静态对象
- **翻译单元（translation unit）**：一个 `.cpp` 源文件加上它所有 `#include` 的头文件

### 问题

**不同翻译单元中非局部静态对象的初始化顺序是未定义的**。如果一个翻译单元中的非局部静态对象在初始化时依赖另一个翻译单元中的非局部静态对象，被依赖的对象可能尚未初始化。

```cpp
// 文件系统库 —— FileSystem.cpp
class FileSystem { ... };
extern FileSystem tfs;              // 全局对象

// 客户端代码 —— Directory.cpp
class Directory { ... };
Directory::Directory(params) {
    std::size_t disks = tfs.numDisks();  // tfs 可能还没初始化！
}
Directory tempDir(params);          // 全局对象
```

这就是经典的 **"static initialization order fiasco"（静态初始化顺序灾难）**。

### 解决方案：用局部静态对象替代非局部静态对象

将全局对象搬进函数内部，返回引用：

```cpp
FileSystem& tfs()
{
    static FileSystem fs;   // 局部静态对象，首次调用时初始化
    return fs;
}

Directory::Directory(params) {
    std::size_t disks = tfs().numDisks();  // 保证 fs 已初始化
}

Directory& tempDir()
{
    static Directory td(params);
    return td;
}
```

**原理**：C++ 保证局部静态对象在函数首次被调用时初始化。通过函数调用访问对象，就能保证使用时对象已经初始化。

**额外好处**：惰性初始化——如果从未调用该函数，对象就不会被创建，节省开销。

**注意**：如果对象 A 和 B 之间存在循环依赖（A 依赖 B 初始化，B 也依赖 A 初始化），这个方案也无法解决。应避免这种设计。

> **现代 C++ 补充（C++11）**：C++11 标准保证了**局部静态变量的初始化是线程安全的**（"magic statics"）。编译器会自动插入同步机制，保证多线程并发调用时也只初始化一次。因此书中提到的多线程隐患在 C++11 之后已不存在。

## Things to Remember

- **手动初始化内置类型的对象**，因为 C++ 只在某些情况下会自动初始化它们。
- **在构造函数中，优先使用成员初始化列表**，而非在构造函数体内赋值。初始化列表中成员的顺序应与类中声明的顺序一致。
- **用局部静态对象替换非局部静态对象**，以避免跨翻译单元的初始化顺序问题。
