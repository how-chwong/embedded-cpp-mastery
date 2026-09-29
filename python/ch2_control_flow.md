# 第2章：控制流与异常处理

上一章我们学习了类型系统。本章我们将探索 Python 的程序流转控制机制（条件、匹配、循环）和异常处理，看看在没有 `&&`、`||` 和 花括号的情况下，Python 是如何进行逻辑控制的。

---

## 1. 条件控制与布尔运算符

在 C# 中，我们使用圆括号 `()` 包裹条件，以及 `&&`, `||`, `!` 执行布尔逻辑。
在 Python 中，条件**不需要圆括号**，并且布尔操作符是极其口语化的英文：`and`, `or`, `not`。
此外，`else if` 在 Python 中缩写为 `elif`。

**C# 示例**：
```csharp
if (age >= 18 && name != null) {
    Console.WriteLine("Welcome adult!");
} else if (age < 18 || !isVip) {
    Console.WriteLine("Forbidden.");
} else {
    Console.WriteLine("Other.");
}
```

**Python 示例**：
```python
if age >= 18 and name is not None:
    print("Welcome adult!")
elif age < 18 or not is_vip:
    print("Forbidden.")
else:
    print("Other.")
```

> **.NET 开发者贴士**：
> Python 中的 "Truthiness"（真值测试）：除了布尔值 `True`/`False`，很多空的对象都会隐式评估为 `False`。例如：`0`, `0.0`, `""`（空字符）, `[]`（空列表）, `{}`（空字典）, `None`。
> 在 C# 中，你需要写 `if (string.IsNullOrEmpty(name))`；而在 Python 中，只需写 `if not name:`。

---

## 2. 现代模式匹配：`match-case` (对标 C# Switch Expressions & Pattern Matching)

C# 近几个版本增强了 Pattern Matching（例如 `switch` 模式匹配、解构）。Python 从 3.10 版本也引入了高级的 **`match-case` 语法**，功能同样非常震撼，支持解构模式、属性匹配和守卫。

**C# 示例**：
```csharp
var text = status switch {
    200 => "OK",
    400 or 404 => "Client Error",
    _ => "Unknown"
};
```

**Python 示例（支持通配、合取与解构）**：
```python
# 1. 基础值匹配与合并
status = 404
match status:
    case 200:
        message = "OK"
    case 400 | 404: # 使用 | 代替 or
        message = "Client Error"
    case _: # 相当于 C# default 或 _ 丢弃模式
        message = "Unknown"

# 2. 高级模式匹配：解构类似 tuple/dictionary 
point = (0, 20)
match point:
    case (0, 0):
        print("At the origin")
    case (0, y): # 匹配 x 为 0，并把 y 的值提取并命名为 y
        print(f"On the Y axis at {y}")
    case (x, y) if x == y: # 带有 if 守卫 (guard)
        print("On the diagonal line x = y")
```

---

## 3. 循环结构与特有的 Built-in 利器

在 Python 中，循环只有两种：`for-in` 和 `while`。
没有 C# 中传统的 C风格 `for(int i=0; i<10; i++)`，也没有 `do-while`。

### A. 数值循环：对标 C# 的 `for (int i = 0;;)`
Python 使用内置函数 `range()` 配合 `for-in` 来实现索引数值循环。
```python
# C#中: for (int i = 0; i < 5; i++)
for i in range(5):
    print(i) # 依次输出 0, 1, 2, 3, 4

# 支持步长和区间：range(start, stop, step)
for i in range(2, 10, 3):
    print(i) # 依次输出 2, 5, 8
```

### B. 遍历循环：对标 C# 的 `foreach`
Python 的 `for-in` 本质上就是 C# 的 `foreach`。
```python
items = ["dotnet", "core", "python"]
for item in items:
    print(item)
```

### C. 黄金搭配：`enumerate` & `zip`
在 .NET 中，如果你需要在 `foreach` 中同时知道**索引**和**元素值**，你通常得通过显式索引；抑或是将两个 List 一起按索引迭代。
在 Python 中，提供了非常多优雅的内置函数。

```python
# 1. 同时获取索引和值：enumerate()
elements = ["Gold", "Silver", "Bronze"]
for index, element in enumerate(elements):
    print(f"Rank {index + 1}: {element}") 
    # 输出: Rank 1: Gold, Rank 2: Silver, ...

# 2. 并行迭代两个等长列表：zip() (等价于 C# 的 IEnumerable.Zip)
users = ["Alice", "Bob"]
passwords = ["A123", "B567"]
for user, password in zip(users, passwords):
    print(f"User: {user}, Pwd: {password}")
```

### D. 奇特魔术：`for-else` 语法
Python 提供了一个对 C# 转型者来说比较新奇的结构：`for-else`。
如果循环**完整地执行完毕（没有被 `break` 提前打断）**，就会在循环结束后去执行 `else` 块内的代码。它免去了你在 C# 中写一个 `bool isFound = false;` 的标志性代码。

```python
# 检测一个列表中是否存在偶数，如果没有，输出一个日志：
numbers = [1, 3, 5, 7]

for num in numbers:
    if num % 2 == 0:
        print(f"Found even: {num}")
        break
else:
    # 只要上面的 for 没有触及 break（也就是没有一个偶数），就会进这里！
    print("C# 中我们需要显式定义 isFound flag，但在 Python 中这简直是一气呵成！")
```

---

## 4. 异常处理：`try-except` (对比 C# `try-catch`)

Python 的异常处理机制在结构上和 .NET 几乎一致，但关键字不同。

| C# 核心词  | Python 核心词 | 差异与说明                                                      |
| :--------- | :------------ | :-------------------------------------------------------------- |
| `try`      | `try`         | 开始可能报错的上下文。                                          |
| `catch`    | `except`      | 捕获指定类型的异常。                                            |
| `finally`  | `finally`     | 无论资源释放还是正常完毕一定执行底部的代码。                    |
| `throw`    | `raise`       | 引发/向上抛出一个异常。                                         |
| *(无对应)* | `else`        | **独有：** 仅当 `try` 块里**未发生**任何异常时，才会走 `else`。 |

### 典型异常捕捉代码

```python
# C# 示例：
# try { ... }
# catch (DivideByZeroException ex) { ... }
# finally { ... }

try:
    x = int(input("请输入数字: "))
    result = 10 / x
except ZeroDivisionError as ex:
    print(f"发生了除零错误: {ex}")
except (ValueError, TypeError) as ex: # 可以用 tuple 捕捉多个异常
    print(f"数据转换或类型错误: {ex}")
except Exception as ex: # 相当于 catch (Exception ex)
    print(f"未知的其他异常: {ex}")
else:
    print(f"太棒了！你的结果是 {result}，无任何异常发生！")
finally:
    print("此处对应 C# Finally，用来清理资源/连接。")
```

### 抛出异常 (Throw)

```python
# C#中: throw new ArgumentException("Must be positive.");
raise ValueError("Must be positive.")
```

### 核心常见异常类映射关系：

| .NET 异常                  | Python 对应异常                 | 触发场景                                     |
| :------------------------- | :------------------------------ | :------------------------------------------- |
| `NullReferenceException`   | `AttributeError` 或 `TypeError` | 试图读取 `None` 的属性，或者对它进行非法操作 |
| `IndexOutOfRangeException` | `IndexError`                    | 列表/元组索引越界                            |
| `KeyNotFoundException`     | `KeyError`                      | 字典中查找了不存在的 key                     |
| `ArgumentException`        | `ValueError`                    | 传入函数的参数不合规                         |
| `NotImplementedException`  | `NotImplementedError`           | 接口或基类方法留空待后续重写                 |

---

现在，你对于 Python 中处理常规控制流与异常已经得心应手了。点击 **[第3章：函数式特性与面向对象](ch3_oop.md)**，我们将触及 C# 开发者的绝对本领领域——面向对象。
