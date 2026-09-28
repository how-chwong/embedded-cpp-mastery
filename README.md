# 嵌入式 C/C++ 全栈精通手册

> **目标**：零基础读者按章节实操后，达到 5 年以上 C / C++ / 嵌入式开发工作经验水平。

---

## 目录总览

| 章节 | 标题 | 难度 |
|------|------|------|
| [第 01 章](docs/01_c_fundamentals.md) | C 语言基础 | ⭐ |
| [第 02 章](docs/02_c_advanced.md) | C 语言进阶 | ⭐⭐ |
| [第 03 章](docs/03_cpp_fundamentals.md) | C++ 基础 | ⭐⭐ |
| [第 04 章](docs/04_cpp_advanced.md) | C++ 进阶（OOP / 模板 / STL） | ⭐⭐⭐ |
| [第 05 章](docs/05_cpp_modern.md) | 现代 C++（C++11/14/17/20） | ⭐⭐⭐ |
| [第 06 章](docs/06_embedded_fundamentals.md) | 嵌入式系统基础 | ⭐⭐ |
| [第 07 章](docs/07_embedded_hardware.md) | 嵌入式硬件与外设编程 | ⭐⭐⭐ |
| [第 08 章](docs/08_rtos.md) | RTOS 与并发编程 | ⭐⭐⭐⭐ |
| [第 09 章](docs/09_debug_test_perf.md) | 调试、测试与性能优化 | ⭐⭐⭐ |
| [第 10 章](docs/10_production.md) | 生产环境落地方案 | ⭐⭐⭐⭐ |
| [附录 A](docs/appendix_a_comparison.md) | C / C++ / C# 横向对比 | — |
| [附录 B](docs/appendix_b_tools.md) | 工具链速查手册 | — |

---

## 如何使用本手册

1. **零基础**：从第 01 章顺序阅读，每节末尾均有动手实验。
2. **有 C 基础**：可跳至第 03 章，或直接阅读嵌入式篇（第 06 章）。
3. **跨语言开发者**：先阅读附录 A 了解语言差异，再按需深入。
4. **代码示例**：所有示例均可在 Linux / macOS / Windows（WSL）上编译运行；嵌入式示例使用 STM32（ARM Cortex-M）为目标板。

---

## 开发环境快速启动

```bash
# Ubuntu / Debian
sudo apt update
sudo apt install -y gcc g++ cmake make gdb \
    gcc-arm-none-eabi binutils-arm-none-eabi \
    openocd minicom git

# 验证
gcc --version
arm-none-eabi-gcc --version
openocd --version
```

---

*本手册持续更新，欢迎 PR 贡献。*
