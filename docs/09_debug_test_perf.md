# 第 09 章：调试、测试与性能优化

> **学习目标**：掌握嵌入式调试技术、单元测试方法和性能分析优化手段。

---

## 目录

1. [调试工具与技术](#1-调试工具与技术)
2. [GDB 调试](#2-gdb-调试)
3. [日志系统设计](#3-日志系统设计)
4. [HardFault 分析](#4-hardfault-分析)
5. [单元测试（Unity / CUnit）](#5-单元测试unity--cunit)
6. [C++ 单元测试（Catch2 / Google Test）](#6-c-单元测试catch2--google-test)
7. [硬件在环测试（HIL）](#7-硬件在环测试hil)
8. [性能分析](#8-性能分析)
9. [代码大小优化](#9-代码大小优化)
10. [CPU 性能优化](#10-cpu-性能优化)

---

## 1. 调试工具与技术

### 1.1 JTAG vs SWD

| 接口 | 引脚数 | 速度 | 功能 |
|------|--------|------|------|
| JTAG | 5+ | 中 | 调试 + 边界扫描 |
| SWD | 2（SWDIO+SWCLK） | 快 | 调试（ARM 专用） |

### 1.2 调试探针

| 探针 | 价格 | 协议 | 特点 |
|------|------|------|------|
| ST-Link V2/V3 | 低 | SWD/JTAG | STM32 专用 |
| J-Link | 中高 | SWD/JTAG | 通用，速度快 |
| CMSIS-DAP | 低 | SWD | 开源固件 |
| Black Magic Probe | 低 | SWD/JTAG | 内置 GDB server |

### 1.3 RTT（实时传输，无侵入日志）

```c
// SEGGER RTT：通过调试探针传输日志，不占用 UART，延迟极低
#include "SEGGER_RTT.h"

SEGGER_RTT_printf(0, "Temp: %.2f°C\n", temperature);

// 主机端查看
// JLinkRTTClient  或  openocd + telnet 到端口 19021
```

### 1.4 SWO 跟踪输出

```c
// 通过 SWO 引脚输出 ITM 数据，速度可达 6 Mbps
#include "core_cm4.h"

void itm_send_char(char c) {
    while (ITM->PORT[0].u32 == 0);  // 等待端口可用
    ITM->PORT[0].u8 = c;
}

// 重定向 printf
int fputc(int c, FILE *f) {
    itm_send_char(c);
    return c;
}
```

---

## 2. GDB 调试

### 2.1 OpenOCD + GDB 调试流程

```bash
# 终端1：启动 OpenOCD
openocd -f interface/stlink.cfg -f target/stm32f4x.cfg

# 终端2：连接 GDB
arm-none-eabi-gdb firmware.elf

(gdb) target remote :3333
(gdb) monitor reset halt
(gdb) load                    # 烧录
(gdb) break main              # 设断点
(gdb) continue                # 运行到断点
```

### 2.2 常用 GDB 命令

```bash
# 断点
break main.c:42               # 行断点
break sensor_read             # 函数断点
watch temperature             # 数据观察点（值变化时停止）
tbreak 100                    # 临时断点（触发一次后删除）

# 执行控制
continue  (c)                 # 继续运行
step      (s)                 # 单步进入
next      (n)                 # 单步跳过
finish                        # 运行到函数返回
until 50                      # 运行到第 50 行

# 查看数据
print x                       # 打印变量
print *ptr                    # 打印指针指向的值
print arr[0]@10               # 打印数组前 10 个元素
display temperature           # 每步自动显示
info registers                # 查看所有寄存器
x/16xw 0x20000000             # 查看内存（16 个 32 位字，十六进制）

# 调用栈
backtrace (bt)                # 显示调用栈
frame 2                       # 切换到第 2 帧
info locals                   # 查看局部变量

# 内存操作
set {int}0x20001000 = 42     # 修改内存
```

### 2.3 GDB TUI 模式

```bash
arm-none-eabi-gdb -tui firmware.elf
# Ctrl+X Ctrl+A 切换 TUI 模式
# 显示源代码、寄存器、内存分割视图
```

---

## 3. 日志系统设计

### 3.1 轻量级日志框架

```c
// log.h
#pragma once
#include <stdio.h>
#include <stdint.h>

typedef enum { LOG_DEBUG=0, LOG_INFO, LOG_WARN, LOG_ERROR } LogLevel;

// 全局日志级别（可运行时修改）
extern LogLevel g_log_level;

#define LOG(level, fmt, ...) \
    do { \
        if ((level) >= g_log_level) { \
            static const char *level_str[] = {"DBG","INF","WRN","ERR"}; \
            printf("[%lu][%s][%s:%d] " fmt "\r\n", \
                   HAL_GetTick(), level_str[level], \
                   __func__, __LINE__, ##__VA_ARGS__); \
        } \
    } while (0)

#define LOG_D(fmt, ...) LOG(LOG_DEBUG, fmt, ##__VA_ARGS__)
#define LOG_I(fmt, ...) LOG(LOG_INFO,  fmt, ##__VA_ARGS__)
#define LOG_W(fmt, ...) LOG(LOG_WARN,  fmt, ##__VA_ARGS__)
#define LOG_E(fmt, ...) LOG(LOG_ERROR, fmt, ##__VA_ARGS__)
```

### 3.2 异步日志（FreeRTOS 版）

```c
// 日志消息放入队列，专用任务异步输出
typedef struct { LogLevel level; char msg[96]; } LogMsg;
static QueueHandle_t xLogQueue;

void log_async(LogLevel level, const char *fmt, ...) {
    LogMsg lm = {.level = level};
    va_list a; va_start(a, fmt);
    vsnprintf(lm.msg, sizeof(lm.msg), fmt, a);
    va_end(a);
    xQueueSend(xLogQueue, &lm, 0);  // 非阻塞，丢弃而不阻塞调用方
}

void log_output_task(void *pv) {
    LogMsg lm;
    while (1) {
        if (xQueueReceive(xLogQueue, &lm, portMAX_DELAY) == pdPASS) {
            // 输出到 UART 或 Flash
            uart_puts(lm.msg);
        }
    }
}
```

---

## 4. HardFault 分析

HardFault 是嵌入式中最常见的崩溃来源。

```c
// HardFault 处理：从栈帧恢复寄存器，打印调试信息
void HardFault_Handler(void) {
    __asm volatile(
        "tst lr, #4         \n"  // 检查 EXC_RETURN，判断用哪个栈
        "ite eq             \n"
        "mrseq r0, msp      \n"  // 使用 MSP
        "mrsne r0, psp      \n"  // 使用 PSP
        "b hard_fault_handler_c\n"
    );
}

void hard_fault_handler_c(uint32_t *hardfault_args) {
    volatile uint32_t r0  = hardfault_args[0];
    volatile uint32_t r1  = hardfault_args[1];
    volatile uint32_t r2  = hardfault_args[2];
    volatile uint32_t r3  = hardfault_args[3];
    volatile uint32_t r12 = hardfault_args[4];
    volatile uint32_t lr  = hardfault_args[5];
    volatile uint32_t pc  = hardfault_args[6];  // 崩溃地址！
    volatile uint32_t psr = hardfault_args[7];

    volatile uint32_t cfsr = SCB->CFSR;  // 故障状态寄存器

    printf("=== HardFault ===\n");
    printf("PC  = 0x%08lX\n", pc);
    printf("LR  = 0x%08lX\n", lr);
    printf("CFSR= 0x%08lX\n", cfsr);

    if (cfsr & SCB_CFSR_IBUSERR_Msk)  printf("指令总线错误\n");
    if (cfsr & SCB_CFSR_DBUSERR_Msk)  printf("数据总线错误\n");
    if (cfsr & SCB_CFSR_MMARVALID_Msk)
        printf("访问地址: 0x%08lX\n", SCB->MMFAR);

    // 用 PC 值在 map 文件中查找崩溃位置
    // arm-none-eabi-addr2line -e firmware.elf 0x080012A4
    while (1);
}
```

```bash
# 根据 PC 地址找出崩溃代码行
arm-none-eabi-addr2line -e firmware.elf -a 0x080012A4 -f
# 输出示例：
# 0x080012a4
# sensor_read
# /home/user/project/src/sensor.c:47
```

---

## 5. 单元测试（Unity / CUnit）

### 5.1 Unity 测试框架（嵌入式 C 首选）

```bash
# 安装
git clone https://github.com/ThrowTheSwitch/Unity.git
```

```c
// test_sensor.c
#include "unity.h"
#include "sensor.h"

void setUp(void)    { /* 每个测试前运行 */ }
void tearDown(void) { /* 每个测试后运行 */ }

void test_temperature_conversion(void) {
    TEST_ASSERT_FLOAT_WITHIN(0.01f, 32.0f,  celsius_to_fahrenheit(0.0f));
    TEST_ASSERT_FLOAT_WITHIN(0.01f, 212.0f, celsius_to_fahrenheit(100.0f));
    TEST_ASSERT_FLOAT_WITHIN(0.01f, -40.0f, celsius_to_fahrenheit(-40.0f));
}

void test_adc_to_voltage(void) {
    TEST_ASSERT_FLOAT_WITHIN(0.001f, 0.0f,   adc_to_voltage(0));
    TEST_ASSERT_FLOAT_WITHIN(0.001f, 1.65f,  adc_to_voltage(2048));
    TEST_ASSERT_FLOAT_WITHIN(0.001f, 3.3f,   adc_to_voltage(4095));
}

void test_ring_buffer(void) {
    RingBuf rb;
    ring_init(&rb);

    TEST_ASSERT_TRUE(ring_empty(&rb));

    uint8_t byte;
    ring_push(&rb, 0xAB);
    TEST_ASSERT_TRUE(ring_pop(&rb, &byte));
    TEST_ASSERT_EQUAL_HEX8(0xAB, byte);
    TEST_ASSERT_TRUE(ring_empty(&rb));

    TEST_ASSERT_FALSE(ring_pop(&rb, &byte));  // 空
}

int main(void) {
    UNITY_BEGIN();
    RUN_TEST(test_temperature_conversion);
    RUN_TEST(test_adc_to_voltage);
    RUN_TEST(test_ring_buffer);
    return UNITY_END();
}
```

```makefile
# Makefile 运行测试（Host 上编译）
test: test_sensor.c src/sensor.c Unity/src/unity.c
	gcc -I Unity/src -I include $^ -o test_runner
	./test_runner
```

---

## 6. C++ 单元测试（Catch2 / Google Test）

### 6.1 Catch2

```cpp
// test_pid.cpp
#define CATCH_CONFIG_MAIN
#include <catch2/catch_test_macros.hpp>
#include <catch2/catch_approx.hpp>
#include "pid.hpp"

TEST_CASE("PID 输出计算", "[pid]") {
    PID pid(1.0f, 0.1f, 0.01f);
    pid.set_limits(-100.0f, 100.0f);

    SECTION("比例项") {
        float out = pid.compute(10.0f, 0.0f, 0.1f);
        REQUIRE(out == Catch::Approx(10.0f).margin(0.1f));
    }

    SECTION("输出限幅") {
        float out = pid.compute(1000.0f, 0.0f, 0.1f);
        REQUIRE(out <= 100.0f);
        REQUIRE(out >= -100.0f);
    }

    SECTION("零误差输出零") {
        float out = pid.compute(0.0f, 0.0f, 0.1f);
        REQUIRE(out == Catch::Approx(0.0f));
    }
}
```

### 6.2 Google Test

```cpp
// test_filter.cpp
#include <gtest/gtest.h>
#include "low_pass_filter.hpp"

class FilterTest : public ::testing::Test {
protected:
    void SetUp() override { filter = LowPassFilter(0.1f); }
    LowPassFilter filter;
};

TEST_F(FilterTest, InitialOutput) {
    EXPECT_FLOAT_EQ(filter.process(5.0f), 0.5f);  // 0 + 0.1*(5-0)
}

TEST_F(FilterTest, StepResponse) {
    for (int i = 0; i < 100; i++) filter.process(10.0f);
    EXPECT_NEAR(filter.output(), 10.0f, 0.01f);  // 最终应收敛
}

int main(int argc, char **argv) {
    testing::InitGoogleTest(&argc, argv);
    return RUN_ALL_TESTS();
}
```

---

## 7. 硬件在环测试（HIL）

```
[被测设备 (DUT)] ←── SWD ──── [J-Link/ST-Link] ────→ [测试主机]
        ↑                                                    ↓
    UART/GPIO ←──────────────────────────────────────── [Python 脚本]
```

```python
# hil_test.py - 自动化硬件测试
import serial
import time
import pytest

@pytest.fixture
def dut():
    ser = serial.Serial('/dev/ttyUSB0', 115200, timeout=1)
    time.sleep(0.5)
    yield ser
    ser.close()

def test_temperature_response(dut):
    dut.write(b"READ_TEMP\r\n")
    response = dut.readline().decode().strip()
    assert response.startswith("TEMP=")
    temp = float(response.split("=")[1])
    assert -10.0 < temp < 85.0, f"Temperature out of range: {temp}"

def test_led_control(dut):
    dut.write(b"LED ON\r\n")
    assert dut.readline().strip() == b"OK"
    # 用摄像头或光传感器验证 LED 状态
```

---

## 8. 性能分析

### 8.1 时间测量

```c
// 精确计时（使用 DWT 循环计数器）
static void dwt_init(void) {
    CoreDebug->DEMCR |= CoreDebug_DEMCR_TRCENA_Msk;
    DWT->CYCCNT  = 0;
    DWT->CTRL   |= DWT_CTRL_CYCCNTENA_Msk;
}

#define TIMER_START()   uint32_t _t0 = DWT->CYCCNT
#define TIMER_STOP(var) var = DWT->CYCCNT - _t0

uint32_t cycles;
TIMER_START();
fft_compute(data, 1024);
TIMER_STOP(cycles);
printf("FFT 耗时: %lu 周期 = %lu ns\n",
       cycles, cycles * 1000 / (SystemCoreClock / 1000000));
```

### 8.2 栈使用分析

```c
// FreeRTOS 运行时统计
void print_task_stats(void) {
    char buf[512];
    vTaskList(buf);          // 任务列表（名称、状态、优先级、栈余量）
    printf("任务状态:\n%s", buf);

    vTaskGetRunTimeStats(buf);   // 运行时间统计
    printf("运行时间:\n%s", buf);
}
```

### 8.3 程序大小分析

```bash
# 查看各段大小
arm-none-eabi-size -A firmware.elf

# 查看最大的函数/符号
arm-none-eabi-nm --size-sort --print-size firmware.elf | tail -20

# 生成可视化映射（需要 Python bloaty）
bloaty firmware.elf --domain=file -d symbols | head -30
```

---

## 9. 代码大小优化

| 优化手段 | 典型节省 | 说明 |
|---------|---------|------|
| `-Os` 代码大小优化 | 20~40% | 而不是 `-O2` |
| `--gc-sections` | 10~30% | 删除未用代码/数据 |
| `-fno-exceptions` | 10~20KB | 禁用 C++ 异常 |
| `-fno-rtti` | 2~5KB | 禁用运行时类型信息 |
| 替换 printf | 5~30KB | 用 tiny printf |
| LTO（链接时优化） | 5~15% | `-flto` |

```cmake
# CMakeLists.txt 代码大小优化配置
target_compile_options(firmware PRIVATE
    -Os -flto
    -ffunction-sections -fdata-sections
    -fno-exceptions -fno-rtti
    -fno-threadsafe-statics
)
target_link_options(firmware PRIVATE
    -Wl,--gc-sections -flto
    -Wl,--print-memory-usage
)
```

---

## 10. CPU 性能优化

### 10.1 编译器优化

```c
// 内联热路径函数
__attribute__((always_inline))
static inline float fast_filter(float prev, float curr, float alpha) {
    return prev + alpha * (curr - prev);
}

// 分支预测提示
#define LIKELY(x)   __builtin_expect(!!(x), 1)
#define UNLIKELY(x) __builtin_expect(!!(x), 0)

if (LIKELY(data_valid)) {
    process(data);
}

// 数据对齐（SIMD/DMA 要求）
uint8_t __attribute__((aligned(4))) dma_buffer[256];

// 将频繁访问的数据放到 CCMRAM（STM32F4，零等待）
uint32_t __attribute__((section(".ccmram"))) fft_buffer[1024];
```

### 10.2 查表法替代浮点计算

```c
// 预计算 sin 表（256 点，Q15 格式）
#define SIN_TABLE_SIZE 256
static const int16_t sin_table[SIN_TABLE_SIZE] = {
    0, 804, 1607, 2410, /* ... */
};

int16_t fast_sin(uint8_t angle) {
    return sin_table[angle];  // O(1) 查表
}
```

### 10.3 定点数运算（避免软件浮点）

```c
// Q16.16 定点数
typedef int32_t fixed_t;
#define FIXED_ONE  (1 << 16)

fixed_t float_to_fixed(float f) { return (fixed_t)(f * FIXED_ONE); }
float   fixed_to_float(fixed_t x) { return (float)x / FIXED_ONE; }

fixed_t fixed_mul(fixed_t a, fixed_t b) {
    return (fixed_t)(((int64_t)a * b) >> 16);
}

// 性能对比（无 FPU 的 Cortex-M0）
// 浮点乘法：~100 周期（软件模拟）
// 定点乘法：~3 周期
```

---

**上一章**：[第 08 章：RTOS 与并发编程 ←](08_rtos.md)  
**下一章**：[第 10 章：生产环境落地方案 →](10_production.md)
