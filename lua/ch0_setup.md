# 第0章：极简工具链、第一行代码与调试沙盒（Lua 篇）

如果你是从 .NET 和 Visual Studio 转型而来，你会发现调试一个 Lua 项目可以说是精简化到了骨子里。因为它不需要庞大的运行时（SDK），也不需要复杂的编译期。我们将在几分钟内在你的 VS Code 中搭建好一个企业级的 Lua 自动提示与 F5 极速调试环境。

---

## 1. 下载并安装 Lua 解释器

不同于 C# 由多兆的 .NET CLR 转译运行，Lua 的解释器和核心库编译出来往往只有 **几百 KB**，运行开销小到可以忽略。

### Windows 安装指南（极速极简版）
对于 Windows 用户，我们可以通过直接下载免安装官方编译版（LuaBinaries）：

1. 访问官方推荐的 Windows 预编译下载源：[SourceForge LuaBinaries](https://sourceforge.net/projects/luabinaries/files/)。
2. 进入最新版本（如 **5.4.2** 或是稳定 5.3 系列），选择类似 `lua-5.4.2_Win64_bin.zip` 的压缩包下载。
3. 在你的电脑（例如 `C:\Program Files\Lua`）创建一个名为 `Lua` 的文件夹，将压缩包解压进去。
4. 解压后你会发现里面只有一个 `lua54.exe`（或者 `lua.exe`）和它的 DLL 链接库文件。
5. **添加环境变量（PATH）**：
   - 按下 `Win + S`，输入并搜索“编辑系统环境变量”。
   - 选择下方的 `环境变量`，并在“系统变量”里找到 `Path`，点击 `编辑` -> `新建` -> 将刚才的解压目录（如 `C:\Program Files\Lua`）添加进去，保存。
6. 打开一个新的终端（PowerShell 或 CMD），物理启动测试：
   ```bash
   lua54 -v  # 或者 lua -v
   ```
   *(根据解压出来的 exe 命名为准，一般可以重命名为 lua.exe，输入即可看到：`Lua 5.4.2  Copyright...`)*

> **Luajit 与标准 Lua的选择**：
> 如果你是在 Unity （xLua 框架）或者 OpenResty 核心上工作，通常它们底层搭载的是 **Luajit**（Lua 5.1 语法底座）。它们的语法有 95% 全平台一致，请安心选用任一版本学习。

---

## 2. VS Code 绝对统治级插件推荐

在 Lua 发型中，如果你没有高质量的静态拼写检验，极容易由于写错一个字母而变成 `nil` 空值（相当于 C# 中的空值错误，且运行时没有任何报错，极难被肉眼捕获）。

因此，立刻进入 VS Code（`Ctrl+Shift+X`），装入以下两款白金扩展插件：

1. **Lua (sumneko)**: 这是由腾讯工程师 sumneko 开发的 Lua 语言服务器。对于 Lua 来说，它就是 Pylance 或者 Visual Studio 级别的存在，提供极致精准的代码提示、函数签名解析、全局与局部变量警示、多文件跳转等。
2. **Local Lua Debugger (TomBlachap)**: 无需任何复杂的 C 编译，这是最简单、最轻量的开箱即用 F5 断点调试器，它完全基于 JS 虚拟引擎和本地管道驱动。

---

## 3. 编写你的第一行 Lua 代码

1. 在你的 Workspace 中，新建或在 `lua` 文件夹下新建一个文件：`main.lua`。
2. 放入以下带变量、函数和控制打印的数据代码：

```lua
-- 这是 Lua 里面的单行注释（对标 C# 中的 //）
--[=[
    这是 Lua 的多行注释（对标 C# 中的 /* ... */）
--]=]

-- 1. 定义一个简单的打招呼函数
local function welcome_dotnet(name)
    -- local 代表局部变量！请记住：在 Lua 中，定义任何变量和函数都默认必须带上 local！
    -- 如果你不加 local，它将默认作为“全局变量”直接注册在全局字典中，极易污染整个运行空间！
    local greeting = "Hello " .. name .. "! Welcome to Lua." -- Lua 中拼接字符串用的是两个英文句号 ".."
    print(greeting)
end

-- 2. 执行逻辑
local dev_name = "NET-Coder"
welcome_dotnet(dev_name)
```

### 运行方式
确保你的命令行能识别 `lua`（或者 `lua54`）命令，在此文件目录下，在终端中输入：
```bash
lua main.lua
```
便会瞬发输出 `Hello NET-Coder! Welcome to Lua.`。

---

## 4. F5 极致调试：无缝捕获变量状态

作为 .NET 程序员，我们调试时离不开监视、单步和条件捕捉。我们刚才安装的 **Local Lua Debugger** 就可以极其完美地支持这一切。

### 调试配置（只需一步）：
1. 在 VS Code 侧边框，点击调试图标 `Run and Debug` -> 点击 `create a launch.json file` -> 选择 **"Local Lua Debugger"**。
2. VS Code 会自动在你工作区下创建并打开 `.vscode/launch.json` 文件。
3. 其中的配置默认即可运行，核心参数如下：
   ```json
   {
       "version": "0.2.0",
       "configurations": [
           {
               "type": "lua-local",
               "request": "launch",
               "name": "Debug Lua File",
               "program": {
                   "lua": "lua54" // 必须与你在命令行终端里可运行的 exe 名字（lua、lua54 或 luajit）保持一致
               }
           }
       ]
   }
   ```

### 开始高能 Debug：
1. 打开 `main.lua` 文件。
2. 悬停在代码第 9 行 `local greeting = "Hello " .. ...` 处，在左侧边缘点一下，生成一个**红色实心断点**（对齐 C# 断点）。
3. 按下键盘上的 **F5**。
4. 你会看到调试控制台伴随着闪烁启动，然后程序完美地挂死在第 9 行（该行高亮变为焦黄色）。
5. 此时此刻：
   - 调试控制台左侧侧边栏的 `Variables -> Local` 区域中，你会在无声中看到 `name` 变量正牢牢绑定着当前值 `"NET-Coder"`。
   - 按下 **F10 (单步跳过)**、**F11 (单步钻入)** 控制你的多行逻辑。

有此沙盒，所有语法执行时到底是在计算还是在空转，你都能在第一时刻探明清楚！

---

现在，调试的红点已经点亮。点击 **[lua/ch1_basics.md](lua/ch1_basics.md)**，我们将了解 Lua 的基本类型，以及它赖以构建全宇宙的唯一数据容器 —— Table 的精妙设计！
