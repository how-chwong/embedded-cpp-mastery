# Three.js 3D 渲染极速通关指南

欢迎来到 Three.js 的世界！

作为一名 .NET 工程师，你可能习惯了使用 Windows 原生的 WPF (3D Viewport)、WinForms 的 GDI+、DirectX 或者 XNA/MonoGame 等相对传统的 3D 工具，甚至是 Unity。当需要实现跨平台、无需任何插件即可在现代浏览器（PC 端、平板、手机）中展示高性能、流畅的纯 Web 2.5D/3D 应用（数字孪生、工厂大屏、游戏）时，**Three.js** 是全球当之无愧的霸主。

本指南将借助你在 C# 中熟悉的基本概念和面向对象原理，帮助你快速理清现代 GPU/WebGL 架构与 Three.js API。

---

## 🌎 核心心智模型对比：C# 3D 渲染 vs Web Three.js

| 维度                   | C# / .NET 传统 3D                                                 | Web Three.js                                                                                                                |
| :--------------------- | :---------------------------------------------------------------- | :-------------------------------------------------------------------------------------------------------------------------- |
| **底层渲染驱动**       | WPF 视口/DirectX / OpenGL。依赖本地显卡驱动，属于重量级宿主原生。 | **WebGL / WebGL 2** (甚至最新的 **WebGPU**)。运行在浏览器受沙机保护的高并发 GPU 线程，轻盈且全跨平台。                      |
| **主要承载容器**       | `System.Windows.Controls.Viewport3D` 或是 DirectX 控件句柄。      | 现代 HTML5 中的 **`<canvas>`** 画布标记像素标签。                                                                           |
| **基础语言与类型**     | C# (静态类型、结构清晰)，可采用 ILGPU 计算。                      | 现代 JavaScript 或 **TypeScript**（强烈建议加入，提供类型跳转和补全体验，神似 C#）。                                        |
| **场景与对象数据结构** | 视觉树（Visual Tree）/ 3D Mesh 继承体树组织。                     | **场景图 (Scene Graph)**，经典的父子嵌套层次树模型。                                                                        |
| **开发与发布**         | 编译成庞大的 `.msi` 部署包，需要极高昂的客户端环境一致性。        | 现代前端打包工具（如 **Vite**）几秒内构建并压缩成极轻的静态 HTML/JS 文件夹，直接扔给 Web nginx 服务器即可，全世界秒级访问。 |

---

## 📖 章节导航

请按照以下章节顺序逐步开展研究：

0. **[threejs/ch0_setup.md](threejs/ch0_setup.md)**：极速开箱与现代工程起步。
   - 配置 Node.js、Vite 最快单页构建架构、强类型静态检测 TypeScript 项目配置，并在 VS Code 中实现实时热重载（HMR）预览。
1. **[threejs/ch1_basics.md](threejs/ch1_basics.md)**：三大基石（Scene、Camera、Renderer）。
   - 图形世界基本架构，如何搭建场景、定义透视或正交相机、使用渲染器将 3D 对象映射在 HTML `<canvas>` 上。
2. **[threejs/ch2_geometries_materials.md](threejs/ch2_geometries_materials.md)**：几何体、贴图与 PBR 物理材质模型。
   - 体积网格 Mesh 拼装、纹理贴图对齐，精解如何使用对标 C# DirectX 里面的基础物理着色机制（MeshStandardMaterial）生成真实反光。
3. **[threejs/ch3_lights_animation.md](threejs/ch3_lights_animation.md)**：光影质感、阴影生成与 RequestAnimationFrame 动画主循环。
   - 主流环境光、环境漫射与聚光灯投射配置，处理对显卡开销最大的 Shadow Map 阴影，配置每秒 60 帧不卡顿的 C# GameLoop 对标渲染主循环。
4. **[threejs/ch4_interactive_deploy.md](threejs/ch4_interactive_deploy.md)**：鼠标拾取交互、外部 3D 模型载入与部署打包。
   - 使用 Raycaster 射线进行鼠标精准 3D 物体点击，利用 GLTFLoader 载入外部 Blender / Max 导出的专业 glTF/glb 模型，并一键部署成超轻 Web 应用。

---

现在，让我们直接翻开第一页。点击阅读 **[threejs/ch0_setup.md](threejs/ch0_setup.md)**，搭建你在现代前端的最强 3D 渲染沙盒！
