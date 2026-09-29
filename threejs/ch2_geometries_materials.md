# 第2章：几何体、纹理贴图与 PBR 物理材质（Three.js 篇）

在 C# 或是传统图形开发中，画一个 3D 模型需要定义**顶点缓存（Vertex Buffer）、法线缓存（Normal Buffer）和 UV 纹理坐标**。
在 Three.js 中，这个繁琐的过程被解耦并拼装成一个极其富有语义的、面向对象的结构。

一个可以在浏览器画面上看到的 3D 实体，物理上被统称为 **Mesh（网格物体）**。它通常是由以下两个底层元件组成：
$$\text{Mesh} = \text{Geometry (几何体)} + \text{Material (材质)}$$

本章我们将像三维雕刻师一样，搞清楚如何操纵物体的骨架（形状）和灵魂（反光材质）。

---

## 1. 骨架：几何体 (Geometry) 的秘密

一具几何骨骼，本质上就是通知 GPU 在显存中建立一块物理的**顶点坐标集、法向向量和 UV 贴图坐标**。

```
                    (Vertex)
                     (0, 1, 0)
                       / \
                      /   \
                     /     \
                    /_______\
               (0,0,0)     (1,0,0) (Vertices)
```

Three.js 提供了极其丰富和完备的开箱即用几何形状（你无需通过顶点数组手算三角剖面）：
- **`BoxGeometry`**: 经典六面立方体形状。
- **`SphereGeometry`**: 高模/低模球体。参数中的长宽分段数（`widthSegments / heightSegments`）越大，球体会越圆润；但开销同样变大。
- **`PlaneGeometry`**: 2D 矩形双面扁平网格，一般拿来当做 3D 世界的地表、墙面、或者背景。
- **`CylinderGeometry`**: 圆柱体/圆锥体。

### 【高阶硬核】BufferGeometry 自定义几何体：对标 C# DirectX 顶点映射缓存
如果你在 C# 中写过客自制 3D 数据引擎，你可能需要根据传感器传回的物理 X/Y/Z 点云序列直接合成形状。在 Three.js 中，你可以通过 `BufferGeometry` 在显存中建立全定制自由 3D 面片。

```typescript
// 建立一个全自制三角形骨骼
const customGeometry = new THREE.BufferGeometry();

// 定义 3 个顶点的 X, Y, Z 三维浮点数组
const vertices = new Float32Array([
  0.0, 1.0, 0.0,  // 第一点 (顶端)
  -1.0, 0.0, 0.0, // 第二点 (左下)
  1.0, 0.0, 0.0   // 第三点 (右下)
]);

// 将顶点数据塞入显存缓存区，命名绑定为 'position' 专有标记（GPU 着色器的经典引接）
customGeometry.setAttribute('position', new THREE.BufferAttribute(vertices, 3));
```

---

## 2. 皮肤：材质 (Material) 家族与现代 PBR 着色

在 graphics 架构中，决定一个模型看起来是像光滑白嫩的瓷器、还是像反射太阳金属光的跑车外壳，这属于底层 **Shader（着色器）** 的工作。

Three.js 帮我们预装好了五个统治级的核心材质类。作为程序员，你需要了解它们的性能和能效对比：

| 材质类名                   | 是否受光照影响 | 性能开销         | 说明与 .NET 对齐                                                                                                                                                                   |
| :------------------------- | :------------- | :--------------- | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **`MeshBasicMaterial`**    | **否**         | 极低（极省 GPU） | 纯平面色。哪怕全场塞满红光，它依然是死板的原色。不进行任何像素光照法向计算。常用于做辅助包围盒或发光线框。                                                                         |
| **`MeshMatcapMaterial`**   | **否**         | 低               | 模拟反光的高妙手段。它直接读取并投影一张球状的环境光常识图片（Matcap），**不依靠物理灯光，就能在 CPU 里贴出一套令人惊艳的拟真反光质感**。极佳的前端移动端大屏性能提速方案！        |
| **`MeshLambertMaterial`**  | **是**         | 中               | 早期传统的非物理材质。采用 Gouraud 明暗着色，反光效果生硬（常有块状色斑）。适合对光影精度要求极低的简易背景模型。                                                                  |
| **`MeshPhongMaterial`**    | **是**         | 中高             | 经典质感。Phong 光照模型（镜面反射）。适合做极其光滑反光、自带高光斑点（Specular Highlights）的材质，如镜子、光油塑料、抛光金属。                                                  |
| **`MeshStandardMaterial`** | **是**         | 高               | **现代工业级首选：PBR (基于物理的渲染)**。它根据现实中真实金属、粗糙度材料的物理反射比进行方程运算。效果逼真耀眼，能模拟出和传统虚幻引擎（Unreal）、Unity URP 极其一致的物理反射。 |

### ⚠️【黄金避坑指南：黑屏大怪兽】
很多新手兴致勃勃地将材质参数设为 **`MeshStandardMaterial`** 或者 **`MeshPhongMaterial`**，却惊奇地发现运行后屏幕直接是一片死黑（看不见魔方）。
**这是因为这两种高精度材质必须依靠“光源（Lights）”来折算漫反射和镜面反光！如果你的场景中没有往 scene 里塞入灯光，那么哪怕显卡再强，像素折算出来也是黑暗状态。** 

---

## 3. PBR 两个大招：金属性（Metalness）与 粗糙度（Roughness）

如果是 PBR 的 `MeshStandardMaterial`，你不需要繁琐的物理方程设定。你只需要调控以下两个位于 `[0.0, 1.0]` 区间内浮点值，就能瞬间实现 90% 物理真实感材料的合成：

- **`roughness` (粗糙度)**: `0.0` 代表像冰面或镜子一样绝对镜面反光；`1.0` 代表像老旧的水泥路或厚纸一样，完全没有明亮反射区（漫反射）。
- **`metalness` (金属感)**: `0.0` 代表像木头、塑料、皮肤等非金属，反光只集中在高光表面；`1.0` 代表像不锈钢、黄金、铜等金属，能够完全像镜子一样镜像和吃透周围物体的反光色彩。

### 实战：配制一个高保真拉丝钢球

```typescript
// 1. 创建高精球体
const sphereGeom = new THREE.SphereGeometry(1.5, 64, 64); // 水平分段 64 会极为柔和圆润

// 2. 装配 PBR 物理材质
const steelMaterial = new THREE.MeshStandardMaterial({
  color: 0xcccccc,      // 基础银灰白色
  roughness: 0.15,      // 极低粗糙度：使其周围具备清晰、光滑的反光区
  metalness: 0.85,      // 高金属感：使反光处吃显外部反光面
  wireframe: false
});

// 3. 融合成实体网格
const steelSphere = new THREE.Mesh(sphereGeom, steelMaterial);
scene.add(steelSphere);

// 4. 【再次提醒】：因为是 PBR 材质，我们必须往 scene 塞一盏大灯，不然依然是黑色的！
const directionalLight = new THREE.DirectionalLight(0xffffff, 1.5);
directionalLight.position.set(5, 5, 5); // 悬停在右上方 45度
scene.add(directionalLight);

// 再塞一盏环境柔和光，模拟空气中漫射的反光，避免暗部完全黑死
const ambientLight = new THREE.AmbientLight(0xffffff, 0.4);
scene.add(ambientLight);
```

---

## 4. 纹理贴图加载：TextureLoader 

如果想要在 3D 特效中生成一个木箱或地球，你不可能仅用单一纯色，你需要把一张 2D 的图片包装覆盖在 3D 立方体各个表面。这就需要 **纹理贴图加载器**。

在 Three.js 中，通过内置的 `TextureLoader` 完美对标 C# 中的 `Bitmap/Texture Image` 加载过程。

```typescript
// 1. 实例化纹理加载器
const textureLoader = new THREE.TextureLoader();

// 2. 加载图片。注意，Vite 默认会让 public 目录下的静态文件可以直接引接
// 所以我们只需保证图片放在 public 目录下即可：如 public/textures/wood.jpg
const woodTexture = textureLoader.load('/textures/wood.jpg');

// 3. 完美结合到 PBR 的颜色通道（.map）中！
const boxGeom = new THREE.BoxGeometry(1.5, 1.5, 1.5);
const boxMat = new THREE.MeshStandardMaterial({
  map: woodTexture, // 这一步会将木纹图片完美糊在立方体上！
  roughness: 0.4
});

const crate = new THREE.Mesh(boxGeom, boxMat);
scene.add(crate);
```

> **.NET 开发者贴士**：
> 在高端 Web 渲染大屏中，PBR 材质还会额外加载：[法线贴图](#) `.normalMap`（用图片 RGB 的色彩通道来模拟凸凹不平的物理小细节如拉丝缝隙，免去几百万的多余顶点计算）、`.roughnessMap`（不同色块粗糙度不一，比如木箱上有泥斑的地方粗糙，其他完好地方光滑）。合理运用它们能让你在浏览器中画出和现代高精主机大作画质完全一致的高画质大屏！

---

现在，完美的模型、逼真的皮肤、反光的拉丝金属已一应俱全。点击 **[threejs/ch3_lights_animation.md](threejs/ch3_lights_animation.md)**，我们将了解图形学中堪称开销最大、同时效果最炸裂的“影子魔法”以及流畅的渲染主环！
