# 附录 B：工具链速查手册

---

## 1. GCC/G++ 常用选项

```bash
# 编译 C
gcc -std=c11 -Wall -Wextra -Wpedantic -O2 -g main.c -o app

# 编译 C++
g++ -std=c++17 -Wall -Wextra -O2 -g main.cpp -o app

# 交叉编译 ARM
arm-none-eabi-gcc -mcpu=cortex-m4 -mthumb -mfpu=fpv4-sp-d16 \
    -mfloat-abi=hard -Os -ffunction-sections -fdata-sections \
    -fno-exceptions -fno-rtti -nostartfiles \
    -T stm32f4.ld startup.s main.c -o firmware.elf \
    -Wl,--gc-sections

# 常用优化级别
-O0  不优化（调试）
-O1  基本优化
-O2  推荐（性能/大小平衡）
-O3  激进优化（可能增大代码）
-Os  代码大小优化（嵌入式推荐）
-Og  调试友好优化

# 生成汇编
gcc -S -O2 -fverbose-asm main.c -o main.s

# 预处理只展开宏
gcc -E main.c -o main.i

# 静态分析
gcc -fanalyzer main.c
```

---

## 2. CMake 速查

```cmake
# 最小 CMakeLists.txt
cmake_minimum_required(VERSION 3.16)
project(MyApp LANGUAGES C CXX)
set(CMAKE_C_STANDARD 11)
set(CMAKE_CXX_STANDARD 17)
add_executable(app main.cpp)

# 常用命令
target_include_directories(app PRIVATE include/)
target_compile_options(app PRIVATE -Wall -Wextra)
target_link_libraries(app PRIVATE pthread m)
target_compile_definitions(app PRIVATE DEBUG=1 VERSION="1.0")

# 构建类型
cmake -B build -DCMAKE_BUILD_TYPE=Release
cmake -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build -j$(nproc)

# 安装
cmake --install build --prefix /usr/local
```

---

## 3. GDB 速查

```bash
# 启动
gdb ./app
gdb -tui ./app                   # TUI 模式
arm-none-eabi-gdb firmware.elf   # 嵌入式

# 连接远程目标
target remote :3333              # OpenOCD
target extended-remote :3333

# 断点
b main                           # 函数断点
b file.c:42                      # 行断点
b *0x080012A4                    # 地址断点
watch var                        # 数据观察点
rwatch var                       # 读观察点
info breakpoints                 # 列出断点
delete 1                         # 删除断点 1

# 执行
run [args]   r                   # 运行
continue     c                   # 继续
step         s                   # 步入
next         n                   # 步过
finish                           # 运行到返回
until 50                         # 运行到行 50
stepi        si                  # 汇编步入
nexti        ni                  # 汇编步过

# 查看
print var    p var               # 打印变量
p *ptr                           # 解引用
p arr[0]@10                      # 数组前 10 个
display var                      # 每步自动显示
info registers                   # 所有寄存器
info locals                      # 局部变量
backtrace    bt                  # 调用栈
frame 2      f 2                 # 切换栈帧
x/10xw 0x20000000                # 内存（10 个 32 位字，十六进制）
x/s 0x08000100                   # 内存作为字符串

# 修改
set var = 42                     # 修改变量
set {int}0x20001000 = 42        # 修改内存

# 其他
monitor reset halt               # OpenOCD 复位
monitor flash write_image ...    # OpenOCD 烧录
shell arm-none-eabi-size app.elf # 执行 shell 命令
quit         q                   # 退出
```

---

## 4. Valgrind / Sanitizers

```bash
# Valgrind（Linux）
valgrind ./app                                  # 基本检查
valgrind --leak-check=full --show-leak-kinds=all ./app
valgrind --tool=callgrind ./app                 # 性能分析
callgrind_annotate callgrind.out.*              # 分析结果

# AddressSanitizer（更快，推荐）
gcc -fsanitize=address,undefined -g -O1 main.c -o app
./app

# ThreadSanitizer（线程安全检查）
gcc -fsanitize=thread -g -O1 main.c -o app -pthread
./app

# UBSanitizer（未定义行为）
gcc -fsanitize=undefined -g main.c -o app
```

---

## 5. OpenOCD 速查

```bash
# 常用目标配置
openocd -f interface/stlink.cfg      -f target/stm32f4x.cfg    # STM32F4 + ST-Link
openocd -f interface/stlink-v2.cfg   -f target/stm32l4x.cfg    # STM32L4
openocd -f interface/jlink.cfg       -f target/stm32h7x.cfg    # STM32H7 + J-Link
openocd -f interface/cmsis-dap.cfg   -f target/nrf52.cfg        # nRF52 + CMSIS-DAP

# 常用命令（telnet localhost 4444）
reset halt                           # 复位并暂停
reset run                            # 复位并运行
halt                                 # 暂停
resume                               # 继续
step                                 # 单步
flash write_image erase firmware.bin 0x08000000  # 烧录
flash verify_image firmware.bin 0x08000000       # 验证
flash erase_sector 0 0 7            # 擦除扇区 0-7

# 一行烧录命令
openocd -f interface/stlink.cfg -f target/stm32f4x.cfg \
    -c "program firmware.elf verify reset exit"
```

---

## 6. arm-none-eabi 工具集

```bash
# 固件信息
arm-none-eabi-size firmware.elf           # 段大小
arm-none-eabi-size -A firmware.elf        # 详细
arm-none-eabi-nm --size-sort firmware.elf # 符号表（按大小）
arm-none-eabi-objdump -d firmware.elf     # 反汇编
arm-none-eabi-objdump -D firmware.elf | grep -A5 "main:"  # 查看 main 函数汇编
arm-none-eabi-readelf -h firmware.elf     # ELF 头信息

# 格式转换
arm-none-eabi-objcopy -O ihex   firmware.elf firmware.hex
arm-none-eabi-objcopy -O binary firmware.elf firmware.bin
arm-none-eabi-objcopy -O srec   firmware.elf firmware.srec

# 从 PC 地址定位源码行
arm-none-eabi-addr2line -e firmware.elf -a 0x080012A4 -f -p
```

---

## 7. 静态分析工具

```bash
# Cppcheck
cppcheck --enable=all --error-exitcode=1 \
         --suppress=missingIncludeSystem \
         -I include/ src/

# clang-tidy
clang-tidy src/*.cpp -- -std=c++17 -I include/

# PC-lint / FlexeLint（商业，MISRA 检查）

# PVS-Studio（商业）
pvs-studio-analyzer analyze -o pvs.log
plog-converter -a GA:1,2 -t errorfile pvs.log
```

---

## 8. 性能分析工具

```bash
# Linux perf
perf record -g ./app              # 采样
perf report                       # 查看报告
perf stat ./app                   # 统计硬件计数器

# gprof（编译时插桩）
gcc -pg -g main.c -o app
./app                             # 运行生成 gmon.out
gprof app gmon.out | head -30

# Heaptrack（堆内存分析）
heaptrack ./app
heaptrack_gui heaptrack.app.*.gz

# Valgrind Massif（堆增长分析）
valgrind --tool=massif ./app
ms_print massif.out.* | head -50
```

---

## 9. 常用开发板与平台

| 板卡 | MCU | Flash | RAM | 调试接口 | 推荐场景 |
|------|-----|-------|-----|---------|---------|
| STM32F4-Nucleo | STM32F411 | 512KB | 128KB | ST-Link | 入门学习 |
| STM32H7-Nucleo | STM32H743 | 2MB | 1MB | ST-Link | 高性能 |
| STM32F103 Blue Pill | STM32F103 | 64KB | 20KB | JTAG/SWD | 低成本 |
| Arduino Due | SAM3X8E | 512KB | 96KB | JTAG | 原型 |
| ESP32-DevKitC | ESP32 | 4MB | 520KB | USB | WiFi/BT IoT |
| Raspberry Pi Pico | RP2040 | 2MB | 264KB | SWD | 教学 |
| Nordic nRF52840-DK | nRF52840 | 1MB | 256KB | J-Link | BLE IoT |

---

## 10. 推荐学习资源

### 书籍

| 书名 | 适合阶段 | 语言 |
|------|---------|------|
| 《C 程序设计语言》（K&R） | C 基础 | C |
| 《C 和指针》 | C 进阶 | C |
| 《C 专家编程》 | C 高级 | C |
| 《C++ Primer》 | C++ 基础 | C++ |
| 《Effective C++》 | C++ 进阶 | C++ |
| 《Effective Modern C++》 | 现代 C++ | C++ |
| 《嵌入式系统设计》 | 嵌入式 | 通用 |
| 《Making Embedded Systems》 | 嵌入式实践 | C |

### 在线资源

| 资源 | 说明 |
|------|------|
| [cppreference.com](https://cppreference.com) | C/C++ 标准库参考 |
| [Godbolt Compiler Explorer](https://godbolt.org) | 在线查看汇编输出 |
| [Quick Bench](https://quick-bench.com) | 在线性能对比 |
| [STM32CubeIDE](https://st.com/stm32cubeide) | ST 官方 IDE |
| [FreeRTOS 文档](https://freertos.org/Documentation) | RTOS 官方文档 |
| [ARM Cortex-M TRM](https://developer.arm.com) | ARM 技术参考手册 |

---

**返回目录**：[README →](../README.md)
