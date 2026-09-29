# 第1章：起步与类型系统

作为一名 .NET 开发者，你对于 C# 的编译、垃圾回收、强类型系统已经驾轻就熟。切换到 Python 时，你可能会对“动态语言”抱有疑虑，例如类型安全或难以代码重构等问题。本章将为你解开这些疑虑，帮助你快速建立 Python 的基础。

---

## 1. 结构与格式：缩进代替花括号

在 C# 中，我们习惯于使用花括号 `{}` 划分作用域和类/函数体，并且每行以分号 `;` 结尾。
在 Python 中，**缩进（Indent）** 是唯一的语法逻辑分界线，物理换行即代表语句结束（无需分号）。

**C# 示例**：
```csharp
public class Basics {
    public static void Main() {
        int val = 10;
        if (val > 5) {
            Console.WriteLine("Big");
        }
    }
}
```

**Python 示例（保存为 `hello.py`）**：
```python
val = 10
if val > 5:
    print("Big") # 四个空格的缩进表示 if 块内的代码。
```

> **.NET 开发者贴士**：
> 1. Python 中没有显式的 `Main` 入口。脚本从上到下顺序执行。
> 2. 请严格使用 **4个空格** 进行缩进，绝不要混用 Tab 和空格（现代 IDE 如 VS Code 会默认将 Tab 转换成4个空格）。

---

## 2. 动态类型 (Dynamic) 还是弱类型 (Weak)？

很多人有个误区，认为“动态语言”就等于“弱类型语言”（如 JavaScript 可以做 `1 + "2"` 的隐式转换）。
实际上，**Python 是动态强类型（Dynamic & Strong）语言**。

- **动态（Dynamic）**：变量本身不用声明类型，可以随时指向任何类型的对象。
- **强类型（Strong）**：类型一旦确定，运行时是不容许隐式类型混合运算的。其行为与 C# 的 `dynamic` 关键字非常相似，但更安全。

```python
x = 10       # x 是一个 int 类型的引用
x = "Hello"  # 正确！x 现在变为了 str 类型的引用（动态）

try:
    result = 1 + "2"  # 报错：TypeError
except TypeError as e:
    print(f"Python 阻止了隐式类型转换: {e}")  # 强类型体现
```

---

## 3. 救星：Type Hints（类型提示）

作为习惯了静态类型安全的 .NET 工程师，写没有类型标注的代码可能会让你感到窒息。Python 3.5 引入了 **Type Hints** 机制，能在保留动态语言灵活性的一同，提供媲美 C# 的静态编译期类型检查。

```python
# C# 定义：int Add(int a, int b) { return a + b; }
def add(a: int, b: int) -> int:
    return a + b

# 如果在 IDE 中传入了字符串，静态分析器（如 Pyright/MyPy）会发出警告
add(1, "2")  # IDE 会标红报错，这非常类似于 C# 编译错误提示！
```
*注：Type Hints 只是“画给 IDE 和 Linter 看”的标签，Python 虚拟机在运行时依然是以动态模式运行，不会在运行时强制拦截。*

---

## 4. 基础数据类型对应表

C# 拥有丰富的值类型和引用类型，而在 Python 中，一切皆为对象（继承于 `object`），通常在分配时在堆上创建。

| C# 类型            | Python 类型 | 说明与对比                                                                                |
| :----------------- | :---------- | :---------------------------------------------------------------------------------------- |
| `int` / `long`     | `int`       | Python 中的 `int` 是**无限精度**的（类似于 .NET 的 `BigInteger`），永远不会发生整数溢出！ |
| `double` / `float` | `float`     | Python 中只有 `float`（双精度，精度等同于 C# 的 `double`）。                              |
| `bool`             | `bool`      | 值取 `True` / `False` （注意：**必须首字母大写**，这与 C# 的小写 `true/false` 不同）。    |
| `string`           | `str`       | Unicode 字符串。Python 的字符串也是不可变的（Immutable）。                                |
| `null`             | `None`      | 代表空值，其类型是 `NoneType`。对应 C# 中的 `null`。                                      |

**基础类型操作代码实例**：
```python
# 1. 整数没有大小限制
large_num = 99999999 * 99999999999999999
print(large_num)  # 照常计算并输出，无需显式使用 BigInteger

# 2. 字符串插值 (C# 的 $"Hello {name}")：在 Python 中称为 f-string
name = "dotnet_coder"
age = 30
greeting = f"Hello, {name}! Your age is {age}." # f 前缀
print(greeting)

# 3. 各种类型的空值判定：C# 使用 val == null，Python 推荐 using 'is'
user = None
if user is None:
    print("User is null")
```

---

## 5. 常用的容器（Collections）对比

在 .NET 开发中，你离不开 `List<T>` 和 `Dictionary<TKey, TValue>`。Python 的内置容器极为强大和简洁，可以不夸张地说：90% 的 Python 开发仅用内置类型即可完成。

### A. List (列表) - 对应 C# `List<T>`
```python
# 创建列表（元素可以不同，但通常相同）
fruits: list[str] = ["apple", "banana"]

fruits.append("orange")         # 相当于 List.Add()
fruits.insert(0, "strawberry")  # 相当于 List.Insert()
item = fruits[1]                # 按索引索引

print(fruits)
```

### B. Dict (字典) - 对应 C# `Dictionary<TKey, TValue>`
```python
# 创建字典
user_scores: dict[str, int] = {"Alice": 90, "Bob": 85}

user_scores["Charlie"] = 95             # 新增或修改
score = user_scores.get("David", 0)     # 安全索取，若不存在返回默认值 0

print(user_scores)
```

### C. Set (集合) - 对应 C# `HashSet<T>`
```python
# 创建唯一且无序的集合
unique_ids: set[int] = {1, 2, 3, 3}  # 重复的 3 会自动去重
unique_ids.add(4)
print(unique_ids)  # {1, 2, 3, 4}
```

### D. Tuple (元组) - 相当于只读的 `ValueTuple` / 数组
Tuple 是不可变的，声明后就不能再修改其大小或替换其内部元素，非常适合用于传递不可篡改的多值。
```python
point: tuple[int, int] = (10, 20)
x, y = point  # 结构/解构赋值，与 C# 极其相似

# point[0] = 5  # 报错：TypeError，不可修改！
```

---

## 6. 【核心避坑】可变 (Mutable) 与 不可变 (Immutable) 的陷阱

这是一个极其经典的 Python 陷阱。
- **不可变类型**：`int`, `float`, `string`, `tuple`。
- **可变类型**：`list`, `dict`, `set`。

当你在一个带有默认参数的函数中，使用可变类型作为默认值时，其在 Python 内部只会被初始化**一次**（属于函数对象元数据本身）。这意味着所有对该函数的调用都将**共享**这同一个实例。

**坏味道代码（千万不要这样做）**：
```python
# 期望：每次调用 add_item 时，不传 lst 会默认新建一个空列表
def add_item(item: str, lst: list[str] = []):
    lst.append(item)
    return lst

print(add_item("apple"))   # 输出: ['apple']
print(add_item("banana"))  # 期望: ['banana']，实际输出: ['apple', 'banana']!
```

**C# 视角的重构方案**（类似于 C# 中我们做 `lst ?? new List<string>()`）：
```python
# 正确写法：默认值为 None
def add_item_correct(item: str, lst: list[str] = None):
    if lst is None:
        lst = []  # 每次调用时真正地新建一个列表
    lst.append(item)
    return lst

print(add_item_correct("apple"))   # ['apple']
print(add_item_correct("banana"))  # ['banana'] - 符合预期！
```

---

现在，你已经掌握了 Python 的大门钥匙。点击 **[第2章：控制流与异常处理](ch2_control_flow.md)**，我们将了解如何在解释型世界中控制代码逻辑的走向。
