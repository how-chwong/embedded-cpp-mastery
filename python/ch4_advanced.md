# 第4章：高级核心特性

如果你想写出极其地道（Pythonic）、高效的 Python 代码，你必须了解本章介绍的高级特性。本章将主要探究：Python 里的 "LINQ"（推导式）、"yield return"（生成器）、原生 AOP 拦截器（装饰器）以及资源管理释放模型。

---

## 1. 列表与字典推导式：Pythonic 的 “LINQ” 语法糖

作为 .NET 开发者，你肯定视 **LINQ** （Language Integrated Query）为无可替代的生产力工具。
在 Python 领域，虽然也存在函数式的 `map()` 和 `filter()`（类似于 Linq 的 `Select()` 和 `Where()`），但社区更推荐使用更简洁、更高效的手法——**推导式 (Comprehensions)**。

让我们看看两者的惊人对应关系：

### A. 数据的过滤与转换 (Where + Select)

**C# / LINQ 方式**：
```csharp
var scores = new List<int> { 45, 80, 52, 95, 100 };
var passedDoubled = scores
    .Where(s => s >= 60)
    .Select(s => s * 2)
    .ToList(); // 结果: [160, 190, 200]
```

**Python 列表推导式方式**：
```python
scores = [45, 80, 52, 95, 100]

# 语法：[ <输出转换> for <变量> in <容器> if <过滤条件> ]
passed_doubled = [s * 2 for s in scores if s >= 60] # 结果: [160, 190, 200]
```

### B. 转换到字典 (ToDictionary)

**C# / LINQ 方式**：
```csharp
var users = new List<User> { new User("Alice", 1), new User("Bob", 2) };
var userDict = users.ToDictionary(u => u.Name, u => u.Id);
```

**Python 字典推导式方式**：
```python
users = [("Alice", 1), ("Bob", 2)] # 元组列表

# 字典推导式：{ key: value for ... }
user_dict = {name: uid for name, uid in users}
print(user_dict) # {'Alice': 1, 'Bob': 2}
```

---

## 2. 迭代器与生成器：对标 C# `yield return`

在 C# 中，如果我们想极度节省内存地、流式地（Lazy Evaluation）处理无穷大数据流，我们会写一个返回 `IEnumerable<T>` 的方法，并在内部使用 `yield return`。

在 Python 中，这非常相似，同样有一个名为 `yield` 的关键字，它能自动生产一个 **生成器 (Generator)** 对象。

**C# 示例**：
```csharp
public IEnumerable<int> Fibonacci(int limit) {
    int a = 0, b = 1;
    while (a < limit) {
        yield return a;
        int temp = a;
        a = b;
        b = temp + b;
    }
}
```

**Python 示例**：
```python
def fibonacci(limit: int):
    a, b = 0, 1
    while a < limit:
        yield a  # 当执行到这里时，函数会“挂起”并产出当前值。下一次请求（如迭代）时，从这里恢复
        a, b = b, a + b

# 循环调用它，没有任何内存负担（哪怕 limit 是十亿级别，其内存使用始终为常数）
for num in fibonacci(100):
    print(num, end=" ") # 0 1 1 2 3 5 8 13 21 34 55 89
```

---

## 3. 装饰器 (Decorators)：Python 原生的 AOP / 切面拦截机制

在 .NET 架构设计中，要实现诸如权限校验、慢查询日志记录等业务不相关逻辑（面向切面 AOP ），我们会使用 Attribute（拦截器），或者通过 ASP.NET Core Mini API 的 ActionFilter 过滤器。通常这非常沉重，且需要繁琐的依赖注入（DI）配置。

而在 Python 中，**装装饰（Decorators）** 是极致轻量、优美且原生支持的最强武器。它本质上只是一个高阶函数：**它接收一个目标函数，对其进行包裹装饰后，返回新的加强版函数。**

### 自定义一个耗时监控装饰器：
```python
import time

# 1. 定义装饰器。接收一个函数 'func' 作为参数
def timer_decorator(func):
    # *args, **kwargs 用来无缝兼容被装饰函数的任何入参
    def wrapper(*args, **kwargs):
        start_time = time.time()
        
        # 执行原函数
        result = func(*args, **kwargs)
        
        end_time = time.time()
        print(f"[Timer] 函数 '{func.__name__}' 耗时: {end_time - start_time:.4f} 秒")
        return result
    return wrapper # 返回包装后的函数

# 2. 通过 @ 语法无缝注入此逻辑（等同于 C# 的 [TimerAttribute] 拦截器）
@timer_decorator
def heavy_database_query(record_id: int):
    print(f"正在读取数据 {record_id}...")
    time.sleep(1.2) # 模拟 IO 堵塞
    return {"id": record_id, "data": "dotnet_to_python"}

# 3. 直接调用，拦截器自动工作
data = heavy_database_query(42)
```

---

## 4. 魔法方法（Dunder Methods）与 资源管理释放

Python 中有一系列以“双下划线（Double Underscore）”包围的方法，被称为 **Dunder Methods**。它们在框架或运行时中由于特定事件而被隐式调用，这也是 Python 高度定制类的钩子（等同于 C# 的 `IComparable` / `IDisposable` 等接口实现）。

### A. 对标 `ToString()`：`__str__` 和 `__repr__`
```python
class Book:
    def __init__(self, title: str):
        self.title = title

    def __str__(self) -> str: # 等同于 C# 的 string ToString()
        return f"书名: 《{self.title}》"

book = Book("CLR via C#")
print(book)  # 自动调用 __str__ -> 输出：书名: 《CLR via C#》
```

### B. 对标 `using(...)` 语句与 `IDisposable` 接口：上下文管理器

在 C# 中，读写文件或连接数据资源时，为防内存/连接溢出，我们使用 `using`：
```csharp
using (var stream = new StreamReader("file.txt")) {
    var content = stream.ReadToEnd();
} // 离开作用域，自动触发 Dispose() 关掉文件句柄
```

在 Python 中，没有 `IDisposable` 接口，只要你的类实现了 **`__enter__`** 和 **`__exit__`** 这两个魔法方法，就可以用极其简美的 **`with`** 关键字做这件事：

```python
class DatabaseConnection:
    def __enter__(self):
        print("【Open】 建立数据库连接池...")
        return self # 这里的返回值将被 under 'as' 的变量绑定

    def __exit__(self, exc_type, exc_val, exc_tb):
        # 参数包括是否发生了异常，可在离开 with 作用域时，决定是否捕获异常并彻底清理连接
        print("【Close】 无论是否发生异常，自动高能回收连接资源！")
        return False # 返回 False 代表不吞掉 Scope 发生的异常，正常向外抛出

# 极度精美的上下文管理器使用：
with DatabaseConnection() as conn:
    print("正在进行增删改查...")
    # 哪怕在此处写 raise ValueError() 出错，也绝对会进入 __exit__ 执行 Close 逻辑！

print("资源已被完美回收。")
```

---

现在，你已经解锁了 Python 核心的魔法特权。点击 **[第5章：包管理与工程实践](ch5_ecosystem.md)**，我们将从具体的语言基础走出来，聊聊项目整体架构、包生态配置（NuGet 对标等）和如何在企业开发中编写规范的 Python 项目。
