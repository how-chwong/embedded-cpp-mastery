# .NET 开发者转型 Python 极速通关指南

欢迎来到 Python 的世界！本指南专为掌握 C# 且拥有 .NET 平台开发经验的工程师定制。我们不从零讲解什么是“变量”或“循环”，而是通过 C# 知识作为参照系（"By Analogy"），帮助你快速建立 Python 的心智模型。

## 核心心智模型对比：C# vs Python

| 维度         | C# / .NET                                             | Python                                                                     |
| :----------- | :---------------------------------------------------- | :------------------------------------------------------------------------- |
| **执行模型** | 编译为 IL，由 CLR (JIT) 编译并执行。                  | 解释型语言，由 CPython 虚拟机逐行解释字节码（ bytecode ）。                |
| **类型系统** | 静态强类型（Static & Strong），主要在编译期安全检查。 | 动态强类型（Dynamic & Strong），运行时检查类型，支持 Type Hints 静态分析。 |
| **平台管理** | .NET SDK, NuGet, 构建为 `.dll`/`.exe`。               | Python 解释器, Pip/Poetry/Conda, 运行源文件或 `.pyc` 字节码。              |
| **包隔离**   | 依靠不同的 Project / Solution 和 bin 目录独立依赖。   | 依靠 **虚拟环境 (Virtual Environment - venv)** 隔离不同项目的全局依赖。    |
| **核心哲学** | 严格、严谨、多范式，依赖接口 (Interface) 和显式契约。 | 优美、简洁、鸭子类型 (Duck Typing)，“允许错误胜于过度防范”。               |

---

## 教程章节目录

为了便于你循序渐进地学习，整个知识路线分为以下章节（从基础工具搭建直到生产部署）。每一章都包含深度的 C# 对比与实例代码。看完它们，你将能够无缝开发 Python 应用程序：

0. **[第0章：极简环境与配置起步](ch0_setup.md)**
   - 安装 Python、IDE 插件全套推荐（VS Code/Pylance/Ruff）、本地开发唯一的刚需：“虚拟环境 (venv)”、编辑器与虚拟环境的打通对接、Hello World 运行机制、以及 F5 极速调试教程。
1. **[第1章：起步与类型系统](ch1_basics.md)**
   - 动静态类型本质、基本数据类型、强类型系统特征、常用的四大容器、可变 (Mutable) 与不可变 (Immutable) 的陷阱。
2. **[第2章：控制流与异常处理](ch2_control_flow.md)**
   - 条件判断、`match-case`（对标 C# Pattern Matching）、三种循环机制、`try-except-finally` 的微妙差异。
3. **[第3章：函数式特性与面向对象](ch3_oop.md)**
   - 独立函数、默认参数、参数包（`*args`, `**kwargs`）、类与实例、构造器、Python 特有的 `self`、继承、鸭子类型与抽象基类。
4. **[第4章：高级核心特性](ch4_advanced.md)**
   - 列表/字典推导式（Python 的 “LINQ” 糖）、迭代器与生成器（`yield return` 的等价物）、装饰器（面向切面/控制反转的 Python 最优雅实现）、魔法方法（运算符重载与特殊属性）。
5. **[第5章：包管理与工程实践](ch5_ecosystem.md)**
   - 项目配置对比（`.csproj` vs `pyproject.toml`）、包管理工具对比（NuGet vs `pip`/`Poetry`）、虚拟环境 `venv` 精要、面向企业级开发的代码规约与 Linter。
6. **[第6章：生态对应与实战](ch6_practice.md)**
   - .NET 生态库在 Python 中的对应物清单、FastAPI 对比 ASP.NET Core Mini API、SQLAlchemy 对比 Entity Framework Core、提供一个完整的极简 Web + 数据库 CRUD 示例。
7. **[第7章：AI 与 机器学习入门实战](ch7_ml.md)**
   - AI 底层粘合胶水揭秘、机器学习经典五步大法、使用 `scikit-learn` 决策树智能预测鸢尾花分类、利用 `YOLOv8` 目标检测模型10行代码实现实时图像识别。

---

## 准备好开始了吗？
我们建议你跟随章节编写测试代码。在 Python 中，不需要复杂的编译与运行步骤，通常只需运行：
```bash
python your_script.py
```
现在，让我们先进入 **[第0章：极简开箱指南（开发环境与工具链）](ch0_setup.md)** 来搭建你的第一盘沙箱，然后开启神奇的 Python 旅程！
