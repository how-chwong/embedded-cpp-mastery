# Vue + TypeScript 前端工程师全栈手册

> 本手册以 **Vue 3 + TypeScript** 为核心技术栈，系统梳理一名前端工程师从入门到能够带领三人小队独立交付项目所需的全部知识体系。内容融合了工程实践、代码技巧与排错经验，并在关键处与 C# 做横向类比，帮助有后端背景的工程师快速建立前端认知。

---

## 阅读建议

| 背景 | 推荐路径 |
|------|----------|
| 前端零基础 | 从第 01 章顺序阅读，跟随示例逐章实操 |
| 有 JS 基础、无 TS/Vue 经验 | 从第 03 章 TypeScript 开始，再读第 04 章 Vue |
| 有 C# / Java 后端背景 | 先读附录中的类比说明，再按需选读 |
| 已有 Vue2 经验 | 直接从第 04 章的 Composition API 部分切入 |
| 准备带队开发 | 重点读第 06、10 章 |

---

## 章节目录

| 章节 | 标题 | 关键词 | 难度 |
|------|------|--------|------|
| [第 01 章](01_frontend_fundamentals.md) | 前端基础知识 | HTML · CSS · JavaScript · 浏览器 | ⭐ |
| [第 02 章](02_dev_tools.md) | 开发工具链 | VSCode · Vite · ESLint · Git · npm | ⭐⭐ |
| [第 03 章](03_typescript.md) | TypeScript 深度指南 | 类型系统 · 泛型 · 装饰器 · 工具类型 | ⭐⭐⭐ |
| [第 04 章](04_vue3_core.md) | Vue 3 核心 | Composition API · 响应式 · 指令 · 生命周期 | ⭐⭐ |
| [第 05 章](05_vue_ecosystem.md) | Vue 生态系统 | Vue Router · Pinia · Axios · i18n · UI库 | ⭐⭐⭐ |
| [第 06 章](06_engineering.md) | 工程化与代码规范 | 项目结构 · 规范 · CI/CD · 单元测试 | ⭐⭐⭐ |
| [第 07 章](07_component_patterns.md) | 组件设计模式 | Composable · Slot · 无渲染组件 · 高阶组件 | ⭐⭐⭐ |
| [第 08 章](08_performance.md) | 性能优化 | 懒加载 · 虚拟列表 · 缓存 · 包体积 | ⭐⭐⭐⭐ |
| [第 09 章](09_debugging.md) | 调试与排错经验 | DevTools · 错误边界 · 常见坑 · 最佳实践 | ⭐⭐⭐ |
| [第 10 章](10_team_and_project.md) | 团队协作与项目管理 | 三人小队 · 需求拆解 · Code Review · 发布 | ⭐⭐⭐⭐ |

---

## 技术栈总览

```
前端技术栈（Vue + TS）
├── 语言层
│   ├── HTML5 / CSS3
│   ├── JavaScript (ES2020+)
│   └── TypeScript 5.x
├── 框架层
│   ├── Vue 3 (Composition API)
│   ├── Vue Router 4
│   └── Pinia (状态管理)
├── 构建工具
│   ├── Vite (开发服务器 + 构建)
│   ├── ESBuild / Rollup
│   └── PostCSS / Tailwind CSS
├── 代码质量
│   ├── ESLint + @typescript-eslint
│   ├── Prettier
│   └── Husky + lint-staged
├── 测试
│   ├── Vitest (单元测试)
│   └── Playwright / Cypress (E2E)
├── HTTP & 数据
│   ├── Axios
│   └── VueUse
└── 工程化
    ├── Git + GitHub/GitLab
    ├── Docker (可选)
    └── CI/CD (GitHub Actions)
```

---

## 快速开始（5 分钟建立项目）

```bash
# 前置条件：Node.js >= 18
node -v

# 使用官方脚手架创建项目
npm create vue@latest my-app
# 选项：TypeScript ✓ | Vue Router ✓ | Pinia ✓ | ESLint ✓ | Prettier ✓

cd my-app
npm install
npm run dev
```

打开 `http://localhost:5173`，即可看到运行中的 Vue 3 + TypeScript 项目。

---

> 每一章都包含：概念讲解 → 代码示例 → 与 C# 类比（适用时）→ 常见错误与排错 → 章节小结。
