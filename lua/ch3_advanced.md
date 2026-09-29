# 第3章：独立模块、协同程序与 C# 协程对比（高阶篇）

一个优秀的 .NET/C# 项目必然是讲究**命名空间（Namespace）**、模块拆分以及高弹性的**异步与流式运算**。
在本章中，我们将研究：
1. Lua 是如何通过全能 Table 实现类似 C# Namespace 与 `File Import` 的**模块化**。
2. Lua 的看家核心本领 —— **协同程序（Coroutine）**，以及它如何完美重做 Unity/C# 中的 `yield return` 异步状态机。

---

## 1. 模块化机制：C# `using / namespace` vs Lua `require`

在 Lua 5.1/5.4 之后，官方推荐使用最清爽的模块形式：**将一个外部 `.lua` 文件打包并返回一个装满方法/变量的局部 Table**，在其他文件中使用 **`require`** 函数引入它（相当于 NuGet DLL 引用、或是 C# 中的 `using`）。

### 建立你的第一个模块：`calc.lua`
在同一目录下，新建编写文件：

```lua
-- calc.lua 内容：
local Calc = {} -- 定义一个局部 Table

-- 内部局部常量（外部绝对无法直接访问，相当于 C# internal / private）
local PI = 3.14159265

function Calc.add(a, b)
    return a + b
end

function Calc.get_circle_area(radius)
    return PI * radius * radius
end

return Calc -- 这一步是核心！必须在文件尾部 return 这个模块 table
```

### 引入并使用该模块：`app.py` 对应的 `main_app.lua`

```lua
-- main_app.lua 内容：
-- 引入 calc.lua 模块，解释器会自动在目录下搜索 calc.lua 并将返回的 table 赋给 local calc
local calculator = require("calc")

local sum = calculator.add(10, 20)
local area = calculator.get_circle_area(5)

print("Add Sum: " .. sum)
print("Circle Area: " .. area)

-- 尝试获取私有成员 PI (会失败)
-- print(calculator.PI) -- 输出: nil （完美实现数据封装隔离）
```

> **.NET 开发者贴士**：
> 1. `require` 是带有一级文件缓存的：即使你在不同的脚本中重叠调用 `require("calc")`，该模块文件在运行时底层实际上**只会被加载并执行一次**。
> 2. 如果你的模块放在了子文件夹中，如 `./utils/calc.lua`，调用方法是将主文件夹到文件名通过`.`连接：`require("utils.calc")`。

---

## 2. 协程机制 (Coroutine)：对标 C# `yield return` 与异步

这是 Lua 的绝对看家特长！
在 C#（尤其在 Unity）开发中，你极度熟悉 `yield return`。例如让一个怪物的 AI 行为循环巡逻、停顿、攻击：
```csharp
public IEnumerator PatrolAI() {
    Console.WriteLine("Step 1: Move left");
    yield return new WaitForSeconds(2.0f); // 挂起并等待。主框架下次迭代时唤醒我
    Console.WriteLine("Step 2: Cast skills");
}
```

在 C# 内部，编译器会在底层自动帮你生成一个极度繁琐的 `IEnumerator` 和 `IAsyncStateMachine` 状态机类。
而而在 Lua 中，**协程是一等公民直接内置的（单线程多任务协作机制）**！你可以在纯原生层面通过 `coroutine` 完成这种不依靠物理多线程的高效异步和状态挂起。

### Coroutine 核心函数体系：
- **`coroutine.create(func)`**: 为一个普通函数创建一个协程，返回协程对象（其类型为 `thread`）。
- **`coroutine.resume(co, ...)`**: **【启动或恢复】**执行协程。当协程未开始，调用它会真正进入函数。当协程被 yield 挂起，调用它会从上次挂起点继续跑。**支持传递参数！**
- **`coroutine.yield(...)`**: **【中停挂起】**当前协程。它会拉闸中停函数的执行，并将参数传回给唤醒它的 `resume` 外部环境。这是最高秒笔的机制。
- **`coroutine.status(co)`**: 查看协程的活动状态：`suspended`（挂起/未开始）、`running`（运行中）、`dead`（已执行结束）。

---

### 实战演示：怪物 NPC 行为巡逻状态机与双向数据流传递

下面是一个闭环、可以让你彻底看懂 `resume` 与 `yield` 双向交互的数据传递机制（极易困惑的新手大坑）：

```lua
-- 1. 定义一个控制怪物工作流的普通函数
local function monster_ai()
    print("👾 [AI] 怪物生成！开始向左巡逻...")
    
    -- 下面这行代码会把当前函数彻底“挂死、暂停执行”。
    -- 括号里的数据 "Ask_Main_To_Wait_2_Seconds" 会作为返回值在外部的 resume 函数执行结果中抛出
    local main_response = coroutine.yield("Ask_Main_To_Wait_2_Seconds")
    
    -- 再次苏醒时，我们会收回外部传递进来的主流程投食的反馈参数
    print("👾 [AI] 怪物已再度苏醒。主事件循环在休息后反馈的数据是: " .. tostring(main_response))
    print("👾 [AI] 怪物开始发起重破攻击！")
    
    coroutine.yield("Ask_Main_To_Save_Data")
    print("👾 [AI] 怪物巡逻结束并退场！")
end

-- 2. 将此函数打包装入协程
local golem_co = coroutine.create(monster_ai)

-- 3. 主框架第一次唤醒协程
print("⏱️ [Main] 第一次唤醒协程...")
local success, msg = coroutine.resume(golem_co) 
-- 此时 monster_ai 启动，运行到第一个 yield，被挂起中停！
-- success 代表执行是否正常无报错，msg 就是 yield 吐出来的 "Ask_Main_To_Wait_2_Seconds"
print("⏱️ [Main] 第一阶段返回结果: " .. msg)

-- 4. 模拟主流程休息了一会后，第二次恢复协程
print("\n⏱️ [Main] 准备第二次恢复协程...")
-- 这一次我们调用 resume，并多传一个参数 "Slept_Well_Boss"。
-- 这个字符串，会被我们在协程内部怪物断点处的 `main_response` 变量稳稳接住！
local success2, msg2 = coroutine.resume(golem_co, "Slept_Well_Boss")
-- 运行到第二个 yield 处再次挂起！
print("⏱️ [Main] 第二阶段返回结果: " .. msg2)

-- 5. 第三次（最后一次）恢复，主流程不需要传参，让它走到底
print("\n⏱️ [Main] 准备恢复收尾...")
coroutine.resume(golem_co)

print("⏱️ [Main] 此时协程状态: " .. coroutine.status(golem_co)) -- dead （功德圆满）
```

---

### 【深度解析】协程与 C# 多线程/异步的本质差别

对于 .NET 程序员，不要混淆 **协程（Coroutine）** 与 **物理进程/多线程（Thread/Task）**：
- **C# 物理线程（CPU 抢占式调度）**：由 OS Kernel 或线程池随时强行中断和唤醒，可能在后台并行（Parallel）利用多核 CPU。必须通过 `lock` 极其严密地防范多个物理线程读写同一个数据引发的数据竞争崩溃。
- **Lua 协程（协作式非强占）**：在宏观和微观上都是 **单线程顺序运行的**。除非协程内部主动高亮执行了 `coroutine.yield()`，否则外部任务永远不可能插队进来。
- **优势**：
  1. **零死锁/锁开销**：不用写任何 `lock` 关键字，不会因为乱抓互斥锁导致崩溃，极度适合编写复杂的离散游戏业务逻辑。
  2. **内存极小**：协程切换仅仅是 CPU 寄存器指针的挪移，开销非常微弱，哪怕本地创建几万个协程并发也不会阻塞内存系统。

---

现在，由 `require` 与 `coroutine` 组成的系统高阶技能已经成功毕业。点击 **[lua/ch4_busted_nlua.md](lua/ch4_busted_nlua.md)**，我们将迈入 Lua 开发的终极实战派：如何像专业工程师一样做自动化测试，以及在真实的 .NET WinForms / ASP.NET 项目中如何高能嵌入和调用 Lua 语法！
