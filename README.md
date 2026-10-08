# Embedded C/C++ Mastery & 技术学习导航

> 本仓库既是嵌入式 C/C++ 全栈学习手册，也是多个技术专题的实战型学习仓库。适合按主题深入学习，并通过章节式实践快速建立工程能力。

---

## 项目概览

这个仓库目前包含两类核心内容：

1. 嵌入式与 C/C++ 体系的系统学习路线：位于 `docs/`。
2. 其他主流技术栈的快速入门与实战路径：位于 `python/`、`lua/`、`threejs/`、`elasticsearch/`、`pytorch/`。

你可以按自己的学习目标选择合适的路径：

- 想建立 C / C++ / 嵌入式工程能力：从 `docs/` 开始。
- 想快速掌握 Python：参考 `python/`。
- 想理解 Lua、脚本语言与宿主互操作：参考 `lua/`。
- 想学习 WebGL / Three.js：参考 `threejs/`。
- 想了解搜索引擎：参考 `elasticsearch/`。
- 想入门深度学习与 PyTorch：参考 `pytorch/`。

---

## 核心学习路线

### 1) 嵌入式 C/C++ 全栈精通手册

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

### 2) 其他专题

| 路径 | 主题 | 入口 |
|------|------|------|
| [python/](python/) | Python 与工程实践 | [python/README.md](python/README.md) |
| [lua/](lua/) | Lua 速成与宿主互操作 | [lua/README.md](lua/README.md) |
| [threejs/](threejs/) | Three.js 3D / WebGL 学习 | [threejs/README.md](threejs/README.md) |
| [elasticsearch/](elasticsearch/) | Elasticsearch 与搜索引擎 | [elasticsearch/README.md](elasticsearch/README.md) |
| [pytorch/](pytorch/) | PyTorch 与深度学习入门 | [pytorch/README.md](pytorch/README.md) |
| [vue/](vue/) | Vue 3 + TypeScript 前端全栈 | [vue/README.md](vue/README.md) |

---

## 推荐学习顺序

1. **零基础**：从 `docs/01_c_fundamentals.md` 开始，按顺序阅读并完成每章的代码练习。
2. **有 C 基础**：可以跳到 `docs/03_cpp_fundamentals.md`，或直接阅读嵌入式篇 `docs/06_embedded_fundamentals.md`。
3. **跨语言开发者**：先阅读附录 A 了解 C / C++ / C# 差异，再按需选读专题目录。
4. **专项提升**：若你专注于某个方向，可直接进入对应目录，如 Python、Lua、Three.js 或 PyTorch。

---

## 开发环境说明

### C / C++ / 嵌入式环境

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

### 通用建议

- Linux / macOS / WSL 下均可进行主线代码实验。
- 嵌入式章节默认以 STM32（ARM Cortex-M）开发板为目标平台。
- 若使用 Python / Lua / JavaScript 等脚本类专题，可按各目录的 README 进行环境搭建。

---

## 贡献方式

本仓库适合长期维护和迭代更新，欢迎通过 PR 或补充案例来完善内容。

---

> 这是一个以学习、实践和知识沉淀为主的技术仓库，重点在于搭建系统化认知和工程化实践能力。
