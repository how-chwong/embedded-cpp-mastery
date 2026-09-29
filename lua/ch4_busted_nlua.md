# 第4章：单元测试、集成打包与 C# 宿主双向交互（实战集成篇）

如果你要在生产中引入 Lua，你必须解决三个工业化问题：
1. 如何为没有编译期安全检查的动态脚本编写**自动单元测试**？
2. 如何实现脚本发布和简单集成？
3. **（重头戏）**：如何在你的 C# (.NET Core/.NET 8) 宿主项目里完美嵌入、操控，并实现 **C# 与 Lua 之间的两界双向高能互操作（NLua 实现）**？

本章将给出最详尽的解决方案。

---

## 1. 自动化单元测试：Busted 框架

在 .NET 中，我们最爱使用 `xUnit`。在 Lua 生态中，最主流、用起来最像自然语言描述（Behavior-Driven Specification）的单元测试框架是 **`Busted`**。

### A. 什么是 Busted 且如何安装？
Lua 领域有一款包管理工具叫 **LuaRocks**（相当于 .NET 的 NuGet，可以通过 `scoop/choco install luarocks` 安装，其作用是方便开发者下载库）。
如果你本地装了 LuaRocks，只需要运行：
```bash
luarocks install busted
```

### B. 一个测试的结构（非常精美）
类似于 C# 的 `SpecFlow`，Busted 将测试组织为 `describe` 和 `it` 断言：

```lua
-- test_spec.lua 示例，用来测试第 3 章的 calc 模块：
local calc = require("calc")

describe("测试 Calc 基础数学模块的行为", function()
    -- 在每个测试用例前会进入此挂钩
    before_each(function()
        print("💡 开启本轮测试数据校验")
    end)

    it("10 加 20 应该完全等于 30", function()
        local result = calc.add(10, 20)
        -- 断言机制，极其类似 C# FluentAssertions (Assert.Equal(30, result))
        assert.are.equal(30, result)
    end)

    it("计算半径为 5 的圆周面积应当正常", function()
        local area = calc.get_circle_area(5)
        -- 判定值是否大于某个值
        assert.is_true(area > 78)
    end)
end)
```
测试运行极其精细，只需要在文件夹下输入命令 `busted .`，就会在终端输出炫酷的钩号进度条。

---

## 2. 核心桥梁：在 C# 中嵌入和双向调用 Lua (NLua 实战)

对于 .NET 工程师，学 Lua 最实用的场景莫过于在 C# 项目中，通过给策划/运维提供写 Lua 来达到不用重新发布 C# 控制台、ASP.NET 服务、或者 Unity 客户端就能热修改、热配置功能。

目前在 .NET 生态中，最全能、功能最稳定的桥梁是 **[NLua](https://github.com/NLua/NLua)** 库（基于 C 原生 Lua 深度粘合至纯 C# 虚拟机中）。

### 🛠️ 动手实现：C# 端对端融合 Lua
让我们在你的电脑上用 C# 创建一个真实的控制台项目，并引入、测试 NLua。

#### 第一步：在本地创建一个极简 C# 控制台项目

1. 新建一个空文件夹，例如命名为 `dotnet_lua_bridge`。
2. 用 VS Code 或 Visual Studio 开启此目录，在终端运行命令一键建站：
   ```bash
   dotnet new console
   ```
3. 引入 NuGet 包 **NLua**（等同于 Python `pip`）：
   ```bash
   dotnet add package NLua
   ```

#### 第二步：替换并编写 `Program.cs`
将项目下的 `Program.cs` 彻底修改为以下代码，来完整感受两界传递：

```csharp
using System;
using NLua; // 引入 NLua 命名空间

namespace DotNetLuaBridge;

// 1. 定义一个提供给 Lua 调用的普通 C# 自定义类
public class GameCharacter
{
    public string Name { get; set; } = "C#_Paladin";
    public int Level { get; set; } = 1;

    public void GainExp(int exp)
    {
        Level += exp / 100;
        Console.WriteLine($"[C# Character Object] {Name} has gained exp! New Level: {Level}");
    }
}

class Program
{
    static void Main(string[] args)
    {
        Console.WriteLine("🛡️ --- 启动 .NET / Lua 交互现场 --- 🛡️\n");

        // 2. 初始化核心 Lua 虚拟机
        using (Lua luaState = new Lua())
        {
            // ===============================================
            // 模式 A：C# 单向通知 Lua 执行代码
            // ===============================================
            Console.WriteLine(">>> 【Step 1】从 C# 启动执行 Lua 简单运算...");
            // 执行纯 Lua 代码，返回一个 object[] 数组。
            object[] res = luaState.DoString("return 100 + 200, 'Lua' .. 'Script'");
            Console.WriteLine($"Lua 返回的数字: {res[0]}, 返回的字符: {res[1]}");

            // ===============================================
            // 模式 B：C# 向 Lua 域注册外部对象（热调用）
            // ===============================================
            Console.WriteLine("\n>>> 【Step 2】将 C# 物理实例注册并开放给 Lua 调用...");
            GameCharacter hero = new GameCharacter { Name = "DotNet_Dragon_Slayer", Level = 10 };

            // 注册！相当于给该实例起了个别名（"hero"），Lua 内部脚本便可以直接把它当全局变量操控！
            luaState["hero"] = hero;

            // 让 Lua 来操作这个 C# 级别对象的方法与属性！
            string hotfixLuaScript = @"
                print('[Lua State] 当前 hero 的名字为: ' .. hero.Name)
                -- 修改 C# 实例属性！
                hero.Name = 'Super_Lua_Slayer'
                -- 直接跨界调用 C# 实例函数！
                hero:GainExp(200) -- C# 级别的方法在 Lua 中默认用『冒号』调用！
            ";
            luaState.DoString(hotfixLuaScript);

            // 在 C# 验证，属性是否被 Lua 真正热修改成功：
            Console.WriteLine($"[C# Confirm] 离开 Lua 作用域。C# 中的原实例名字变为了: {hero.Name}");

            // ===============================================
            // 模式 C：C# 获取并显式调用 Lua 定义的函数
            // ===============================================
            Console.WriteLine("\n>>> 【Step 3】C# 从宿主方向直接提取并调用 Lua 闭包函数...");
            
            // 在 Lua 中定一个打怪分配经验的函数
            string defineLuaFunc = @"
                function GetBattleReward(base_xp, multiplier)
                    local final_xp = base_xp * multiplier
                    local message = '战斗结束！奖励经验值为: ' .. final_xp
                    return final_xp, message
                end
            ";
            luaState.DoString(defineLuaFunc);

            // C# 显式抓取这个 Lua 全局函数
            LuaFunction rewardFunction = luaState.GetFunction("GetBattleReward");
            
            if (rewardFunction != null)
            {
                // C# 带有类型参数地调用它：传入 150 和 1.5 倍数
                object[] output = rewardFunction.Call(150, 1.5);
                double finalXp = (double)output[0];
                string rewardMsg = (string)output[1];

                Console.WriteLine($"[C# Received] 计算结果: {finalXp}");
                Console.WriteLine($"[C# Received] 消息摘要: {rewardMsg}");
            }
        }

        Console.WriteLine("\n🎉 --- 两界互操作完美收官 --- 🎉");
    }
}
```

#### 第三步：运行 C# 项目
在控制台输入并运行：
```bash
dotnet run
```
你将看到控制台高能输出了从 **「Lua 内部打印出了 C# 注册过的 hero 的初始名字」**、**「C# 的 hero 数据真的被 Lua 脚本修改并顺利升级」** 以及 **「C# 成功获取 Lua 自定函数算出了 225 点经验值」** 的闭环报告！

---

## 3. Lua 极致学习结业总结：

此时此刻，恭喜你已经一网打尽了有关 Lua 的全功能底座：
1. **[第0章：极简开箱与环境调试](ch0_setup.md)** 精致地配置了 sumneko 白金插件，打通了 IDE 级别的断点挂起调试。
2. **[第1章：基本概念与元表](ch1_basics.md)** 重点指出了 1-based 索引习惯并见证了 `__add` / `__tostring` 元方法的运算符重载力量。
3. **[第2章：面向对象实现](ch2_oop.md)** 利用极其巧妙的原型 `__index` 级联，完成了类模型和构造器的还原。
4. **[第3章：模块与协程](ch3_advanced.md)** 展示了局部局部包的封装以及单线程并发、绝无同步死锁的极致状态机。
5. **本章** 更是带我们在 C# 中以宿主身份引入并对撞了 Lua 脚本逻辑，让你彻底拥有了架构热更新、脚本解耦的实战大招。

你可以随时以这些模板开启和游戏引擎或其他中间件的 Lua 融合。那么现在，让我们关闭并打包 Lua，带着对于 OOP、面向对象和动态特性的自如掌控，进军我们三大通关大山里的最后一站 —— 重量级的 Web 端 3D 渲染器 **Three.js** 吧！
