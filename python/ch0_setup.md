# 第0章：极简开箱指南——开发环境、工具与环境管理

如果你是第一次接触 Python，不用担心！对于已经有 .NET 经验的你，搭建 Python 的开发环境实际上比安装重量级的 Visual Studio 2022 还要轻量和快速。本章将一步步带你从零组装一个完美的 Python 生产环境，涵盖安装、IDE 配置、环境管理、以及如何运行与调试你的第一个程序。

---

## 1. 安装 Python 核心程序

Python 是一门解释型语言，你需要下载并安装 Python 解释器（CPython）来执行代码。

### 各平台安装指南
我们推荐使用 **Python 3.10 或更新版本**（如 3.11 或 3.12 最佳）。

#### Windows（最常见平台）
1. 访问 Python 官方下载页面：[https://www.python.org/downloads/windows/](https://www.python.org/downloads/windows/)。
2. 下载稳定版的 **Windows installer (64-bit)**。
3. 双击打开安装包，**【极其重要：黄金避坑第一步】**：
   - 务必勾选最下方的 **"Add python.exe to PATH"**（将 Python 添加至环境变量）。
   - 如果不勾选，你在终端里输入 `python` 命令将会报错“找不到该程序”。
4. 点击 **"Install Now"**，等待安装完成。
5. （可选）安装完成后，点击出现的 **"Disable path length limit"**（解除 Windows 路径最大 260 字符的限制，能避免复杂的嵌套项目路径报错）。

#### macOS
* 推荐使用 macOS 包管理器 Homebrew 安装，打开终端执行：
  ```bash
  brew install python
  ```

#### Linux (Ubuntu/Debian)
* 打开终端，执行：
  ```bash
  sudo apt update
  sudo apt install python3 python3-pip python3-venv
  ```

### 验证安装
打开一个新的命令行终端（PowerShell 或 CMD），输入以下命令并回车：
```bash
python --version
```
如果输出显示类似 `Python 3.11.x` 类似的版本，说明 Python 已经成功入驻你的电脑！

---

## 2. 打造你的金牌开发工具：VS Code (或 Visual Studio)

作为 .NET 工程师，你熟悉 Visual Studio 的强大。但在 Python 领域，全球 70% 以上的开发者都在使用极其轻盈而扩展丰富的 **Visual Studio Code (VS Code)**。

### A. 全套开发插件推荐
首先安装 VS Code 之后，点击左侧导航栏的扩展市场（Extensions，快捷键 `Ctrl+Shift+X`），搜索并安装以下三个神器插件：

1. **Python** (Publisher: Microsoft)：官方提供的最核心插件，提供调试、运行、代码重构、代码跳转等全套支持。
2. **Pylance** (Publisher: Microsoft)：官方强力推荐的语言服务器（Language Server），不仅速度极快，还提供了优秀的 **Type Hints（类型提示）** 自动补全。
3. **Ruff** (Publisher: Astral Software)：极快、秒级的代码格式化和规范检测插件（对标 .NET 里面的 Roslyn 语法分析与格式化）。

---

## 3. Python 唯一的刚需：环境管理 (venv) 与 VS Code 关联

在 C# 中，不同项目的 NuGet 依赖关系被隔离在每个项目的 `bin/` 目录下。
但在 Python 中，如果不做任何隔离，所有的包都会被安装在系统全局里，这会导致多个项目共用包时发生灾难性的“版本覆灭”。

因此，Python 引入了 **“虚拟环境 (venv)”**。**请记住这条黄金法则：在 Python 中，永远不要在系统全局下直接使用 `pip install`！每一个新项目都必须建立自己的虚拟环境。**

### 第一步：新建项目目录并创建虚拟环境
1. 在你的电脑任意位置，新建一个干净的文件夹，例如命名为 `my_first_python`。
2. 在 VS Code 中打开该文件夹：`File -> Open Folder...`
3. 使用快捷键 <code>Ctrl + ` </code>（反引号，通常在 Esc 下方）在 VS Code 底部打开内置终端（默认通常为 PowerShell）。
4. 在终端中，输入并运行以下命令，这将在你当前的项目文件夹下，自动新建一个名为 `.venv` 的隔离文件夹：
   ```powershell
   # Windows 平台：
   python -m venv .venv
   ```

### 第二步：激活虚拟环境
为了让命令知道后续的操作都是在这个局部沙盒中，运行以下命令（请根据你手头的终端选择）：

* **PowerShell (VS Code 默认)**：
  ```powershell
  .\.venv\Scripts\Activate.ps1
  ```
  *(如果 Windows 提示权限受限禁止脚本执行，可先执行 `Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope Process` 允许当前会话激活脚本)*
* **Windows CMD (经典命令行)**：
  ```cmd
  .\.venv\Scripts\activate.bat
  ```
* **Git Bash / macOS / Linux**：
  ```bash
  source .venv/bin/activate
  ```

**激活成功的重要标志**：你的命令行终端路径前部，会出现一个圆括号包裹的标记 **`(.venv)`**，这表明所有命令均已安全隔离。

### 第三步：【小白极易迷失】在 VS Code 中关联虚拟环境
仅仅在终端里激活是不够的，你必须告诉 VS Code 的代码编辑器（Pylance 语言服务器）该从哪里去寻找代码高亮和跳转提示。

1. 在 VS Code 顶端菜单栏点击并运行快捷键：`Ctrl + Shift + P`（打开命令面板）。
2. 输入并搜索：`Python: Select Interpreter`。
3. 在弹出的列表中，VS Code 会智能扫描并标出带有 **"('venv':  .venv)"** 或 **"./.venv/Scripts/python.exe"** 的选项，点击选中它！
4. 此时，你看向 VS Code 右下角的状态栏，会看到显示有类似 `3.11.x (.venv)` 的字样。恭喜！你的编辑器和后台环境已经天衣无缝地对接好了，所有的代码跳转和高亮瞬间被彻底点亮。

---

## 4. 实战演示：编写并运行第一行代码

现在，万事俱备，让我们像真正的程序员一样，通过文件执行。

### 1. 新建并编写 `app.py`
在 VS Code 侧边栏的文件浏览器中，新建一个文件并命名为 **`app.py`**（注意后缀为 `.py`）。
放入以下最简单的变量声明和计算代码：

```python
# 声明一个打招呼函数
def say_hello(name: str):
    message = f"Hello, {name}! Welcome to Python."
    print(message)

# 执行并测试该函数
my_name = "DotNet-Developer"
say_hello(my_name)
```

### 2. 运行代码
运行代码有两种最简单的方式：

* **方式 A：终端执行（原汁原味）**
  确保你在终端中已经进入了当前目录，并且虚拟环境是 `(.venv)` 激活状态下。直接输入并运行：
  ```bash
  python app.py
  ```
* **方式 B：VS Code 一键播放**
  看下你 VS Code 的编辑器右上角，由于安装了 Python 插件，会有一个类似播放磁带的“三角播放图标”：
  - 点击那个播放图标，VS Code 会帮你自动组装指令、并在底部终端执行完毕输出 `Hello, DotNet-Developer! Welcome to Python.`。

---

## 5. C# 调试经验平替：在 VS Code 中进行 Debug

作为高级程序员，Debug 是我们必不可少的手段。Python 在 VS Code 里的调试体验几乎与 Visual Studio 无出其右。

### 如何像 C# 程序员一样 Debug：
1. **加断点 (Set Breakpoints)**：将你的鼠标指针悬停在 `app.py` 的第 3 行，即 `message = f"Hello..."` 的左侧。你会看到一个若隐若现的红点，点击它，你会发现第 3 行左侧出现了一个**红色的实心圆点** —— 这就是断点。
2. **启动调试 (F5)**：按下键盘上的 **F5**。或者在左侧侧边栏中点击 `Run and Debug` 按钮（第三个看起来带甲壳虫加播放标志的图标），然后点击 **"Run and Debug"** -> 在顶部选择 **"Python File"**（调试当前活动文件）。
3. **触屏与捕获**：
   - 调试器将瞬间启动并在你标红的第 3 行停下来。这非常像 C# 中的调试时黄条高亮。
   - 在左侧的 `VARIABLES`（变量）窗格中，你能完整追踪到传入的 `name` 值为 `"DotNet-Developer"`。
4. **控制流导航快捷键（与 Visual Studio 高度重合）**：
   * **F10 (Step Over / 单步跳过)**：执行本行，跳到下一行代码。
   * **F11 (Step Into / 单步进入)**：如果当前行是函数，能直接钻进该函数的函数体继续追踪。
   * **Shift + F11 (Step Out / 退出函数)**：瞬间执行完函数，跳出该函数返回调用点。
   * **F5 (Continue / 继续执行)**：直接跳过断点，一直执行到下一个断点或结束。

有了这些，你就再也不用通过简单地 `print` 来找寻 Bug 来源了，调试、侦测异常、看变量状态在一行行之间尽显掌控。

---

现在，你已经成功地组装好了你的第一个 Python 开发和调试沙盒。点击 **[第1章：起步与类型系统](ch1_basics.md)**，我们将正式开启 Python 变量与类型的心智模型转型之旅！
