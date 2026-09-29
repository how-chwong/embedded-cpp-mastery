# 第7章：人工智能与人工智能实战——scikit-learn 与 YOLO 极速入门

对于不少 .NET 工程师而言，转型 Python 的最大诱因往往就是 **AI 和机器学习（Machine Learning）**。
虽然 .NET 生态有 ML.NET，但不可否认，全球 AI 的前沿科研、模型训练和生态完全由 Python 垄断。

在本章中，我们将通过两个极具代表性的 AI 实战，带你跨入这个世界的大门：
1. **经典机器学习：使用 `scikit-learn` 预测数据分类（鸢尾花分类）**。
2. **现代深度学习/计算机视觉：使用著名的 `YOLOv8` 模型进行图像目标检测**。

---

## 1. 机器学习时代的“瑞士军刀”：`scikit-learn`

在传统数据挖掘与机器学习中，[scikit-learn](https://scikit-learn.org/)（简称 `sklearn`）是绝对的霸主。它内置了数据预处理、特征工程、回归、分类、聚类、降维等全套算法。

### 核心开发步骤（机器学习的标准流水线）
几乎所有的机器学习任务，都遵循以下五步大法：
1. **获取数据**：加载数据集。
2. **切分数据集**：分为“训练集”（Training Set）用于让模型学习，和“测试集”（Test Set）用于闭卷考试评估成绩。
3. **选择模型并训练**：创建模型实例，调用 `.fit(X_train, y_train)` 进行拟合（训练）。
4. **测试并预测**：调用 `.predict(X_test)` 测试模型对未知数据的猜测能力。
5. **评估成绩**：计算准确率（Accuracy）等指标。

### 实战：10行代码实现鸢尾花（Iris）种类智能预测
鸢尾花数据集是机器学习界最经典的“Hello World”。它包含了 150 朵鸢尾花的四个物理特征：花萼长度、花萼宽度、花瓣长度、花瓣宽度，并据此将它们分为三类。

#### 1. 安装依赖包
在激活的 `.venv` 虚拟环境中，安装 `scikit-learn` 和基础数据处理库 `pandas`（这还会自动下载数据基础库 `numpy`）：
```bash
pip install scikit-learn pandas
```

#### 2. 新建并编写 `iris_ml.py`
创建文件并填入以下代码，我们使用轻量、经典的“决策树（Decision Tree）”分类器：

```python
# 引入 sklearn 的数据集、模型以及分割工具
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split
from sklearn.tree import DecisionTreeClassifier
from sklearn.metrics import accuracy_score

# 1. 载入内置的鸢尾花数据集
iris = load_iris()
X = iris.data  # 特征矩阵 (花萼/花瓣长宽)
y = iris.target  # 目标类别 (标签: 0, 1, 2 分别代表三种不同的鸢尾花)

# 2. 将数据随机切分：80% 用于训练模型，20% 用于保留测试（闭卷考试）
# random_state 相当于随机数种子，确保每次运行切分结果一致
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

# 3. 选择决策树模型作为我们的大脑（类似于 C# Class 实例化）
model = DecisionTreeClassifier(max_depth=3)

# 4. 训练模型（".fit" 是机器学习的核心，模型在这里寻找特征与标签的数学关系）
model.fit(X_train, y_train)

# 5. 用测试集对未知的花进行预测
y_pred = model.predict(X_test)

# 6. 计算成绩
accuracy = accuracy_score(y_test, y_pred)
print(f"🎉 决策树模型预测准确率: {accuracy * 100:.2f}%")

# 7. 模拟一个全新的未知花朵数据（长宽物理数据），让模型猜它是哪一种
new_flower = [[5.1, 3.5, 1.4, 0.2]] # 类似一朵山鸢尾 (Setosa)
predicted_class = model.predict(new_flower)
predicted_name = iris.target_names[predicted_class[0]]
print(f"🔮 输入特征 {new_flower}，模型智能诊断该花为: {predicted_name}")
```

#### 3. 运行它
在控制台输入：
```bash
python iris_ml.py
```
这会秒级训练完毕，并在屏幕上打印出接近 **100% 准确率**的预测分类输出！

---

## 2. 现代计算机视觉巨擘：`YOLO` 目标检测

`YOLO` (You Only Look Once) 是近年计算机视觉（CV）中声名赫赫的实时目标检测算法。它能够以极高的帧率，瞬间识别出图像或视频中的人、车、猫、狗等上百类日常物体，并返回它们的坐标。

这里我们使用 [Ultralytics](https://github.com/ultralytics/ultralytics) 团队维护的目前最易用、最先进的 **`YOLOv8`**（或更新版本）。

### 目标检测的工作原理
你无需从头训练一个千亿参数的 CNN（卷积神经网络）。YOLO 官方提供了丰富的**预训练模型（Pre-trained Models）**（是在巨量全球图片上已经训练好、具备常识认知的骨架文件，通常为 `.pt` 后缀）。
你只需直接加载骨架文件并进行**推理（Inference）**，就能快速获得识别结果。

### 实战：10行代码检测图片中所有的物体

#### 1. 安装 YOLO 依赖包
```bash
pip install ultralytics pillow
```
*注：`ultralytics` 包含了完整的 PyTorch 深度学习引擎，首次安装会下载较大的包（一般为几十到几百兆，请耐心等待）。*

#### 2. 新建并编写 `yolo_cv.py`
创建此文件，在此示例中，模型会**自动下载**一张测试图片和基础版预训练小模型 `yolov8n.pt`（仅数兆大小），接着在本地完成物体定位：

```python
from ultralytics import YOLO
import urllib.request

# 1. 下载一张带有多种日常物体的示例图片 (比如猫和狗)
image_url = "https://ultralytics.com/images/bus.jpg"
image_path = "bus.jpg"
print("正在从网络获取测试图片...")
urllib.request.urlretrieve(image_url, image_path)

# 2. 加载一个官方预训练的 YOLOv8-nano 目标检测模型 (轻量极速，适合本地 CPU 运行)
# 第一次运行此代码时，它会自动从官方下载 yolov8n.pt 权重文件到当前目录
model = YOLO("yolov8n.pt")

# 3. 对图片进行目标检测（推理）
results = model(image_path)

# 4. 遍历识别出的每个物体信息
print("\n--- 🕵️‍♂️ 图像物体检测报告 ---")
for result in results:
    for box in result.boxes:
        # 获取物体的类别数字（如 0 代表人，5 代表公交车 等）
        class_id = int(box.cls[0])
        # 将数字映射为物体的英文名字
        class_name = model.names[class_id]
        # 获取模型的置信度（Confidence, 即有百分之多少的把握）
        confidence = float(box.conf[0])
        # 目标在图片中的像素坐标边界框：[左上x, 左上y, 右下x, 右下y]
        coordinates = box.xyxy[0].tolist()
        
        print(f"[{class_name.upper()}] 置信度: {confidence:.2%}, 坐标位置: {[round(c, 1) for c in coordinates]}")

# 5. 可选：YOLO 会自动把框画在图片上，并保存一张可视化后的渲染图在 runs/detect/predict/ 目录下
result.save()
print("\n🎉 检测完成！画好标记坐标的合成图已经保存在当前目录下的 'runs/' 文件夹内。")
```

#### 3. 运行它
在命令行输入并运行：
```bash
python yolo_cv.py
```
你将看到控制台高能输出了图片里的乘客（`PERSON`）、公交车（`BUS`）的置信度与对应像素方框坐标！
同时项目根目录下会自动生成一个 `runs` 文件夹，里面那张画好了漂亮置信度框框的图片就是 AI 识别的最佳见证。

---

## 3. 为什么是 Python？从 C# 和 C++ 的视角来看 AI 帝国

许多 .NET 开发者会问：C# 性能这么强，为什么深度学习在 C# 却无法流行？
1. **底层全 C++**：深度学习的大型引擎（如 PyTorch、TensorFlow）底层都是极其复杂的 C++ 代码以及 GPU 自带的 **CUDA/CUDNN** 库。
2. **胶水粘合的艺术**：Python 拥有和 C++ 进行二进制互相调用（C-Bindings）最成熟的机制（如 `C-Extensions` / `Pybind11`）。深度学习库表面上是用 Python 编写，但每一次 `.fit()` 或 `model()`，都是在用 Python 充当“胶水”，瞬间通知下层的 C++ 甚至直接命令 GPU 显卡执行高维矩阵乘法运算。
3. **完美的动态解释性**：由于 AI 算法研究需要频繁修改参数、热改动网络结构并即时观察矩阵维度变化，Python 的动态类型与交互式特性（配合 Jupyter Notebook）天然比需要严格编译、严格类型匹配的 C# 具有百倍的研发迭代效率。

通过本章的洗礼，你已经用极低的代码代价掌握了分类预测和深度感知两套武功！如果你准备好成为一个 AI 应用层的弄潮儿，请返回主目录，继续探索更多充满想象力的 Python 架构！
