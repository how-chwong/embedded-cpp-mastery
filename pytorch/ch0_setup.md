# 第0章：极速起步、GPU 显卡验证与调试沙盒（PyTorch篇）

在传统 .NET 开发中，你要写显卡 GPU 级计算通常需要复杂的 **DirectX / DirectCompute** 编写或是利用极其生硬的 C# CUDA C-Bindings 翻译。
但在 Python 下，**PyTorch** 早就把这一切包装得极其雅致：你甚至可以在完全不需要知道 GPU 底层任何寄存器细节的情况下，一行代码调用数万门 CUDA 核并发。

本章我们将：
1. 学习如何下载最适合你电脑配置的 PyTorch。
2. 验证本地的英伟达显卡（**NVIDIA CUDA**）是否能为 PyTorch 驱动加速（如果不支持，如何优雅、无缝地在 CPU 下开展学习调试）。
3. 学习调试深度学习模型张量数据形态的最佳控制台技巧。

---

## 1. 安装 PyTorch 解释包

深度学习环境通常体积庞大。因为 PyTorch 不仅包含 Python 表面命令，还默默地整合了 **PyTorch C++ 底层 C10 计算库** 和针对 CUDA 显卡的大尺寸二进制内核。

### 一秒看透：英伟达 CUDA 对应关系
如果你拥有好用的 NVIDIA 独立显卡，开启 GPU 选项可以让你的神经网络运算体验到几百倍的提速：
- 前往命令行运行命令检查 NVIDIA 是否正常：`nvidia-smi`
- 如果看到支持的 CUDA 最高版本号（例如 `CUDA Version: 12.x` 或 `11.x` ），你就拥有了最牛的加速器。

---

### 🎨 下载命令（极简精选）

进入你之前创建好的局域 **`.venv` 虚拟环境**（或者用 Anaconda 创建的纯净隔离沙盒），运行以下命令。如果你不确定显卡，直接选择 CPU 或者经典 CUDA 11.x/12.x：

#### 选项 A：拥有 NVIDIA 独显（GPU 豪华加速版）
```bash
# Windows / Linux：下载支持 CUDA 12.1 加速的高精 PyTorch
pip install torch --index-url https://download.pytorch.org/whl/cu121
```

#### 选项 B：普通电脑、核显电脑（最稳妥 CPU 版）
对于我们快速学习基本原理，CPU 算力就已经绝对足够！
```bash
pip install torch
```

---

## 2. 验证你的 GPU 状态：一击验证脚本

在工程项目中，我们最希望的是：**写一份代码，在有显卡的地方自动用显卡跑，没显卡的地方自动无缝降级到 CPU，不影响业务流。**

让我们新建并编写物理文件：**`test_torch_gpu.py`** 并在本地运行：

```python
import torch

print("🚀 --- 开始 PyTorch 环境健康侦测 --- 🚀\n")

# 1. 验证 PyTorch 核心库版本
print(f"1. 当前本地安装的 PyTorch 版本是: {torch.__version__}")

# 2. 检查 CUDA (GPU 加速硬件) 底层在当前 Python 环境下是否可连
cuda_available = torch.cuda.is_available()
print(f"2. 英伟达 GPU CUDA 核心在虚拟机中是否启用: {cuda_available}")

if cuda_available:
    # 3. 打印当前独立显卡的物理名称：
    device_name = torch.cuda.get_device_name(0)
    print(f"  ├─ 物理显卡型号: {device_name}")
    print(f"  ├─ 可用 CUDA 显卡数: {torch.cuda.device_count()}")
    
    # 4. 创建一记可以直接在 GPU 显存上并发做运算的纯显卡 Tensor！
    gpu_tensor = torch.tensor([1.0, 2.0, 3.0]).cuda() # 一键 .cuda() 秒入显存
    print(f"  └─ GPU Tensor 测试计算成功，位置: {gpu_tensor.device}")
else:
    print("  ⚠️ 本地未检索到 NVIDIA 显卡或 CUDA 驱动未开启。")
    print("  💡 不要担心！PyTorch 会极其优雅地启用【纯 CPU 计算模式】，对我们后续的所有神经网络学习与编码测试完全没有任何阻碍。")

# 5. 【高保真】推荐写一份随时准备自适应切换 GPU/CPU 的代码结构：
# 这在深度学习界被称为定义 "device" 黄金底线（以后要让哪个模型在显卡跑，直接写 .to(device) 即可！）
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
print(f"\n3. 以后项目中的黄金自适应运算大本营设定为: {device}")
```

### 运行结果：
在终端中，运行此命令：
```bash
python test_torch_gpu.py
```
你将直接看透你自己本地电脑的 AI 底盘健康报告。

---

## 3. 【极易忽视】深度学习的调试特点：查看张量（Tensor）的形状与位置

作为习惯了 Visual Studio 强类型 C# 编译大厦的你，需要知道：
- 在传统 C# 中，我们只要双击变量就能在 IDE 观察到完整的类型、长度和物理数据。
- 但在深度学习中，绝大部分报错，**百分之八十以上，全都是因为相乘的两个矩阵“维度不匹配”（Shape Mismatch）**。比如一个 $3 \times 4$ 的矩阵，强行要去跟一个 $5 \times 2$ 的矩阵发生相乘运算。
- 此时，在任何 PyTorch 断点调试中，你最常用、最重要的两个“上帝之眼属性”就是：
  * **`.shape`**：查看张量的**各个维度长度**（如显示 `torch.Size([32, 1, 28, 28])` 代表有 32 张单通道 28x28 像素的图片）。
  * **`.device`**：查看张量正躺在 **`cpu`** 内存，还是已经飞入 **`cuda:0`** 显卡。

---

现在，健康的 AI 底牌健康校验已通过，代码也已验证就绪。点击阅读 **[pytorch/ch1_tensors.md](pytorch/ch1_tensors.md)**，我们将正式跨过数学屏障，揭秘构成 PyTorch 甚至神经网络大模型整个物理大厦的唯一黏砖 —— **张量（Tensor）与自动梯度求导机制（Autograd）**！
