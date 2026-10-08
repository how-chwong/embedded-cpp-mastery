# 第 05 章：Vue 生态系统

> 目标：掌握 Vue Router、Pinia、Axios、VueUse 等核心生态库，具备构建完整前端应用的能力。

---

## 5.1 Vue Router 4

### 安装与基础配置

```typescript
// router/index.ts
import { createRouter, createWebHistory } from 'vue-router';
import type { RouteRecordRaw } from 'vue-router';

const routes: RouteRecordRaw[] = [
  {
    path: '/',
    name: 'Home',
    component: () => import('@/views/HomeView.vue'), // 懒加载
  },
  {
    path: '/users',
    name: 'Users',
    component: () => import('@/views/UsersView.vue'),
    meta: { requiresAuth: true, title: '用户列表' },
  },
  {
    path: '/users/:id',
    name: 'UserDetail',
    component: () => import('@/views/UserDetailView.vue'),
    props: true, // 将路由参数作为 Props 传入
  },
  {
    path: '/admin',
    component: () => import('@/layouts/AdminLayout.vue'),
    children: [
      { path: '', redirect: '/admin/dashboard' },
      { path: 'dashboard', component: () => import('@/views/admin/Dashboard.vue') },
      { path: 'users', component: () => import('@/views/admin/Users.vue') },
    ],
  },
  {
    path: '/:pathMatch(.*)*',
    name: 'NotFound',
    component: () => import('@/views/NotFound.vue'),
  },
];

const router = createRouter({
  history: createWebHistory(import.meta.env.BASE_URL),
  routes,
  scrollBehavior(to, from, savedPosition) {
    if (savedPosition) return savedPosition;
    return { top: 0, behavior: 'smooth' };
  },
});

export default router;
```

### 导航守卫（路由鉴权）

```typescript
// router/index.ts
import { useAuthStore } from '@/stores/auth';

// 全局前置守卫（鉴权）
router.beforeEach(async (to, from, next) => {
  const authStore = useAuthStore();
  
  // 设置页面标题
  document.title = (to.meta.title as string) || 'My App';
  
  if (to.meta.requiresAuth && !authStore.isLoggedIn) {
    // 未登录，跳转到登录页，并记录目标路由
    next({ name: 'Login', query: { redirect: to.fullPath } });
    return;
  }
  
  next();
});

// 全局后置钩子（数据上报、进度条关闭）
router.afterEach((to, from) => {
  console.log(`导航到: ${to.path}`);
});
```

### 组件内使用

```vue
<script setup lang="ts">
import { useRouter, useRoute } from 'vue-router';

const router = useRouter();
const route = useRoute();

// 读取路由参数
const userId = computed(() => route.params.id as string);
const queryFilter = computed(() => route.query.filter as string);

// 编程式导航
function goToUser(id: number) {
  router.push({ name: 'UserDetail', params: { id } });
}

function goBack() {
  router.back();
}

// 替换（不留历史记录）
function replaceRoute() {
  router.replace({ name: 'Home' });
}
</script>

<template>
  <!-- 声明式导航 -->
  <RouterLink to="/">首页</RouterLink>
  <RouterLink :to="{ name: 'Users' }" active-class="active">用户</RouterLink>
  
  <!-- 路由出口 -->
  <RouterView />
</template>
```

---

## 5.2 Pinia 状态管理

### 与 Vuex 对比

| 特性 | Vuex 4 | Pinia |
|------|--------|-------|
| TypeScript | 较差 | **原生支持** |
| DevTools | 支持 | **支持** |
| 学习曲线 | 陡峭 | **平缓** |
| Mutation | 必须 | **不需要** |
| 模块化 | 复杂 | **自动** |
| 代码量 | 多 | **少** |

### 创建 Store

```typescript
// stores/user.ts
import { defineStore } from 'pinia';
import { ref, computed } from 'vue';
import { userApi } from '@/services/api';
import type { User } from '@/types';

// Option Store（类似 Vue 2 Options API）
export const useUserStoreOptions = defineStore('user-options', {
  state: () => ({
    users: [] as User[],
    currentUser: null as User | null,
    loading: false,
  }),
  getters: {
    activeUsers: (state) => state.users.filter(u => u.active),
    userCount: (state) => state.users.length,
  },
  actions: {
    async fetchUsers() {
      this.loading = true;
      try {
        this.users = await userApi.getAll();
      } finally {
        this.loading = false;
      }
    },
  },
});

// Setup Store（推荐，类似 Composition API）
export const useUserStore = defineStore('user', () => {
  // state
  const users = ref<User[]>([]);
  const currentUser = ref<User | null>(null);
  const loading = ref(false);

  // getters
  const activeUsers = computed(() => users.value.filter(u => u.active));
  const userCount = computed(() => users.value.length);

  // actions
  async function fetchUsers() {
    loading.value = true;
    try {
      users.value = await userApi.getAll();
    } finally {
      loading.value = false;
    }
  }

  async function updateUser(id: number, data: Partial<User>) {
    const updated = await userApi.update(id, data);
    const index = users.value.findIndex(u => u.id === id);
    if (index !== -1) {
      users.value[index] = updated;
    }
    return updated;
  }

  function $reset() {
    users.value = [];
    currentUser.value = null;
    loading.value = false;
  }

  return {
    users,
    currentUser,
    loading,
    activeUsers,
    userCount,
    fetchUsers,
    updateUser,
    $reset,
  };
});
```

### 组件中使用

```vue
<script setup lang="ts">
import { storeToRefs } from 'pinia';
import { useUserStore } from '@/stores/user';

const userStore = useUserStore();

// ✅ 使用 storeToRefs 保持响应性（解构时不丢失）
const { users, loading, activeUsers } = storeToRefs(userStore);

// Actions 直接解构（函数不需要 storeToRefs）
const { fetchUsers, updateUser } = userStore;

onMounted(() => {
  fetchUsers();
});
</script>

<template>
  <div v-if="loading">加载中...</div>
  <ul v-else>
    <li v-for="user in activeUsers" :key="user.id">
      {{ user.name }}
    </li>
  </ul>
</template>
```

### Pinia 持久化

```bash
pnpm add pinia-plugin-persistedstate
```

```typescript
// main.ts
import piniaPluginPersistedstate from 'pinia-plugin-persistedstate';
const pinia = createPinia();
pinia.use(piniaPluginPersistedstate);

// stores/auth.ts
export const useAuthStore = defineStore('auth', () => {
  const token = ref<string | null>(null);
  const user = ref<User | null>(null);
  // ...
}, {
  persist: {
    key: 'auth',
    storage: localStorage,
    paths: ['token', 'user'],
  },
});
```

---

## 5.3 Axios HTTP 客户端

### 封装 Axios 实例

```typescript
// services/http.ts
import axios from 'axios';
import type { AxiosInstance, AxiosRequestConfig, AxiosResponse } from 'axios';
import { useAuthStore } from '@/stores/auth';
import router from '@/router';

const http: AxiosInstance = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  timeout: 10000,
  headers: {
    'Content-Type': 'application/json',
  },
});

// 请求拦截器：自动添加 Token
http.interceptors.request.use(
  (config) => {
    const authStore = useAuthStore();
    if (authStore.token) {
      config.headers.Authorization = `******;
    }
    return config;
  },
  (error) => Promise.reject(error),
);

// 响应拦截器：统一错误处理
http.interceptors.response.use(
  (response: AxiosResponse) => {
    // 解包后端统一响应格式 { code, data, message }
    const { code, data, message } = response.data;
    if (code === 0) return data;
    return Promise.reject(new Error(message));
  },
  async (error) => {
    const { response } = error;
    
    if (response?.status === 401) {
      const authStore = useAuthStore();
      authStore.$reset();
      router.push({ name: 'Login' });
    } else if (response?.status === 403) {
      router.push({ name: 'Forbidden' });
    } else if (response?.status >= 500) {
      console.error('服务器错误:', response.data);
    }
    
    return Promise.reject(error);
  },
);

export default http;
```

### API 服务层

```typescript
// services/api/user.ts
import http from '../http';
import type { User, CreateUserDto, UpdateUserDto } from '@/types';

interface PaginatedResponse<T> {
  items: T[];
  total: number;
  page: number;
  pageSize: number;
}

export const userApi = {
  getAll(params?: { page?: number; pageSize?: number; search?: string }) {
    return http.get<PaginatedResponse<User>>('/users', { params });
  },

  getById(id: number) {
    return http.get<User>(`/users/${id}`);
  },

  create(data: CreateUserDto) {
    return http.post<User>('/users', data);
  },

  update(id: number, data: UpdateUserDto) {
    return http.put<User>(`/users/${id}`, data);
  },

  delete(id: number) {
    return http.delete<void>(`/users/${id}`);
  },

  uploadAvatar(id: number, file: File) {
    const form = new FormData();
    form.append('avatar', file);
    return http.post<{ url: string }>(`/users/${id}/avatar`, form, {
      headers: { 'Content-Type': 'multipart/form-data' },
    });
  },
};
```

---

## 5.4 VueUse（组合式工具集）

```bash
pnpm add @vueuse/core
```

```typescript
import {
  useLocalStorage,
  useSessionStorage,
  useDebounceFn,
  useThrottleFn,
  useIntersectionObserver,
  useResizeObserver,
  useFetch,
  useEventListener,
  useWindowSize,
  useBreakpoints,
  onClickOutside,
  useClipboard,
  useMediaQuery,
} from '@vueuse/core';

// 本地存储（自动响应式）
const theme = useLocalStorage('theme', 'light');

// 防抖搜索
const search = useDebounceFn(async (query: string) => {
  results.value = await searchApi(query);
}, 300);

// 窗口尺寸
const { width, height } = useWindowSize();
const isMobile = computed(() => width.value < 768);

// 媒体查询
const isSmall = useMediaQuery('(max-width: 640px)');

// 剪贴板
const { copy, copied } = useClipboard();
// copy('Hello')，copied 在 2 秒后重置为 false

// 无限滚动
const target = ref(null);
const { stop } = useIntersectionObserver(target, ([{ isIntersecting }]) => {
  if (isIntersecting) loadMore();
});

// 点击外部
const popup = ref(null);
onClickOutside(popup, () => { isOpen.value = false; });

// 简单 Fetch
const { data, isFetching, error } = useFetch('/api/users').json<User[]>();
```

---

## 5.5 UI 组件库

### 主流选择对比

| 组件库 | 特点 | 适用场景 |
|--------|------|---------|
| Element Plus | 功能全面，中文文档 | B端管理后台（首选）|
| Naive UI | TypeScript 友好，性能好 | 中后台、创业项目 |
| Ant Design Vue | 规范严格，大厂背书 | 企业级应用 |
| Vuetify | Material Design | Google 风格应用 |
| shadcn-vue | 无样式基础组件，高度可定制 | 设计要求高的项目 |
| Tailwind CSS | 原子化 CSS | 自定义设计系统 |

### Element Plus 快速集成

```bash
pnpm add element-plus
pnpm add -D unplugin-vue-components unplugin-auto-import
```

```typescript
// vite.config.ts（按需导入，减少包体积）
import AutoImport from 'unplugin-auto-import/vite';
import Components from 'unplugin-vue-components/vite';
import { ElementPlusResolver } from 'unplugin-vue-components/resolvers';

export default defineConfig({
  plugins: [
    AutoImport({ resolvers: [ElementPlusResolver()] }),
    Components({ resolvers: [ElementPlusResolver()] }),
  ],
});
```

```vue
<!-- 无需导入直接使用 -->
<template>
  <el-button type="primary" :loading="loading" @click="submit">
    提交
  </el-button>
  
  <el-table :data="users" stripe>
    <el-table-column prop="name" label="姓名" />
    <el-table-column prop="email" label="邮箱" />
    <el-table-column label="操作">
      <template #default="{ row }">
        <el-button @click="edit(row)">编辑</el-button>
      </template>
    </el-table-column>
  </el-table>
  
  <el-pagination
    v-model:current-page="page"
    v-model:page-size="pageSize"
    :total="total"
    layout="total, sizes, prev, pager, next"
  />
</template>
```

---

## 5.6 国际化（vue-i18n）

```bash
pnpm add vue-i18n
```

```typescript
// i18n/index.ts
import { createI18n } from 'vue-i18n';

const messages = {
  zh: {
    common: { save: '保存', cancel: '取消', delete: '删除' },
    user: { title: '用户管理', count: '共 {count} 位用户' },
  },
  en: {
    common: { save: 'Save', cancel: 'Cancel', delete: 'Delete' },
    user: { title: 'User Management', count: '{count} users total' },
  },
};

const i18n = createI18n({
  legacy: false, // Composition API 模式
  locale: localStorage.getItem('locale') || 'zh',
  fallbackLocale: 'en',
  messages,
});

export default i18n;
```

```vue
<script setup lang="ts">
import { useI18n } from 'vue-i18n';

const { t, locale } = useI18n();

function switchLocale() {
  locale.value = locale.value === 'zh' ? 'en' : 'zh';
  localStorage.setItem('locale', locale.value);
}
</script>

<template>
  <h1>{{ t('user.title') }}</h1>
  <p>{{ t('user.count', { count: 100 }) }}</p>
  <button @click="switchLocale">切换语言</button>
</template>
```

---

## 5.7 章节小结

| 库 | 用途 | 优先级 |
|----|------|--------|
| Vue Router | 路由管理 | ⭐⭐⭐⭐⭐ |
| Pinia | 全局状态 | ⭐⭐⭐⭐⭐ |
| Axios + 封装 | HTTP 请求 | ⭐⭐⭐⭐⭐ |
| VueUse | 组合式工具 | ⭐⭐⭐⭐⭐ |
| Element Plus / Naive | UI 组件库 | ⭐⭐⭐⭐⭐ |
| vue-i18n | 国际化 | ⭐⭐⭐ |

> **下一步**：阅读 [第 06 章 - 工程化与代码规范](06_engineering.md)
