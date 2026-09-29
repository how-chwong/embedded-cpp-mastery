# Lua 极速升级通关指南

欢迎来到 Lua 的世界！
作为一名熟练掌握 C# 和 .NET 的工程师，你可能在以下地方听说过 Lua 或是与它擦肩而过：
1. **游戏开发（重点）**：Unity 的热更新方案（如 xLua、toLua），绝大部分游戏逻辑都是用 Lua 实现的，以绕过 iOS/Android 只能运行 JIT 或非编译代码的阻断。
2. **中间件扩展**：网关核心 Nginx / OpenResty 中使用 Lua 编写高性能 WAF 防火墙（或流控防火墙）。
3. **数据库自定操作**：Redis 中通过内置的 Lua 脚本（如 `EVAL`）来保证多个命令的强原子性操作。

本指南通过将 Lua 这个极致小巧的**轻量级脚本语言**与 C# 进行全方位对比（By Analogy），帮助你瞬间消除跨界壁垒。

---

## 🌓 核心心智模型对比：C# vs Lua

| 维度         | C# / .NET                                           | Lua                                                                                                     |
| :----------- | :-------------------------------------------------- | :------------------------------------------------------------------------------------------------------ |
| **设计核心** | 重量级、高静态工程化、现代多范式大厦。              | 极致轻量、极简规则、纯胶水粘合、适合作为嵌入宿主。                                                      |
| **执行模型** | C# -> IL -> CLR JIT / NativeAOT 机器码。            | Lua 源码 -> 虚机字节码（由 C 编写的解释器解释，或 Luajit 高速 JIT 编译）。                              |
| **类型系统** | 强类型、静态检查。拥有全套值类型/引用类型。         | 弱类型（动态弱类型）、运行时转换、极其简约共 8 种基础基本类型。                                         |
| **数据集合** | `List<T>`, `Dictionary<TKey, TValue>`, `Array` 等。 | **仅有唯一一个复合数据结构**：**Table (表)**。它既是数组，又是 Dictionary，甚至是 Class，还是 Package。 |
| **核心机制** | 继承、接口（Interface）、虚方法表。                 | **元表（Metatable）与元方法（Metamethod）**，一切扩展皆由它演生。                                       |

---

## 📖 章节导航

请按照以下章节顺序展开研读：

0. **[lua/ch0_setup.md](lua/ch0_setup.md)**：极简开箱与开发环境搭建。
   - Lua 版本选择、依赖隔离、第一个 Lua Hello World 以及在 VS Code 下通过 F5 极速调试 Lua 项目。
1. **[lua/ch1_basics.md](lua/ch1_basics.md)**：基本语法及单数据结构 Table 与元表。
   - 数字字符运算、全能的 Table、通过元表（Metatables）重载运算符与 C# 对标。
2. **[lua/ch2_oop.md](lua/ch2_oop.md)**：基于原型链的面向对象（OOP）。
   - C# 经典的成员变量与函数如何用 Lua 翻译、原型字典共享。
3. **[lua/ch3_advanced.md](lua/ch3_advanced.md)**：模块化、协同程序。
   - 包导入对标、Coroutines（协程）对标 C# `yield return` 的底座原理。
4. **[lua/ch4_busted_nlua.md](lua/ch4_busted_nlua.md)**：测试及 C# 宿主互操作。
   - 在 C# WinForms/Console 或 ASP.NET 中如何直接加载、修改、调用 Lua 函数，以及 Busted 测试框架的简易引导。

---

现在，让我们从最首要的开发沙箱准备开始，点击阅读 **[lua/ch0_setup.md](lua/ch0_setup.md)**！
