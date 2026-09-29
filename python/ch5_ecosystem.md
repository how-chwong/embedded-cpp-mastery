# 第5章：包管理与工程实践

在 C# / .NET 中，我们日常工作都伴随着 `dotnet build`、`.csproj` 的依赖声明、以及用 NuGet 下载公共库。
到了 Python 领域，包管理、打包模式、依赖隔离也有完全对应的一套体系。本章将帮助你快速掌握这些工程实践，避开常见的包版本冲突问题。

---

## 1. 核心概念：为什么必须引入 “虚拟环境 (venv)”？

作为 .NET 开发者，你需要知道：
- 在 C# 中，NuGet 将包下载到全局缓存。但是在构建项目时，每一个项目的依赖都会被隔离输出在自己项目根目录下的 `bin/[Debug|Release]/` 文件夹中。因此不同的 C# 项目依赖不同版本的类库一般不会相互影响。
- 但在 Python 中，如果默认全局使用 `pip install` 安装类库，它会安装在系统级的 `/site-packages` 全局目录下。**这意味着当你有两个项目分别依赖 `SqlAlchemy 1.4` 和 `SqlAlchemy 2.0` 时，系统内后安装的版本会直接覆盖先前的版本，造成整个开发环境崩溃！**

为了解决这一问题，Python 提出了 **虚拟环境 (Virtual Environment)** 的概念。

### 什么是虚拟环境？
虚拟环境（一般命名为 `.venv` 或是 `venv`）本质上是在你当前**项目的根目录下，克隆一个极轻量的 Python 纯物理程序副本**。在这个文件夹内通过 `pip` 安装的所有包，都会留在该项目局部里，彻底不与系统全局和其他项目发生冲突。

### 虚拟环境实战命令清单 (Windows Power-Shell / command 必备)：

```bash
# 1. 在项目目录下创建一个叫 .venv 的虚拟环境副本
python -m venv .venv

# 2. 激活虚拟环境 (这一步将在你的终端前加上 (.venv) 的字样，表示后续所有的 Python 资源和命令都属于这个局部环境)
# 终端：Windows PowerShell：
.\.venv\Scripts\Activate.ps1

# 3. 退出虚拟环境（要退回系统默认环境时直接在控制台敲以下命令即可）
deactivate
```

---

## 2. 工程管理文件对比表

| .NET 规范/工具                     | Python 对应 (经典)           | Python 对应 (现代首选)       | 说明与对比                           |
| :--------------------------------- | :--------------------------- | :--------------------------- | :----------------------------------- |
| `*.csproj`                         | `requirements.txt`           | `pyproject.toml`             | 依赖、元数据、工具版本的声明卡片。   |
| **NuGet**                          | `pip`                        | `Poetry` / `pipenv`          | 包的分发与获取工具。                 |
| [nuget.org](https://www.nuget.org) | [pypi.org](https://pypi.org) | [pypi.org](https://pypi.org) | 全球最大公共依赖包共享中心（PyPI）。 |

### A. 经典工作流：`requirements.txt`
这是很多老旧项目和传统算法项目依然在采用的最简单模式：其纯粹是一份写满包名和版本的“配方列表”。

```text
# requirements.txt 内容：
fastapi==0.110.0
sqlalchemy>=2.0.0
pydantic>=2.5.0
```

安装运行：
```bash
# 从配方清单一键下载安装
pip install -r requirements.txt

# 将你当前项目里本地测试出来的所有包装入 requirements.txt 以便交付他人：
pip freeze > requirements.txt
```

### B. 现代标准：`pyproject.toml` 和 Poetry
自 PEP 518 规范发布后，现代 Python 项目逐渐演进为使用 **`pyproject.toml`** 单一配置文件（类似于 `.csproj` 结合了 `nuget.config` 和 `editorconfig`）。
在诸多包依赖解决方案中，**Poetry** 或 **Ruff**（见下文）已经是现代 Python 高质量团队的御用主力工具。

---

## 3. 企业代码规约与质量守卫

在 .NET 平台，由于 C# 本身是严格的强静态编译型，其编译器 Roslyn、`.editorconfig` 和 MSBuild 分析器的存在能够过滤 90% 以上的代码笔误，并强力推行全平台一致的格式化规范。

在 Python 动态语言里，格式化和 Linter 的作用是高精防护。

### A. 核心代码规范：PEP 8
[PEP 8](https://peps.python.org/pep-0008/) 是 Python 社区官方制定的代码风格指南。它硬性统一了类名用大驼峰（`CamelCase`）、函数/变量用小蛇形（`snake_case`）、常量用大写、最大行宽限制等一系列标准。

### B. 代码格式化工匠：Black
[Black](https://github.com/psf/black) 是目前 Python 社区首屈一指的自动代码美化控制工具（人称“铁面无私格式化器”）。它不给你任何配置妥协的选择，只奉行一种代码美学。
- C# 拥有 IDE 的 `Ctrl+K, Ctrl+D`
- Python 可以直接通过 pip 安装 `black`：
  ```bash
  pip install black
  black YourProjectFolder/ # 瞬间自动纠正所有的缩进、单双引号使用、行尾等
  ```

### C. 最强性能合并：Ruff (现代首选利器)
[Ruff](https://github.com/astral-sh/ruff) 是近一两年异军突起的一个用 **Rust** 编写的 Python Linter & Formatter。它完美合并了以前经典但极慢的工具集（如 `flake8`, `isort`, `autoflake`, `pyupgrade` ），它的执行速度比原先提升了 **几十到上百倍**（一眨眼就能秒级分析完毕数万行大型项目）。

只需安装并在项目根目录下做如下简单的 `pyproject.toml` 配置：
```toml
# pyproject.toml 示例：
[tool.ruff]
line-length = 88 # 设定最大宽
select = ["E", "F", "I"] # 检查基础拼写错误、无效引用 import、未定义变量等
```

---

有了这个包隔离、工程标准和 Linter 的武器，你已彻底具备开发一个专业 Python 项目的后台实力。点击 **[第6章：生态对应与实战](ch6_practice.md)**，我们将直接呈上一个大型系统的“生还沙盒”——一个 FastAPI 配合 SQLAlchemy 数据库的经典工程实例！
