# 第1章：基本概念、全能 Table 与魔法元表（Lua 篇）

作为 C# 程序员，你习惯了 `C#` 中分门别类的内置容器（`List`、`Dictionary`、`Array`、`Stack`）。
但在 Lua 中，设计者奉行极致的简约概念 —— **只提供 8 种基础基本类型**。并且在这之中，**复合数据结构只有唯一的一个：就是 Table（表）**。

本章将带你领略 Table 的全功能魅力，以及类似于 C# 运算符重载的神秘武器 —— **元表（Metatables）**。

---

## 1. Lua 的 8 种基础类型对照表

当你使用 `type(val)` 运行时，Lua 最多只会返回以下 8 种类型之一：

| Lua 返回值 | C# 基础对照                     | 说明                                                                       |
| :--------- | :------------------------------ | :------------------------------------------------------------------------- |
| `nil`      | `null`                          | 代表空值或未定义的变量。                                                   |
| `boolean`  | `bool`                          | 值只有 `true` 或 `false`。                                                 |
| `number`   | `double` / `int`                | 在 Lua 5.3 之后，内置区分了长整型和双精度浮点数，但对用户统称为 `number`。 |
| `string`   | `string`                        | 字符序列。注意：Lua 中的字符串是字节流，天然不可变。                       |
| `function` | `Func<...>` / `delegate`        | 一等公民函数。可以作为变量、入参和返回值传递。                             |
| `table`    | `Dictionary` / `List` / `class` | **全能表。** 是一切复杂结构的根源。                                        |
| `thread`   | 类似并发状态机                  | 代表协同程序（Coroutine）。注意：非操作系统的物理线程。                    |
| `userdata` | 宿主原生 C 结构指针             | 外部 C/C++（或嵌入在 C# 中交互）注入的原生对象内存布局。                   |

---

## 2. 全能 Table：一表治天下

Lua 的 Table 本质上是一个 **关联数组 (Associative Array)**。这意味着它既可以用做 **按数字索引的数组/列表**，也可以用做 **按 Key-Value 索引的键值对字典（Map）**。

### A. 作为数组（对标 C# `List<T>` / `Array`）
**【高能核警：Lua 的索引是从 1 开始的！】**
对于习惯了 C/C++、C# 的程序员，这绝对是最需要扭置的心智习惯。在 Lua 中，数组的第一个元素索引是 `1`，而不再是 `0`。

```lua
-- 创建一个纯数组 Table
local skills = { "C#", "SQL", "Lua" }

-- 1. 获取数组长度：使用井号运算符 `#` (相当于 C# List.Count)
print("数组长度为: " .. #skills) -- 3

-- 2. 读取第一项（索引是 1 ！）
print("第一项技能是: " .. skills[1]) -- C#

-- 3. 添加新元素（对标 List.Add）
table.insert(skills, "TypeScript")

-- 4. 遍历数组（对标 foreach 遍历）
-- ipairs 是专门为按 1 到 N 连续递增数字数组设计的遍历器
for index, val in ipairs(skills) do
    print("索引: " .. index .. ", 值: " .. val)
end
```

### B. 作为键值对字典（对标 C# `Dictionary<string, object>`）
当你的 Table 采用了不连续的、非数字或字符串作为 key 时，它就化身为一张哈希地图。

```lua
-- 创建一个字典 Table
local player = {
    name = "Solomon",
    level = 50,
    is_vip = true
}

-- 1. 访问元素有两种等价方式：
print(player["name"]) -- 经典字典查找 key
print(player.name)    -- 【优雅写法：点号运算符】等价于 string 查找，看起来非常接近 C# 对象的属性访问！

-- 2. 加入或修改元素（对标 Dict[key] = value）
player.gold = 9999 -- 动态增加
player.level = 51  -- 修改值

-- 3. 遍历字典（对标 foreach KeyValuePair）
-- pairs 是万能哈希遍历器，它能搜出所有没有数字序列规律的 Key-Value
for key, value in pairs(player) do
    print(key .. " = " .. tostring(value))
end
```

---

## 3. 魔法元表（Metatables）与元方法（Metamethods）

在 C# 中，我们可以通过重载运算符来实现自定义类型的加减（例如让两个 `Vector3D` 实例直接通过 `+` 拼接）：
```csharp
public static Vector3D operator +(Vector3D a, Vector3D b) {
    return new Vector3D(a.X + b.X, a.Y + b.Y);
}
```

在 Lua 中，没有任何显式的重载语法，但有一套精简绝伦的降维打击武器 —— **元表 (Metatable)**。
- 每一个普通 Table，都可以任意绑定另一个 Table 作为它的 **“元表”**。
- 元表内放着一系列特定的、以双下划线开头的函数（称为 **元方法 Metamethods**）。
- 类似 C# 的运算符重载：当解释器在对普通对象做加法、减法、或者转换为字符串等操作时，发现它们属于 Table 而无法直接预算，解释器会在第一时间去查找其元表中是否有对应的元方法，如果有就调用它。

### 核心元方法对照：
- `__add`: 处理加法操作（`+`）
- `__sub`: 处理减法操作（`-`）
- `__tostring`: 转换为字符串操作（对标 C# 的 `override string ToString()`）
- `__index`: **【全宇宙最核心】** 属性查找缺省处理器。当我们在一个 Table 查找某个 key 找不到时，它会自动去委托给 `__index` 所指定的候补字典。这也是 Lua 中实现面向对象（OOP）的最核心技术！

### 实战：通过元表重载三维矢量（Vector3）的相加与打印

```lua
-- 1. 创建普通表（VectorA 和 VectorB）
local vector_a = { x = 10, y = 20 }
local vector_b = { x = 5,  y = 15 }

-- 2. 创建一个元表，充当“方法的承载和运算符重载大本营”
local vector_meta = {}

-- 3. 重载加法元方法 __add
function vector_meta.__add(a, b)
    -- 返回一个全新的坐标表
    return { x = a.x + b.x, y = a.y + b.y }
end

-- 4. 重载 ToString 方法 __tostring
function vector_meta.__tostring(obj)
    return "Vector(x = " .. obj.x .. ", y = " .. obj.y .. ")"
end

-- 5. 【画龙点睛之笔】将两个运算矢量表的元表，设定为 vector_meta
setmetatable(vector_a, vector_meta)
setmetatable(vector_b, vector_meta)

-- 6. 现在，奇迹发生了！我们可以直接执行拼接、做加法运算！
local vector_sum = vector_a + vector_b  -- 自动寻找 __add 元方法
print(tostring(vector_sum)) -- 自动寻找 __tostring 元方法 -> 输出: Vector(x = 15, y = 35)
```

> **.NET 开发者贴士**：
> 元表机制极为自由，它甚至允许你在运行时动态剥离、替换或者挂载（通过 `setmetatable` 或 `getmetatable`）。这可以让你的对象在运行时发生脱胎换骨的行为改变，展示了极佳的灵活性。

---

现在，元表与 Table 的组合拳已经打下。点击 **[lua/ch2_oop.md](lua/ch2_oop.md)**，我们将见证这两个极简元素是如何拼装在一起，从而完成对 C# Class（经典面向对象继承模型）的全面逆袭！
