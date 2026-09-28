# 第 05 章：现代 C++（C++11/14/17/20）

> **学习目标**：掌握现代 C++ 核心特性，写出简洁、安全、高效的代码。

---

## 目录

1. [移动语义与右值引用](#1-移动语义与右值引用)
2. [智能指针](#2-智能指针)
3. [Lambda 表达式](#3-lambda-表达式)
4. [类型推导 auto / decltype](#4-类型推导-auto--decltype)
5. [constexpr 与编译期计算](#5-constexpr-与编译期计算)
6. [std::optional / variant / any](#6-stdoptional--variant--any)
7. [结构化绑定（C++17）](#7-结构化绑定c17)
8. [并发编程（std::thread / future）](#8-并发编程stdthread--future)
9. [Concepts（C++20）](#9-conceptsc20)
10. [Ranges（C++20）](#10-rangesc20)
11. [嵌入式中的现代 C++ 实践](#11-嵌入式中的现代-c-实践)

---

## 1. 移动语义与右值引用

### 1.1 左值与右值

```cpp
int x = 42;        // x 是左值（有名字，可取地址）
int y = x + 1;     // x+1 是右值（临时值，无地址）

int &lref = x;     // 左值引用：绑定左值
int &&rref = 42;   // 右值引用：绑定右值（C++11）
```

### 1.2 移动构造与移动赋值

```cpp
class Buffer {
public:
    explicit Buffer(size_t size)
        : data_(new uint8_t[size]()), size_(size) {}

    // 移动构造（窃取资源，不分配内存）
    Buffer(Buffer &&other) noexcept
        : data_(other.data_), size_(other.size_) {
        other.data_ = nullptr;
        other.size_ = 0;
    }

    // 移动赋值
    Buffer &operator=(Buffer &&other) noexcept {
        if (this != &other) {
            delete[] data_;
            data_       = other.data_;
            size_       = other.size_;
            other.data_ = nullptr;
            other.size_ = 0;
        }
        return *this;
    }

    ~Buffer() { delete[] data_; }

private:
    uint8_t *data_;
    size_t   size_;
};

Buffer a(1024);
Buffer b = std::move(a);  // a 的资源转移到 b，a 变为空
```

> 📌 **Rule of Five**（C++11）：析构 + 拷贝构造 + 拷贝赋值 + 移动构造 + 移动赋值。

### 1.3 std::move 与完美转发

```cpp
// std::move：将左值转换为右值引用
std::string s1 = "Hello";
std::string s2 = std::move(s1);  // s1 变为空字符串，零拷贝

// std::forward：保持值类别（用于模板）
template<typename T, typename... Args>
std::unique_ptr<T> make(Args&&... args) {
    return std::unique_ptr<T>(new T(std::forward<Args>(args)...));
}
```

---

## 2. 智能指针

### 2.1 unique_ptr（独占所有权）

```cpp
#include <memory>

// 创建（推荐用 make_unique，C++14）
auto p = std::make_unique<int>(42);
auto buf = std::make_unique<uint8_t[]>(1024);  // 数组版本

// 所有权转移
auto p2 = std::move(p);  // p 变为空
if (!p) std::cout << "p is empty\n";

// 自动释放（离开作用域）
{
    auto sensor = std::make_unique<TempSensor>(0);
    sensor->read();
}  // sensor 在此处自动 delete
```

### 2.2 shared_ptr（共享所有权）

```cpp
auto sp1 = std::make_shared<std::string>("Hello");
auto sp2 = sp1;               // 引用计数 = 2
auto sp3 = sp1;               // 引用计数 = 3

sp1.use_count();              // 3
sp2.reset();                  // 引用计数 = 2
// 所有 shared_ptr 析构时，对象才被释放
```

### 2.3 weak_ptr（避免循环引用）

```cpp
struct Node {
    std::shared_ptr<Node> next;
    std::weak_ptr<Node>   prev;  // 使用 weak_ptr 打破循环
    int value;
};
```

### 2.4 嵌入式中的智能指针

> 嵌入式中 `shared_ptr` 引用计数需要原子操作，开销较大。推荐：
> - 小型 MCU：使用 `unique_ptr`（零开销抽象）
> - 或使用自定义删除器的 `unique_ptr`

```cpp
// 自定义删除器（用于内存池）
auto deleter = [](Sensor *s) { pool_free(s); };
auto p = std::unique_ptr<Sensor, decltype(deleter)>(
    new(pool_alloc()) Sensor(), deleter);
```

---

## 3. Lambda 表达式

```cpp
// 基本形式
auto add = [](int a, int b) -> int { return a + b; };
std::cout << add(3, 4) << "\n";  // 7

// 捕获外部变量
int threshold = 10;
auto above = [threshold](int x) { return x > threshold; };  // 值捕获
auto above2 = [&threshold](int x) { return x > threshold; };// 引用捕获
auto above3 = [=](int x) { return x > threshold; };         // 全部值捕获
auto above4 = [&](int x) { return x > threshold; };         // 全部引用捕获

// 与 STL 算法结合
std::vector<int> v = {1, 5, 3, 8, 2};
std::sort(v.begin(), v.end(), [](int a, int b) { return a > b; });  // 降序

int count = std::count_if(v.begin(), v.end(),
                          [](int x) { return x % 2 == 0; });

// 泛型 lambda（C++14）
auto print = [](const auto &x) { std::cout << x << "\n"; };
print(42);
print(3.14);
print("Hello");

// mutable lambda（修改值捕获的副本）
int counter = 0;
auto inc = [counter]() mutable { return ++counter; };
```

---

## 4. 类型推导 auto / decltype

```cpp
// auto
auto x = 42;                        // int
auto y = 3.14;                      // double
auto z = std::vector<int>{1,2,3};   // std::vector<int>

// auto 与引用
auto &ref = x;                      // int&
const auto &cref = x;               // const int&

// decltype：获取表达式类型
int a = 5;
decltype(a) b = 10;                 // int
decltype(a + 3.0) c = 8.0;         // double

// 返回类型推导（C++14）
auto multiply(int a, int b) { return a * b; }

// 尾置返回类型（C++11，用于依赖参数类型的返回值）
template<typename T, typename U>
auto add(T a, U b) -> decltype(a + b) { return a + b; }
```

---

## 5. constexpr 与编译期计算

```cpp
// constexpr 函数（C++11 限制较多，C++14 放宽）
constexpr int factorial(int n) {
    return n <= 1 ? 1 : n * factorial(n - 1);
}
constexpr int f5 = factorial(5);  // 编译期计算，结果 120

// constexpr 变量
constexpr double PI = 3.141592653589793;
constexpr size_t BUFFER_SIZE = 1024;

// if constexpr（C++17，编译期条件）
template<typename T>
void process(T val) {
    if constexpr (std::is_integral_v<T>) {
        std::cout << "Integer: " << val << "\n";
    } else if constexpr (std::is_floating_point_v<T>) {
        std::cout << "Float: " << std::fixed << val << "\n";
    }
}

// consteval（C++20，强制编译期求值）
consteval int square(int x) { return x * x; }
constexpr int s = square(7);  // 必须编译期计算
```

### 5.1 编译期查表（嵌入式常用）

```cpp
// 预计算 CRC 查找表
constexpr uint8_t crc8_table[256] = []() constexpr {
    std::array<uint8_t, 256> table{};
    for (int i = 0; i < 256; i++) {
        uint8_t crc = i;
        for (int j = 0; j < 8; j++) {
            crc = (crc & 0x80) ? (crc << 1) ^ 0x07 : (crc << 1);
        }
        table[i] = crc;
    }
    return table;
}();  // 注意：C++20 才支持 std::array 的 constexpr 完整支持
```

---

## 6. std::optional / variant / any

### 6.1 optional（可选值，替代指针 + NULL）

```cpp
#include <optional>

std::optional<float> read_sensor(int id) {
    if (id < 0 || id > 15) return std::nullopt;  // 无效
    return 25.0f;  // 有效值
}

auto result = read_sensor(0);
if (result.has_value()) {
    std::cout << "Temp: " << result.value() << "\n";
}
// 或使用 value_or 提供默认值
float temp = read_sensor(99).value_or(-1.0f);
```

### 6.2 variant（类型安全的联合体）

```cpp
#include <variant>

using Measurement = std::variant<float, int, std::string>;

Measurement m = 36.6f;
std::visit([](auto val) { std::cout << val << "\n"; }, m);

m = 42;
if (auto *p = std::get_if<int>(&m)) {
    std::cout << "Int: " << *p << "\n";
}
```

### 6.3 expected（C++23，错误处理新范式）

```cpp
#include <expected>

std::expected<float, std::string> read() {
    if (/* error */) return std::unexpected("I2C timeout");
    return 25.0f;
}

auto r = read();
if (r) std::cout << *r;
else   std::cerr << r.error();
```

---

## 7. 结构化绑定（C++17）

```cpp
// 绑定 pair
auto [min_val, max_val] = std::minmax({3, 1, 4, 1, 5, 9});

// 绑定 map 迭代
std::map<std::string, int> scores{{"Alice", 95}, {"Bob", 87}};
for (auto &[name, score] : scores) {
    std::cout << name << ": " << score << "\n";
}

// 绑定结构体
struct Point { float x, y, z; };
Point p{1.0f, 2.0f, 3.0f};
auto [x, y, z] = p;
```

---

## 8. 并发编程（std::thread / future）

```cpp
#include <thread>
#include <future>
#include <atomic>
#include <mutex>

// 基本线程
std::thread t([]{ std::cout << "Worker\n"; });
t.join();

// 带参数
void work(int id, int n) {
    for (int i = 0; i < n; i++) std::cout << id << ":" << i << "\n";
}
std::thread t2(work, 1, 5);
t2.join();

// atomic（无锁原子操作）
std::atomic<int> counter{0};
auto inc = [&]{ for (int i=0; i<1000; i++) counter++; };
std::thread t3(inc), t4(inc);
t3.join(); t4.join();
std::cout << counter;  // 2000（线程安全）

// mutex
std::mutex mtx;
std::vector<int> shared_vec;

auto push = [&](int v) {
    std::lock_guard<std::mutex> lock(mtx);
    shared_vec.push_back(v);
};

// future/promise（异步结果）
std::future<int> fut = std::async(std::launch::async, []{
    std::this_thread::sleep_for(std::chrono::seconds(1));
    return 42;
});
std::cout << fut.get() << "\n";  // 阻塞等待结果
```

---

## 9. Concepts（C++20）

```cpp
#include <concepts>

// 定义 Concept
template<typename T>
concept Numeric = std::integral<T> || std::floating_point<T>;

template<typename T>
concept Sensor = requires(T s) {
    { s.read()    } -> std::convertible_to<float>;
    { s.init()    } -> std::same_as<bool>;
    { s.is_ready()} -> std::same_as<bool>;
};

// 使用 Concept 约束模板
template<Numeric T>
T clamp(T val, T lo, T hi) {
    return val < lo ? lo : (val > hi ? hi : val);
}

// Abbreviated function template（C++20）
auto add(Numeric auto a, Numeric auto b) { return a + b; }
```

---

## 10. Ranges（C++20）

```cpp
#include <ranges>
#include <algorithm>

std::vector<int> v = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

// 管道语法
auto result = v
    | std::views::filter([](int x) { return x % 2 == 0; })  // 偶数
    | std::views::transform([](int x) { return x * x; })     // 平方
    | std::views::take(3);                                    // 取前 3

for (int n : result) std::cout << n << " ";  // 4 16 36

// 懒求值：不创建临时容器
auto squares = std::views::iota(1, 11)
             | std::views::transform([](int x) { return x * x; });
```

---

## 11. 嵌入式中的现代 C++ 实践

### 11.1 推荐使用的特性

| 特性 | 理由 |
|------|------|
| `constexpr` | 零运行时开销，编译期计算 |
| `static_assert` | 编译期检查，无代码体积影响 |
| `enum class` | 类型安全，防止隐式转换 |
| `std::array` | 安全的 C 数组替代品 |
| `unique_ptr` | 零开销 RAII 资源管理 |
| `std::optional` | 替代可空指针，更安全 |
| `[[nodiscard]]` | 强制调用方处理返回值 |

### 11.2 谨慎使用的特性

| 特性 | 原因 | 替代方案 |
|------|------|---------|
| 异常 | 代码膨胀，不确定延迟 | `std::expected`，错误码 |
| `shared_ptr` | 原子引用计数开销 | `unique_ptr` |
| 虚函数（热路径） | vtable 间接调用 | CRTP 静态多态 |
| `std::string` | 堆分配，不确定性 | `string_view`，固定缓冲区 |
| `iostream` | 代码膨胀（100KB+） | `printf`（嵌入式） |
| 动态内存（裸堆） | 碎片化 | 内存池，栈分配 |

### 11.3 编译器配置示例（嵌入式 C++17）

```cmake
target_compile_options(firmware PRIVATE
    -std=c++17
    -fno-exceptions          # 禁用异常
    -fno-rtti                # 禁用运行时类型信息
    -ffunction-sections      # 每个函数单独段，方便链接器删除未用代码
    -fdata-sections
    -Os                      # 代码大小优化
    -Wall -Wextra
)

target_link_options(firmware PRIVATE
    -Wl,--gc-sections        # 删除未使用的段
)
```

---

**上一章**：[第 04 章：C++ 进阶 ←](04_cpp_advanced.md)  
**下一章**：[第 06 章：嵌入式系统基础 →](06_embedded_fundamentals.md)
