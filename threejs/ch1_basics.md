# 第1章：3D 世界的三大基石与场景级联树（Three.js 篇）

在 C# 或者是 Unity 中，要呈现一个 3D 空间，你至少需要：一个 3D 容器、一个决定人眼观察视角的镜头以及底层的 Direct3D 或 Vulkan 驱动。

在 Three.js 中，这完美的对应了我们称之为 **3D 渲染黄金三角** 的“三大基石”：
1. **Scene (场景)**：无限大、装载一切的容器大舞台（包含几何体、灯光）。
2. **Camera (相机)**：导演的眼睛，决定我们把 3D 世界切下来多宽并如何看它（透视 vs 正交）。
3. **Renderer (渲染器)**：幕后工匠，负责将 3D 物体换算为屏幕 Canvas HTML 上的 2D 彩色像素。

本章将带你深入理清这三者的心智模型，并解释对标 C# 的 3D 父子空间级联。

---

## 1. 场景 (Scene) 与“场景图 (Scene Graph)”

在 C# 的 UI 系统（如 WPF）中，你习惯了使用 **Visual Tree (视觉树)** 或 **Logical Tree** 来层层包含对象（例如 Panel 里面放 Canvas，Canvas 里面放 Button）。
在 Three.js 中，管理一切 3D 对象的底层数据结构叫 **“场景图 (Scene Graph)”**。它本质上是一棵 **n 叉树**。

- 树的根节点是我们的 `Scene` 实例。
- 每一个节点都是一个 3D 物体（基于 `Object3D` 基类，比如 `Mesh`, `Group`, `Light`, `Camera`）。
- **空间级联继承（Transform Nesting）**：这也是本章的核心概念。如果物体 A 是物体 B 的父节点，那么**当 A 发生移动、旋转或拉伸时，B 也会带着自己局部的相对坐标，一同跟着父节点漂移！**

### 实战：父子级联旋转（模拟“地球绕着太阳转”）
这也是展示场景树的经典例子。我们创建一个空节点“太阳”，再创建一个“地球”作为子节点并拉开偏移。只旋转太阳，“地球”就会由于级联关系自动绕着太阳完成完美的公转。

```typescript
import * as THREE from 'three';

// 假设我们已经按照第 0 章初始化好了 scene 和 camera、renderer

// 1. 创建一个空节点群组 Group（代表“太阳系”旋转中心体系，对标 C# 中的 3D Transform Node）
const solarSystemGroup = new THREE.Group();
scene.add(solarSystemGroup); // 放入主场景根节点

// 2. 创建核心主星“太阳”（金色球体）
const sunGeometry = new THREE.SphereGeometry(0.8, 16, 16);
const sunMaterial = new THREE.MeshBasicMaterial({ color: 0xffaa00, wireframe: true });
const sun = new THREE.Mesh(sunGeometry, sunMaterial);
// 我们直接把太阳塞进太阳系 Group 内
solarSystemGroup.add(sun);

// 3. 创建绕转的“地球”（海蓝色球体）
const earthGeometry = new THREE.SphereGeometry(0.3, 12, 12);
const earthMaterial = new THREE.MeshBasicMaterial({ color: 0x0088ff, wireframe: true });
const earth = new THREE.Mesh(earthGeometry, earthMaterial);

// 【核心高能点】：不把地球塞入主 scene，而是把地球设置为太阳系 Group 的子节点
solarSystemGroup.add(earth);

// 将地球在这个旋转坐标系内，往右侧推开偏移 3 个单位。这就是它跟太阳的相对距离！
earth.position.x = 3;

// 4. 下在每一帧的动画主循环中：
function tick() {
  requestAnimationFrame(tick);

  // 仅仅旋转整个主群组，由于地球作为子节点挂在它身上，会自动实现绕原点公转！
  solarSystemGroup.rotation.y += 0.015;

  // 地球也可以有自己的局部自转：
  earth.rotation.y += 0.05;

  renderer.render(scene, camera);
}
tick();
```

---

## 2. 摄像机 (Camera)：透视（Perspective） vs 正交（Orthographic）

我们在 C# 或者 WPF 里要将 3D 物体投映到 2D 的窗口需要声明。在 Three.js 中，最常使用的摄像机主要有两个，它们面对完全不同的业务场景：

### A. PerspectiveCamera (透视相机) - 首选最像人眼体验
* **特点**：**“近大远小”**。离摄像机越近的物体看起来越大，离得越远看起来越小（类似于铁轨在远方相交于一点）。
* **应用类型**：3D 游戏、室内仿真漫游、数字孪生工厂、3D 视觉展示。
* **构造器参数详解**：
  ```typescript
  const camera = new THREE.PerspectiveCamera(fov, aspect, near, far);
  ```
  - `fov` (Field of View)：**视场角（角度）**。决定视野有多宽（就像换单反相机的广角还是长焦镜头，一般 `45` - `75` 最佳）。
  - `aspect`：**横纵比例**。必须与你画布的宽高（`width/height`）强绑定，否则渲染出来的魔方或圆球会发生令人崩溃的拉伸变形。
  - `near` & `far`：**近端和远端距离裁剪面**。超出这两个限制区间的模型会被 GPU 自动剔除丢弃以节省显卡算力。

### B. OrthographicCamera (正交相机) - 类似微缩平面
* **特点**：**“大小与距离无关”**。无论物体离镜头多远，其渲染出来的像素尺寸纹丝不动。
* **应用类型**：工业 CAD 软件建模视图、2.5D 平铺策略设计图大屏、对排版和比例要求极其死板的平面数据图。
* **构造器参数详解**：
  ```typescript
  // 必须传入视锥体的左、右、上、下、近、远六个物理边界值
  const camera = new THREE.OrthographicCamera(left, right, top, bottom, near, far);
  ```

---

## 3. 渲染器 (Renderer) 与 高清视网膜 DPI 适配

很多人写完 3D 代码运行后，发现虽然能转，但模型的边缘有很多不平滑的“白边锯齿”或者在高分屏（Retina / 4K / 2K 屏）上看起来一派模糊。
这通常是因为：
1. 没有开启抗锯齿。
2. 渲染器的输出像素比例与屏幕的硬件物理像素比例（Device Pixel Ratio）不匹配（这在 C# 开发中对应 **DPI-Awareness DPI感知机制**）。

### 最佳推荐：高质量 WebGL 渲染器初始化标准：
在写新 Three.js 项目时，请直接带上这套高能物理 DPI 识别初始化模板，它能确保不论是在普通的 4K 显示器还是在最新的 iPhone Retina 屏上，你的 3D 模型都能渲染得如丝般顺滑、刀锋般锐利：

```typescript
// 1. 初始化 WebGL 渲染器并开启抗锯齿
const renderer = new THREE.WebGLRenderer({
  canvas: canvas,
  antialias: true,            // 允许抗锯齿：利用硬件和算法抹平多重物体的边缘毛刺
  powerPreference: "high-performance" // 强推荐：呼唤移动端或双显卡笔电优先调用高性能独立显卡，而不是核显
});

// 2. 物理画布自适应大小与物理 DPI 渲染匹配（极其重要！）
const updateSize = () => {
  const width = window.innerWidth;
  const height = window.innerHeight;

  // 更新摄像机的物理宽高比
  camera.aspect = width / height;
  camera.updateProjectionMatrix();

  // 更新渲染器所占用的 HTML 样式大小
  renderer.setSize(width, height);

  // 【避坑核精】：将渲染器的物理像素输出比限制为屏幕的高清物理像素比例
  // 为了防止某些极高 DPI 屏幕导致 GPU 功耗过载，我们限制最高到设备的 2 倍分辨率，这就已经达到了人眼无法分辨高清画质的最佳均衡点！
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
};

updateSize();
window.addEventListener('resize', updateSize);
```

---

现在，三大基石的底层逻辑已经在你的大脑中扎下了坚不可摧的基础。点击 **[threejs/ch2_geometries_materials.md](threejs/ch2_geometries_materials.md)**，我们将触及真正的视觉艺术 —— 让我们看看如何雕琢出各种复杂的几何体并给它挂载现代游戏标准的物理 PBR 反光材质！
