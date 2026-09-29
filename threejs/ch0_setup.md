# 第0章：现代前端 3D 工程起步与 HMR 渲染沙盒（Three.js 篇）

如果你习惯了使用 **NuGet** 和 **MSBuild** 来构建和部署 .NET 项目，当你跨入前端 3D 渲染世界时，可能会对前端令人眼花缭乱的框架（Webpack, Rollup, Babel）感到头疼。

别担心！今天我们将使用目前全球公认最轻量、编译执行速度堪称光速的构建引擎 —— **Vite**，结合微软强类型语法 **TypeScript**（提供与 C# 近乎 100% 完美的自动感知、代码跳转和编译排错优势），用两分钟搭建起你的 Three.js 开发调试沙箱。

---

## 1. 准备 Node.js 宿主（前端的 “.NET SDK”）

如同你需要安装 .NET Core SDK 来编译运行 C# 代码一样，现代 Web 开源生态极其依赖 **Node.js** 的运行时与配套包管理器 **npm**（对标 NuGet）来拉取并构建前端第三方库。

### 下载与安装
1. 访问 Node.js 官方网站：[https://nodejs.org/](https://nodejs.org/)。
2. 建议下载极高稳定度的 **LTS (长期官方支持版)**。
3. 一路点击 "Next" 安装完成。
4. 打开你的终端输入以下命令验证，如果看到版本号显示证明安装成功：
   ```bash
   node -v
   npm -v  # NuGet 包管理的前端平替
   ```

---

## 2. 闪电级工程脚手法一键搭建：Vite + TypeScript

在以往，前端搭建项目需要写动辄几百行的配置文件，但今天我们用 **Vite** 做到秒级开箱。

### 🛠️ 跟着执行：五阶段建站法

1. 选择一个干净的目录文件夹（如 `d:\Workspace\threejs_project`）。
2. 在终端进入此根目录，运行 Vite 官方最快速脚手架命令：
   ```bash
   # 新建一个基于 TypeScript 纯净工程
   npm create vite@latest my-three-app -- --template vanilla-ts
   ```
3. 过程仅需一秒！成功后进入该文件夹：
   ```bash
   cd my-three-app
   ```
4. **添加 Three.js 核心依赖**（这一步相当于在 C# dotnet cli 中 `dotnet add package Three`）：
   ```bash
   # 1. 安装核心 Three.js 3D 源码包
   npm install three
   
   # 2. 安装给 TypeScript 使用的“强类型声明头文件包”（这样 VS Code 才能获得像写 C# 一样强大的写代码自动类型推断！）
   npm install --save-dev @types/three
   ```
5. 完成后，在 VS Code 中点开当前 `my-three-app` 文件夹：
   ```bash
   code .
   ```

---

## 3. 【极速测试】编写你的第一个“旋转绿魔方”

为了测试这套工具链是否正常，让我们清空默认脚手架文件，实现一个自旋立方体。

### 1. 修改 `index.html`（容器板）
在 VS Code 中双击打开根目录下的 [index.html](index.html)。将其整行替换为以下内容（提供承载 3D 画面所必须的 `<canvas>` 画布标签）：

```html
<!DOCTYPE html>
<html lang="zh-CN">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Three.js 3D Sandbox for .NET Developers</title>
    <style>
      /* 干净清爽背景：让 360 度画布占满整个浏览器视窗并且没有边距滚动条 */
      body {
        margin: 0;
        overflow: hidden;
      }
      #webgl-canvas {
        width: 100vw;
        height: 100vh;
        display: block;
      }
    </style>
  </head>
  <body>
    <!-- 所有的 3D 图形像素，都将渲染映射在这个轻巧的画布标签中 -->
    <canvas id="webgl-canvas"></canvas>
    
    <!-- 引入我们的 TypeScript 控制逻辑脚本 -->
    <script type="module" src="/src/main.ts"></script>
  </body>
</html>
```

### 2. 替换 `src/main.ts`（逻辑核心）
在侧边栏找到 `src/main.ts` 并打开它，清空内容，将以下具备强类型注释的 3D 自走渲染脚本完全复制过去：

```typescript
import * as THREE from 'three';

// 1. 获取我们刚才在 index.html 里定义的物理 Canvas DOM 元素
const canvas = document.getElementById('webgl-canvas') as HTMLCanvasElement;

// 2. 实例化 3D 黄金三大基石
// a. 场景 (对标 C# 的 3D 空间，用来放置各种物体和光源)
const scene = new THREE.Scene();
scene.background = new THREE.Color('#1a1a1a'); // 设定暗黑雅致背景

// b. 摄像机 (对标我们观察 3D 空间的大脑视窗)：此处选用最像人眼透视的 PerspecticeCamera
const camera = new THREE.PerspectiveCamera(
  75, // 垂直视野角度（FOV）：75度
  window.innerWidth / window.innerHeight, // 屏幕宽高比宽高纵横比（Aspect Ratio）
  0.1, // 近端裁剪面：距离小于 0.1 的物体不渲染
  1000 // 远端裁剪面：距离大于 1000 的物体不渲染
);
camera.position.z = 5; // 将摄像机往屏幕外稍微拉开 5 个单位，不然默认会卡在物体核心内部

// c. 渲染器 (对标 C# 的 DirectX DrawingEngine。用于将 3D 物体转化为网页上的像素图案)
const renderer = new THREE.WebGLRenderer({
  canvas: canvas,
  antialias: true // 开启抗锯齿，使立方体边缘显得圆润和锐利
});
renderer.setSize(window.innerWidth, window.innerHeight);

// 3. 往场景中塞入一个自制立方体
// 建立几何骨架（Geometry）
const geometry = new THREE.BoxGeometry(1.5, 1.5, 1.5);
// 建立基础绿宝石颜色的感光材质（Material）
const material = new THREE.MeshBasicMaterial({ color: 0x00ff88, wireframe: true });
// 融合成物理 3D 实实体网格（Mesh = Geometry + Material）
const cube = new THREE.Mesh(geometry, material);
scene.add(cube); // 将精心装配好立方体扔入场景树。此时物体的默认世界坐标为原点 (0,0,0)

// 4. 实现无级不卡顿 C# GameLoop 式动画主循环（每1/60秒刷新一次界面）
function animate() {
  // 请求浏览器下次重绘时（1帧）执行我，由此保持自动不卡顿刷新
  requestAnimationFrame(animate);

  // 每一帧自加一点自旋弧角速度
  cube.rotation.x += 0.01;
  cube.rotation.y += 0.01;

  // 将场景和摄像机的交汇通过核芯引擎进行重绘输出！
  renderer.render(scene, camera);
}

// 5. 监听浏览器窗口拉扯变化（自适应视窗大小）
window.addEventListener('resize', () => {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix(); // 更新投影矩阵
  renderer.setSize(window.innerWidth, window.innerHeight);
});

// 6. 激活执行循环引擎！
animate();
```

---

## 4. 运行并体验高科技：Vite 极速热重载 (HMR)

现在所有逻辑已经备齐，让我们在你的项目路径终端输入：
```bash
npm run dev
```

### 闪速起跳体验：
1. Vite 会在几乎 **0 到几十毫秒内**，瞬间在本地启动一个局部的 Web 预览服务。
2. 终端会高亮输出服务地址端口，如：`  ➜  Local:   http://localhost:5173/`。
3. 用浏览器（Chrome、Edge 等）打开该网址，一个散发着炫酷黑客绿、平滑、不带有任何卡顿自转的 3D 立方体就此诞生！
4. **【神奇的热重载（HMR）体验】**：
   - 不要关闭浏览器。返回 VS Code 编辑器中，修改 `src/main.ts` 第 24 行的立方体颜色，例如把 `0x00ff88` 改成深海蓝 `0x00aaff`。
   - 按下 `Ctrl + S` 保存。
   - **观察浏览器，你根本不用手动刷新网页，也几乎看不到网页闪烁重载的过程，立方体的自旋和状态完全保持，颜色却在瞬间变为了纯正的深海蓝！** 
   - 这对于以往在 WPF 3D 或者是 DirectX C# 项目中做修改需要重新编译、重新拉起流程的你，绝对是极具有视觉震撼力的高极度开发和修改体验。

---

有了这个强悍的热重载渲染沙盒，你已经彻底将脚踏实地的跨过了最复杂的配置门槛。点击 **[threejs/ch1_basics.md](threejs/ch1_basics.md)**，我们将从经典的 C# 3D 理论出发，详细拆解让你可以任意自由拼装 3D 世界版图的“三大基石”！
