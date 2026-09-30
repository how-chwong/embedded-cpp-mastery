# 第 02 章：Vue 3 核心架构与 Composition API 深度实践

现代前端框架的核心目标是实现 **数据与视图的声明式绑定**，让开发者能够专注于状态的控制，而无需手动去写极其繁琐的 DOM 增删改查。

在本章中，我们将深入 Vue 3 的核心运行机制，解析其底层响应式系统（ES6 Proxy），学习 Composition API 的最佳工程实践，并构建起路由与全局状态管理的黄金架构（Vue Router + Pinia），同时保持与 C# 底层机制的对照。

---

## 2.1 Vue 3 响应式系统底层原理与 C# 深度对比

Vue 3 的底层最核心的卖点是其全新的**响应式系统**。

### 2.1.1 依赖收集（Dependency Tracking）的运行机制

在 Vue 3 中，当你使用 `ref` 或 `reactive` 定义一个数据时，Vue 在底层通过 ES6 **`Proxy`** 拦截了该对象的所有读取（`get`）和写入（`set`）操作。

* **依赖收集（Track）**：当视图模板（或 `watchEffect`）在渲染过程中，读取了某个响应式属性的值，就会触发 `Proxy` 的 `get` 拦截。此时，Vue 会把当前的渲染任务（称为一个 `Effect`）保存起来，记录为“这个属性的忠实订阅者”。
* **触发更新（Trigger）**：当你在业务代码中修改了该响应式属性的值，触发了 `Proxy` 的 `set` 拦截。此时，Vue 会根据之前记录的映射关系，找到所有依赖于该属性的 `Effect`（渲染任务），并将它们全部放入微任务队列中进行异步刷新。

```
[数据 A] ──触发 get 拦截──> [ 依赖收集 Track ] ──> 将当前组件渲染 Effect 记录到 [ Dep Map ]
   │
被修改 (A = newValue)
   │
   └───触发 set 拦截──> [ 触发更新 Trigger ] ──> 从 [ Dep Map ] 中取出 Effect ──> 放入微任务队列异步渲染
```

---

### 2.1.2 Vue 3 (Proxy) vs Vue 2 (defineProperty)

如果你之前了解过 Vue 2，理解这两者的变化对高级面试至关重要：

| 维度 | Vue 2 (`Object.defineProperty`) | Vue 3 (`Proxy`) |
| :--- | :--- | :--- |
| **拦截广度** | 只能拦截属性的 `get` / `set`。 | **可以拦截整个对象**的所有操作，包括新增属性、删除属性（`deleteProperty`）、属性枚举等 13 种陷阱方法。 |
| **数组支持** | 无法完美拦截数组的索引修改和 `length` 变化。需要对数组原型方法（`push`/`pop` 等）进行手动重写。 | 完美的、全天然的数组和集合（Map, Set）响应式拦截支持。 |
| **性能开销** | 必须在初始化时，**递归遍历**对象的所有层级和每一个属性进行响应式拦截。如果数据极大，初始化时会造成明显的卡顿（CPU 耗时）。 | **懒惰拦截（Lazy Proxy）**。只有在开发人员真正访问了深层嵌套属性时，Vue 才会动态地为该层子对象创建 Proxy 包装，极大地降低了首屏开销。 |

---

### 2.1.3 🔍 与 C# 双向绑定机制深度对比

如果你熟悉 WPF 或是 Blazor 的 MVVM 开发，你一定对数据绑定（Data Binding）不陌生。

1. **C# WPF 中的绑定**：
   * 需要实体类实现 **`INotifyPropertyChanged`** 接口。
   * 在属性的 `set` 访问器中，开发人员必须手动编写或者利用编译器魔法触发 `OnPropertyChanged(nameof(MyProperty))` 事件。
   * **差异点**：C# 属于**命令式/半自动**的声明。每一次写属性你都得主动发通知。而在 Vue 3 中，借助于动态代理（Proxy），属性通知是**完全静默且全自动**的。
2. **C# Blazor 中的绑定**：
   * Blazor 采用组件级状态更新，在发生事件（例如点击、输入）后，Blazor 引擎在底层自动运行组件的生命周期并重新评估 DOM 树，通常伴随着组件的整体重绘或手写 `StateHasChanged()` 强制刷新。

---

## 2.2 Composition API 与 Reactivity 最佳实践

Vue 3 引入了 **Composition API（组合式 API）**。在 Vue 2 的 Options API（选项式 API）中，一个组件的逻辑分散在 `data`、`methods`、`computed`、`mounted` 中，当文件超过 1000 行时，修改一个功能需要上下反复横跳，维护极其痛苦。

Composition API 允许我们**按照业务功能维度**，将相关的数据、计算属性、方法和生命周期打包聚合在一起，甚至提取成独立的可复用函数（称为 **Composables**）。

### 2.2.1 `ref` 与 `reactive` 的选用标准与工程守则
在编写 Vue 3 组件时，新手最容易在 `ref` 和 `reactive` 的选择上陷入迷茫。以下是前端架构组制定的**严苛工程规范**：

```typescript
import { ref, reactive } from 'vue';

// 1. ref：适合所有基本类型（number, string, boolean 等）以及“可能需要直接替换整个对象”的场景
const userId = ref<number | null>(null);
const userInfo = ref({ name: 'Alice', age: 20 });
userInfo.value = { name: 'Bob', age: 25 }; // 合法！直接替换整个对象，依然能保持响应式。

// 2. reactive：仅适用于“内部属性极其稳定，绝对不需要整体替换”的大型复杂对象或状态表单
const formState = reactive({
  username: '',
  password: '',
  isRemember: false
});
// ❌ 错误反例：不要直接对 reactive 对象赋值，这会丢失它的 Proxy 代理！
// formState = { username: '张三', ... }; // 这将彻底破坏响应式绑定！
```

**💡 团队铁律（Best Practices）：**
* **优先使用 `ref`**：因为 `.value` 提供了清晰的线索，明确告诉阅读代码的工程师这是一个“响应式引用”，同时直接整体赋值不会导致响应式丢失。
* **只有在定义组件内部的多字段状态表单（Form State）时**，才考虑使用 `reactive`。

---

### 2.2.2 高级计算属性 `computed` 与副作用侦听 `watch` / `watchEffect`
* **`computed`**：具有**缓存性**。只有当它依赖的响应式数据发生改变时，它才会重新计算。绝对不能在计算属性中执行副作用操作（如发起 API 请求、修改全局状态）。
* **`watch`**：惰性侦听。需要明确指定侦听源，能够获取到新值（`newValue`）和旧值（`oldValue`），适合在状态变化时执行异步操作。
* **`watchEffect`**：立即运行并自动收集其内部引用的所有响应式属性作为依赖。只要其中任意属性发生改变，函数便会重新运行，适合做自动同步逻辑。

```typescript
import { ref, computed, watchEffect } from 'vue';

const query = ref('');
const list = ref(['Apple', 'Banana', 'Orange']);

// 自动缓存的过滤列表
const filteredList = computed(() => {
  return list.value.filter(item => item.toLowerCase().includes(query.value.toLowerCase()));
});

// watchEffect 会立即执行一次，并自动侦听 query.value 的变化
watchEffect((onCleanup) => {
  console.log('当前搜索关键字已变更为:', query.value);
  
  const timer = setTimeout(() => {
    // 模拟搜索打点分析
  }, 500);
  
  // 清理函数：如果 query 连续高频变更（防抖），会在下一次 watchEffect 触发前运行，清理掉上一次的未完成定时器
  onCleanup(() => clearTimeout(timer));
});
```

---

## 2.3 Vue Router 企业级路由管理与权限守卫

在单页应用（SPA）中，所有页面切换其实都是**同一页面内的 DOM 替换**。负责控制这一切换逻辑的核心机制就是 **Vue Router**。

### 2.3.1 路由配置与动态路由（Dynamic Routing）
在大型后台管理系统中，我们不能把所有菜单和路由都硬编码在本地文件里，因为不同的用户角色（管理员、普通员工）看到的界面完全不同。我们通常采用**动态路由**方案：

```typescript
import { createRouter, createWebHistory, RouteRecordRaw } from 'vue-router';

// 基础路由：所有人都能访问
const constantRoutes: RouteRecordRaw[] = [
  {
    path: '/login',
    name: 'Login',
    component: () => import('@/views/login/index.vue'), // 路由懒加载：提升首屏加载性能
  },
  {
    path: '/404',
    component: () => import('@/views/error/404.vue'),
  }
];

export const router = createRouter({
  history: createWebHistory(),
  routes: constantRoutes,
});
```

---

### 2.3.2 全局路由导航守卫（Navigation Guards）与权限控制流程
通过全局前置守卫 `beforeEach`，我们可以像 .NET 中的 **Middleware（中间件）** 一样拦截路由请求：

```typescript
// src/router/permission.ts
import { router } from './index';
import { useUserStore } from '@/store/user';

const WHITE_LIST = ['/login', '/404'];

router.beforeEach(async (to, from, next) => {
  const userStore = useUserStore();
  const token = userStore.token;

  if (token) {
    if (to.path === '/login') {
      next({ path: '/' }); // 已登录，免密进入首页
    } else {
      // 检查当前 store 是否已经加载了用户信息和角色权限列表
      const hasRoles = userStore.roles && userStore.roles.length > 0;
      if (hasRoles) {
        next();
      } else {
        try {
          // 异步获取用户信息与角色列表
          const roles = await userStore.getUserInfo();
          
          // 💡 核心：根据角色动态生成符合权限的路由表，并动态添加到路由系统
          const accessRoutes = await userStore.generateRoutes(roles);
          accessRoutes.forEach(route => {
            router.addRoute(route); // 动态注入路由！
          });
          
          // 确保路由添加完成后再进行跳转
          next({ ...to, replace: true });
        } catch (error) {
          // 获取用户信息失败，清除 Token 并重定向至登录页
          userStore.resetToken();
          next(`/login?redirect=${to.path}`);
        }
      }
    }
  } else {
    // 未登录
    if (WHITE_LIST.includes(to.path)) {
      next();
    } else {
      next(`/login?redirect=${to.path}`);
    }
  }
});
```

---

## 2.4 Pinia 状态管理架构设计（与 C# DI 容器类比）

在复杂的单页应用中，组件是层层嵌套的。如果“组件 A”想要修改“组件 Z”的数据，只能通过层层事件传递（Props Drilling），这会导致代码极难维护。为此，我们引入了 **Pinia（官方推荐的新一代状态管理库）**。

### 2.4.1 Pinia 核心心智：Store 到底是什么？
Pinia 的 Store 就像是一个全局的、任何组件都能直接访问的“**内存数据库**”与“**核心业务服务**”。
* **State**：定义状态数据（类似于组件的 `data`）。
* **Getters**：定义依赖状态的计算属性（类似于组件的 `computed`）。
* **Actions**：定义修改状态的方法（可以是同步的，也可以是异步的，类似于组件的 `methods`）。

```typescript
// src/store/counter.ts
import { defineStore } from 'pinia';
import { ref, computed } from 'vue';

// 采用 Setup Store 写法，与 Composition API 保持高度统一，极易理解
export const useCounterStore = defineStore('counter', () => {
  // 1. State
  const count = ref(0);
  
  // 2. Getter
  const doubleCount = computed(() => count.value * 2);
  
  // 3. Action
  function increment() {
    count.value++;
  }
  
  function decrement() {
    count.value--;
  }

  return { count, doubleCount, increment, decrement };
});
```

---

### 2.4.2 🔍 与 C# DI / IoC 容器及单例（Singleton）的类比

对于 C# 开发者，你可以将 **Pinia Store** 完美地等同于 .NET 中的 **单例依赖注入服务（Singleton Registered Services）**。

1. **依赖生命周期**：
   在 ASP.NET Core 中，你通过 `builder.Services.AddSingleton<ICounterService, CounterService>()` 注册一个全局服务。任何 Controller 或 Service 只要在构造函数中声明，就能获取到这个共享的单例实例。
   * **在 Pinia 中**：你通过 `const counterStore = useCounterStore()` 获取 Store 实例。只要是同一个 Store ID（如 `'counter'`），不管你在 100 个不同的组件里调用多少次 `useCounterStore()`，返回的都是**同一个内存实例**，任何组件对其状态的修改都会立刻共享。
2. **状态保护与并发**：
   * **C# 单例服务**：在多线程环境下，修改 C# 单例属性时，必须小心锁机制（`lock`, `SemaphoreSlim`），防止线程冲突与脏数据。
   * **Pinia Store**：运行在单线程的 JavaScript 事件循环中，**绝无多线程并发冲突**的问题！你可以非常优雅地进行状态变更，无需任何加锁操作。

---

## 2.5 课后实操：手写高复用强类型 `useFetch` 异步加载组合式函数

作为未来的高级前端/架构师，你必须有能力抽取通用的业务逻辑。接下来，我们将手写一个高复用、带强类型约束、带**请求防重（AbortController 自动取消）**以及**重试机制（Retry）** 的企业级通用数据异步获取 Composable。

### 🎯 实操目标
1. 掌握如何封装高内聚、低耦合的 Composable 函数。
2. 支持泛型 `<T>`，实现完美的返回值类型推导。
3. 内置防抖/防重逻辑：如果前一次请求还未完成又触发了新的请求，自动调用 `AbortController` 取消前一次请求，防止竞态（Race Condition）引起数据混乱。
4. 内置重试机制，可配置最大重试次数。

### 💻 核心实现代码

```typescript
// src/composables/useFetch.ts
import { ref, Ref, watch } from 'vue';

interface FetchOptions {
  immediate?: boolean; // 是否立即执行
  retryCount?: number; // 失败重试次数
  retryDelay?: number; // 失败重试延迟(ms)
}

interface FetchResult<T> {
  data: Ref<T | null>;
  loading: Ref<boolean>;
  error: Ref<Error | null>;
  execute: () => Promise<void>;
}

export function useFetch<T>(
  urlRef: Ref<string> | (() => string),
  options: FetchOptions = {}
): FetchResult<T> {
  const { immediate = true, retryCount = 0, retryDelay = 1000 } = options;

  const data = ref<T | null>(null) as Ref<T | null>;
  const loading = ref(false);
  const error = ref<Error | null>(null) as Ref<Error | null>;

  let abortController: AbortController | null = null;

  async function execute(): Promise<void> {
    // 1. 如果之前有正在执行的请求，直接取消它！(Abort)
    if (abortController) {
      abortController.abort();
    }
    
    // 2. 创建全新的 AbortController 并保存
    abortController = new AbortController();
    
    loading.value = true;
    error.value = null;

    const currentUrl = typeof urlRef === 'function' ? urlRef() : urlRef.value;

    let attempts = 0;

    async function attemptFetch(): Promise<void> {
      try {
        const response = await fetch(currentUrl, {
          signal: abortController?.signal
        });

        if (!response.ok) {
          throw new Error(`HTTP 异常！状态码: ${response.status}`);
        }

        const json = await response.json();
        data.value = json as T;
      } catch (err: any) {
        if (err.name === 'AbortError') {
          console.log('请求被主动取消:', currentUrl);
          return; // 被主动取消的请求，不进行错误反馈与重试
        }

        if (attempts < retryCount) {
          attempts++;
          console.warn(`请求失败，准备第 ${attempts} 次重试...`);
          await new Promise(resolve => setTimeout(resolve, retryDelay));
          return attemptFetch();
        }

        error.value = err as Error;
      } finally {
        loading.value = false;
      }
    }

    await attemptFetch();
  }

  // 3. 响应式监听 url 变化：如果 Url 改变，自动重新发起请求
  if (typeof urlRef !== 'function') {
    watch(urlRef, () => {
      execute();
    });
  }

  // 4. 控制是否立即执行
  if (immediate) {
    execute();
  }

  return { data, loading, error, execute };
}
```

### 📝 在 Vue 3 组件中引用该 Composable：

```vue
<!-- src/views/UserList.vue -->
<template>
  <div class="user-list">
    <h3>用户数据列表 (Composable 示例)</h3>
    <div v-if="loading" class="loading">加载中，请稍候...</div>
    <div v-else-if="error" class="error-msg">
      数据获取失败：{{ error.message }}
      <button @click="execute">点击重试</button>
    </div>
    <ul v-else-if="data">
      <li v-for="user in data" :key="user.id">
        {{ user.name }} - {{ user.email }}
      </li>
    </ul>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue';
import { useFetch } from '@/composables/useFetch';

interface User {
  id: number;
  name: string;
  email: string;
}

// 模拟动态变更 API 的基础 Url
const apiUrl = ref('https://jsonplaceholder.typicode.com/users');

// 调用高复用 useFetch，传入泛型 <User[]>，获得完美的类型推导！
const { data, loading, error, execute } = useFetch<User[]>(apiUrl, {
  immediate: true,
  retryCount: 3,       // 失败自动重试 3 次
  retryDelay: 1500     // 重试延迟 1.5 秒
});

// 在 VS Code 中，如果你访问 data.value[0]，你会得到完整的 id/name/email 智能自动补全！
</script>
```

本章中我们对 Vue 3 的底层原理、组件编写守则、路由守卫与全局 Pinia Store 架构进行了系统化提炼。在实际开发中，这些业务代码需要放置在一个健壮、规范的工程底座之上。在下一章，我们将讲解如何从零配置这样的底座 —— **[第 03 章：企业级工程化脚手架与 Monorepo 架构设计](./ch3_architecture_tools.md)**！
