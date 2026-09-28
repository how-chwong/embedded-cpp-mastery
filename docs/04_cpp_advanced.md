# 第 04 章：C++ 进阶（OOP / 模板 / STL）

> **学习目标**：深入掌握继承、多态、模板编程和 STL，编写工程级 C++ 代码。

---

## 目录

1. [继承](#1-继承)
2. [多态与虚函数](#2-多态与虚函数)
3. [抽象类与接口](#3-抽象类与接口)
4. [多重继承与虚继承](#4-多重继承与虚继承)
5. [异常处理](#5-异常处理)
6. [模板（Templates）](#6-模板templates)
7. [STL 容器](#7-stl-容器)
8. [STL 算法](#8-stl-算法)
9. [迭代器](#9-迭代器)
10. [实战：传感器驱动框架](#10-实战传感器驱动框架)

---

## 1. 继承

```cpp
class Animal {
public:
    Animal(std::string name) : name_(std::move(name)) {}
    virtual ~Animal() = default;

    std::string name() const { return name_; }
    virtual void speak() const { std::cout << name_ << ": ...\n"; }

protected:
    std::string name_;
};

class Dog : public Animal {
public:
    Dog(std::string name) : Animal(std::move(name)) {}
    void speak() const override { std::cout << name_ << ": Woof!\n"; }
    void fetch() const { std::cout << name_ << " fetches the ball!\n"; }
};

class Cat : public Animal {
public:
    Cat(std::string name) : Animal(std::move(name)) {}
    void speak() const override { std::cout << name_ << ": Meow!\n"; }
};
```

### 1.1 继承访问控制

| 基类访问 | public 继承 | protected 继承 | private 继承 |
|---------|------------|---------------|-------------|
| public  | public     | protected     | private |
| protected | protected | protected   | private |
| private | 不可访问   | 不可访问       | 不可访问 |

---

## 2. 多态与虚函数

```cpp
void make_speak(const Animal &a) {
    a.speak();  // 运行时多态，调用实际类型的 speak
}

Dog d("Rex");
Cat c("Whiskers");

make_speak(d);  // Rex: Woof!
make_speak(c);  // Whiskers: Meow!

// 基类指针/引用指向派生类对象
std::vector<std::unique_ptr<Animal>> zoo;
zoo.push_back(std::make_unique<Dog>("Buddy"));
zoo.push_back(std::make_unique<Cat>("Felix"));

for (auto &animal : zoo) {
    animal->speak();  // 多态调用
}
```

### 2.1 虚函数表（vtable）原理

```
Dog 对象内存布局：
┌──────────────────┐
│ vptr ──────────→ │ vtable for Dog
│                  │ [0]: Dog::speak()
│ name_            │ [1]: ~Dog()
└──────────────────┘
```

> ⚠️ **嵌入式注意**：虚函数有少量运行时开销（一次指针间接寻址）和额外内存（每对象一个 vptr）。资源极度受限时考虑 CRTP（静态多态）。

### 2.2 CRTP 静态多态（嵌入式推荐）

```cpp
// 编译期多态，零运行时开销
template<typename Derived>
class SensorBase {
public:
    float read() {
        return static_cast<Derived*>(this)->read_impl();
    }
};

class TempSensor : public SensorBase<TempSensor> {
public:
    float read_impl() { return 25.0f; }  // 实际硬件读取
};

TempSensor ts;
float t = ts.read();  // 直接调用，无虚表查找
```

---

## 3. 抽象类与接口

```cpp
// 纯虚函数 = 抽象方法
class IDriver {
public:
    virtual ~IDriver() = default;
    virtual bool init() = 0;
    virtual bool write(const uint8_t *buf, size_t len) = 0;
    virtual bool read(uint8_t *buf, size_t len) = 0;
    virtual void deinit() = 0;
};

class UartDriver : public IDriver {
public:
    bool init() override { /* 初始化 UART 硬件 */ return true; }
    bool write(const uint8_t *buf, size_t len) override { /* 发送 */ return true; }
    bool read(uint8_t *buf, size_t len) override { /* 接收 */ return true; }
    void deinit() override { /* 关闭 */ }
};

// 通过接口操作，不依赖具体实现（依赖倒置原则）
void send_packet(IDriver &driver, const uint8_t *data, size_t len) {
    driver.write(data, len);
}
```

---

## 4. 多重继承与虚继承

```cpp
class Flyable { public: virtual void fly() = 0; };
class Swimmable { public: virtual void swim() = 0; };

class Duck : public Flyable, public Swimmable {
public:
    void fly()  override { std::cout << "Duck flying\n"; }
    void swim() override { std::cout << "Duck swimming\n"; }
};

// 菱形继承问题与虚继承解决方案
class A { public: int x = 1; };
class B : virtual public A {};
class C : virtual public A {};
class D : public B, public C {};  // D 中只有一个 A 的副本
```

---

## 5. 异常处理

```cpp
#include <stdexcept>

class HardwareException : public std::runtime_error {
public:
    explicit HardwareException(const std::string &msg, int error_code)
        : std::runtime_error(msg), error_code_(error_code) {}

    int error_code() const { return error_code_; }

private:
    int error_code_;
};

float read_sensor(int id) {
    if (id < 0 || id > 15) {
        throw std::out_of_range("Sensor ID out of range: " + std::to_string(id));
    }
    // 模拟硬件故障
    if (id == 5) {
        throw HardwareException("I2C read timeout", -110);
    }
    return 25.0f;
}

try {
    float temp = read_sensor(5);
} catch (const HardwareException &e) {
    std::cerr << "HW Error [" << e.error_code() << "]: " << e.what() << "\n";
} catch (const std::exception &e) {
    std::cerr << "Error: " << e.what() << "\n";
} catch (...) {
    std::cerr << "Unknown exception\n";
}
```

> ⚠️ **嵌入式注意**：异常处理增加代码体积（约 10-30%）和不确定延迟。MISRA C++ 和许多嵌入式项目禁止使用异常，改用错误码。编译时用 `-fno-exceptions` 禁用。

---

## 6. 模板（Templates）

### 6.1 函数模板

```cpp
template<typename T>
T max_val(T a, T b) { return (a > b) ? a : b; }

auto m1 = max_val(3, 5);        // int
auto m2 = max_val(3.14, 2.71);  // double
auto m3 = max_val<float>(1, 2); // 显式指定类型
```

### 6.2 类模板

```cpp
template<typename T, size_t N>
class FixedArray {
public:
    T &operator[](size_t i) { return data_[i]; }
    const T &operator[](size_t i) const { return data_[i]; }
    size_t size() const { return N; }

    T *begin() { return data_; }
    T *end()   { return data_ + N; }

private:
    T data_[N];
};

FixedArray<int, 8> arr;   // 栈上分配，无动态内存
arr[0] = 42;
for (auto &v : arr) { /* ... */ }
```

### 6.3 模板特化

```cpp
template<typename T>
struct Serializer {
    static void serialize(const T &val, uint8_t *buf) {
        memcpy(buf, &val, sizeof(T));
    }
};

// 针对 float 的特化（IEEE 754 大端序转换）
template<>
struct Serializer<float> {
    static void serialize(const float &val, uint8_t *buf) {
        uint32_t raw;
        memcpy(&raw, &val, 4);
        buf[0] = (raw >> 24) & 0xFF;
        buf[1] = (raw >> 16) & 0xFF;
        buf[2] = (raw >>  8) & 0xFF;
        buf[3] = (raw >>  0) & 0xFF;
    }
};
```

### 6.4 可变参数模板（C++11）

```cpp
// 类型安全的 printf
template<typename T, typename... Args>
void log(const char *fmt, T first, Args... rest) {
    // 递归展开
}

// 折叠表达式（C++17）
template<typename... Args>
auto sum(Args... args) { return (args + ...); }

sum(1, 2, 3, 4, 5);  // 15
```

---

## 7. STL 容器

### 7.1 顺序容器

```cpp
#include <vector>
#include <deque>
#include <list>
#include <array>

// vector：动态数组，O(1) 随机访问
std::vector<int> v = {1, 2, 3};
v.push_back(4);          // 尾部追加
v.emplace_back(5);       // 就地构造，更高效
v.insert(v.begin(), 0);  // 头部插入，O(n)
v.erase(v.begin());      // 删除
v.reserve(100);          // 预分配，避免频繁 realloc
v.size();                // 元素数量
v.capacity();            // 已分配容量

// array：固定大小，栈上分配（嵌入式推荐）
std::array<int, 5> arr = {1, 2, 3, 4, 5};
arr.size();              // 5（编译期常量）
```

### 7.2 关联容器

```cpp
#include <map>
#include <unordered_map>
#include <set>

// map：有序，O(log n) 查找
std::map<std::string, int> scores;
scores["Alice"] = 95;
scores["Bob"]   = 87;
scores.find("Alice")->second;  // 95

// unordered_map：哈希，O(1) 平均查找
std::unordered_map<std::string, float> config;
config["timeout_ms"] = 1000.0f;
config["retry_count"] = 3.0f;

// set：唯一有序元素
std::set<int> ids = {5, 3, 1, 4, 2};  // 自动排序：{1,2,3,4,5}
ids.count(3);   // 1（存在）
ids.count(9);   // 0（不存在）
```

### 7.3 容器适配器

```cpp
#include <stack>
#include <queue>

std::stack<int> stk;
stk.push(1); stk.push(2); stk.push(3);
stk.top();   // 3
stk.pop();

std::queue<std::string> q;
q.push("msg1"); q.push("msg2");
q.front();   // "msg1"
q.pop();
```

---

## 8. STL 算法

```cpp
#include <algorithm>
#include <numeric>

std::vector<int> v = {5, 3, 1, 4, 2};

// 排序
std::sort(v.begin(), v.end());                    // 升序
std::sort(v.begin(), v.end(), std::greater<>());  // 降序

// 查找
auto it = std::find(v.begin(), v.end(), 3);
if (it != v.end()) std::cout << *it << "\n";

// 转换
std::vector<int> doubled(v.size());
std::transform(v.begin(), v.end(), doubled.begin(),
               [](int x) { return x * 2; });

// 归约
int sum = std::accumulate(v.begin(), v.end(), 0);
int max = *std::max_element(v.begin(), v.end());

// 过滤（C++20 ranges 更优雅）
v.erase(std::remove_if(v.begin(), v.end(),
        [](int x) { return x % 2 == 0; }),  // 删除偶数
        v.end());

// 统计
int count = std::count_if(v.begin(), v.end(),
                          [](int x) { return x > 3; });
```

---

## 9. 迭代器

```cpp
// 迭代器分类
std::vector<int> v = {1,2,3,4,5};

// 正向迭代
for (auto it = v.begin(); it != v.end(); ++it) { *it *= 2; }

// 反向迭代
for (auto it = v.rbegin(); it != v.rend(); ++it) {
    std::cout << *it << " ";  // 10 8 6 4 2
}

// 常量迭代器（只读）
for (auto it = v.cbegin(); it != v.cend(); ++it) {
    std::cout << *it << " ";
}

// 插入迭代器
std::vector<int> dest;
std::copy(v.begin(), v.end(), std::back_inserter(dest));
```

---

## 10. 实战：传感器驱动框架

```cpp
// sensor_framework.hpp
#pragma once
#include <cstdint>
#include <optional>
#include <string_view>

// 传感器接口
class ISensor {
public:
    virtual ~ISensor() = default;
    virtual bool     init() = 0;
    virtual float    read() = 0;
    virtual bool     is_ready() const = 0;
    virtual std::string_view name() const = 0;
};

// 温度传感器实现
class NTC_Thermistor : public ISensor {
public:
    NTC_Thermistor(uint8_t adc_channel, float r_ref)
        : adc_ch_(adc_channel), r_ref_(r_ref), ready_(false) {}

    bool init() override {
        // 初始化 ADC...
        ready_ = true;
        return true;
    }

    float read() override {
        // 读 ADC，转换为温度
        uint16_t raw = read_adc(adc_ch_);
        float r = r_ref_ * (4095.0f / raw - 1.0f);
        return steinhart_hart(r);
    }

    bool is_ready() const override { return ready_; }
    std::string_view name() const override { return "NTC_Thermistor"; }

private:
    uint8_t adc_ch_;
    float   r_ref_;
    bool    ready_;

    uint16_t read_adc(uint8_t ch) { return 2048; /* 模拟值 */ }

    float steinhart_hart(float r) {
        const float A = 1.009249522e-3f;
        const float B = 2.378405444e-4f;
        const float C = 2.019202697e-7f;
        float log_r = std::log(r);
        return 1.0f / (A + B * log_r + C * log_r * log_r * log_r) - 273.15f;
    }
};

// 传感器管理器
template<size_t N>
class SensorManager {
public:
    bool add(ISensor *s) {
        if (count_ >= N) return false;
        sensors_[count_++] = s;
        return true;
    }

    void init_all() {
        for (size_t i = 0; i < count_; i++) {
            if (!sensors_[i]->init()) {
                // 处理初始化失败
            }
        }
    }

    void print_all() {
        for (size_t i = 0; i < count_; i++) {
            if (sensors_[i]->is_ready()) {
                printf("[%s] = %.2f\n",
                       sensors_[i]->name().data(),
                       sensors_[i]->read());
            }
        }
    }

private:
    ISensor *sensors_[N]{};
    size_t   count_ = 0;
};
```

---

**上一章**：[第 03 章：C++ 基础 ←](03_cpp_fundamentals.md)  
**下一章**：[第 05 章：现代 C++ →](05_cpp_modern.md)
