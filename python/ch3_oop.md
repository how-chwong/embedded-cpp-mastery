# 第3章：函数式特性与面向对象

在 .NET 世界，一切皆是“类”和“方法”，自 C# 脱胎起就充斥着丰富的类结构。
但在 Python 中，它是一个**多范式语言**。在这里，既有非常轻量好用的**一等公民函数（First-Class Functions）**，也有一套极富动态特色的**面向对象系统（OOP）**。

下面，我们将对比这两者的设计差异。

---

## 1. 函数是一等公民 (First-Class Functions)

在 C# 中，所有能执行的逻辑基本上必须依赖类：即使是一个单纯的工具函数，也必须嵌套在 `public static class Helper` 中作为静态方法。
在 Python 中，**函数可以直接自由定义在文件任何地方**，并且可以像参数一样任意传递（甚至可以把一个函数赋给另一个变量，这与 C# 的 `Func<T>` 委托或 Lambda 非常相似）。

```python
# 一个单纯的全局函数，不属于任何 Class
def system_logger(message: str) -> None:
    print(f"[LOG] {message}")

# 可以将函数赋给变量 (类似于 C# Action<string> delegate)
log_caller = system_logger
log_caller("Hello, Free Function!")
```

### 多样化的参数机制（极致灵活）
Python 的参数传递远远比 C# 灵活，有以下三大特殊法宝：

1. **关键字传参 (Named Arguments)**：你可以不按顺序，通过指定参数名来调用（C# 也支持，但 Python 用得极普遍）。
2. **位置参数包 (`*args`)**：对标 C# 的 `params object[] args`，将不限长度的多余参数当做一个 **Tuple** 接收。
3. **命名参数包 (`**kwargs`)**：把传入的命名键值对当做一个 **Dict** 接收。

```python
# *args 收集溢出的位置参数；**kwargs 收集溢出的命名键值对参数
def super_flex_func(normal_arg, *args, **kwargs):
    print(f"Normal: {normal_arg}")
    print(f"Args (Tuple): {args}")
    print(f"Kwargs (Dict): {kwargs}")

super_flex_func("A", 1, 2, 3, debug=True, db_host="localhost")
# 输出: 
# Normal: A
# Args (Tuple): (1, 2, 3)
# Kwargs (Dict): {'debug': True, 'db_host': 'localhost'}
```

---

## 2. 类与实例：C# Class vs Python Class

Python 的类模型与 C# 的最大区别在于：**显式构造函数** 和 **`self` 的机制**。

**C# 示例**：
```csharp
public class Employee {
    public string Name { get; set; } // 隐式 this.Name
    public int Age;

    public Employee(string name, int age) {
        this.Name = name;
        this.Age = age;
    }

    public void Work() {
        Console.WriteLine($"{Name} is working.");
    }
}
```

**Python 示例**：
```python
class Employee:
    # 类的静态成员（相当于 C# static 字段）在这里声明：
    company = "Awesome DotNet Tech" 

    # __init__ 方法是 Python 的构造函数（Constructor），前后双下划线是内置魔术方法的标志
    # 所有的实例方法，第一个参数必须显式声明为 'self' (对标 C# 中的 'this')
    def __init__(self, name: str, age: int):
        self.name = name  # 实例属性直接通过 self 附加
        self.age = age

    def work(self) -> None:
        # 在内部访问自己的属性，也必须带上 self
        print(f"{self.name} is working.")

# 实例化对象（在 Python 中实例化不需要像 C# 一样写 'new'）
emp = Employee("Bob", 30)
emp.work() # 在外部调用时，不需要显式传入 self 参数，解释器会自动绑定
```

### 【解惑】为什么必须显式写 `self`？
在 C# 中，`this` 是一个关键字，编译器在后台帮你完成了类字段的变量作用域查找。
在 Python 哲学中，“**显式优于隐式（Explicit is better than implicit）**”。在定义实例方法时，显式将第一个入参绑定为正在调用的实例对象，降低了运行时的混淆。

---

## 3. 属性：`Getter/Setter` 对比

在 C# 中，我们常用自动属性来做属性访问拦截：
```csharp
public class BankAccount {
    private decimal _balance;
    public decimal Balance {
        get { return _balance; }
        set {
            if (value >= 0) _balance = value;
        }
    }
}
```

在 Python 中，可以通过内置装饰器 `@property` 完美对标 C# 的 `get/set`。

```python
class BankAccount:
    def __init__(self):
        # 惯例：在 Python 中，以单下划线 _ 开头的变量代表“私有变量”（约定，非严格物理阻止）
        # 双下划线 __balance 则是强私有（会触发名称改写，让外部无法轻易直接访问）
        self._balance = 0.0

    @property
    def balance(self) -> float: # 相当于 C# 的 getter
        return self._balance

    @balance.setter
    def balance(self, value: float) -> None: # 相当于 C# 的 setter
        if value >= 0:
            self._balance = value
        else:
            raise ValueError("存款不能为负数")

# 外部调用 (依然像访问字段一样简洁，而无需类似 () 的函数调用)
account = BankAccount()
account.balance = 100.0  # 触发 setter
print(account.balance)  # 触发 getter -> 100.0
```

---

## 4. 鸭子类型（Duck Typing）与抽象基类

C# 是极其依赖**接口(Interface)**的平台：如果你想实现契约模式，就必须声明诸如 `IDisposable`、`IEnumerable` 等接口。
而在 Python 这样的强动态语言中，有一条广为人知的圣经般的法则：
> **「如果它走起路来像鸭子，叫起来也像鸭子，那么它就是鸭子。」**

在 Python 中，你不必声明 `IReader` 接口。任何一个类，只要实现了 `read()` 方法，那我们在传递时，它就可以直接充当“Reader”使用了。

**鸭子类型实例：不需要接口就能实现的契约调用**
```python
class FileStream:
    def read(self) -> str:
        return "Reading from local file..."

class HttpStream:
    def read(self) -> str:
        return "Reading from HTTP packets..."

# 这个函数对传入对象的类一无所知，他只看这个对象有没有 "read" 属性。
# 只要有，就能调用！
def fetch_logs(stream_obj) -> None:
    content = stream_obj.read()
    print(f"Log content: {content}")

fetch_logs(FileStream())  # 正常读取！
fetch_logs(HttpStream())  # 正常读取！
```

### 如果 C# 程序员依然想要严格的接口契约怎么办？
Python 从不剥夺你的乐趣。如果你希望能像 C# 的 `abstract class` 或 `interface` 一样对子类进行严格约束（缺少实现就报错），你可以使用内置的 `abc`（Abstract Base Classes） 模块。

```python
from abc import ABC, abstractmethod

# 定义一个抽象类（对标 C# 里面的 interface / abstract class）
class DatabaseConnector(ABC):
    @abstractmethod
    def connect(self) -> None:
        pass

# 继承于抽象类
class PostgresConnector(DatabaseConnector):
    # 若不重写 connect 方法，在初始化此子类时会直接抛出 TypeError 报错！
    def connect(self) -> None:
        print("Connected to PostgreSQL databases successfully.")

conn = PostgresConnector()
conn.connect()
```

---

恭喜！你已经拥有了 Python 全套主流的逻辑、函数与面向对象的视角。点击 **[第4章：高级核心特性](ch4_advanced.md)**，我们将触及 Python 的高阶高级特性——这绝对能让你的代码写起来不仅少，而且快。
