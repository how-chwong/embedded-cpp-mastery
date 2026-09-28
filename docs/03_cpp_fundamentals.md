# 第 03 章：C++ 基础

> **学习目标**：掌握 C++ 相对 C 的核心扩展，能够编写面向对象的 C++ 程序。

---

## 目录

1. [C++ 简史与 C 的关系](#1-c-简史与-c-的关系)
2. [C++ 开发环境](#2-c-开发环境)
3. [C++ 对 C 的增强](#3-c-对-c-的增强)
4. [引用（Reference）](#4-引用reference)
5. [函数重载与默认参数](#5-函数重载与默认参数)
6. [命名空间](#6-命名空间)
7. [类与对象基础](#7-类与对象基础)
8. [构造函数与析构函数](#8-构造函数与析构函数)
9. [拷贝语义](#9-拷贝语义)
10. [运算符重载](#10-运算符重载)
11. [实战：实现 String 类](#11-实战实现-string-类)

---

## 1. C++ 简史与 C 的关系

| 版本 | 年份 | 重大特性 |
|------|------|---------|
| C++ 98/03 | 1998/2003 | STL、异常、模板 |
| C++ 11 | 2011 | 移动语义、lambda、智能指针 |
| C++ 14 | 2014 | 泛型 lambda、返回类型推导 |
| C++ 17 | 2017 | 结构化绑定、std::optional、if constexpr |
| C++ 20 | 2020 | Concepts、Ranges、Coroutines、Modules |
| C++ 23 | 2023 | std::expected、更多标准库改进 |

**C++ 与 C 的关系**：
- C++ 向上兼容大部分 C 代码
- C++ 添加了 OOP、泛型编程、异常处理、RAII 等机制
- 嵌入式中常使用"C++ without exceptions/RTTI"的子集

```bash
# 编译 C++（推荐标准）
g++ -std=c++17 -Wall -Wextra -O2 main.cpp -o app
```

---

## 2. C++ 开发环境

```bash
# 安装
sudo apt install -y g++ cmake ninja-build

# CMakeLists.txt 最小示例
cmake_minimum_required(VERSION 3.16)
project(MyApp LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_executable(app main.cpp)
target_compile_options(app PRIVATE -Wall -Wextra)
```

```bash
cmake -B build -G Ninja
cmake --build build
./build/app
```

---

## 3. C++ 对 C 的增强

### 3.1 类型系统增强

```cpp
// bool 类型（C 需要 <stdbool.h>，C++ 原生支持）
bool flag = true;

// nullptr（取代 NULL，类型安全）
int *p = nullptr;

// auto 类型推导
auto x = 42;        // int
auto y = 3.14;      // double
auto z = "hello";   // const char*

// 范围 for 循环
int arr[] = {1, 2, 3, 4, 5};
for (auto v : arr) {
    std::cout << v << " ";
}
```

### 3.2 输入输出

```cpp
#include <iostream>
#include <iomanip>

// 输出
std::cout << "Hello, " << "World!" << std::endl;
std::cout << std::fixed << std::setprecision(2) << 3.14159 << "\n";

// 输入
int n;
std::cin >> n;

std::string name;
std::getline(std::cin, name);
```

### 3.3 string 类

```cpp
#include <string>

std::string s1 = "Hello";
std::string s2 = "World";
std::string s3 = s1 + ", " + s2 + "!";  // 拼接

s3.length();          // 13
s3.substr(7, 5);      // "World"
s3.find("World");     // 7
s3.replace(7, 5, "C++");  // "Hello, C++!"

// 转换
int n = std::stoi("42");
std::string str = std::to_string(3.14);
```

---

## 4. 引用（Reference）

```cpp
int x = 10;
int &ref = x;    // ref 是 x 的别名

ref = 20;        // 修改 ref 就是修改 x
std::cout << x;  // 20

// 引用 vs 指针
void swap_ptr(int *a, int *b) { int t=*a; *a=*b; *b=t; }
void swap_ref(int &a, int &b) { int t= a;  a= b;  b=t; }  // 更简洁

int a=1, b=2;
swap_ref(a, b);   // a=2, b=1
```

### 4.1 const 引用

```cpp
// 接受任意表达式，避免拷贝
void print(const std::string &s) {
    std::cout << s << "\n";
}

print("Hello");            // 字符串字面量也可以
print(std::string("Hi")); // 临时对象也可以
```

---

## 5. 函数重载与默认参数

```cpp
// 函数重载：名称相同，参数不同
int    add(int a, int b)       { return a + b; }
double add(double a, double b) { return a + b; }
int    add(int a, int b, int c){ return a + b + c; }

// 默认参数（必须从右向左设置）
void connect(const std::string &host, int port = 8080, bool tls = false);

connect("192.168.1.1");           // 使用默认 port 和 tls
connect("192.168.1.1", 443, true);
```

---

## 6. 命名空间

```cpp
namespace sensors {
    struct Reading { float value; int id; };
    void init();
    Reading read(int id);
}

namespace actuators {
    void set_pwm(int channel, float duty);
}

// 使用
sensors::Reading r = sensors::read(0);
actuators::set_pwm(1, 0.75f);

// using 声明
using sensors::Reading;
Reading r2 = sensors::read(1);

// 嵌套命名空间（C++17）
namespace hal::gpio {
    void set_pin(int pin, bool state);
}
hal::gpio::set_pin(13, true);
```

---

## 7. 类与对象基础

```cpp
class Temperature {
public:
    // 构造函数
    Temperature(double celsius) : celsius_(celsius) {}

    // 成员函数
    double get_celsius() const { return celsius_; }
    double get_fahrenheit() const { return celsius_ * 9.0 / 5.0 + 32.0; }
    double get_kelvin() const { return celsius_ + 273.15; }

    void set(double celsius) {
        if (celsius < -273.15) throw std::invalid_argument("Below absolute zero");
        celsius_ = celsius;
    }

private:
    double celsius_;    // 成员变量用下划线后缀
};

Temperature t(100.0);
std::cout << t.get_fahrenheit() << "°F\n";  // 212°F
```

### 7.1 访问控制

| 访问修饰符 | 当前类 | 派生类 | 外部 |
|-----------|--------|--------|------|
| `public` | ✅ | ✅ | ✅ |
| `protected` | ✅ | ✅ | ❌ |
| `private` | ✅ | ❌ | ❌ |

### 7.2 静态成员

```cpp
class Counter {
public:
    Counter() { ++total_; }
    ~Counter() { --total_; }
    static int get_total() { return total_; }

private:
    static int total_;  // 所有实例共享
};

int Counter::total_ = 0;  // 类外定义静态成员

Counter c1, c2, c3;
std::cout << Counter::get_total();  // 3
```

---

## 8. 构造函数与析构函数

### 8.1 构造函数类型

```cpp
class Buffer {
public:
    // 默认构造
    Buffer() : data_(nullptr), size_(0) {}

    // 参数构造
    explicit Buffer(size_t size)
        : data_(new uint8_t[size]()), size_(size) {}

    // 析构函数（释放资源）
    ~Buffer() { delete[] data_; }

    size_t size() const { return size_; }
    uint8_t *data() { return data_; }

private:
    uint8_t *data_;
    size_t   size_;
};

Buffer b(1024);  // 自动申请和释放内存
// b 离开作用域时，析构函数自动调用
```

### 8.2 初始化列表

```cpp
class Point {
public:
    // 初始化列表（推荐，效率更高）
    Point(double x, double y) : x_(x), y_(y) {}
    // 注意：初始化顺序按照成员声明顺序，与列表顺序无关

private:
    double x_, y_;
};
```

### 8.3 委托构造（C++11）

```cpp
class Config {
public:
    Config() : Config("localhost", 8080) {}  // 委托给下面的构造
    Config(std::string host, int port)
        : host_(std::move(host)), port_(port) {}

private:
    std::string host_;
    int         port_;
};
```

---

## 9. 拷贝语义

```cpp
class Buffer {
public:
    explicit Buffer(size_t size)
        : data_(new uint8_t[size]()), size_(size) {}

    // 拷贝构造（深拷贝）
    Buffer(const Buffer &other)
        : data_(new uint8_t[other.size_]()), size_(other.size_) {
        std::copy(other.data_, other.data_ + size_, data_);
    }

    // 拷贝赋值运算符
    Buffer &operator=(const Buffer &other) {
        if (this == &other) return *this;  // 自赋值检查
        delete[] data_;
        size_ = other.size_;
        data_ = new uint8_t[size_]();
        std::copy(other.data_, other.data_ + size_, data_);
        return *this;
    }

    ~Buffer() { delete[] data_; }

private:
    uint8_t *data_;
    size_t   size_;
};
```

> 📌 **Rule of Three**：如果需要自定义析构函数、拷贝构造或拷贝赋值，三个通常都要自定义。

---

## 10. 运算符重载

```cpp
class Vector2D {
public:
    double x, y;
    Vector2D(double x=0, double y=0) : x(x), y(y) {}

    // 加法
    Vector2D operator+(const Vector2D &rhs) const {
        return {x + rhs.x, y + rhs.y};
    }

    // 复合赋值
    Vector2D &operator+=(const Vector2D &rhs) {
        x += rhs.x; y += rhs.y;
        return *this;
    }

    // 比较
    bool operator==(const Vector2D &rhs) const {
        return x == rhs.x && y == rhs.y;
    }

    // 流输出（友元函数）
    friend std::ostream &operator<<(std::ostream &os, const Vector2D &v) {
        return os << "(" << v.x << ", " << v.y << ")";
    }
};

Vector2D a(1, 2), b(3, 4);
Vector2D c = a + b;           // (4, 6)
std::cout << c << "\n";       // (4, 6)
```

---

## 11. 实战：实现 String 类

```cpp
// my_string.hpp
#pragma once
#include <cstring>
#include <stdexcept>
#include <iostream>

class MyString {
public:
    MyString() : data_(new char[1]()), len_(0) {}

    MyString(const char *str) {
        len_ = std::strlen(str);
        data_ = new char[len_ + 1];
        std::strcpy(data_, str);
    }

    MyString(const MyString &other) {
        len_ = other.len_;
        data_ = new char[len_ + 1];
        std::strcpy(data_, other.data_);
    }

    MyString &operator=(const MyString &other) {
        if (this == &other) return *this;
        delete[] data_;
        len_ = other.len_;
        data_ = new char[len_ + 1];
        std::strcpy(data_, other.data_);
        return *this;
    }

    ~MyString() { delete[] data_; }

    size_t length() const { return len_; }
    const char *c_str() const { return data_; }

    char &operator[](size_t idx) {
        if (idx >= len_) throw std::out_of_range("index out of range");
        return data_[idx];
    }

    MyString operator+(const MyString &rhs) const {
        char *buf = new char[len_ + rhs.len_ + 1];
        std::strcpy(buf, data_);
        std::strcat(buf, rhs.data_);
        MyString result(buf);
        delete[] buf;
        return result;
    }

    friend std::ostream &operator<<(std::ostream &os, const MyString &s) {
        return os << s.data_;
    }

private:
    char  *data_;
    size_t len_;
};

// 测试
int main() {
    MyString s1("Hello, ");
    MyString s2("C++!");
    MyString s3 = s1 + s2;
    std::cout << s3 << "\n";  // Hello, C++!
    std::cout << s3.length() << "\n";  // 11
    return 0;
}
```

---

**上一章**：[第 02 章：C 语言进阶 ←](02_c_advanced.md)  
**下一章**：[第 04 章：C++ 进阶 →](04_cpp_advanced.md)
