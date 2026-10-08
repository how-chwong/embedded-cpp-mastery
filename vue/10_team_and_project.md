# 第 10 章：团队协作与项目管理

> 目标：掌握带领三人小队开展项目开发的核心方法，包括需求拆解、任务分工、技术决策、Code Review、风险管控和发布管理。

---

## 10.1 三人小队的角色分工

```
典型的三人前端小队
├── 技术负责人（TL / 你）
│   职责：架构设计、技术决策、Code Review、进度把控、阻塞解决
│   日常：30% 开发 + 30% 评审 + 20% 设计 + 20% 沟通
│
├── 开发工程师 A（中级）
│   职责：核心业务功能开发、单元测试
│   关注：功能实现、代码质量
│
└── 开发工程师 B（初级/中级）
    职责：UI 组件开发、辅助功能、Bug 修复
    关注：按规范实现、及时同步进展
```

---

## 10.2 项目启动（Kickoff）

### 技术选型决策框架

```markdown
## 技术选型清单

### 框架层
- [ ] Vue 3 + TypeScript（确认团队熟悉度）
- [ ] UI 库：Element Plus（B 端）/ Naive UI（新项目）
- [ ] 状态管理：Pinia（Vue 3 首选）

### 工程层
- [ ] 构建工具：Vite（默认选择）
- [ ] 代码规范：ESLint + Prettier + Husky
- [ ] 测试：Vitest（单元）+ Playwright（E2E）
- [ ] CSS 方案：Scoped CSS / Tailwind / UnoCSS

### 后端对接
- [ ] API 风格：RESTful / GraphQL
- [ ] 认证方案：JWT / Session
- [ ] 实时通信：WebSocket / SSE

### 部署
- [ ] 托管：Nginx / CDN
- [ ] CI/CD：GitHub Actions / GitLab CI
- [ ] 监控：Sentry / 自建

选型原则：
1. 团队熟悉 > 最新最好
2. 社区活跃 > 功能完善
3. 渐进演进 > 一步到位
```

### 项目初始化 CheckList

```bash
# 创建并配置项目
npm create vue@latest project-name
cd project-name

# 基础配置
pnpm install
git init
git add .
git commit -m "chore: project initialization"

# 推送到远端
git remote add origin <repo-url>
git push -u origin main

# 创建分支保护规则（在 GitHub/GitLab 上设置）
# - main 分支必须通过 CI 才能合并
# - 至少 1 人 Review 才能合并
# - 禁止直接 push 到 main
```

---

## 10.3 需求管理

### 需求拆解方法（任务树）

```markdown
## 需求：用户管理模块

### Epic：用户管理
├── Story：用户列表
│   ├── Task: 设计用户列表 UI（0.5d）
│   ├── Task: 实现用户列表 API 对接（0.5d）
│   ├── Task: 分页、搜索、排序功能（1d）
│   └── Task: 单元测试（0.5d）
│
├── Story：用户创建/编辑
│   ├── Task: 用户表单组件（1d）
│   ├── Task: 表单验证（0.5d）
│   └── Task: 提交/更新 API（0.5d）
│
└── Story：用户权限
    ├── Task: 角色下拉选择（0.5d）
    ├── Task: 权限配置（1d）
    └── Task: 前端权限拦截（0.5d）

总估时：6.5 天 → 预留 20% Buffer = 8 天
```

### 任务分配原则

```
分配策略：
1. 按能力匹配：核心/复杂任务给高级工程师
2. 按模块划分：减少代码冲突，提高专注度
3. 留有 Buffer：实际时间 = 估时 × 1.2~1.5
4. 明确完成标准：DoD（Definition of Done）

每个任务的 DoD：
- [ ] 功能按需求实现
- [ ] 代码通过 ESLint 检查
- [ ] 关键逻辑有单元测试
- [ ] 自测通过（包含边界情况）
- [ ] PR 已创建并@TL 审查
```

---

## 10.4 Git 协作规范

### 分支管理

```bash
# 分支命名规范
main                          # 生产分支（受保护）
develop                       # 开发主分支
feature/user-management       # 功能分支
feature/dashboard-charts
fix/table-pagination-bug      # 修复分支
fix/login-redirect
hotfix/critical-security-fix  # 紧急修复（从 main 切出）
release/v1.2.0               # 发布分支

# 工作流
git checkout develop
git pull origin develop
git checkout -b feature/my-feature

# 开发完成
git add .
git commit -m "feat(user): 添加用户搜索功能"
git push origin feature/my-feature

# 创建 PR：feature/my-feature → develop
```

### PR（Pull Request）规范

```markdown
## PR 模板（.github/pull_request_template.md）

## 变更描述
简要说明本次变更的目的和内容

## 变更类型
- [ ] feat: 新功能
- [ ] fix: Bug 修复
- [ ] refactor: 重构
- [ ] docs: 文档
- [ ] test: 测试

## 变更截图（UI 变更必填）


## 测试清单
- [ ] 功能正常（主流程）
- [ ] 边界情况已处理
- [ ] 无控制台报错
- [ ] 单元测试通过

## 关联 Issue
Closes #xxx
```

### Code Review 规范

```markdown
## 审查要点（TL 审查清单）

### 功能正确性
- [ ] 逻辑是否按需求实现
- [ ] 边界情况是否覆盖（null, empty, error）

### 代码质量
- [ ] TypeScript 类型是否完整（无不必要的 any）
- [ ] 命名是否清晰
- [ ] 是否有重复代码（应提取复用）
- [ ] 注释是否有必要

### Vue 最佳实践
- [ ] 是否使用 `<script setup>`
- [ ] v-for 是否有 key
- [ ] 是否在 onUnmounted 中清理资源
- [ ] 是否避免了直接修改 Props

### 性能
- [ ] 是否有不必要的计算在模板中
- [ ] 大数据是否使用了虚拟列表

## 评论规范（避免主观判断伤人）
❌ "这段代码写的很烂"
✅ "这里可以用 computed 代替，这样性能更好（因为...）"

❌ "你应该知道这里要用 xxx"
✅ "建议使用 xxx，这样可以...，参考 Vue 官方文档 [链接]"
```

---

## 10.5 每日站会与进度管理

### 站会模板（每天 15 分钟）

```
每人回答 3 个问题：
1. 昨天完成了什么？
2. 今天计划做什么？
3. 有什么阻碍？

注意：
- 不要在站会中解决问题，记录后另约时间
- TL 重点关注风险和阻塞，及时干预
- 超过 1 天的任务，检查是否需要拆分或帮助
```

### 进度追踪看板

```
Backlog → To Do → In Progress → Code Review → Testing → Done

规则：
- In Progress 每人最多 2 个任务（防止多任务切换）
- Code Review 超过 24 小时未完成，TL 主动推进
- Done 必须满足 DoD 标准
```

---

## 10.6 前后端协作

### API 文档规范

```markdown
## 接口定义（在开发前约定好）

GET /api/users
Query: { page: number, pageSize: number, search?: string }
Response: {
  code: 0,
  data: {
    items: User[],
    total: number
  }
}

POST /api/users
Body: { name: string, email: string, role: 'admin' | 'user' }
Response: { code: 0, data: User }

## 统一响应格式
{ code: number, data: T, message: string }
code 0 = 成功，其他 = 错误
```

```typescript
// 前端可以先 Mock 数据，不等后端
// 使用 MSW（Mock Service Worker）
import { http, HttpResponse } from 'msw';

export const handlers = [
  http.get('/api/users', () => {
    return HttpResponse.json({
      code: 0,
      data: {
        items: [{ id: 1, name: 'Alice', email: 'alice@example.com' }],
        total: 1,
      },
    });
  }),
];
```

---

## 10.7 发布管理

### 版本号规范（语义化版本）

```
v主版本.次版本.补丁版本
v1.0.0

主版本（MAJOR）：不兼容的 API 变更
次版本（MINOR）：向后兼容的新功能
补丁版本（PATCH）：向后兼容的 Bug 修复

示例：
v1.0.0 → v1.1.0（新增用户管理功能）
v1.1.0 → v1.1.1（修复分页 Bug）
v1.1.1 → v2.0.0（重构整体架构）
```

### 发布流程

```bash
# 1. 创建发布分支
git checkout -b release/v1.2.0 develop

# 2. 更新版本号
npm version minor  # 自动更新 package.json

# 3. 更新 CHANGELOG.md
# 4. 最终测试
# 5. 合并到 main
git checkout main
git merge --no-ff release/v1.2.0
git tag v1.2.0

# 6. 合并回 develop
git checkout develop
git merge --no-ff release/v1.2.0

# 7. 触发 CI/CD 部署
git push origin main --tags
```

### CHANGELOG 维护

```markdown
# Changelog

## [1.2.0] - 2024-03-01

### Added
- 用户管理模块（列表、创建、编辑、删除）
- 数据导出 Excel 功能

### Changed
- 登录页面 UI 优化
- 分页组件改为统一的 BasePagination

### Fixed
- 修复搜索结果不更新的问题（#123）
- 修复表格排序在 Safari 下异常（#125）

### Performance
- 用户列表引入虚拟滚动，10 万数据渲染流畅
```

---

## 10.8 常见团队问题与解决方案

### 代码风格不统一

```
问题：团队成员有不同的编码习惯
解决：
1. 配置 ESLint + Prettier 并强制执行（Husky pre-commit）
2. 提供组件模板（snippets 或脚手架命令）
3. 定期 Code Review，统一纠正
4. 建立内部组件库和代码示例库
```

### 分支合并冲突频繁

```
问题：多人同时修改同一文件导致冲突
解决：
1. 按模块划分工作，减少文件交叉
2. 频繁从 develop 同步（每天 git pull --rebase）
3. 及时合并 PR，避免长期存在的 feature 分支
4. 将大文件拆分为小文件（组件拆分）
```

### 新成员上手慢

```
解决：
1. 维护 README.md（环境搭建、项目运行、目录说明）
2. 提供项目架构文档
3. 建立代码规范文档（本手册就是一个例子）
4. 安排 1 天 pair programming（结对编程）熟悉代码
5. 第一周分配简单且独立的任务
```

---

## 10.9 技术债务管理

```markdown
## 技术债务登记表

| 编号 | 描述 | 影响 | 优先级 | 负责人 | 计划版本 |
|------|------|------|--------|--------|---------|
| TD-001 | UserList 组件超过 500 行，需要拆分 | 维护困难 | P2 | Alice | v1.3.0 |
| TD-002 | 部分组件使用了 Options API | 风格不统一 | P3 | Bob | v1.4.0 |
| TD-003 | 缺少 E2E 测试 | 发布风险 | P1 | All | v1.2.0 |

原则：
- 每个迭代留出 10-20% 时间处理技术债
- P1（高风险）：下个迭代必须处理
- P2（中等）：安排在未来 2 个迭代
- P3（低优先级）：排期处理或接受
```

---

## 10.10 前端安全基础

```typescript
// 1. XSS 防御：避免 v-html 使用不可信内容
// ❌
const userInput = '<script>alert("xss")</script>';
// <div v-html="userInput" />  危险！

// ✅ 如果必须用 v-html，先清理
import DOMPurify from 'dompurify';
const safeHtml = DOMPurify.sanitize(userInput);

// 2. 敏感信息不存前端
// ❌ 不要在 localStorage 存明文密码
// ✅ 只存 Token，且设置合理过期时间

// 3. HTTPS
// 生产环境必须使用 HTTPS

// 4. 内容安全策略（CSP）
// 在 index.html 或服务器配置中
// <meta http-equiv="Content-Security-Policy" content="default-src 'self'">

// 5. 敏感操作二次确认
// 删除、修改权限等操作需要二次确认

// 6. 路由权限
// 前端权限只是 UI 层保护，真正的权限验证在后端
router.beforeEach((to) => {
  if (to.meta.roles && !hasRole(to.meta.roles)) {
    return { name: 'Forbidden' };
  }
});
```

---

## 10.11 章节小结

```
带领三人小队的核心能力：

技术维度：
✅ 做好技术选型和架构决策
✅ 建立并执行代码规范（自动化优于人工审查）
✅ 系统性 Code Review
✅ 识别和处理技术风险

管理维度：
✅ 合理拆解需求、分配任务
✅ 保障每日进度可视化（站会 + 看板）
✅ 快速解决团队阻塞
✅ 保护团队不被频繁需求变更打乱节奏

文化维度：
✅ 建立心理安全感（鼓励提问和暴露问题）
✅ Code Review 以学习为目的，不是挑错
✅ 定期回顾（Retrospective），持续改进
✅ 知识分享（技术分享、文档沉淀）
```

---

> 恭喜你完成了全部 10 章的学习！🎉
>
> **完成所有章节后，你应该能够：**
> - 使用 Vue 3 + TypeScript 独立构建复杂的单页应用
> - 建立完整的工程化体系（构建、测试、CI/CD）
> - 设计可复用的组件和 Composable 函数
> - 优化应用性能到生产级标准
> - 系统排查和解决前端问题
> - 带领三人小队高效完成项目交付
>
> **持续进阶建议：**
> - 阅读 [Vue 官方文档](https://cn.vuejs.org/)
> - 学习 [TypeScript 官方手册](https://www.typescriptlang.org/docs/handbook/intro.html)
> - 关注 [Vue 生态动态](https://github.com/vuejs/awesome-vue)
> - 实战：用本手册的技术栈完成一个完整项目
