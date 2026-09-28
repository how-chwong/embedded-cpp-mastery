# 第 06 章：嵌入式系统基础

> **学习目标**：理解嵌入式系统概念、MCU 架构、工具链，能独立搭建嵌入式开发环境。

---

## 目录

1. [嵌入式系统概述](#1-嵌入式系统概述)
2. [MCU 架构](#2-mcu-架构)
3. [存储器映射](#3-存储器映射)
4. [嵌入式开发工具链](#4-嵌入式开发工具链)
5. [启动流程（Boot Sequence）](#5-启动流程boot-sequence)
6. [链接脚本（Linker Script）](#6-链接脚本linker-script)
7. [中断系统](#7-中断系统)
8. [时钟与定时器](#8-时钟与定时器)
9. [低功耗设计](#9-低功耗设计)
10. [实战：STM32 LED 闪烁（裸机）](#10-实战stm32-led-闪烁裸机)

---

## 1. 嵌入式系统概述

### 1.1 定义

嵌入式系统是以应用为中心、以计算机技术为基础，软硬件可裁剪，对功能、可靠性、成本、体积、功耗有严格约束的专用计算机系统。

### 1.2 分类

| 类别 | CPU 频率 | RAM | 典型芯片 | 应用 |
|------|---------|-----|---------|------|
| 8 位 MCU | 4~32 MHz | <1 KB | AVR, PIC | 简单控制 |
| 32 位 MCU | 32~480 MHz | 32 KB~2 MB | STM32, ESP32 | IoT, 工控 |
| 应用处理器 | GHz | MB~GB | i.MX, RPi CM | Linux 嵌入式 |
| DSP | 300~1200 MHz | — | TMS320 | 信号处理 |
| FPGA+SoC | — | — | Zynq, Cyclone | 高性能定制 |

### 1.3 与桌面开发的差异

| 方面 | 桌面开发 | 嵌入式开发 |
|------|---------|-----------|
| 内存 | GB 级 | KB~MB |
| 存储 | TB 级 | KB~MB |
| OS | Windows/Linux | 无 OS 或 RTOS |
| 调试 | 软件调试器 | JTAG/SWD 硬件调试 |
| 编译 | 本机编译 | 交叉编译 |
| 功耗 | 无限制 | μA~mA 级别 |
| 实时性 | 非实时 | 常需硬实时 |

---

## 2. MCU 架构

### 2.1 ARM Cortex-M 系列

| 内核 | 指令集 | 特性 | 典型应用 |
|------|--------|------|---------|
| Cortex-M0/M0+ | ARMv6-M | 超低功耗，最简单 | 传感器节点 |
| Cortex-M3 | ARMv7-M | 硬件乘除法 | 通用控制 |
| Cortex-M4 | ARMv7E-M | DSP 指令，可选 FPU | 电机控制 |
| Cortex-M7 | ARMv7E-M | 双发射，I/D Cache | 高性能控制 |
| Cortex-M33 | ARMv8-M | TrustZone 安全 | IoT 安全 |

### 2.2 寄存器组

ARM Cortex-M 有 16 个 32 位寄存器：

```
R0-R12   通用寄存器
R13(SP)  栈指针（MSP 主栈，PSP 进程栈）
R14(LR)  链接寄存器（保存函数返回地址）
R15(PC)  程序计数器
xPSR     程序状态寄存器（N,Z,C,V 标志位）
```

### 2.3 Thumb-2 指令集

```asm
; ARM 汇编示例（理解即可）
        PUSH    {R4-R7, LR}     ; 保存寄存器
        MOV     R4, #0          ; R4 = 0
        LDR     R5, =0x40020000 ; R5 = GPIOA 基地址
loop:
        LDR     R6, [R5, #20]   ; 读取 BSRR
        ORR     R6, #(1<<5)     ; 置位 bit 5（LED）
        STR     R6, [R5, #20]   ; 写回
        BL      delay_ms        ; 调用延时函数
        B       loop
        POP     {R4-R7, PC}     ; 恢复并返回
```

---

## 3. 存储器映射

ARM Cortex-M 使用统一地址空间（4 GB）：

```
0xFFFFFFFF ┌──────────────────────────────┐
           │ 厂商特定（Cortex-M 调试组件）   │
0xE0000000 ├──────────────────────────────┤
           │ 外设（AHB/APB Peripherals）    │  GPIO, UART, SPI 等
0x40000000 ├──────────────────────────────┤
           │ SRAM（RAM 区域）               │  全局变量、堆、栈
0x20000000 ├──────────────────────────────┤
           │ 代码区（Flash）                │  程序代码、常量
0x00000000 └──────────────────────────────┘
```

### STM32F4 具体映射示例

| 区域 | 起始地址 | 大小 | 说明 |
|------|---------|------|------|
| Flash | 0x08000000 | 1 MB | 用户代码存放 |
| SRAM | 0x20000000 | 192 KB | 变量、堆栈 |
| APB1 | 0x40000000 | 64 KB | UART, SPI, I2C |
| APB2 | 0x40010000 | 64 KB | GPIO |
| AHB1 | 0x40020000 | — | GPIO, DMA, RCC |

```c
// 直接通过地址访问寄存器
#define GPIOA_BASE   0x40020000UL
#define GPIOA_MODER  (*(volatile uint32_t*)(GPIOA_BASE + 0x00))
#define GPIOA_ODR    (*(volatile uint32_t*)(GPIOA_BASE + 0x14))

// 配置 PA5 为输出
GPIOA_MODER &= ~(3U << 10);   // 清除 bit[11:10]
GPIOA_MODER |=  (1U << 10);   // 设置为通用输出模式

// 控制 LED
GPIOA_ODR |= (1U << 5);   // 点亮
GPIOA_ODR &= ~(1U << 5);  // 熄灭
```

---

## 4. 嵌入式开发工具链

### 4.1 工具链组成

```
源代码(.c/.cpp)
    ↓ arm-none-eabi-gcc/g++（交叉编译器）
目标文件(.o)
    ↓ arm-none-eabi-ld（链接器）
ELF 文件（调试）
    ↓ arm-none-eabi-objcopy
bin/hex 文件（烧录）
```

### 4.2 安装工具链

```bash
# Ubuntu
sudo apt install gcc-arm-none-eabi binutils-arm-none-eabi

# 验证
arm-none-eabi-gcc --version
arm-none-eabi-size --help

# macOS
brew install arm-none-eabi-gcc
```

### 4.3 CMake 嵌入式配置

```cmake
# toolchain-arm-none-eabi.cmake
set(CMAKE_SYSTEM_NAME Generic)
set(CMAKE_SYSTEM_PROCESSOR arm)

set(CMAKE_C_COMPILER   arm-none-eabi-gcc)
set(CMAKE_CXX_COMPILER arm-none-eabi-g++)
set(CMAKE_ASM_COMPILER arm-none-eabi-gcc)
set(CMAKE_OBJCOPY      arm-none-eabi-objcopy)
set(CMAKE_SIZE         arm-none-eabi-size)

set(CPU_FLAGS "-mcpu=cortex-m4 -mthumb -mfpu=fpv4-sp-d16 -mfloat-abi=hard")

set(CMAKE_C_FLAGS   "${CPU_FLAGS} -Os -ffunction-sections -fdata-sections")
set(CMAKE_CXX_FLAGS "${CPU_FLAGS} -Os -ffunction-sections -fdata-sections -fno-exceptions -fno-rtti")
set(CMAKE_EXE_LINKER_FLAGS "-T${CMAKE_SOURCE_DIR}/linker.ld -Wl,--gc-sections -nostartfiles")
```

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.16)
project(firmware C CXX ASM)

set(CMAKE_TOOLCHAIN_FILE toolchain-arm-none-eabi.cmake)

add_executable(firmware.elf
    src/main.c
    src/startup_stm32f4xx.s
)

# 生成 bin 和 hex
add_custom_command(TARGET firmware.elf POST_BUILD
    COMMAND ${CMAKE_OBJCOPY} -O ihex   firmware.elf firmware.hex
    COMMAND ${CMAKE_OBJCOPY} -O binary firmware.elf firmware.bin
    COMMAND ${CMAKE_SIZE} firmware.elf
)
```

### 4.4 烧录工具

```bash
# OpenOCD + ST-Link
openocd -f interface/stlink.cfg -f target/stm32f4x.cfg \
        -c "program firmware.bin verify reset exit 0x08000000"

# STM32CubeProgrammer（图形界面）
STM32_Programmer_CLI -c port=SWD -w firmware.hex -v -rst

# J-Link
JLinkExe -device STM32F407VG -if SWD -speed 4000 -autoconnect 1
```

---

## 5. 启动流程（Boot Sequence）

```
上电/复位
    ↓
1. 从 0x00000000 读取初始 MSP（主栈指针）
2. 从 0x00000004 读取复位向量（Reset_Handler 地址）
3. 跳转到 Reset_Handler
    ↓
Reset_Handler（汇编启动代码）
    ├── 设置栈指针
    ├── 拷贝 .data 段：Flash → RAM
    ├── 清零 .bss 段
    ├── 调用系统初始化（SystemInit）
    └── 调用 main()
    ↓
main()
    ├── HAL/BSP 初始化
    ├── 外设初始化
    └── 主循环 / RTOS 启动
```

### 启动代码示例（精简版）

```c
// startup.c（精简版，实际工程用汇编）
extern uint32_t _sidata;  // .data 段在 Flash 中的起始
extern uint32_t _sdata;   // .data 段在 RAM 中的起始
extern uint32_t _edata;   // .data 段在 RAM 中的结束
extern uint32_t _sbss;    // .bss 段起始
extern uint32_t _ebss;    // .bss 段结束

void Reset_Handler(void) {
    // 1. 拷贝 .data 段
    uint32_t *src = &_sidata, *dst = &_sdata;
    while (dst < &_edata) *dst++ = *src++;

    // 2. 清零 .bss 段
    for (uint32_t *p = &_sbss; p < &_ebss; p++) *p = 0;

    // 3. 调用 main
    SystemInit();
    main();
    while (1);  // main 不应返回
}
```

---

## 6. 链接脚本（Linker Script）

```ld
/* stm32f4.ld */
MEMORY {
    FLASH (rx)  : ORIGIN = 0x08000000, LENGTH = 1024K
    RAM   (rwx) : ORIGIN = 0x20000000, LENGTH = 192K
}

SECTIONS {
    /* 代码段（存放在 Flash） */
    .text : {
        KEEP(*(.isr_vector))    /* 中断向量表必须在最前 */
        *(.text .text.*)
        *(.rodata .rodata.*)
        _sidata = .;            /* .data 在 Flash 中的位置 */
    } > FLASH

    /* 已初始化数据段（Flash 存储，RAM 运行） */
    .data : AT(_sidata) {
        _sdata = .;
        *(.data .data.*)
        _edata = .;
    } > RAM

    /* 未初始化数据段（RAM，启动时清零） */
    .bss : {
        _sbss = .;
        *(.bss .bss.*)
        *(COMMON)
        _ebss = .;
    } > RAM

    /* 堆和栈 */
    .heap : {
        _heap_start = .;
        . += 0x2000;            /* 8 KB 堆 */
        _heap_end = .;
    } > RAM

    _estack = ORIGIN(RAM) + LENGTH(RAM);  /* 栈顶 */
}
```

---

## 7. 中断系统

### 7.1 向量表

```c
// 中断向量表（部分）
typedef void (*IRQHandler)(void);

__attribute__((section(".isr_vector")))
const IRQHandler vector_table[] = {
    (IRQHandler)0x20030000,  // 初始 MSP（RAM 末尾）
    Reset_Handler,           // 0x04: 复位
    NMI_Handler,             // 0x08: 不可屏蔽中断
    HardFault_Handler,       // 0x0C: 硬件故障
    // ...更多系统中断...
    TIM2_IRQHandler,         // 外设中断
    USART1_IRQHandler,
    // ...
};
```

### 7.2 NVIC（嵌套向量中断控制器）

```c
// 使用 CMSIS
#include "stm32f4xx.h"

// 配置中断优先级和使能
NVIC_SetPriority(TIM2_IRQn, 2);  // 优先级 2
NVIC_EnableIRQ(TIM2_IRQn);

// 中断处理函数
void TIM2_IRQHandler(void) {
    if (TIM2->SR & TIM_SR_UIF) {  // 更新事件
        TIM2->SR &= ~TIM_SR_UIF;  // 清除标志
        // 处理定时器中断
        toggle_led();
    }
}
```

### 7.3 中断安全编程

```c
// 进入/退出临界区
#define ENTER_CRITICAL()  __disable_irq()
#define EXIT_CRITICAL()   __enable_irq()

// 或使用 PRIMASK 保存状态
#define CRITICAL_SECTION_ENTER(mask) \
    do { (mask) = __get_PRIMASK(); __disable_irq(); } while(0)
#define CRITICAL_SECTION_EXIT(mask) \
    do { __set_PRIMASK(mask); } while(0)

// 共享变量用 volatile 修饰
volatile uint32_t tick_count = 0;

void SysTick_Handler(void) {
    tick_count++;  // 原子写（32 位对齐）
}
```

---

## 8. 时钟与定时器

```c
// STM32 SysTick 配置（1ms 中断）
void systick_init(uint32_t cpu_freq_hz) {
    SysTick->LOAD  = cpu_freq_hz / 1000 - 1;  // 重载值
    SysTick->VAL   = 0;                         // 清除当前值
    SysTick->CTRL  = SysTick_CTRL_CLKSOURCE_Msk |
                     SysTick_CTRL_TICKINT_Msk   |
                     SysTick_CTRL_ENABLE_Msk;   // 使能
}

volatile uint32_t sys_tick_ms = 0;
void SysTick_Handler(void) { sys_tick_ms++; }

void delay_ms(uint32_t ms) {
    uint32_t start = sys_tick_ms;
    while ((sys_tick_ms - start) < ms);
}

uint32_t get_tick_ms(void) { return sys_tick_ms; }
```

---

## 9. 低功耗设计

```c
// 睡眠模式
__WFI();   // Wait For Interrupt（最浅睡眠，保留时钟）
__WFE();   // Wait For Event

// STM32 低功耗模式
HAL_PWR_EnterSLEEPMode(PWR_MAINREGULATOR_ON, PWR_SLEEPENTRY_WFI);
HAL_PWR_EnterSTOPMode(PWR_LOWPOWERREGULATOR_ON, PWR_STOPENTRY_WFI);
HAL_PWR_EnterSTANDBYMode();  // 最低功耗，几 μA

// 时钟门控（关闭不用外设的时钟）
__HAL_RCC_GPIOB_CLK_DISABLE();
__HAL_RCC_SPI1_CLK_DISABLE();
```

| 模式 | 唤醒时间 | 电流 | 保留状态 |
|------|---------|------|---------|
| Sleep | μs | ~mA | CPU 停，外设运行 |
| Stop | ms | ~μA | RAM/寄存器保留 |
| Standby | ms | ~1μA | RAM 丢失 |
| Shutdown | ms | ~10nA | 全部丢失 |

---

## 10. 实战：STM32 LED 闪烁（裸机）

```c
// main.c - 无 HAL，纯寄存器操作
#include <stdint.h>

// RCC 寄存器
#define RCC_BASE    0x40023800UL
#define RCC_AHB1ENR (*(volatile uint32_t*)(RCC_BASE + 0x30))

// GPIOA 寄存器
#define GPIOA_BASE  0x40020000UL
#define GPIOA_MODER (*(volatile uint32_t*)(GPIOA_BASE + 0x00))
#define GPIOA_ODR   (*(volatile uint32_t*)(GPIOA_BASE + 0x14))
#define GPIOA_BSRR  (*(volatile uint32_t*)(GPIOA_BASE + 0x18))

// PA5 = LED（STM32F4 Nucleo 板）
#define LED_PIN  5

volatile uint32_t tick = 0;

void SysTick_Handler(void) { tick++; }

void delay_ms(uint32_t ms) {
    uint32_t start = tick;
    while ((tick - start) < ms);
}

int main(void) {
    // 1. 使能 GPIOA 时钟
    RCC_AHB1ENR |= (1U << 0);

    // 2. 配置 PA5 为输出模式
    GPIOA_MODER &= ~(3U << (LED_PIN * 2));
    GPIOA_MODER |=  (1U << (LED_PIN * 2));

    // 3. 配置 SysTick（168MHz 系统时钟，1ms 中断）
    SysTick->LOAD = 168000 - 1;
    SysTick->VAL  = 0;
    SysTick->CTRL = 7;  // 使能 + 时钟 + 中断

    while (1) {
        GPIOA_BSRR = (1U << LED_PIN);          // 置高（LED 亮）
        delay_ms(500);
        GPIOA_BSRR = (1U << (LED_PIN + 16));   // 置低（LED 灭）
        delay_ms(500);
    }
}
```

```bash
# 编译
arm-none-eabi-gcc -mcpu=cortex-m4 -mthumb -mfpu=fpv4-sp-d16 \
    -mfloat-abi=hard -Os -nostartfiles \
    -T stm32f4.ld startup.s main.c \
    -o firmware.elf

# 查看大小
arm-none-eabi-size firmware.elf

# 烧录
openocd -f interface/stlink.cfg -f target/stm32f4x.cfg \
    -c "program firmware.elf verify reset exit"
```

---

**上一章**：[第 05 章：现代 C++ ←](05_cpp_modern.md)  
**下一章**：[第 07 章：嵌入式硬件与外设编程 →](07_embedded_hardware.md)
