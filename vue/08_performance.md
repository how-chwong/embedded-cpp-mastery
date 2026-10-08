# 第 08 章：性能优化

> 目标：掌握前端性能优化的核心方法，能够识别和解决 Vue 应用中的性能瓶颈，将应用优化到生产级标准。

---

## 8.1 性能优化思维框架

```
性能优化方向
├── 加载性能（首屏速度）
│   ├── 减少资源体积（Tree-shaking、代码分割、压缩）
│   ├── 减少请求数量（合并、缓存、预加载）
│   └── 加快传输速度（CDN、HTTP/2、Gzip/Brotli）
├── 运行性能（交互流畅）
│   ├── 减少渲染次数（memo、v-memo、shallowRef）
│   ├── 减少渲染工作量（虚拟列表、懒加载）
│   └── 避免主线程阻塞（Web Worker、分批任务）
└── 感知性能（用户体验）
    ├── 加载动画 / 骨架屏
    ├── 乐观更新
    └── 预加载 / 预渲染
```

---

## 8.2 Vue 渲染优化

### v-memo（跳过子树更新）

```vue
<template>
  <!-- 只有 user.id 或 selected 变化时才重新渲染 -->
  <div v-for="user in users" :key="user.id" v-memo="[user.id, selected]">
    <UserRow :user="user" :selected="selected === user.id" />
  </div>
</template>
```

### shallowRef / shallowReactive（浅响应式）

```typescript
// 对于大型只读数据，使用 shallowRef 避免深度追踪
const tableData = shallowRef<User[]>([]);

// 更新时必须替换整个对象触发更新
tableData.value = [...newData]; // ✅
tableData.value.push(newItem);  // ❌ 不触发更新

// shallowReactive：只追踪第一层属性
const state = shallowReactive({
  config: { /* 大型配置对象，不需要深度响应 */ },
  count: 0,
});
```

### 避免不必要的组件更新

```vue
<script setup lang="ts">
// ❌ 每次父组件渲染，子组件都重渲染
// 因为函数每次都是新引用

// ✅ 使用 computed 或 useCallback 式写法
const handleClick = useCallback(() => { ... }); // VueUse

// ✅ 对于纯展示组件，使用 functional
// 或者确保 Props 是稳定引用
</script>

<template>
  <!-- ❌ 内联 handler 每次都新建函数 -->
  <ChildComponent :on-click="() => handleItem(item)" />
  
  <!-- ✅ 提前绑定参数 -->
  <ChildComponent :on-click="handlers[item.id]" />
</template>
```

### 计算属性缓存

```typescript
// ✅ computed 有缓存，依赖不变则不重新计算
const expensiveResult = computed(() => {
  return heavyCalculation(rawData.value); // 只在 rawData 变化时执行
});

// ❌ 方法调用每次渲染都执行
function expensiveMethod() {
  return heavyCalculation(rawData.value); // 每次都执行
}
```

---

## 8.3 代码分割与懒加载

### 路由级别分割（最重要）

```typescript
// router/index.ts
// 每个路由组件都懒加载 → 首屏只加载当前页面代码
const routes = [
  {
    path: '/dashboard',
    component: () => import('@/views/Dashboard.vue'),
    // Vite 魔法注释：自定义 chunk 名
    component: () => import(/* webpackChunkName: "dashboard" */ '@/views/Dashboard.vue'),
  },
  // 将相关路由打包到同一 chunk
  {
    path: '/admin/users',
    component: () => import(/* webpackChunkName: "admin" */ '@/views/admin/Users.vue'),
  },
  {
    path: '/admin/settings',
    component: () => import(/* webpackChunkName: "admin" */ '@/views/admin/Settings.vue'),
  },
];
```

### 组件级别懒加载

```typescript
// 大型第三方组件按需加载
const RichEditor = defineAsyncComponent(() => import('@/components/RichEditor.vue'));
const HeavyChart = defineAsyncComponent(() => import('@/components/HeavyChart.vue'));

// 仅在用户交互时加载
const dialog = ref(false);
const LazyDialog = shallowRef<Component | null>(null);

async function openDialog() {
  if (!LazyDialog.value) {
    const { default: comp } = await import('@/components/HeavyDialog.vue');
    LazyDialog.value = comp;
  }
  dialog.value = true;
}
```

### 动态 import 分析

```typescript
// vite.config.ts：自定义分包策略
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks(id) {
          if (id.includes('node_modules')) {
            // 第三方库分离打包
            if (id.includes('element-plus')) return 'element-plus';
            if (id.includes('echarts')) return 'echarts';
            if (id.includes('vue')) return 'vue-vendor';
            return 'vendor';
          }
        },
      },
    },
  },
});
```

---

## 8.4 虚拟列表（大数据量渲染）

```bash
pnpm add vue-virtual-scroller
```

```vue
<script setup lang="ts">
import { RecycleScroller } from 'vue-virtual-scroller';
import 'vue-virtual-scroller/dist/vue-virtual-scroller.css';
</script>

<template>
  <!-- 渲染 10 万条数据，只渲染可视区域内的 DOM -->
  <RecycleScroller
    :items="largeList"
    :item-size="60"
    key-field="id"
    style="height: 600px"
  >
    <template #default="{ item }">
      <div class="list-item">{{ item.name }}</div>
    </template>
  </RecycleScroller>
</template>
```

### 手写简单虚拟列表（理解原理）

```vue
<script setup lang="ts">
const ITEM_HEIGHT = 50;
const VISIBLE_COUNT = 10;

const items = ref(Array.from({ length: 100000 }, (_, i) => ({ id: i, name: `Item ${i}` })));
const scrollTop = ref(0);

const startIndex = computed(() => Math.floor(scrollTop.value / ITEM_HEIGHT));
const endIndex = computed(() => Math.min(startIndex.value + VISIBLE_COUNT + 2, items.value.length));
const visibleItems = computed(() => items.value.slice(startIndex.value, endIndex.value));
const translateY = computed(() => startIndex.value * ITEM_HEIGHT);
const totalHeight = computed(() => items.value.length * ITEM_HEIGHT);

function handleScroll(e: Event) {
  scrollTop.value = (e.target as HTMLElement).scrollTop;
}
</script>

<template>
  <div class="virtual-list-container" style="height: 500px; overflow-y: auto;" @scroll="handleScroll">
    <div :style="{ height: `${totalHeight}px`, position: 'relative' }">
      <div :style="{ transform: `translateY(${translateY}px)` }">
        <div v-for="item in visibleItems" :key="item.id" :style="{ height: `${ITEM_HEIGHT}px` }">
          {{ item.name }}
        </div>
      </div>
    </div>
  </div>
</template>
```

---

## 8.5 图片与资源优化

```vue
<template>
  <!-- 1. 图片懒加载（原生） -->
  <img src="large-image.jpg" loading="lazy" alt="示例" />
  
  <!-- 2. 响应式图片 -->
  <picture>
    <source srcset="image.avif" type="image/avif" />
    <source srcset="image.webp" type="image/webp" />
    <img src="image.jpg" alt="示例" />
  </picture>
  
  <!-- 3. 使用 VueUse 的 useIntersectionObserver 懒加载 -->
</template>

<script setup lang="ts">
import { ref } from 'vue';
import { useIntersectionObserver } from '@vueuse/core';

const imgEl = ref<HTMLImageElement | null>(null);
const isVisible = ref(false);

const { stop } = useIntersectionObserver(imgEl, ([{ isIntersecting }]) => {
  if (isIntersecting) {
    isVisible.value = true;
    stop(); // 加载后停止观察
  }
});
</script>
```

---

## 8.6 包体积分析与优化

```bash
# 安装包分析工具
pnpm add -D rollup-plugin-visualizer

# vite.config.ts
import { visualizer } from 'rollup-plugin-visualizer';
plugins: [
  visualizer({ open: true, filename: 'dist/stats.html' }),
]

# 运行构建后自动打开分析报告
pnpm build
```

### 常见优化手段

```typescript
// 1. Element Plus 按需导入（自动导入插件已处理）
// 避免整包导入：import ElementPlus from 'element-plus'

// 2. Lodash 按需导入
import debounce from 'lodash-es/debounce'; // ✅ 仅导入需要的函数
import { debounce } from 'lodash';         // ❌ 整包导入（60KB+）

// 3. Day.js 替代 Moment.js（体积小 10 倍）
import dayjs from 'dayjs';
import 'dayjs/locale/zh-cn';
dayjs.locale('zh-cn');

// 4. 图标按需加载
// 使用 unplugin-icons 或 iconify 按需打包 SVG 图标
```

---

## 8.7 缓存策略

```typescript
// HTTP 缓存（配合后端）
// 静态资源（hash 文件名）：Cache-Control: max-age=31536000
// HTML 文件：Cache-Control: no-cache（每次验证）

// 内存缓存（前端）
const cache = new Map<string, any>();

async function fetchWithCache<T>(key: string, fetcher: () => Promise<T>, ttl = 60000): Promise<T> {
  const cached = cache.get(key);
  if (cached && Date.now() - cached.time < ttl) {
    return cached.data;
  }
  const data = await fetcher();
  cache.set(key, { data, time: Date.now() });
  return data;
}

// Service Worker 离线缓存（PWA）
// vite-plugin-pwa 可快速接入
```

---

## 8.8 Web Vitals 与性能指标

```typescript
// 核心 Web Vitals
// LCP（最大内容绘制）< 2.5s      → 加快资源加载
// FID（首次输入延迟）< 100ms     → 减少 JS 执行时间
// CLS（累积布局偏移）< 0.1       → 为图片预留尺寸

// 测量性能
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.log(`${entry.name}: ${entry.startTime}ms`);
  }
});
observer.observe({ type: 'largest-contentful-paint', buffered: true });

// 使用 web-vitals 库
import { onCLS, onFID, onLCP } from 'web-vitals';

function sendToAnalytics({ name, value }: { name: string; value: number }) {
  console.log(`${name}: ${value}`);
}

onCLS(sendToAnalytics);
onFID(sendToAnalytics);
onLCP(sendToAnalytics);
```

---

## 8.9 常见性能陷阱

```typescript
// ❌ 陷阱 1：在 computed 中有副作用
const result = computed(() => {
  fetchData(); // 绝对不要在 computed 中发请求
  return processData(rawData.value);
});

// ❌ 陷阱 2：watch 深度监听大对象
watch(hugeObject, handler, { deep: true }); // 非常慢
// ✅ 监听具体字段
watch(() => hugeObject.value.specificField, handler);

// ❌ 陷阱 3：reactive 包裹大型只读数据
const config = reactive(hugeReadonlyData); // 浪费 Proxy 开销
// ✅ 用 shallowReactive 或 readonly
const config = readonly(hugeReadonlyData);

// ❌ 陷阱 4：v-for 不加 key 或用 index 作 key
<div v-for="(item, index) in items" :key="index"> <!-- 会导致错误复用 -->
// ✅ 使用唯一稳定 ID
<div v-for="item in items" :key="item.id">

// ❌ 陷阱 5：同时使用 v-if 和 v-for
<div v-for="item in items" v-if="item.active"> <!-- Vue 3 报错，且性能差 -->
// ✅ 先过滤数据
const activeItems = computed(() => items.value.filter(i => i.active));
<div v-for="item in activeItems" :key="item.id">
```

---

## 8.10 章节小结

| 优化方向 | 手段 | 收益 |
|---------|------|------|
| 首屏加载 | 路由懒加载 + 代码分割 | ⭐⭐⭐⭐⭐ |
| 大数据渲染 | 虚拟列表 | ⭐⭐⭐⭐⭐ |
| 包体积 | Tree-shaking + 按需导入 | ⭐⭐⭐⭐ |
| 重渲染 | v-memo + shallowRef | ⭐⭐⭐⭐ |
| 图片 | 懒加载 + WebP | ⭐⭐⭐⭐ |
| 缓存 | HTTP 缓存 + 内存缓存 | ⭐⭐⭐⭐ |

> **下一步**：阅读 [第 09 章 - 调试与排错](09_debugging.md)
