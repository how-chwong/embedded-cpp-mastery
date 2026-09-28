# 附录 A：C / C++ / C# 横向对比

> 本附录面向有 C# 背景的开发者，帮助快速建立 C/C++ 思维模型。

---

## 目录

1. [语言定位与生态](#1-语言定位与生态)
2. [基本语法对比](#2-基本语法对比)
3. [内存管理对比](#3-内存管理对比)
4. [面向对象对比](#4-面向对象对比)
5. [泛型/模板对比](#5-泛型模板对比)
6. [异常处理对比](#6-异常处理对比)
7. [并发编程对比](#7-并发编程对比)
8. [字符串处理对比](#8-字符串处理对比)
9. [I/O 与格式化对比](#9-io-与格式化对比)
10. [性能与资源占用对比](#10-性能与资源占用对比)
11. [常见陷阱：C# 开发者转 C/C++](#11-常见陷阱c-开发者转-cc)

---

## 1. 语言定位与生态

| 特性 | C | C++ | C# |
|------|---|-----|----|
| 范式 | 过程式 | 多范式 | 多范式（OOP 为主） |
| 运行时 | 无（裸机） | 无/极小 | .NET CLR（重量级） |
| GC | ❌ | ❌ | ✅ |
| 跨平台 | 全平台（含嵌入式） | 全平台（含嵌入式） | Windows/Linux/macOS（需 .NET） |
| 编译目标 | 本机 | 本机 | IL（JIT/AOT） |
| 嵌入式适用 | ✅ 首选 | ✅ 常用 | ⚠️ 受限（.NET nanoFramework） |
| 开发速度 | 慢 | 中 | 快 |
| 运行速度 | 极快 | 极快 | 快（略慢于 C/C++） |

---

## 2. 基本语法对比

### 2.1 变量声明

```csharp
// C#
int    count = 42;
double pi    = 3.14159;
var    name  = "Alice";   // 类型推导
bool   flag  = true;
```

```c
// C
int    count = 42;
double pi    = 3.14159;
// C 没有 var，C11 也没有
_Bool  flag  = 1;   // 或 #include <stdbool.h> 使用 bool
```

```cpp
// C++
int    count = 42;
double pi    = 3.14159;
auto   name  = std::string("Alice");  // auto 类型推导
bool   flag  = true;
```

### 2.2 控制流

```csharp
// C# foreach
foreach (var item in list) { Console.WriteLine(item); }

// C# switch 表达式（C# 8+）
string result = x switch {
    1 => "one",
    2 => "two",
    _ => "other"
};
```

```cpp
// C++ 范围 for
for (auto &item : vec) { std::cout << item; }

// C++ switch（传统）
switch (x) {
    case 1: /* ... */ break;
    default: /* ... */
}
```

### 2.3 函数

```csharp
// C# 方法（必须在类中）
public static int Add(int a, int b) => a + b;

// out 参数
bool TryParse(string s, out int result) { ... }
```

```cpp
// C++ 函数（可在类外）
int add(int a, int b) { return a + b; }

// 引用参数（类似 ref/out）
bool try_parse(const std::string &s, int &result) { ... }
```

---

## 3. 内存管理对比

```csharp
// C# - GC 自动管理
var obj = new MyClass();  // 无需 delete
// GC 在适当时机自动回收
// using 语句管理非托管资源
using (var stream = new FileStream(...)) { ... }
```

```cpp
// C++ - 手动管理（裸指针）❌ 不推荐
MyClass *p = new MyClass();
p->DoWork();
delete p;   // 必须手动释放，否则内存泄漏

// C++ - RAII 智能指针 ✅ 推荐
auto p = std::make_unique<MyClass>();
p->do_work();
// 自动释放，类似 C# using
```

```c
// C - malloc/free
MyStruct *p = malloc(sizeof(MyStruct));
init_my_struct(p);
use(p);
free(p);  // 必须手动释放
p = NULL; // 防止悬空指针
```

### 关键差异

| 方面 | C# | C++ | C |
|------|----|----|---|
| 分配 | `new` | `new` / `malloc` / 栈 | `malloc` / 栈 |
| 释放 | GC 自动 | `delete` / 智能指针 | `free` |
| 内存泄漏 | 几乎不可能 | 需要谨慎 | 非常常见 |
| 悬空指针 | 不存在 | 可能 | 可能 |
| GC 暂停 | 存在（ms~s） | 不存在 | 不存在 |

---

## 4. 面向对象对比

### 4.1 类定义

```csharp
// C#
public class Temperature {
    private double _celsius;

    public Temperature(double celsius) => _celsius = celsius;

    public double Celsius => _celsius;
    public double Fahrenheit => _celsius * 9 / 5 + 32;

    public static Temperature FromFahrenheit(double f)
        => new Temperature((f - 32) * 5 / 9);
}
```

```cpp
// C++
class Temperature {
public:
    explicit Temperature(double celsius) : celsius_(celsius) {}

    double celsius() const { return celsius_; }
    double fahrenheit() const { return celsius_ * 9.0 / 5.0 + 32.0; }

    static Temperature from_fahrenheit(double f) {
        return Temperature((f - 32.0) * 5.0 / 9.0);
    }

private:
    double celsius_;
};
```

### 4.2 继承与接口

```csharp
// C# 接口
public interface ISensor {
    bool Init();
    float Read();
}

public class TempSensor : ISensor {
    public bool Init() { ... }
    public float Read() { ... }
}
```

```cpp
// C++ 抽象类（没有 interface 关键字）
class ISensor {
public:
    virtual ~ISensor() = default;
    virtual bool  init() = 0;   // 纯虚函数 = C# 接口方法
    virtual float read() = 0;
};

class TempSensor : public ISensor {
public:
    bool  init() override { ... }
    float read() override { ... }
};
```

### 4.3 属性 vs 访问器

```csharp
// C# - 属性语法糖
public string Name { get; set; }
public int Age {
    get => _age;
    set { if (value >= 0) _age = value; }
}
```

```cpp
// C++ - 需要手写 getter/setter
std::string name() const { return name_; }
void set_name(std::string n) { name_ = std::move(n); }

int age() const { return age_; }
void set_age(int a) { if (a >= 0) age_ = a; }
```

---

## 5. 泛型/模板对比

```csharp
// C# 泛型（运行时类型擦除）
public class Stack<T> {
    private List<T> items = new();
    public void Push(T item) => items.Add(item);
    public T Pop() { var t = items[^1]; items.RemoveAt(items.Count-1); return t; }
}

// 泛型约束
public T Max<T>(T a, T b) where T : IComparable<T>
    => a.CompareTo(b) > 0 ? a : b;
```

```cpp
// C++ 模板（编译期代码生成，无运行时开销）
template<typename T>
class Stack {
public:
    void push(T item) { items_.push_back(std::move(item)); }
    T    pop()        { T t = items_.back(); items_.pop_back(); return t; }
private:
    std::vector<T> items_;
};

// 模板约束（C++20 Concepts）
template<std::totally_ordered T>
T max_val(T a, T b) { return a > b ? a : b; }
```

---

## 6. 异常处理对比

```csharp
// C# - 异常是主流错误处理方式
try {
    var result = DivideNumbers(10, 0);
} catch (DivideByZeroException e) {
    Console.WriteLine($"错误: {e.Message}");
} finally {
    // 始终执行（类似 RAII）
    Cleanup();
}

// C# 异常类型丰富，携带 StackTrace
```

```cpp
// C++ - 异常语法相同，但嵌入式常禁用
try {
    auto result = divide(10, 0);
} catch (const std::exception &e) {
    std::cerr << "错误: " << e.what() << "\n";
}

// 嵌入式中常用返回码替代
std::expected<int, ErrorCode> divide(int a, int b) {
    if (b == 0) return std::unexpected(ErrorCode::DivByZero);
    return a / b;
}
```

---

## 7. 并发编程对比

```csharp
// C# - async/await
public async Task<string> FetchDataAsync(string url) {
    var response = await httpClient.GetAsync(url);
    return await response.Content.ReadAsStringAsync();
}

// Task 并行
var tasks = new[] { Task.Run(() => Work1()), Task.Run(() => Work2()) };
await Task.WhenAll(tasks);

// lock
lock (lockObject) { sharedData++; }
```

```cpp
// C++ - 基于线程的并发
std::future<std::string> fetch_async(std::string url) {
    return std::async(std::launch::async, [url] {
        // HTTP 请求（同步）
        return std::string("result");
    });
}

// std::jthread（C++20，自动 join）
std::jthread t1(work1);
std::jthread t2(work2);

// mutex
std::mutex mtx;
std::lock_guard<std::mutex> lock(mtx);
shared_data++;
```

---

## 8. 字符串处理对比

```csharp
// C# - string 是引用类型，不可变，GC 管理
string s = "Hello, World!";
s.Length;                    // 13
s.ToUpper();                 // "HELLO, WORLD!"
s.Substring(7, 5);           // "World"
s.Contains("World");         // true
string.Format("{0}: {1}", "x", 42);  // "x: 42"
$"x: {42}";                  // C# 插值字符串
```

```cpp
// C++ - std::string 是值类型，堆分配
std::string s = "Hello, World!";
s.length();                  // 13
// 无 ToUpper，需手动
std::transform(s.begin(), s.end(), s.begin(), ::toupper);
s.substr(7, 5);              // "World"
s.find("World") != std::string::npos;  // true
std::to_string(42);          // "42"
std::format("x: {}", 42);   // C++20

// C++17 string_view（零拷贝，嵌入式推荐）
std::string_view sv = s;
sv.substr(7, 5);  // 不分配内存
```

```c
// C - 字符数组 + 函数
char s[32] = "Hello, World!";
strlen(s);                   // 13
toupper(s[0]);               // 需逐字符处理
strncpy(dest, s + 7, 5);     // 手动截取
strstr(s, "World") != NULL;  // 查找
snprintf(buf, sizeof(buf), "x: %d", 42);
```

---

## 9. I/O 与格式化对比

```csharp
Console.WriteLine("Hello");
Console.WriteLine($"x = {x:F2}");   // 格式化
int n = int.Parse(Console.ReadLine());
```

```cpp
std::cout << "Hello\n";
std::cout << std::fixed << std::setprecision(2) << x << "\n";
int n; std::cin >> n;

// C 风格（嵌入式常用）
printf("Hello\n");
printf("x = %.2f\n", x);
scanf("%d", &n);
```

---

## 10. 性能与资源占用对比

| 指标 | C | C++ | C# |
|------|---|-----|----|
| 启动时间 | μs 级 | μs 级 | 100ms~s 级（CLR 初始化） |
| 内存占用（Hello World） | ~1 KB | ~5 KB | ~30 MB（.NET 运行时） |
| GC 暂停 | N/A | N/A | 0.1ms~100ms |
| 函数调用开销 | 极低 | 低（虚函数略高） | 低（JIT 优化后） |
| 嵌入式最小 Flash | 4 KB | 8 KB | 1 MB+（nanoFramework） |
| 代码生成质量 | 最高 | 高 | 高（JIT） |

---

## 11. 常见陷阱：C# 开发者转 C/C++

### 陷阱 1：字符串不是引用类型

```cpp
// C# 中 string 是不可变引用类型
// C++ 中 std::string 是值类型（有拷贝开销）

void wrong_modify(std::string s) {  // ❌ 拷贝了整个字符串
    s += "!";
}

void correct_modify(std::string &s) {  // ✅ 引用传递
    s += "!";
}

void readonly_param(const std::string &s) {  // ✅ 常量引用（C# in 参数）
    std::cout << s;
}
```

### 陷阱 2：没有 null 检查的自动保护

```csharp
// C# 中访问 null 对象会抛出 NullReferenceException
object obj = null;
obj.ToString();  // 运行时异常，有保护
```

```cpp
// C++ 中解引用空指针是未定义行为（程序直接崩溃）
int *p = nullptr;
*p = 42;  // ❌ 直接段错误，无保护

// 必须手动检查
if (p != nullptr) *p = 42;
```

### 陷阱 3：整数溢出不会抛出异常

```csharp
// C# 中 checked 会检查溢出
checked { int x = int.MaxValue + 1; }  // 抛出 OverflowException
```

```cpp
// C++ 中有符号整数溢出是未定义行为（可能不检测）
int x = INT_MAX;
x++;  // UB！编译器可能优化掉溢出检查
```

### 陷阱 4：数组越界无自动检测

```csharp
int[] arr = new int[5];
arr[10] = 1;  // 抛出 IndexOutOfRangeException
```

```cpp
int arr[5];
arr[10] = 1;  // ❌ 未定义行为，可能改写其他内存

// 使用 std::array 或 std::vector 的 at()
std::array<int,5> arr2;
arr2.at(10) = 1;  // ✅ 抛出 std::out_of_range
```

### 陷阱 5：没有 foreach 的默认安全遍历

```csharp
foreach (var item in list) { Console.WriteLine(item); }
```

```cpp
// C++ 范围 for（现代写法，安全）
for (const auto &item : vec) { std::cout << item; }

// 老式写法（索引越界风险）
for (int i = 0; i <= vec.size(); i++) {  // ❌ <= 应该是 <
    std::cout << vec[i];
}
```

### 陷阱 6：析构函数不等于 Dispose

```csharp
// C# Dispose 需要手动调用或 using 语句
public class MyResource : IDisposable {
    public void Dispose() { /* 释放非托管资源 */ }
}
```

```cpp
// C++ 析构函数在对象生命周期结束时自动调用（RAII）
class MyResource {
public:
    MyResource()  { acquire(); }
    ~MyResource() { release(); }  // 自动调用，无需手动
};

// 栈上对象：函数返回时自动释放
{
    MyResource r;
    // 使用 r
}  // r.~MyResource() 在此处自动调用
```

---

**返回目录**：[README →](../README.md)  
**附录 B**：[工具链速查手册 →](appendix_b_tools.md)
