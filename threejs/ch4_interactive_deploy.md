# 第4章：鼠标事件射线拾取、外部 3D 模型载入与极速生产部署（Three.js 篇）

终于来到了最后一章！
在这一章中，我们将攻克两个最具实用性的技术天险：
1. **交互**：在网页端，鼠标点击的只是 2D 屏幕画布 Canvas。我们如何像 C# 面向对象一样，**知道玩家点击了 3D 物理空间中的哪一个球体或箱子**（射线碰撞检测）？
2. **模型载入**：如何把 professional 的 3D 美术设计师在 Blender、3ds Max 中折腾出来的精美高模（**glTF / GLB** 格式）无缝拖入我们的 Web 页面？
3. **打包部署**：如何像 `dotnet publish` 一样，将大型 3D 页面一键压缩并扔到本地 Nginx 或是云端上，让全球秒级秒开？

---

## 1. 3D 鼠标拾取：Raycaster（光影射线追踪）的奥秘

在普通网页中，我们要想监听用户点击了哪一个按钮，直接写按钮的 `onclick` 事件监听即可。
但在 Three.js 里面，整个 3D 大千世界仅仅是一个 `<canvas>` 标签里呈现的。不管你点击球体还是立方体，对 HTML 来说，你不过是点击了主画布上的一个 `(clientX, clientY)` 二维平面坐标点。

### 是怎么计算点击的？ (Raycasting)
Three.js 使用 **`Raycaster` (光线投射器)** 技术完美对标 C# Unity 里面的 `Physics.Raycast` 机制：
1. 当鼠标在 Canvas 上挪动并点击时，将点击的二维屏幕坐标换算为 GPU 喜欢的 **“规格化设备坐标（Normalized Device Coordinates, NDC）”**，即其 $X$ 和 $Y$ 值的范围全部归并到 $[-1, 1]$。
2. 射线发射器会**从摄像机视角（你的眼睛）为起点**，迎着你点击的屏幕 NDC 物理焦点，**向 3D 空间内部深深地发射出一束看不见的三维物理光导涉线**。
3. 测试这束视线在行进路上，**与哪些场景内的三维物体的三角面片发生了穿透碰撞**。
4. 返回被碰撞到的所有物体（按距离由近到远排序的数组）。排在第一个的，就是你点击到的那个最表面的物体！

```
     [Camera] -----(Ray shot through click position)-----> [Object A] (Hits first!)
                                                            \
                                                             -----> [Object B] (Hits behind)
```

### 实战：点击立方体瞬间随机变色

```typescript
import * as THREE from 'three';

// 假设我们已经按照第 1 章创建好了 scene, camera, renderer 并塞进了一个名为 boxMesh 的红色立方体

// 1. 实例化一根物理射线探测器与一个保存规格化坐标的 Vector2
const raycaster = new THREE.Raycaster();
const mouse = new THREE.Vector2();

// 2. 监听整张网页的点击事件 (Canvas Click Event)
window.addEventListener('click', (event) => {
  // 必须将屏幕物理像素坐标映射规范到精确的 [-1, 1] 视锥体区间中
  mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
  mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

  // 3. 将射线发射源设定为：以当前摄像机视角，瞄准刚才规范化过的鼠标 NDC 点
  raycaster.setFromCamera(mouse, camera);

  // 4. 让射线深入场景去探察并扫描场景树中的所有 Mesh
  // 第二个参数 true 代表是否深度递归搜寻 Group 里面的子网格
  const intersects = raycaster.intersectObjects(scene.children, true);

  // 5. 判断有没有扫到任何物体？如果有，数组 length 必然 > 0
  if (intersects.length > 0) {
    // 数组第一项 intersects[0] 就是我们碰到的最前面的第一个物体！
    const firstHitObject = intersects[0].object as THREE.Mesh;
    
    // 如果这个物体有材质 (并且是可改写材质)，我们让他随机换个色！
    if (firstHitObject.material && 'color' in firstHitObject.material) {
      const randomColor = Math.random() * 0xffffff;
      (firstHitObject.material as THREE.MeshStandardMaterial).color.setHex(randomColor);
      print("🎯 射线瞬间刺穿物体！成功完成 3D 空间交互并随机染色。");
    }
  }
});
```

---

## 2. 拥抱工业级标准：载入 glTF / GLB 三维精美模型

我们不可能仅用简单的立方体和圆球拼装出整个前卫大屏。我们需要在 Blender 或是 Max 下精心烘培好灯光、骨骼和网格的外部模型（被公认为 **“三维世界的 JPEG/PNG”——现代 glTF 或是 GLB 模型**）载入前端。

### 🛠️ 跟着执行：GLTF 极速调入

Vite 会默认托管 `public` 文件夹下的纯文本资源。
1. 在你的项目文件夹下，新建 `public/models` 目录，将你的 3D 美术设计师提供的 `factory.glb`（或是 `car.gltf`）放进该文件夹中。
2. 在 `src/main.ts` 中引入并使用内置的 **`GLTFLoader`** 读入模型：

```typescript
// 记得在文件最顶端引入
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';

// 1. 创建加载器实例
const gltfLoader = new GLTFLoader();

// 2. 调用 .load 接口（Vite 会直接自适应 public 目录路径）
gltfLoader.load(
  '/models/factory.glb', // 文件路径
  (gltf) => {
    // 3. 成功加载后的 callback 回调：
    // 加载成功的 3D 模型骨架、网格等全套层层嵌套在 gltf.scene 节点群组中
    const factoryModel = gltf.scene;
    
    // 【细节避坑】：有些外部模型默认很大很远，我们稍微做一些初始等比物理缩放并推到原点
    factoryModel.scale.set(0.1, 0.1, 0.1); 
    
    // 如果模型带有内部影子效果，我们可以一键遍历所有网格并开启它们
    factoryModel.traverse((child) => {
        if ((child as THREE.Mesh).isMesh) {
            child.castShadow = true;
            child.receiveShadow = true;
        }
    });

    scene.add(factoryModel); // 一线秒入，宏伟的大型工厂 3D 模型瞬间展现在网页大屏中！
    print("🛸 模型加载大本营正常：模型成功装配进入场景。");
  },
  (progress) => {
    // 4. 正在加载的进度条：对标 C# 的 IProgress 进度加载，反馈百分比
    const percentage = (progress.loaded / progress.total) * 100;
    console.log(`⏳ 3D 资源正在高速下载中: ${percentage.toFixed(0)}%`);
  },
  (error) => {
    // 5. 错误捕获处理
    console.error("❌ 模型加载失败，请检查文件编码或格式损坏: ", error);
  }
);
```

---

## 3. 一键发布与工业级部署

当我们的 3D 调试项目编写完毕后，我们如何将它发布？
在 C# 中，你需要运行：
```bash
dotnet publish -c Release -o ./publish
```

而在 Vite 下，你会体验到什么叫真正的轻盈。直接在项目根目录下终端一行运行：
```bash
npm run build
```

### Vite 后台发生的极速打包魔术：
1. Vite 会开启先进的 **编译期摇树优化 (Tree-Shaking)**。在这个极其短暂的过程中，它在后台疯狂检索你没有调用的 Three.js 方法和组件并且将它们剔除干净。
2. 压缩（Uglify/Minify）你所有的 TypeScript 类型标注和变量。
3. 瞬间在你的当前项目目录下自动生成一个极度纯净、极其精小的 **`dist`** 文件夹！
4. **这个 `dist` 里面没有任何的 Node.js 杂质或者依赖。它里面只有：一个极为紧凑的 `index.html` 以及压缩后极小的 `dist/assets/main.xxxx.js` 代码文件。** 

### 本地部署极其简单：
你可以把 `dist` 文件夹下的所有内容像复制静态文本文档一样，直接扔进你的 Windows Nginx/IIS 下或者是 Linux Cloud 上挂载，甚至是上传到静态托管平台。全世界的网民连任何运行插件都不需要，只需用浏览器点开这个网页链接，显卡便会瞬息满速激活，展示你惊人流畅的 3D 实境大作！

---

## 🏁 全能开发者通关：最后的里程碑结语

至此，恭喜你不仅在 **Python 模块和人工智能实战** 中自如游走，也在 **Lua 的超精微嵌入式开发** 中得心应手，更是在现代 Web 后端高水准的 **Three.js 3D 渲染** 体系下，掌握了从骨架材质、物理光影阴影、高帧计算，直到模型引入和部署的完整知识版图！

我们为你建立起的“C# 视角对照镜”已经成功帮助你打破了这些最坚硬的壁垒。后面的大千世界，已经任由你在你的 VS Code 中肆意开天辟地。祝你的探索征途一往无前，未来无可限度！
