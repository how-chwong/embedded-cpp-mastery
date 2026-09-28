# 第 02 章：C 语言进阶

> **学习目标**：深入掌握指针、内存管理、预处理器、并发基础，编写工程级 C 代码。

---

## 目录

1. [高级指针技术](#1-高级指针技术)
2. [动态内存管理](#2-动态内存管理)
3. [预处理器与宏](#3-预处理器与宏)
4. [作用域、链接性与存储类](#4-作用域链接性与存储类)
5. [位域与硬件寄存器建模](#5-位域与硬件寄存器建模)
6. [C 标准库深入](#6-c-标准库深入)
7. [C11 多线程](#7-c11-多线程)
8. [错误处理模式](#8-错误处理模式)
9. [内存安全编码规范](#9-内存安全编码规范)
10. [实战：通用动态数组](#10-实战通用动态数组)

---

## 1. 高级指针技术

### 1.1 const 与指针

```c
int x = 10;
const int *p1 = &x;    // 指向常量的指针（不能通过 p1 修改值）
int * const p2 = &x;   // 常量指针（地址不能变，值可变）
const int * const p3 = &x;  // 既不能改地址，也不能改值

*p1 = 20;  // ❌ 编译错误
p1 = NULL; // ✅ 允许
*p2 = 20;  // ✅ 允许
p2 = NULL; // ❌ 编译错误
```

### 1.2 函数指针数组（状态机）

```c
typedef void (*StateFunc)(void);

void state_idle(void)    { printf("IDLE\n"); }
void state_active(void)  { printf("ACTIVE\n"); }
void state_error(void)   { printf("ERROR\n"); }

StateFunc state_table[] = {
    state_idle,
    state_active,
    state_error,
};

int current_state = 0;
state_table[current_state]();   // 调用当前状态处理函数
```

### 1.3 void 指针（泛型）

```c
// 类似 C++ 的泛型，通过 void* 实现
void print_value(const void *data, char type) {
    switch (type) {
        case 'i': printf("%d\n", *(int*)data);    break;
        case 'f': printf("%.2f\n", *(float*)data); break;
        case 's': printf("%s\n", (char*)data);     break;
    }
}

int   n = 42;
float f = 3.14f;
print_value(&n, 'i');
print_value(&f, 'f');
print_value("hello", 's');
```

### 1.4 restrict 关键字（C99 性能优化）

```c
// 告知编译器两个指针不重叠，允许更激进的优化
void copy(int * restrict dst, const int * restrict src, size_t n) {
    for (size_t i = 0; i < n; i++) dst[i] = src[i];
}
```

---

## 2. 动态内存管理

### 2.1 malloc / calloc / realloc / free

```c
#include <stdlib.h>

// malloc：分配未初始化内存
int *arr = malloc(10 * sizeof(int));
if (!arr) { perror("malloc"); exit(EXIT_FAILURE); }

// calloc：分配并清零
double *vec = calloc(100, sizeof(double));

// realloc：扩容/缩容
arr = realloc(arr, 20 * sizeof(int));
if (!arr) { /* 原指针仍有效，需单独处理 */ }

// 使用完毕释放
free(arr); arr = NULL;
free(vec); vec = NULL;
```

### 2.2 内存布局

```
高地址
┌──────────────┐
│    栈 (Stack) │ ← 局部变量，自动管理，向下增长
│      ↓        │
│               │
│      ↑        │
│   堆 (Heap)   │ ← malloc/free，向上增长
├──────────────┤
│  BSS 段       │ ← 未初始化全局/静态变量（清零）
├──────────────┤
│  数据段(.data)│ ← 已初始化全局/静态变量
├──────────────┤
│  代码段(.text)│ ← 程序代码（只读）
低地址
```

### 2.3 内存泄漏检测

```bash
# 使用 Valgrind（Linux）
valgrind --leak-check=full --show-leak-kinds=all ./app

# 使用 AddressSanitizer（编译期注入）
gcc -fsanitize=address -g -O1 app.c -o app
./app
```

### 2.4 内存池（嵌入式最佳实践）

嵌入式系统中堆碎片化危险，常用固定大小内存池：

```c
#define POOL_BLOCK_SIZE  64
#define POOL_BLOCK_COUNT 32

static uint8_t pool_mem[POOL_BLOCK_SIZE * POOL_BLOCK_COUNT];
static bool    pool_used[POOL_BLOCK_COUNT] = {0};

void *pool_alloc(void) {
    for (int i = 0; i < POOL_BLOCK_COUNT; i++) {
        if (!pool_used[i]) {
            pool_used[i] = true;
            return &pool_mem[i * POOL_BLOCK_SIZE];
        }
    }
    return NULL;    // 池满
}

void pool_free(void *ptr) {
    int idx = ((uint8_t*)ptr - pool_mem) / POOL_BLOCK_SIZE;
    if (idx >= 0 && idx < POOL_BLOCK_COUNT)
        pool_used[idx] = false;
}
```

---

## 3. 预处理器与宏

### 3.1 对象宏 vs 函数宏

```c
// 对象宏（常量）
#define MAX_SIZE     256
#define PI           3.14159265358979f
#define ARRAY_SIZE(a) (sizeof(a) / sizeof((a)[0]))

// 函数宏（注意加括号防止优先级错误）
#define MIN(a, b)    ((a) < (b) ? (a) : (b))
#define BIT(n)       (1U << (n))
#define SET_BIT(reg, bit)   ((reg) |=  BIT(bit))
#define CLR_BIT(reg, bit)   ((reg) &= ~BIT(bit))
#define TST_BIT(reg, bit)   (((reg) >>  (bit)) & 1U)

// 多行宏（用 do-while(0) 包裹，保证语句完整性）
#define LOG(fmt, ...) \
    do { \
        printf("[%s:%d] " fmt "\n", __FILE__, __LINE__, ##__VA_ARGS__); \
    } while (0)
```

### 3.2 条件编译

```c
#ifndef MY_HEADER_H     // Include guard
#define MY_HEADER_H

#ifdef DEBUG
#  define DPRINT(x)  printf x
#else
#  define DPRINT(x)  /* nothing */
#endif

#if defined(__arm__)
#  define ARCH "ARM"
#elif defined(__x86_64__)
#  define ARCH "x86_64"
#endif

#endif  // MY_HEADER_H
```

### 3.3 X-Macro 技巧

```c
// 定义错误码表，一处维护，多处使用
#define ERROR_TABLE(X) \
    X(ERR_OK,      0,  "成功")      \
    X(ERR_TIMEOUT, 1,  "超时")      \
    X(ERR_NOMEM,   2,  "内存不足")  \

typedef enum {
#define ENUM_ENTRY(name, val, str) name = val,
    ERROR_TABLE(ENUM_ENTRY)
#undef ENUM_ENTRY
} ErrorCode;

const char *error_str(ErrorCode e) {
    switch (e) {
#define STR_ENTRY(name, val, str) case name: return str;
        ERROR_TABLE(STR_ENTRY)
#undef STR_ENTRY
        default: return "未知错误";
    }
}
```

---

## 4. 作用域、链接性与存储类

| 存储类 | 作用域 | 生命周期 | 说明 |
|--------|--------|---------|------|
| `auto`（默认） | 块内 | 函数调用 | 栈上分配 |
| `static`（局部） | 块内 | 程序运行期 | 只初始化一次 |
| `static`（文件级） | 文件内 | 程序运行期 | 内部链接，不可被其他文件访问 |
| `extern` | 跨文件 | 程序运行期 | 外部链接 |
| `register` | 块内 | 函数调用 | 提示存寄存器（现代编译器自动优化） |
| `volatile` | — | — | 防止编译器优化（硬件寄存器必用） |

```c
// volatile 示例：硬件寄存器
volatile uint32_t *const GPIOA_IDR = (volatile uint32_t*)0x48000010;

// 读取时每次都从内存地址读，不缓存在寄存器
uint32_t val = *GPIOA_IDR;

// static 局部变量：计数器不重置
int count_calls(void) {
    static int count = 0;
    return ++count;
}
```

---

## 5. 位域与硬件寄存器建模

```c
// 建模 STM32 USART 状态寄存器
typedef union {
    uint32_t word;
    struct {
        uint32_t PE    : 1;   // bit 0: Parity error
        uint32_t FE    : 1;   // bit 1: Framing error
        uint32_t NF    : 1;   // bit 2: Noise flag
        uint32_t ORE   : 1;   // bit 3: Overrun error
        uint32_t IDLE  : 1;   // bit 4: IDLE line detected
        uint32_t RXNE  : 1;   // bit 5: Read data register not empty
        uint32_t TC    : 1;   // bit 6: Transmission complete
        uint32_t TXE   : 1;   // bit 7: Transmit data register empty
        uint32_t       : 24;  // 保留
    } bits;
} USART_SR;

volatile USART_SR *const USART1_SR = (volatile USART_SR*)0x40011000;

if (USART1_SR->bits.RXNE) {
    // 有新数据可读
}
```

> ⚠️ 位域的比特顺序是实现定义的，跨平台需谨慎；实际工程中建议使用位运算宏保持可移植性。

---

## 6. C 标准库深入

### 6.1 `<string.h>` 安全用法

```c
// 推荐使用带 n 限制版本
strncpy(dst, src, sizeof(dst) - 1);
dst[sizeof(dst) - 1] = '\0';   // 确保 NUL 终止

strncat(dst, src, sizeof(dst) - strlen(dst) - 1);

// memcpy vs memmove
memcpy(dst, src, n);     // 不处理重叠
memmove(dst, src, n);    // 安全处理重叠
memset(buf, 0, n);       // 填充字节
```

### 6.2 `<stdlib.h>` 常用函数

```c
// 字符串转数值
int    atoi("42");
long   atol("123456");
double atof("3.14");

// 更安全的版本（可检测错误）
char *end;
long val = strtol("42abc", &end, 10);  // val=42, end->"abc"

// 排序与搜索
int cmp(const void *a, const void *b) {
    return (*(int*)a - *(int*)b);
}
int arr[] = {5, 3, 1, 4, 2};
qsort(arr, 5, sizeof(int), cmp);

int key = 3;
int *found = bsearch(&key, arr, 5, sizeof(int), cmp);
```

### 6.3 `<stdarg.h>` 可变参数

```c
#include <stdarg.h>

int my_printf(const char *fmt, ...) {
    va_list args;
    va_start(args, fmt);
    int ret = vprintf(fmt, args);
    va_end(args);
    return ret;
}
```

---

## 7. C11 多线程

```c
#include <threads.h>  // C11

typedef struct { int id; int count; } Args;

int worker(void *arg) {
    Args *a = (Args*)arg;
    for (int i = 0; i < a->count; i++) {
        printf("Thread %d: %d\n", a->id, i);
    }
    return 0;
}

int main(void) {
    thrd_t t1, t2;
    Args a1 = {1, 3}, a2 = {2, 3};

    thrd_create(&t1, worker, &a1);
    thrd_create(&t2, worker, &a2);
    thrd_join(t1, NULL);
    thrd_join(t2, NULL);
    return 0;
}
```

互斥锁：

```c
mtx_t lock;
mtx_init(&lock, mtx_plain);

mtx_lock(&lock);
// 临界区
mtx_unlock(&lock);

mtx_destroy(&lock);
```

---

## 8. 错误处理模式

### 8.1 返回码模式

```c
typedef enum {
    RET_OK    =  0,
    RET_ERROR = -1,
    RET_BUSY  = -2,
} RetCode;

RetCode read_sensor(uint8_t id, float *out) {
    if (!out) return RET_ERROR;
    if (id > MAX_SENSORS) return RET_ERROR;
    // ...
    *out = 25.0f;
    return RET_OK;
}

// 调用方
float temp;
RetCode rc = read_sensor(0, &temp);
if (rc != RET_OK) {
    LOG("Sensor read failed: %d", rc);
    return rc;
}
```

### 8.2 goto 清理模式

```c
int process_file(const char *path) {
    int   ret = -1;
    FILE *fp  = NULL;
    char *buf = NULL;

    fp = fopen(path, "r");
    if (!fp) { perror("fopen"); goto cleanup; }

    buf = malloc(4096);
    if (!buf) { goto cleanup; }

    // 正常处理...
    ret = 0;

cleanup:
    free(buf);
    if (fp) fclose(fp);
    return ret;
}
```

---

## 9. 内存安全编码规范

| 规则 | 说明 |
|------|------|
| 初始化所有变量 | 未初始化变量是 UB 来源 |
| 检查所有 malloc 返回值 | malloc 可能返回 NULL |
| free 后立即置 NULL | 防止悬空指针二次 free |
| 永不使用 gets/sprintf | 改用 fgets/snprintf |
| 数组访问前检查边界 | 缓冲区溢出是最常见漏洞 |
| 指针使用前检查非 NULL | 空指针解引用是崩溃常因 |
| 使用 `__attribute__((nonnull))` | 编译期检查非空参数 |

```c
// 编译期断言（C11）
_Static_assert(sizeof(int) == 4, "int must be 4 bytes");

// 运行时断言
#include <assert.h>
assert(ptr != NULL);
```

---

## 10. 实战：通用动态数组

```c
// vector.h
#pragma once
#include <stddef.h>
#include <stdbool.h>

typedef struct {
    void   *data;
    size_t  elem_size;
    size_t  length;
    size_t  capacity;
} Vector;

bool   vec_init(Vector *v, size_t elem_size, size_t init_cap);
bool   vec_push(Vector *v, const void *elem);
void  *vec_get(const Vector *v, size_t idx);
void   vec_free(Vector *v);
```

```c
// vector.c
#include "vector.h"
#include <stdlib.h>
#include <string.h>

bool vec_init(Vector *v, size_t elem_size, size_t init_cap) {
    v->data = malloc(elem_size * init_cap);
    if (!v->data) return false;
    v->elem_size = elem_size;
    v->length    = 0;
    v->capacity  = init_cap;
    return true;
}

bool vec_push(Vector *v, const void *elem) {
    if (v->length == v->capacity) {
        size_t new_cap = v->capacity * 2;
        void *p = realloc(v->data, v->elem_size * new_cap);
        if (!p) return false;
        v->data     = p;
        v->capacity = new_cap;
    }
    memcpy((uint8_t*)v->data + v->length * v->elem_size, elem, v->elem_size);
    v->length++;
    return true;
}

void *vec_get(const Vector *v, size_t idx) {
    if (idx >= v->length) return NULL;
    return (uint8_t*)v->data + idx * v->elem_size;
}

void vec_free(Vector *v) {
    free(v->data);
    v->data = NULL;
    v->length = v->capacity = 0;
}
```

```c
// main.c
#include <stdio.h>
#include "vector.h"

int main(void) {
    Vector v;
    vec_init(&v, sizeof(int), 4);

    for (int i = 0; i < 10; i++) {
        vec_push(&v, &i);
    }
    for (size_t i = 0; i < v.length; i++) {
        printf("%d ", *(int*)vec_get(&v, i));
    }
    printf("\n");
    vec_free(&v);
    return 0;
}
```

---

**上一章**：[第 01 章：C 语言基础 ←](01_c_fundamentals.md)  
**下一章**：[第 03 章：C++ 基础 →](03_cpp_fundamentals.md)
