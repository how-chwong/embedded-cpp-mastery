# 第2章：在 Lua 中还原 C# 面向对象（OOP 篇）

作为 C# 工程师，你熟悉了 `class`、接口、继承和多态。
当听到 Lua 没有 `class` 关键字时，你可能会感到惊讶甚至不知所措。

然而，在上一章中我们见证了 **`__index`** 这个强大的元方法。利用 **“元表 + 普通表”**，Lua 能够通过极其精巧的 **“原型链模式（Prototyping）”**（类似于 JavaScript 早期模型），完美实现类的定义、实例化、构造器和继承！

---

## 1. 点号 `.` 与冒号 `:` 的大门钥匙：隐式 `self`

在 Lua 对象的调用中，你经常能看到 `obj:method()` 这样的**冒号**调用。
冒号 `:` 只是一个语法糖：它可以帮你**隐式传递实例自己（`self`）**作为第一个参数。这完全对标了 C# 代码中我们可以省略的隐式 `this`。

### C# 示例：
```csharp
public class Soldier {
    public string Name;
    // this.Name 是编译器隐式定位的
    Public void Attack() {
        Console.WriteLine($"{Name} attacks!");
    }
}
```

### Lua 示例（手工传递 self vs 冒号语法糖）：

```lua
local soldier = { name = "Arasor" }

-- A. 使用点号（.）定义与调用：必须自己显式声明 self
function soldier.attack(self)
    print(self.name .. " 斩出一剑！")
end

soldier.attack(soldier) -- 正常调用，但必须把实例自己作为参数手动灌进去，非常累赘！

-- B. 冒号（:）语法糖强势介入：
function soldier:shield_block() # 注意：这里用冒号，即使这里没声明 self，函数中也能直接使用 self！
    print(self.name .. " 举起盾牌格挡！")
end

soldier:shield_block() -- 也是冒号调用！不需要再手动传入 soldier 实例，解释器在后台自动灌入了 self。
```

---

## 2. 构造函数与类实例化

在 Lua 中，要创造一个“类”（Class），我们通常会：
1. 建立一个普通的 Table，比如叫 `BaseAccount`，让它保存所有的**方法**。
2. 将 `BaseAccount.__index` 指向它自己。
3. 提供一个类似 `new` 的普通方法。在该方法中，创建一个空实例 Table，并把它的元表指向 `BaseAccount`。这个空表就是我们的“实例”。

**完整的“类与构造函数”实现**：

```lua
-- 1. 定义一个充当模板的 table（相当于 C# Class 壳子）
local BankAccount = {}

-- 2. 关键核心：设定寻找 fallback 的源。
-- 当子实例找不到某种行为时，让它来我这个 BankAccount 模板大本营里找
BankAccount.__index = BankAccount

-- 3. 编写构造函数 new（对标 C# 中的 public BankAccount() 构造）
function BankAccount:new(owner, balance)
    -- a. 创建一个干净的无行为空表，用来装载纯独立的“实例数据”
    local instance = {
        owner = owner,         -- 实例特有字段
        balance = balance or 0 -- 实例特有字段
    }
    
    -- b. 将 instance 实例的元表，设定为 BankAccount 类模板
    setmetatable(instance, self) -- 此处的 self 在调用 BankAccount:new 时指的是 BankAccount 本身
    
    -- c. 返回这个绑定了元方法fallback的实例
    return instance
end

-- 4. 编写类的方法
function BankAccount:deposit(amount)
    self.balance = self.balance + amount
    print(self.owner .. " 存入 " .. amount .. " 元，当前总额: " .. self.balance)
end

-- 5. 【执行实例化】对标 C# 的 var acc = new BankAccount("Alice", 100);
local my_acc = BankAccount:new("Alice", 100)

-- my_acc 此时是一个普通的表，本身只有 {owner="Alice", balance=100}。
-- 当调用 deposit 时，my_acc 本身没有 deposit 属性，
-- 解释器会因为元表的 __index，追查到 BankAccount 的方法大本营。
my_acc:deposit(50) -- 输出: Alice 存入 50 元，当前总额: 150
```

---

## 3. 实现派生与多态（对标 C# `Inheritance`）

既然通过 `__index` 我们可以连接“实例”和“母类（Class）”，同样的原理，我们只要做一次**二级链接**，把“子类”的元表指向“父类”，就能实现极其顺滑的类继承！

**实战：让 VIPBankAccount 继承 BankAccount 并覆写方法（多态）**

```lua
-- 1. 声明子类容器表 (对标 C# public class VipAccount : BankAccount)
local VipBankAccount = {}

-- 2. 通过让子类的元表面向 BankAccount，继承 BankAccount 的所有方法
setmetatable(VipBankAccount, { __index = BankAccount })
VipBankAccount.__index = VipBankAccount

-- 3. 定义子类的构造函数
function VipBankAccount:new(owner, balance, discount_rate)
    -- 调用父类的核心初始化
    -- (利用点号显式穿入 self 来模拟 C# base 构造函数调用)
    local instance = BankAccount.new(self, owner, balance)
    
    -- 附加子类特有属性
    instance.discount_rate = discount_rate or 0.9
    
    -- 将其升级为 VipBankAccount 实例
    setmetatable(instance, self)
    return instance
end

-- 4. 【多态/覆写 (Override)】重写 deposit 方法
function VipBankAccount:deposit(amount)
    -- VIP 用户存款可以获得额外 10% 的利息赠送！
    local bonus = amount * 1.1
    self.balance = self.balance + bonus
    print("[VIP 专享] " .. self.owner .. " 存入 " .. amount .. " 元, 享受赠送后总额: " .. self.balance)
end

-- ==========================================
-- 测试运行：继承与多态展示
-- ==========================================
local normal_user = BankAccount:new("老张", 100)
local vip_user = VipBankAccount:new("小李", 100)

normal_user:deposit(100) -- 调用基类：老张 存入 100 元，当前总额: 200
vip_user:deposit(100)     -- 调用子类覆写：[VIP 专享] 小李 存入 100 元, 享受赠送后总额: 210
```

> **.NET 开发者贴士**：
> 通过以上多层链接，解释器在执行方法查找时的递归链条是：
> `实例(vip_user) -> 子类模板(VipBankAccount) -> 基类模板(BankAccount)`。
> 这说明 Lua 的继承在内部完全是不依赖硬性编译结构、自下而上极为灵活和通透的可拓展链条。

---

现在，你已经掌握了如何在解释器世界里游刃有余地构建你的大型面向对象系统。点击进入 **[lua/ch3_advanced.md](lua/ch3_advanced.md)**，我们将解决模块化、作用域、以及对标 C# `yield return` 的硬核多任务发动机 —— **协程（Coroutine）**！
