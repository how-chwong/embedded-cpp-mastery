# 第 02 章：开发工具链

> 目标：配置一套高效的前端开发环境，掌握 VSCode、Node.js、Vite、ESLint、Prettier、Git 等工具的使用与配置。

---

## 2.1 Node.js 与包管理器

### 安装 Node.js

```bash
# 推荐使用 nvm 管理多个 Node 版本
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash

# 安装并使用 LTS 版本
nvm install --lts
nvm use --lts
node -v   # v20.x.x
npm -v    # 10.x.x
```

### npm / pnpm / yarn 对比

| 特性 | npm | pnpm | yarn |
|------|-----|------|------|
| 速度 | 中 | 最快 | 快 |
| 磁盘占用 | 多 | 最少（硬链接） | 多 |
| Monorepo 支持 | 一般 | 优秀 | 良好 |
| 推荐场景 | 默认/通用 | 新项目首选 | 老项目维护 |

```bash
# 安装 pnpm（推荐）
npm install -g pnpm

# 常用命令对比
npm install          pnpm install         # 安装所有依赖
npm install axios    pnpm add axios       # 添加依赖
npm install -D vitest pnpm add -D vitest  # 添加开发依赖
npm run dev          pnpm dev             # 运行脚本
npm uninstall axios  pnpm remove axios    # 删除依赖
```

### package.json 关键字段

```json
{
  "name": "my-vue-app",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vue-tsc && vite build",
    "preview": "vite preview",
    "test": "vitest",
    "lint": "eslint . --ext .ts,.vue",
    "format": "prettier --write ."
  },
  "dependencies": {
    "vue": "^3.4.0",
    "vue-router": "^4.3.0",
    "pinia": "^2.1.0",
    "axios": "^1.6.0"
  },
  "devDependencies": {
    "@vitejs/plugin-vue": "^5.0.0",
    "@vue/tsconfig": "^0.5.0",
    "typescript": "^5.3.0",
    "vite": "^5.0.0",
    "vue-tsc": "^1.8.0",
    "eslint": "^8.57.0",
    "@typescript-eslint/eslint-plugin": "^7.0.0",
    "prettier": "^3.2.0",
    "vitest": "^1.3.0"
  },
  "engines": {
    "node": ">=18.0.0"
  }
}
```

---

## 2.2 VSCode 配置

### 必装插件

```
# 核心插件
Vue - Official (原 Volar)     # Vue 3 语言支持
TypeScript Vue Plugin (Volar) # TS + Vue 集成
ESLint                        # 代码检查
Prettier - Code formatter     # 格式化
GitLens                       # Git 增强

# 效率插件
Auto Import                   # 自动导入
Path Intellisense             # 路径补全
Error Lens                    # 内联显示错误
Todo Tree                     # TODO 注释追踪
Rest Client                   # HTTP 请求测试（替代 Postman）
```

### .vscode/settings.json（项目配置）

```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "[vue]": {
    "editor.defaultFormatter": "Vue.volar"
  },
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit"
  },
  "typescript.tsdk": "node_modules/typescript/lib",
  "typescript.enablePromptUseWorkspaceTsdk": true,
  "editor.tabSize": 2,
  "editor.insertSpaces": true,
  "files.eol": "\n",
  "editor.rulers": [100]
}
```

### .vscode/extensions.json（推荐插件）

```json
{
  "recommendations": [
    "Vue.volar",
    "dbaeumer.vscode-eslint",
    "esbenp.prettier-vscode",
    "eamodio.gitlens"
  ]
}
```

---

## 2.3 Vite 构建工具

### 为什么选 Vite

- **开发时**：利用原生 ESM，按需编译，启动时间 <100ms（webpack 需要几十秒）
- **构建时**：基于 Rollup，生产环境输出优化
- **HMR**（热模块替换）：极速，仅更新变更模块

### vite.config.ts

```typescript
import { defineConfig } from 'vite';
import vue from '@vitejs/plugin-vue';
import { fileURLToPath, URL } from 'node:url';

export default defineConfig({
  plugins: [vue()],
  
  resolve: {
    alias: {
      // @ 指向 src 目录
      '@': fileURLToPath(new URL('./src', import.meta.url)),
    },
  },
  
  server: {
    port: 5173,
    open: true,
    proxy: {
      // 开发时代理 API 请求，解决跨域
      '/api': {
        target: 'http://localhost:8080',
        changeOrigin: true,
        rewrite: (path) => path.replace(/^\/api/, ''),
      },
    },
  },
  
  build: {
    outDir: 'dist',
    sourcemap: false,
    rollupOptions: {
      output: {
        // 手动分块
        manualChunks: {
          vendor: ['vue', 'vue-router', 'pinia'],
          ui: ['element-plus'],
        },
      },
    },
  },
  
  // 环境变量前缀
  envPrefix: 'VITE_',
});
```

### 环境变量

```bash
# .env（所有环境）
VITE_APP_TITLE=My App

# .env.development（开发环境）
VITE_API_BASE_URL=http://localhost:8080

# .env.production（生产环境）
VITE_API_BASE_URL=https://api.myapp.com

# .env.staging（自定义环境）
VITE_API_BASE_URL=https://staging-api.myapp.com
```

```typescript
// 在代码中使用
const apiUrl = import.meta.env.VITE_API_BASE_URL;
const isProd = import.meta.env.PROD;
const isDev = import.meta.env.DEV;
const mode = import.meta.env.MODE; // 'development' | 'production' | 'staging'

// TypeScript 类型提示：src/env.d.ts
interface ImportMetaEnv {
  readonly VITE_API_BASE_URL: string;
  readonly VITE_APP_TITLE: string;
}
interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

---

## 2.4 ESLint 配置

### 安装

```bash
pnpm add -D eslint @typescript-eslint/eslint-plugin @typescript-eslint/parser \
  eslint-plugin-vue vue-eslint-parser eslint-config-prettier
```

### eslint.config.js（ESLint v9 flat config）

```javascript
import globals from 'globals';
import pluginJs from '@eslint/js';
import tsPlugin from '@typescript-eslint/eslint-plugin';
import tsParser from '@typescript-eslint/parser';
import pluginVue from 'eslint-plugin-vue';
import vueParser from 'vue-eslint-parser';

export default [
  pluginJs.configs.recommended,
  {
    files: ['**/*.{ts,tsx,vue}'],
    languageOptions: {
      parser: vueParser,
      parserOptions: {
        parser: tsParser,
        ecmaVersion: 'latest',
        sourceType: 'module',
      },
      globals: { ...globals.browser, ...globals.node },
    },
    plugins: {
      '@typescript-eslint': tsPlugin,
      vue: pluginVue,
    },
    rules: {
      // TypeScript 规则
      '@typescript-eslint/no-explicit-any': 'warn',
      '@typescript-eslint/explicit-function-return-type': 'off',
      '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
      
      // Vue 规则
      'vue/multi-word-component-names': 'error',
      'vue/component-api-style': ['error', ['script-setup']],
      'vue/define-macros-order': ['error', {
        order: ['defineOptions', 'defineProps', 'defineEmits', 'defineSlots'],
      }],
      
      // 通用规则
      'no-console': ['warn', { allow: ['warn', 'error'] }],
      'prefer-const': 'error',
    },
  },
  {
    ignores: ['dist/**', 'node_modules/**'],
  },
];
```

---

## 2.5 Prettier 配置

```json
// .prettierrc
{
  "semi": false,
  "singleQuote": true,
  "printWidth": 100,
  "tabWidth": 2,
  "trailingComma": "all",
  "bracketSpacing": true,
  "arrowParens": "always",
  "endOfLine": "lf",
  "vueIndentScriptAndStyle": false
}
```

```
// .prettierignore
dist
node_modules
*.min.js
```

---

## 2.6 TypeScript 配置

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    
    // 严格模式（必须开启）
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    
    // 路径别名（与 vite.config.ts 保持一致）
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    },
    
    // Vue 特殊配置
    "jsx": "preserve",
    "jsxImportSource": "vue",
    
    "skipLibCheck": true,
    "esModuleInterop": true,
    "resolveJsonModule": true,
    "isolatedModules": true
  },
  "include": ["src/**/*.ts", "src/**/*.tsx", "src/**/*.vue"],
  "exclude": ["node_modules", "dist"]
}
```

---

## 2.7 Git 工作流

### Husky + lint-staged（提交前自动检查）

```bash
pnpm add -D husky lint-staged
npx husky init
```

```json
// package.json
{
  "lint-staged": {
    "*.{ts,tsx,vue}": ["eslint --fix", "prettier --write"],
    "*.{json,md,css}": ["prettier --write"]
  }
}
```

```bash
# .husky/pre-commit
npx lint-staged
```

### Commitlint（规范提交信息）

```bash
pnpm add -D @commitlint/cli @commitlint/config-conventional
echo "export default { extends: ['@commitlint/config-conventional'] };" > commitlint.config.js
```

```bash
# .husky/commit-msg
npx --no -- commitlint --edit $1
```

### 提交信息规范

```
# 格式
<type>(<scope>): <subject>

# type 类型
feat:     新功能
fix:      修复 Bug
docs:     文档变更
style:    代码格式（不影响功能）
refactor: 重构
test:     测试相关
chore:    构建/工具链
perf:     性能优化
ci:       CI 配置

# 示例
feat(auth): 添加 JWT 登录功能
fix(table): 修复分页组件数据不更新的问题
docs(readme): 更新安装说明
```

### Git 分支策略（三人小队）

```
main          # 生产分支，只通过 PR 合入
  └── develop # 开发主分支
        ├── feature/user-login    # 功能分支
        ├── feature/dashboard     # 功能分支
        └── fix/table-pagination  # 修复分支
```

---

## 2.8 项目初始化完整流程

```bash
# 1. 创建项目
npm create vue@latest my-project
cd my-project

# 2. 安装依赖
pnpm install

# 3. 安装额外工具
pnpm add axios pinia vue-router @vueuse/core
pnpm add -D husky lint-staged @commitlint/cli @commitlint/config-conventional

# 4. 初始化 Husky
npx husky init

# 5. 验证运行
pnpm dev
pnpm build
pnpm test
```

---

## 2.9 章节小结

| 工具 | 用途 | 重要程度 |
|------|------|---------|
| Node.js + pnpm | 运行时 + 包管理 | ⭐⭐⭐⭐⭐ |
| VSCode + Volar | 编辑器 + Vue 支持 | ⭐⭐⭐⭐⭐ |
| Vite | 开发服务器 + 构建 | ⭐⭐⭐⭐⭐ |
| TypeScript | 类型安全 | ⭐⭐⭐⭐⭐ |
| ESLint + Prettier | 代码质量 | ⭐⭐⭐⭐ |
| Husky + lint-staged | 提交前检查 | ⭐⭐⭐⭐ |
| Git + Commitlint | 版本管理 | ⭐⭐⭐⭐ |

> **下一步**：阅读 [第 03 章 - TypeScript 深度指南](03_typescript.md)
