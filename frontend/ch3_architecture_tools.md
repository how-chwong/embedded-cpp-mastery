# 第 03 章：企业级工程化脚手架与 Monorepo 架构设计

在大型企业中，一个架构师的核心价值，在于为团队提供稳定、高效、可扩展的“基础设施”。如果每个组员开发时的代码格式不同、随意引入依赖、请求封装不统一、甚至多个关联项目反复复制代码，那么项目很快就会走向崩溃。

本章将带你从零构建一套工业级的工程化底座，涵盖 **Vite 构建体系**、**代码质量守卫（ESLint + Husky）**、**Monorepo 仓库管理**，以及**强类型 Axios 封装**，并将其与 .NET 工程化结构做深度类比。

---

## 3.1 现代构建利器：Vite 深度配置与原理解析

在过去，Webpack 几乎是前端打包的唯一选择。但由于 Webpack 是基于**全量打包**，随着项目体积变大，启动和热更新（HMR）慢得令人难以忍受。

**Vite 的革命性突破**：
* **开发环境（Development）**：Vite 充分利用了现代浏览器原生支持的 **ES Modules (ESM)**。当浏览器请求某个组件时，Vite 会动态转译并按需返回该文件，几乎达到了“毫秒级启动”和热重载。
* **生产环境（Production）**：底层调用 **Rollup** 进行打包。Rollup 的 Tree-shaking 机制比 Webpack 更加轻量、产物更加干净。

### 3.1.1 经典 Vite 企业级配置文件 (`vite.config.ts`)

以下是一个开箱即用的、高度契合企业实战的 Vite 配置模板：

```typescript
// vite.config.ts
import { defineConfig, loadEnv } from 'vite';
import vue from '@vitejs/plugin-vue';
import { resolve } from 'path';

export default defineConfig(({ mode }) => {
  // 根据当前环境加载 .env.[mode] 文件，类似 C# 的 appsettings.json 与 appsettings.Development.json
  const env = loadEnv(mode, process.cwd());

  return {
    plugins: [vue()],
    resolve: {
      alias: {
        // 配置路径别名：使用 '@' 代替 'src'，方便深层引用，类似于 C# 中的根命名空间
        '@': resolve(__dirname, 'src'),
      },
    },
    css: {
      preprocessorOptions: {
        scss: {
          // 全局引入公共 SCSS 变量，无需在每个组件中重复 @import
          additionalData: `@use "@/styles/variables.scss" as *;`,
        },
      },
    },
    server: {
      port: 3000,
      open: true,
      // 💡 解决开发环境跨域（CORS）的核心配置：反向代理
      proxy: {
        [env.VITE_API_BASE_URL]: {
          target: env.VITE_PROXY_TARGET, // 代理的真实后端服务地址 (如 http://localhost:5000)
          changeOrigin: true,            // 允许跨域
          rewrite: (path) => path.replace(new RegExp(`^${env.VITE_API_BASE_URL}`), ''),
        },
      },
    },
    build: {
      outDir: 'dist',
      sourcemap: mode === 'development',
      chunkSizeWarningLimit: 1500, // 报错体积限制
      rollupOptions: {
        output: {
          // 💡 关键：对打包体积进行分包拆分（Vendor Splitting），防止生成单个巨大 JS 文件
          manualChunks(id) {
            if (id.includes('node_modules')) {
              // 将第三方库(如 vue, vue-router, element-plus)打包到单独的 vendor 模块中，利用浏览器强缓存
              return id.toString().split('node_modules/')[1].split('/')[0].toString();
            }
          },
        },
      },
    },
  };
});
```

---

## 3.2 代码质量守卫：ESLint + Prettier + Husky + lint-staged

当 3 人及以上的团队开发时，必须有一种强制机制来保证代码风格的绝对一致。

1. **ESLint**：代码质量校验工具（负责寻找逻辑 Bug，例如定义了未使用的变量、`no-implicit-any` 等）。
2. **Prettier**：代码格式化工具（负责统一单双引号、分号、换行符、缩进空格）。
3. **Husky & lint-staged**：本地 Git 钩子拦截器。

### 3.2.1 💡 为什么需要 Husky？它扮演了什么角色？
如果没有 Husky，团队成员完全可以不安装 ESLint 插件，直接修改代码并提交到 Git。一旦这些不规范的代码被推送到云端，合并时就会产生大量的样式冲突。

**Husky 的作用**：在你的本地电脑执行 `git commit` 时，**强行拦截**并先运行一次本地校验脚本（如代码扫描或单元测试）。只有校验通过了，才允许你生成 Commit。
**lint-staged 的作用**：只对你**当前修改并处于 Git 暂存区（Staged）** 的文件进行 ESLint 扫描和 Prettier 格式化，而不是去扫描整个项目（极大提升校验速度，通常小于 3 秒）。

### 3.2.2 核心配置文件设计

#### 1. ESLint 配置 (`eslint.config.js` - Flat Config 现代格式)
```javascript
import js from '@eslint/js';
import tseslint from 'typescript-eslint';
import pluginVue from 'eslint-plugin-vue';

export default [
  { ignored: ['dist', 'node_modules'] },
  js.configs.recommended,
  ...tseslint.configs.recommended,
  ...pluginVue.configs['flat/essential'],
  {
    rules: {
      'vue/multi-word-component-names': 'off', // 允许组件名为单个单词（如 Home.vue）
      '@typescript-eslint/no-explicit-any': 'warn', // 警告 any 类型的使用
      'no-debugger': process.env.NODE_ENV === 'production' ? 'error' : 'off',
    },
  },
];
```

#### 2. Package.json 中的拦截配置 (`package.json`)
```json
{
  "scripts": {
    "prepare": "husky install"
  },
  "husky": {
    "hooks": {
      "pre-commit": "lint-staged"
    }
  },
  "lint-staged": {
    "*.{ts,tsx,vue}": [
      "eslint --fix",
      "prettier --write"
    ]
  }
}
```

---

## 3.3 企业级 Monorepo 多包管理架构（与 C# Solution 深度对比）

在开发大型企业应用时，我们通常会有“管理后台”、“客户端 Web”、“APP 嵌 H5”等多套应用，但它们会大量共用底层的**接口定义、数据工具函数、通用 UI 组件、样式变量**。

如果采用传统多仓库开发，你就需要把公用代码抽成 npm 包，每次修改都要“发布包 -> 所有项目更新依赖”，开发效率极低。
为此，我们引入 **Monorepo（单仓多包）** 架构。

### 3.3.1 🔍 与 C# / .NET 解决方案的绝妙类比

C# 开发者对 Monorepo 其实拥有天生的直觉。在 .NET 中，我们最习惯的结构是：

```
MyEnterpriseSolution.sln          (解决方案 - 相当于 Monorepo 根目录)
├── src/MyEnterprise.WebAdmin/    (Web 应用 - 相当于前端 apps/admin)
├── src/MyEnterprise.WebClient/   (Web 客户端 - 相当于前端 apps/client)
└── src/MyEnterprise.Shared/      (通用类库 - 相当于前端 packages/shared)
```

在 C# 中，`WebAdmin` 想调用 `Shared`，只需要在 `.csproj` 里加一行项目引用：`<ProjectReference Include="..\MyEnterprise.Shared\MyEnterprise.Shared.csproj" />`。不需要发布任何 NuGet 包，只要在解决方案根目录编译，所有项目立刻同步更新。

**在前端，基于 `pnpm workspaces` 的 Monorepo 实现了完全一模一样的体验！**

---

### 3.3.2 基于 `pnpm workspaces` 的配置

`pnpm` 具有天然的支持。在 Monorepo 根目录下创建 `pnpm-workspace.yaml`：

```yaml
# pnpm-workspace.yaml
packages:
  # 所有的应用项目放在 apps 下
  - 'apps/*'
  # 所有的共享库、工具库、公用组件放在 packages 下
  - 'packages/*'
```

#### 根目录 `package.json`：
```json
{
  "name": "enterprise-monorepo",
  "private": true,
  "engines": {
    "node": ">=18.0.0",
    "pnpm": ">=8.0.0"
  },
  "devDependencies": {
    "husky": "^8.0.0",
    "prettier": "^3.0.0"
  }
}
```

#### 共享工具库：`packages/shared`
它也有自己的 `package.json`：
```json
{
  "name": "@enterprise/shared",
  "version": "1.0.0",
  "main": "src/index.ts",
  "dependencies": {
    "axios": "^1.6.0"
  }
}
```

#### 应用项目引入共享库：`apps/admin-app`
在 `apps/admin-app/package.json` 中，你可以这样引入：
```json
{
  "name": "@enterprise/admin-app",
  "dependencies": {
    "vue": "^3.3.0",
    "@enterprise/shared": "workspace:*"  // 💡 关键：表示直接引用本地工作区共享库
  }
}
```
此时，只要你在根目录下运行 `pnpm install`，pnpm 会自动在 `node_modules` 中建立一个**软链接（Symbolic Link）**，将 `@enterprise/shared` 指向本地的 `packages/shared` 目录。
当你在 `packages/shared` 里修改了任何代码，`apps/admin-app` 在开发时会**实时热更新**，无需执行任何发布打包操作！

---

## 3.4 Axios 拦截器与 TS 强类型通用网络请求客户端设计

在前后端分离的开发模式中，前端需要高频与后端 API 进行交互。如果我们在每个组件中都随意调用原生的 `fetch` 或是零散的 `axios`，会导致 Token 刷新、全局统一错误弹窗、状态码重定向难以归拢。

作为架构师，你需要提供一个高度工程化的 **Http Client**。

### 3.4.1 标准 API 返回值格式约定
在企业级架构中，后端 API 返回的 JSON 通常遵循以下统一数据模型：
```typescript
export interface ApiResponse<T = any> {
  code: number;      // 业务状态码 (如 200 成功，401 未登录，403 无权限，500 系统异常)
  data: T;           // 业务数据载荷，支持泛型
  message: string;   // 业务提示消息
}
```

---

### 3.4.2 强类型 Axios 请求类的完整封装（带 Token 自动刷新）

我们采用面向对象的思想，对 Axios 进行封装。它不仅支持泛型、拦截器，更内置了**网络断开/服务器异常的统一弹窗提示**，以及 **Token 过期后的无感刷新机制**。

```typescript
// packages/shared/src/api/httpClient.ts
import axios, { AxiosInstance, AxiosRequestConfig, AxiosResponse, InternalAxiosRequestConfig } from 'axios';

class HttpClient {
  private instance: AxiosInstance;
  private isRefreshing = false; // 是否正在刷新 Token
  private retryQueue: Array<(token: string) => void> = []; // 因 Token 过期等待刷新的请求队列

  constructor(baseURL: string, timeout = 10000) {
    this.instance = axios.create({
      baseURL,
      timeout,
      headers: {
        'Content-Type': 'application/json;charset=utf-8'
      }
    });

    this.setupInterceptors();
  }

  // 1. 初始化拦截器
  private setupInterceptors() {
    // 请求拦截器 (Request Interceptor)
    this.instance.interceptors.request.use(
      (config: InternalAxiosRequestConfig) => {
        const token = localStorage.getItem('ACCESS_TOKEN');
        if (token && config.headers) {
          // 统一向请求头注入 JWT ****** (与 C# [Authorize] 高效配对)
          config.headers['Authorization'] = `******;
        }
        return config;
      },
      (error) => Promise.reject(error)
    );

    // 响应拦截器 (Response Interceptor)
    this.instance.interceptors.response.use(
      async (response: AxiosResponse) => {
        const res = response.data; // 后端约定的 ApiResponse
        
        // 如果后端业务状态码 200，认为完全成功，直接返回业务层需要的数据
        if (res.code === 200) {
          return res;
        }

        // 💡 针对特定业务错误码的统一拦截处理
        switch (res.code) {
          case 401:
            // 401 代表 Token 过期或未登录，调用刷新 Token 逻辑
            return this.handleUnauthorized(response.config);
          case 403:
            console.error('权限不足，拒绝访问');
            // 可以触发全局弹窗或者跳转到 403 页面
            break;
          default:
            console.error(`业务异常: ${res.message || '未知错误'}`);
        }

        return Promise.reject(new Error(res.message || 'Error'));
      },
      (error) => {
        // 处理常规网络/HTTP 状态码异常（500，504，404等）
        let message = '网络连接异常';
        if (error.response) {
          switch (error.response.status) {
            case 500:
              message = '服务器内部崩溃 (500)';
              break;
            case 404:
              message = '请求接口未找到 (404)';
              break;
            default:
              message = `系统异常 (${error.response.status})`;
          }
        }
        console.error(message);
        return Promise.reject(error);
      }
    );
  }

  // 2. Token 过期后的无感双 Token 刷新算法
  private async handleUnauthorized(config: any) {
    if (!this.isRefreshing) {
      this.isRefreshing = true;
      try {
        const refreshToken = localStorage.getItem('REFRESH_TOKEN');
        // 调用刷新 Token 的接口
        const res = await axios.post(`${config.baseURL}/auth/refresh`, { refreshToken });
        const { accessToken, newRefreshToken } = res.data.data;
        
        localStorage.setItem('ACCESS_TOKEN', accessToken);
        localStorage.setItem('REFRESH_TOKEN', newRefreshToken);
        
        // 刷新成功，重放所有在队列中等待的请求
        this.retryQueue.forEach((callback) => callback(accessToken));
        this.retryQueue = [];
        this.isRefreshing = false;

        // 重新发送本次请求
        config.headers['Authorization'] = `******;
        return this.instance(config);
      } catch (err) {
        // 刷新 Token 彻底失败，说明用户离线太久，清除存储，强制返回登录页
        this.isRefreshing = false;
        this.retryQueue = [];
        localStorage.removeItem('ACCESS_TOKEN');
        localStorage.removeItem('REFRESH_TOKEN');
        window.location.href = '/login';
        return Promise.reject(err);
      }
    } else {
      // 如果正处于刷新中，将后续请求挂起并缓存，等待刷新完成后触发
      return new Promise((resolve) => {
        this.retryQueue.push((token: string) => {
          config.headers['Authorization'] = `******;
          resolve(this.instance(config));
        });
      });
    }
  }

  // 3. 封装统一强类型请求方法
  public get<T>(url: string, config?: AxiosRequestConfig): Promise<ApiResponse<T>> {
    return this.instance.get(url, config);
  }

  public post<T>(url: string, data?: any, config?: AxiosRequestConfig): Promise<ApiResponse<T>> {
    return this.instance.post(url, data, config);
  }

  public put<T>(url: string, data?: any, config?: AxiosRequestConfig): Promise<ApiResponse<T>> {
    return this.instance.put(url, data, config);
  }

  public delete<T>(url: string, config?: AxiosRequestConfig): Promise<ApiResponse<T>> {
    return this.instance.delete(url, config);
  }
}

// 导出全局单例客户端实例
export const api = new HttpClient('/api');
```

---

## 3.5 课后实操：构建基于 Monorepo 与 Axios 强类型契约的应用

### 🎯 实操目标
1. 掌握如何在 Monorepo 中跨包共享强类型模型。
2. 体验从 `packages/shared` 中导出 Http Client，在 `apps/admin-app` 中导入并配合 TypeScript 类型进行安全调用。

### 💻 核心实现步骤

#### 步骤 1：在 `packages/shared/src/types/user.ts` 中定义实体类型
```typescript
// C# 习惯：类似于实体类库
export interface UserEntity {
  id: number;
  username: string;
  nickName: string;
  departmentName: string;
  lastLoginTime: string;
  isOnline: boolean;
}
```

#### 步骤 2：在 `packages/shared/src/index.ts` 集中导出
```typescript
// 导出 API 客户端
export * from './api/httpClient';
// 导出公共类型
export * from './types/user';
```

#### 步骤 3：在应用端 `apps/admin-app/src/views/Dashboard.vue` 中安全调用
```vue
<template>
  <div class="dashboard-page">
    <h2>系统管理员工作区</h2>
    <div v-if="loading">加载中...</div>
    <div v-else>
      <div v-for="user in userList" :key="user.id" class="user-card">
        <h4>{{ user.nickName }} ({{ user.username }})</h4>
        <p>部门: {{ user.departmentName }}</p>
        <p>状态: <span :class="{ 'online': user.isOnline }">{{ user.isOnline ? '在线' : '离线' }}</span></p>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from 'vue';
// 💡 直接从共享 Monorepo 模块导入网络请求实例与强类型契约！
import { api, UserEntity } from '@enterprise/shared';

const loading = ref(false);
// 声明强类型响应式数组
const userList = ref<UserEntity[]>([]);

const fetchUserData = async () => {
  loading.value = true;
  try {
    // 💡 传入泛型 <UserEntity[]>！Axios 会自动推导 api.get() 返回的值具有 ApiResponse<UserEntity[]> 格式
    const res = await api.get<UserEntity[]>('/users/list');
    
    // res 的类型在编译期被完美识别为 ApiResponse<UserEntity[]>
    // res.data 的类型是 UserEntity[]，极度安全，防写错单词
    userList.value = res.data;
  } catch (error) {
    console.error('获取用户数据失败:', error);
  } finally {
    loading.value = false;
  }
};

onMounted(() => {
  fetchUserData();
});
</script>

<style scoped>
.user-card {
  border: 1px solid #ddd;
  padding: 15px;
  margin-bottom: 10px;
  border-radius: 4px;
}
.online {
  color: green;
  font-weight: bold;
}
</style>
```

本章我们彻底打通了现代前端脚手架、代码规范、多项目 Monorepo 软链接调用以及 Axios 工业级无感刷新 Token 的实现。一个强大的架构师不仅要把控项目的代码和结构，更要具备极强的团队领导力、高标准发布交付，以及极高安全性和性能意识。我们将在下一章深入探讨 —— **[第 04 章：前端架构师的团队领导力与工程落地](./ch4_engineering_team.md)**！
