# 第2章：定义神经网络（nn.Module）与工业级 5 步训练循环主干（PyTorch篇）

作为一名称职的 .NET 工程师，你习惯了使用面向对象、依赖注入（DI）、多层架构来建立严谨的主系统工程。
在 PyTorch 中，编写神经网络绝对不是无规律的写随性脚本，它同样也有一套极规范的**工业级结构模型**。

本章我们将：
1. 学习如何通过继承 **`torch.nn.Module`** 基类来层层叠叠拼装出你的核心网络架构模型。
2. 详解横扫深度学习全平台的 **“5 步训练黄金循环法（The 5-Step Training Loop）”**，让你不管在写房价回归、图像识别，还是大型 Transformer 微调时，都能像标准的 C# 业务流一样清晰掌控全盘生命周期进程。

---

## 1. 拼装核心模型骨架（nn.Module）

在 PyTorch 生态中，所有的神经网络（无论是简单的单层线性回归，还是超复杂的 ResNet、Transformer）在代码继承树里面，都**必须百分之百继承 `torch.nn.Module` 这个核心基类**。

这个基类有两个绝对首要的构成重任：
- **`__init__(self)`** 构造器：你得在此声明并实例化整个网络中所需要用到的各类**待学习参数神经层**（如 Linear 线性全连接层、Conv2d 卷积图像处理层等）。这相当于在 C# Entity 或 Service 构造函数里进行成员属性依赖注入。
- **`forward(self, x)`** 前向传播方法：当你的数据 $x$ 顺次送入该模型时，它该如何串行、级联、跳跃地流经你上面的定义好层，并最终返回一个预测输出 $y_{pred}$。这就是你神经网络的工作管线流！

### 实战：定义一个三层全连接前馈神经网络

```python
import torch
import torch.nn as nn # nn 是神经网络（Neural Networks）的极简缩写

# 1. 继承 nn.Module (对齐 C# class Model : BaseModule)
class DotNetFirstNet(nn.Module):
    def __init__(self, input_dim: int, hidden_dim: int, output_dim: int):
        # 必须显式激活并运行父类 nn.Module 核心底层逻辑的初始化 (C# base())
        super().__init__()
        
        # 声明我们的各种待学习权重神经网络硬件元件：
        # nn.Linear 对标经典的 W*x + b 的线性映射权重层
        self.hidden_layer = nn.Linear(input_dim, hidden_dim)  # 输入层 -> 隐蔽层
        self.relu = nn.ReLU()                                 # 负值归零的非线性激活激活器
        self.output_layer = nn.Linear(hidden_dim, output_dim) # 隐蔽层 -> 最终结果输出层

    # 2. 定义数据顺流前向传播的流过逻辑
    def forward(self, x: torch.Tensor) -> torch.Tensor:
        # a. 顺次流过隐藏全连接
        out = self.hidden_layer(x)
        # b. 经过非线性过滤（没有非线性激活，你几万层全连接本质上也只是一层线性，将毫无智能）
        out = self.relu(out)
        # c. 顺流到输出层，吐出模型算出来的预测猜测值
        out = self.output_layer(out)
        return out

# 测试：创造一台带有 8 个输入特征、16层中间特征、1层输出的三层模型电脑模型
model = DotNetFirstNet(input_dim=8, hidden_dim=16, output_dim=1)
print(f"🤖 模型装配完毕，骨架信息如下:\n{model}")
```

---

## 2. 深度学习不得不谈的两大核心伴侣：损失函数与优化器

要让上面的模型能够自我学习更新，你必须为它配置在训练场里的两位贴身助理：
- **损失函数 (Loss Function)**：用来当标卷考官。在它眼里，会物理比对你模型的**猜测值**与真实的**事实标准答案**（Label），算出一个“偏差大到多么离谱的亏损数字值（Loss）”。（如常用于回归预测的 **均方误差 MSELoss**）。
- **优化器 (Optimizer)**：用来当指导改进系数的导师（如著名的 **`SGD` 梯度下降** 或者是 **`Adam` 自适应算法**）。
  - **核心职能**：当后台 `Loss.backward()` 自动求得每一层所有变量当期的微积分梯度偏导后，优化器负责**领头执行修正动作**：用新一期最佳的物理差值，去挨个修正那几亿个 requires_grad 参数。由此完成反向优化。

---

## 3. 金金招牌：经典 5 步训练循环标准流

作为 .NET 架构师，你肯定知道：写项目需要遵循标准的工业骨干流（GameLoop、Request-Response Pipeline、或者 EventLoop）。
深度学习的世界极其讲究，每一轮迭代（Episode / Epoch），在工程代码中，都**被硬性规范为极其稳妥、环环相扣的“5 步大循环”**。

```mermaid
flowchain
    A[第1步: 前向计算 Forward] --> B[第2步: 计算误差 Loss]
    B --> C[第3步: 梯度清零 zero_grad]
    C --> D[第4步: 反向传播 backward]
    D --> E[第5步: 优化更新 step]
```

### 【硬核高能】5 步标准模板精细详解：

```python
# 假设我们手里已经有了模型 model、输入数据 X_train、事实标准答案 y_train、
# 以及优化器 optimizer（它绑定了 model 的所有 requires_grad 权重）、损失评估器 criterion。

# ====================================================================
# 【5 步循环黄金内核，千锤百炼不移其志！】
# ====================================================================

# 第 1 步：模型根据特写当前的能力，迎头进行一次前向计算
y_pred = model(X_train)

# 第 2 步：考官比对预测猜测值与真实黄金标准答案，算得当前总落后偏差 Loss 
loss = criterion(y_pred, y_train)

# 第 3 步：【核心黄金避坑第一秒】：这一行极其致命！
# 必须显式清空上一次残留在内存/GPU 显卡深处的旧导数梯度！
# 因为在 PyTorch 的默认特性中，如果不显式清空，新一轮 backward 出来的梯度会“一并不断累加（Accumulate）”到旧梯度上，从而导致参数失序乱跳，无法正常训练！
optimizer.zero_grad()

# 第 4 步：影子魔法发出指令！自动动态反向求导！
# 瞬间在每一个被绑定的 requires_grad 参数物理插槽中，算好并注入当前偏导微积分梯度值
loss.backward()

# 第 5 步：导师大刀阔斧执行修正！
# 优化器（Adam/SGD）利用第 4 步新鲜出炉的最新梯度，配合设定的学习率，集体把网络的全部参数向误差最小、最佳的反方向拽升一步更新！
optimizer.step()
```

> **.NET 开发者贴士**：
> 不论你在进行多么伟大的 AI 革命（是在做人脸检索、还是微调类似于 Llama 3 这样的大型文本对话大模型），这套 5 步大闭环在框架的底层永远原封不动、金石不易地在后台奔跑。只要你深深把它们印刻进大脑心智中，你就已经彻底掌控了大型 AI 的运行轨迹。

---

现在，神经网络的核心拼装与工业级标准 5 步大循环已被你全部收服。点击阅读最后的金牌实战 **[pytorch/ch3_regression_demo.md](pytorch/ch3_regression_demo.md)**，我们将正式使用 NumPy 自制一批拟真物理数据，真正起动一具线性回归预测脑，并亲眼见证损失值是如何一步步平顺收敛、降阶并获得极高精准预测能力的奇妙视觉过程！
