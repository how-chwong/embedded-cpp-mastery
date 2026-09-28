# 第 01 章：C 语言基础

> **学习目标**：掌握 C 语言核心语法，能独立编写、编译、调试简单程序。

---

## 目录

1. [C 语言简史与定位](#1-c-语言简史与定位)
2. [开发环境搭建](#2-开发环境搭建)
3. [程序结构与编译流程](#3-程序结构与编译流程)
4. [基本数据类型](#4-基本数据类型)
5. [运算符与表达式](#5-运算符与表达式)
6. [控制流](#6-控制流)
7. [函数](#7-函数)
8. [数组与字符串](#8-数组与字符串)
9. [指针基础](#9-指针基础)
10. [结构体与联合体](#10-结构体与联合体)
11. [文件 I/O](#11-文件-io)
12. [实战练习](#12-实战练习)

---

## 1. C 语言简史与定位

| 年份 | 事件 |
|------|------|
| 1972 | Dennis Ritchie 在贝尔实验室创造 C |
| 1978 | K&R C 出版（《The C Programming Language》） |
| 1989 | ANSI C（C89/C90）标准化 |
| 1999 | C99 引入 `//` 注释、变长数组、`stdint.h` |
| 2011 | C11 引入多线程、泛型、匿名结构体 |
| 2017 | C17（缺陷修订） |
| 2023 | C23 标准发布 |

**C 的核心优势**：
- 贴近硬件，可直接操作内存
- 运行时开销极小，适合嵌入式、OS 内核
- 移植性强，几乎所有平台都有 C 编译器
- 是学习 C++、Go、Rust 等语言的最佳基础

---

## 2. 开发环境搭建

### 2.1 Linux（推荐）

```bash
sudo apt update && sudo apt install -y gcc gdb make vim
gcc --version   # 应输出 gcc 11.x 或更高
```

### 2.2 macOS

```bash
xcode-select --install          # 安装 Clang/LLVM
brew install gcc gdb            # 可选：安装 GNU GCC
```

### 2.3 Windows

推荐使用 WSL2（Windows Subsystem for Linux）：

```powershell
wsl --install -d Ubuntu-22.04
```

或安装 [MSYS2](https://www.msys2.org/)：

```bash
pacman -S mingw-w64-x86_64-gcc mingw-w64-x86_64-gdb
```

### 2.4 IDE 选择

| IDE | 平台 | 特点 |
|-----|------|------|
| VS Code + C/C++ 插件 | 全平台 | 轻量、扩展丰富 |
| CLion | 全平台 | 智能补全强，收费 |
| Eclipse CDT | 全平台 | 老牌，嵌入式插件多 |
| Vim/Neovim + clangd | Linux | 极速，高手常用 |

### 2.5 第一个程序

```c
// hello.c
#include <stdio.h>

int main(void) {
    printf("Hello, World!\n");
    return 0;
}
```

```bash
gcc hello.c -o hello    # 编译
./hello                  # 运行
```

---

## 3. 程序结构与编译流程

### 3.1 编译四阶段

```
源文件(.c) → [预处理] → (.i) → [编译] → (.s) → [汇编] → (.o) → [链接] → 可执行文件
```

```bash
gcc -E hello.c -o hello.i    # 预处理
gcc -S hello.i -o hello.s    # 编译为汇编
gcc -c hello.s -o hello.o    # 汇编为目标文件
gcc hello.o -o hello         # 链接
```

### 3.2 常用编译选项

| 选项 | 说明 |
|------|------|
| `-Wall -Wextra` | 开启所有警告 |
| `-std=c11` | 指定 C 标准 |
| `-O2` | 优化级别 2 |
| `-g` | 包含调试信息 |
| `-I<path>` | 头文件搜索路径 |
| `-L<path> -l<lib>` | 链接库 |

### 3.3 多文件项目结构

```
project/
├── src/
│   ├── main.c
│   └── math_utils.c
├── include/
│   └── math_utils.h
└── Makefile
```

**Makefile 示例**：

```makefile
CC      = gcc
CFLAGS  = -Wall -Wextra -std=c11 -Iinclude
TARGET  = app

SRCS    = $(wildcard src/*.c)
OBJS    = $(SRCS:.c=.o)

$(TARGET): $(OBJS)
	$(CC) $^ -o $@

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

clean:
	rm -f $(OBJS) $(TARGET)
```

---

## 4. 基本数据类型

### 4.1 整数类型

| 类型 | 大小（典型） | 范围 |
|------|-------------|------|
| `char` | 1 字节 | -128 ~ 127 |
| `unsigned char` | 1 字节 | 0 ~ 255 |
| `short` | 2 字节 | -32768 ~ 32767 |
| `int` | 4 字节 | -2³¹ ~ 2³¹-1 |
| `long` | 4/8 字节 | 平台相关 |
| `long long` | 8 字节 | -2⁶³ ~ 2⁶³-1 |

> ⚠️ **嵌入式注意**：int 大小与平台相关，**始终使用 `<stdint.h>` 中的固定宽度类型**：

```c
#include <stdint.h>

uint8_t  a = 255;           // 无符号 8 位
int16_t  b = -1000;         // 有符号 16 位
uint32_t c = 0xDEADBEEF;    // 无符号 32 位
int64_t  d = 1234567890LL;  // 有符号 64 位
```

### 4.2 浮点类型

```c
float  f = 3.14f;           // 32 位，约 6-7 位有效数字
double d = 3.141592653589;  // 64 位，约 15-16 位有效数字
```

> ⚠️ 嵌入式中浮点运算代价高，无 FPU 的 MCU 用软件浮点，速度慢 10~100 倍。

### 4.3 类型转换

```c
int   a = 10;
float b = (float)a / 3;   // 显式转换，结果 3.333...

// 隐式转换陷阱
uint8_t x = 200;
uint8_t y = 100;
uint8_t z = x + y;        // 溢出！结果为 44（300 % 256）
int     w = x + y;        // 先提升为 int，结果为 300
```

### 4.4 `sizeof` 运算符

```c
printf("%zu\n", sizeof(int));        // 4
printf("%zu\n", sizeof(double));     // 8
printf("%zu\n", sizeof(char *));     // 8（64 位系统）
```

---

## 5. 运算符与表达式

### 5.1 算术运算符

```c
int a = 17, b = 5;
printf("%d\n", a + b);   // 22
printf("%d\n", a - b);   // 12
printf("%d\n", a * b);   // 85
printf("%d\n", a / b);   // 3  （整数除法，截断）
printf("%d\n", a % b);   // 2  （取余）
```

### 5.2 位运算符（嵌入式核心）

```c
uint8_t reg = 0b00110101;

// 置位（set bit 3）
reg |=  (1 << 3);    // reg = 0b00111101

// 清位（clear bit 2）
reg &= ~(1 << 2);    // reg = 0b00111001

// 翻转（toggle bit 5）
reg ^=  (1 << 5);    // reg = 0b00011001

// 检测位
if (reg & (1 << 4)) {
    // bit 4 为 1
}

// 移位
uint32_t val = 0x01;
val <<= 8;   // 0x00000100
val >>= 4;   // 0x00000010
```

### 5.3 比较与逻辑运算符

```c
int x = 5;
x == 5;    // true  (不要写成 x = 5!)
x != 3;    // true
x > 3 && x < 10;   // true（逻辑与）
x < 0 || x > 3;    // true（逻辑或）
!(x == 5);          // false（逻辑非）
```

### 5.4 三目运算符

```c
int max = (a > b) ? a : b;
```

---

## 6. 控制流

### 6.1 if / else if / else

```c
int score = 85;
if (score >= 90) {
    printf("优秀\n");
} else if (score >= 60) {
    printf("及格\n");
} else {
    printf("不及格\n");
}
```

### 6.2 switch

```c
int day = 3;
switch (day) {
    case 1: printf("周一\n"); break;
    case 2: printf("周二\n"); break;
    case 3: printf("周三\n"); break;
    default: printf("其他\n"); break;
}
```

### 6.3 循环

```c
// for 循环
for (int i = 0; i < 10; i++) {
    printf("%d ", i);
}

// while 循环
int n = 0;
while (n < 5) {
    printf("%d ", n++);
}

// do-while（至少执行一次）
int x;
do {
    printf("请输入正整数：");
    scanf("%d", &x);
} while (x <= 0);

// break / continue
for (int i = 0; i < 10; i++) {
    if (i == 3) continue;   // 跳过 3
    if (i == 7) break;      // 到 7 停止
    printf("%d ", i);
}
```

---

## 7. 函数

### 7.1 函数定义与声明

```c
// 声明（放在 .h 文件或使用前）
int add(int a, int b);

// 定义
int add(int a, int b) {
    return a + b;
}
```

### 7.2 传值 vs 传指针

```c
// 传值：函数内修改不影响外部
void wrong_swap(int a, int b) {
    int tmp = a; a = b; b = tmp;
}

// 传指针：函数内修改影响外部
void swap(int *a, int *b) {
    int tmp = *a; *a = *b; *b = tmp;
}

int x = 1, y = 2;
swap(&x, &y);   // x=2, y=1
```

### 7.3 递归

```c
// 阶乘
unsigned long long factorial(unsigned int n) {
    if (n <= 1) return 1;
    return n * factorial(n - 1);
}
```

> ⚠️ **嵌入式注意**：递归消耗栈空间，MCU 栈通常只有几 KB，谨慎使用递归。

### 7.4 内联函数

```c
static inline int max(int a, int b) {
    return (a > b) ? a : b;
}
```

### 7.5 函数指针

```c
int (*fp)(int, int) = add;    // 函数指针
int result = fp(3, 4);        // 调用，结果 7

// 作为回调
void apply(int *arr, int len, int (*fn)(int)) {
    for (int i = 0; i < len; i++) {
        arr[i] = fn(arr[i]);
    }
}
```

---

## 8. 数组与字符串

### 8.1 一维数组

```c
int arr[5] = {10, 20, 30, 40, 50};
int len = sizeof(arr) / sizeof(arr[0]);   // 5

for (int i = 0; i < len; i++) {
    printf("%d\n", arr[i]);
}
```

### 8.2 二维数组

```c
int matrix[3][4] = {
    {1, 2, 3, 4},
    {5, 6, 7, 8},
    {9,10,11,12}
};

// 遍历
for (int i = 0; i < 3; i++) {
    for (int j = 0; j < 4; j++) {
        printf("%3d", matrix[i][j]);
    }
    printf("\n");
}
```

### 8.3 字符串（字符数组）

```c
#include <string.h>

char name[32] = "Alice";          // 字符数组
const char *greeting = "Hello";   // 字符串字面量（只读）

// 常用字符串函数
strlen(name);                     // 5（不含 \0）
strcpy(dest, src);                // 复制（危险！）
strncpy(dest, src, sizeof(dest)-1); // 安全复制
strcat(dest, src);                // 拼接
strcmp(a, b);                     // 比较：0 表示相等
snprintf(buf, sizeof(buf), "val=%d", val); // 格式化（安全）
```

> ⚠️ **永远不要用 `gets()`，改用 `fgets()`**；`strcpy()` 也应替换为 `strncpy()` 或 `strlcpy()`。

---

## 9. 指针基础

### 9.1 指针概念

```c
int x = 42;
int *p = &x;    // p 指向 x 的地址

printf("值：%d\n",  *p);    // 解引用：42
printf("地址：%p\n",  p);   // 地址：0x...
printf("p 的地址：%p\n", &p);

*p = 100;       // 通过指针修改 x 的值
printf("%d\n", x);  // 100
```

### 9.2 指针与数组

```c
int arr[] = {1, 2, 3, 4, 5};
int *p = arr;   // 数组名即首元素地址

printf("%d\n", *(p + 2));  // 3（指针算术）
printf("%d\n", p[3]);      // 4（等价于 *(p+3)）

// 遍历
for (int *it = arr; it < arr + 5; it++) {
    printf("%d\n", *it);
}
```

### 9.3 指针的指针

```c
int x = 5;
int *p  = &x;
int **pp = &p;

printf("%d\n", **pp);   // 5
```

### 9.4 空指针与野指针

```c
int *p = NULL;     // 空指针，安全初始化

// 使用前检查
if (p != NULL) {
    *p = 10;
}

// 野指针：指向已释放/未初始化内存，严禁使用
int *wild;         // ❌ 未初始化
free(p); p = NULL; // ✅ 释放后立即置 NULL
```

---

## 10. 结构体与联合体

### 10.1 结构体

```c
#include <stdint.h>

typedef struct {
    uint8_t  id;
    char     name[32];
    float    temperature;
} Sensor;

Sensor s1 = {.id = 1, .name = "TempSensor", .temperature = 25.3f};
printf("ID: %d, Temp: %.1f\n", s1.id, s1.temperature);

// 指向结构体的指针
Sensor *sp = &s1;
printf("Name: %s\n", sp->name);  // 箭头运算符
```

### 10.2 内存对齐与填充

```c
typedef struct {
    uint8_t  a;   // 1 字节
    // 3 字节填充
    uint32_t b;   // 4 字节，从 4 字节对齐处开始
    uint8_t  c;   // 1 字节
    // 3 字节填充
} BadLayout;  // sizeof = 12，非 6

typedef struct {
    uint32_t b;   // 4 字节
    uint8_t  a;   // 1 字节
    uint8_t  c;   // 1 字节
    // 2 字节填充
} GoodLayout;  // sizeof = 8

// 嵌入式中强制紧凑布局（慎用，可能影响性能）
typedef struct __attribute__((packed)) {
    uint8_t  a;
    uint32_t b;
    uint8_t  c;
} Packed;  // sizeof = 6
```

### 10.3 联合体

```c
// 联合体：所有成员共享同一块内存
typedef union {
    uint32_t word;
    uint8_t  bytes[4];
} U32;

U32 val;
val.word = 0x12345678;
printf("byte[0] = 0x%02X\n", val.bytes[0]);  // 小端：0x78
```

### 10.4 枚举

```c
typedef enum {
    LED_OFF = 0,
    LED_ON,
    LED_BLINK
} LedState;

LedState state = LED_BLINK;
if (state == LED_BLINK) {
    printf("Blinking...\n");
}
```

---

## 11. 文件 I/O

```c
#include <stdio.h>

// 写文件
FILE *fp = fopen("data.txt", "w");
if (fp == NULL) {
    perror("fopen 失败");
    return -1;
}
fprintf(fp, "温度: %.2f\n", 36.5f);
fclose(fp);

// 读文件
fp = fopen("data.txt", "r");
char line[128];
while (fgets(line, sizeof(line), fp)) {
    printf("%s", line);
}
fclose(fp);

// 二进制读写
typedef struct { int id; float value; } Record;
Record rec = {1, 98.6f};

fp = fopen("data.bin", "wb");
fwrite(&rec, sizeof(rec), 1, fp);
fclose(fp);
```

---

## 12. 实战练习

### 练习 1：温度转换器

编写一个程序，从命令行接收摄氏温度，输出华氏和开尔文温度。

```c
// temperature.c
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char *argv[]) {
    if (argc != 2) {
        fprintf(stderr, "Usage: %s <celsius>\n", argv[0]);
        return 1;
    }
    
    double celsius = atof(argv[1]);
    double fahrenheit = celsius * 9.0 / 5.0 + 32.0;
    double kelvin = celsius + 273.15;
    
    printf("摄氏: %.2f°C\n", celsius);
    printf("华氏: %.2f°F\n", fahrenheit);
    printf("开尔文: %.2fK\n", kelvin);
    return 0;
}
```

```bash
gcc -Wall -std=c11 temperature.c -o temperature
./temperature 100
```

### 练习 2：单链表实现

实现一个整数单链表，支持头插、尾插、查找、删除和打印。

```c
// linked_list.c
#include <stdio.h>
#include <stdlib.h>

typedef struct Node {
    int data;
    struct Node *next;
} Node;

Node *create_node(int data) {
    Node *n = malloc(sizeof(Node));
    if (!n) { perror("malloc"); exit(1); }
    n->data = data;
    n->next = NULL;
    return n;
}

void push_front(Node **head, int data) {
    Node *n = create_node(data);
    n->next = *head;
    *head = n;
}

void print_list(const Node *head) {
    for (const Node *p = head; p; p = p->next)
        printf("%d -> ", p->data);
    printf("NULL\n");
}

void free_list(Node **head) {
    while (*head) {
        Node *tmp = *head;
        *head = (*head)->next;
        free(tmp);
    }
}

int main(void) {
    Node *head = NULL;
    for (int i = 1; i <= 5; i++) push_front(&head, i * 10);
    print_list(head);   // 50->40->30->20->10->NULL
    free_list(&head);
    return 0;
}
```

---

**下一章**：[第 02 章：C 语言进阶 →](02_c_advanced.md)
