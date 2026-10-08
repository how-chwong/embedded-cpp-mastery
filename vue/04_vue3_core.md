# 第 04 章：Vue 3 核心

> 目标：全面掌握 Vue 3 的 Composition API、响应式系统、模板语法、指令、生命周期等核心概念，能够独立构建中大型单页应用。

---

## 4.1 Vue 3 vs Vue 2 核心区别

| 特性 | Vue 2 | Vue 3 |
|------|-------|-------|
| API 风格 | Options API | Composition API（推荐）|
| 响应式 | Object.defineProperty | Proxy（更强大）|
| TypeScript | 勉强支持 | 原生支持 |
| 性能 | 基准 | 提升约 55% |
| 根节点 | 必须单根 | 支持多根（Fragment）|
| 全局配置 | Vue.xxx | app.xxx |

---

## 4.2 项目结构

```
src/
├── main.ts              # 应用入口
├── App.vue              # 根组件
├── assets/              # 静态资源
├── components/          # 通用组件
│   └── base/            # 基础组件（Button、Input等）
├── views/               # 页面组件（与路由对应）
│   ├── HomeView.vue
│   └── UserView.vue
├── composables/         # 可复用逻辑（Composable函数）
│   └── useAuth.ts
├── stores/              # Pinia 状态管理
│   └── user.ts
├── router/              # 路由配置
│   └── index.ts
├── services/            # API 调用层
│   └── api.ts
├── types/               # TypeScript 类型定义
│   └── index.ts
└── utils/               # 工具函数
    └── format.ts
```

---

## 4.3 应用入口

```typescript
// main.ts
import { createApp } from 'vue';
import { createPinia } from 'pinia';
import App from './App.vue';
import router from './router';
import './assets/main.css';

const app = createApp(App);

app.use(createPinia());
app.use(router);

// 全局错误处理
app.config.errorHandler = (error, instance, info) => {
  console.error('Vue 错误:', error, info);
  // 可发送到错误监控服务（如 Sentry）
};

app.mount('#app');
```

---

## 4.4 单文件组件（SFC）结构

```vue
<!-- UserCard.vue -->
<script setup lang="ts">
// 1. 导入
import { ref, computed, onMounted } from 'vue';
import type { User } from '@/types';

// 2. Props & Emits
interface Props {
  user: User;
  showEmail?: boolean;
}

const props = withDefaults(defineProps<Props>(), {
  showEmail: true,
});

const emit = defineEmits<{
  edit: [user: User];
  delete: [id: number];
}>();

// 3. 响应式状态
const isExpanded = ref(false);

// 4. 计算属性
const displayName = computed(() =>
  `${props.user.name} (ID: ${props.user.id})`
);

// 5. 方法
function handleEdit() {
  emit('edit', props.user);
}

// 6. 生命周期
onMounted(() => {
  console.log('UserCard 已挂载');
});
</script>

<template>
  <div class="user-card" :class="{ expanded: isExpanded }">
    <h3>{{ displayName }}</h3>
    <p v-if="showEmail">{{ user.email }}</p>
    
    <button @click="isExpanded = !isExpanded">
      {{ isExpanded ? '收起' : '展开' }}
    </button>
    
    <template v-if="isExpanded">
      <button @click="handleEdit">编辑</button>
      <button @click="emit('delete', user.id)">删除</button>
    </template>
  </div>
</template>

<style scoped>
.user-card {
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  padding: 16px;
}

.expanded {
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}
</style>
```

---

## 4.5 响应式系统

### ref vs reactive

```typescript
import { ref, reactive, toRef, toRefs } from 'vue';

// ref：用于基本类型 / 单个值
const count = ref(0);
count.value++;          // JS 中需要 .value
// 模板中自动解包：{{ count }}（不需要 .value）

const user = ref<User | null>(null); // 对象也可以用 ref

// reactive：用于对象（内部用 Proxy 实现）
const state = reactive({
  count: 0,
  users: [] as User[],
  loading: false,
});
state.count++;           // 直接访问，不需要 .value

// ❌ 常见错误：解构 reactive 会失去响应性
const { count: c } = state; // c 不是响应式的

// ✅ 使用 toRefs 保持响应性
const { count: c2, loading } = toRefs(state);
c2.value++; // 正确

// toRef：将 reactive 对象的某个属性转为 ref
const countRef = toRef(state, 'count');
```

### computed

```typescript
import { ref, computed } from 'vue';

const firstName = ref('John');
const lastName = ref('Doe');

// 只读计算属性
const fullName = computed(() => `${firstName.value} ${lastName.value}`);

// 可写计算属性
const fullNameWritable = computed({
  get: () => `${firstName.value} ${lastName.value}`,
  set: (value: string) => {
    const parts = value.split(' ');
    firstName.value = parts[0];
    lastName.value = parts[1] || '';
  },
});

fullNameWritable.value = 'Jane Smith';
// 自动更新 firstName 和 lastName
```

### watch 与 watchEffect

```typescript
import { ref, watch, watchEffect } from 'vue';

const query = ref('');
const results = ref<string[]>([]);

// watch：明确指定监听源
watch(query, async (newVal, oldVal) => {
  if (newVal.trim()) {
    results.value = await search(newVal);
  }
}, {
  immediate: false,  // 立即执行一次
  deep: false,       // 深度监听
  flush: 'post',     // 在 DOM 更新后执行
});

// 监听多个源
watch([query, page], ([newQuery, newPage]) => {
  loadData(newQuery, newPage);
});

// watchEffect：自动追踪依赖（更简洁）
watchEffect(async () => {
  // 自动追踪 query.value 的变化
  if (query.value) {
    results.value = await search(query.value);
  }
});

// 停止监听
const stop = watchEffect(() => { ... });
stop(); // 手动停止
```

---

## 4.6 模板语法

### 插值与绑定

```vue
<template>
  <!-- 文本插值 -->
  <p>{{ message }}</p>
  
  <!-- v-html（谨慎使用，XSS 风险）-->
  <div v-html="rawHtml"></div>
  
  <!-- v-bind（动态属性）-->
  <img :src="imageUrl" :alt="imageAlt" />
  <button :disabled="isLoading">提交</button>
  
  <!-- 对象语法批量绑定 -->
  <input v-bind="inputAttrs" />
  <!-- 等价于 -->
  <input :type="inputAttrs.type" :placeholder="inputAttrs.placeholder" />
  
  <!-- 动态属性名 -->
  <div :[dynamicProp]="value"></div>
</template>
```

### 条件渲染

```vue
<template>
  <!-- v-if / v-else-if / v-else（条件为假时 DOM 不存在）-->
  <div v-if="status === 'loading'">加载中...</div>
  <div v-else-if="status === 'error'">{{ errorMessage }}</div>
  <div v-else>{{ data }}</div>
  
  <!-- v-show（条件为假时 display:none，DOM 始终存在）-->
  <!-- 适合频繁切换的场景 -->
  <div v-show="isVisible">频繁切换的内容</div>
  
  <!-- template 包裹多个元素 -->
  <template v-if="isLoggedIn">
    <Avatar />
    <UserMenu />
  </template>
</template>
```

### 列表渲染

```vue
<template>
  <!-- 基础列表 -->
  <ul>
    <li v-for="item in items" :key="item.id">
      {{ item.name }}
    </li>
  </ul>
  
  <!-- 带索引 -->
  <div v-for="(item, index) in items" :key="item.id">
    {{ index + 1 }}. {{ item.name }}
  </div>
  
  <!-- 对象遍历 -->
  <div v-for="(value, key, index) in userObject" :key="key">
    {{ key }}: {{ value }}
  </div>
  
  <!-- 数字范围 -->
  <span v-for="n in 5" :key="n">{{ n }}</span>
  
  <!-- ⚠️ key 必须是唯一且稳定的值，不能用 index（会导致更新问题）-->
</template>
```

### 事件处理

```vue
<template>
  <!-- 内联处理 -->
  <button @click="count++">+1</button>
  
  <!-- 方法引用 -->
  <button @click="handleClick">点击</button>
  
  <!-- 带参数 -->
  <button @click="handleClick($event, 'extra')">点击</button>
  
  <!-- 修饰符 -->
  <form @submit.prevent="handleSubmit">...</form>
  <div @click.stop="handleClick">...</div>     <!-- 阻止冒泡 -->
  <input @keyup.enter="handleSearch" />        <!-- 键盘 -->
  <button @click.once="handleOnce">一次性</button>
</template>
```

### v-model（双向绑定）

```vue
<template>
  <!-- 基础 -->
  <input v-model="text" />
  <!-- 等价于 -->
  <input :value="text" @input="text = $event.target.value" />
  
  <!-- 修饰符 -->
  <input v-model.trim="text" />      <!-- 去首尾空格 -->
  <input v-model.number="age" />     <!-- 转数字 -->
  <input v-model.lazy="text" />      <!-- change 而非 input 触发 -->
  
  <!-- 组件上的 v-model -->
  <MyInput v-model="searchText" />
  <!-- 等价于 -->
  <MyInput :modelValue="searchText" @update:modelValue="searchText = $event" />
  
  <!-- 多个 v-model -->
  <UserForm v-model:name="name" v-model:email="email" />
</template>

<script setup lang="ts">
// 组件内部实现 v-model
const props = defineProps<{
  modelValue: string;
}>();
const emit = defineEmits<{
  'update:modelValue': [value: string];
}>();

// 或者使用 defineModel（Vue 3.4+）
const modelValue = defineModel<string>();
</script>
```

---

## 4.7 生命周期

```typescript
import {
  onBeforeMount,
  onMounted,
  onBeforeUpdate,
  onUpdated,
  onBeforeUnmount,
  onUnmounted,
  onErrorCaptured,
} from 'vue';

// 创建阶段（setup 本身就是 beforeCreate + created 的替代）
console.log('初始化');

// 挂载阶段
onBeforeMount(() => {
  // DOM 尚未创建
});

onMounted(() => {
  // DOM 已挂载，可以操作 DOM、发起请求
  fetchData();
  initChart(); // 初始化图表库
});

// 更新阶段
onBeforeUpdate(() => {
  // 数据变化，DOM 更新前
});

onUpdated(() => {
  // DOM 已更新
  // ⚠️ 避免在此修改响应式数据（会导致循环）
});

// 卸载阶段
onBeforeUnmount(() => {
  // 组件卸载前，清理资源
});

onUnmounted(() => {
  clearInterval(timer);     // 清除定时器
  observer.disconnect();    // 断开 Observer
  eventBus.off('event');    // 移除事件监听
});

// 错误捕获
onErrorCaptured((error, instance, info) => {
  console.error(error);
  return false; // 阻止错误继续向上传递
});
```

---

## 4.8 组件通信

### Props & Emits（父子通信）

```vue
<!-- 父组件 -->
<template>
  <UserCard
    :user="currentUser"
    @edit="handleEdit"
    @delete="handleDelete"
  />
</template>

<!-- 子组件 -->
<script setup lang="ts">
const props = defineProps<{ user: User }>();
const emit = defineEmits<{ edit: [user: User]; delete: [id: number] }>();
</script>
```

### provide / inject（跨层级通信）

```typescript
// 祖先组件
import { provide, ref } from 'vue';

const theme = ref('light');

// 提供响应式值
provide('theme', theme);
provide('toggleTheme', () => {
  theme.value = theme.value === 'light' ? 'dark' : 'light';
});

// 类型安全的 provide/inject（使用 InjectionKey）
import type { InjectionKey } from 'vue';

const ThemeKey: InjectionKey<Ref<string>> = Symbol('theme');
provide(ThemeKey, theme);

// 后代组件
import { inject } from 'vue';

const theme = inject(ThemeKey);                    // 有类型推断
const theme2 = inject('theme', ref('light'));      // 带默认值
const toggle = inject<() => void>('toggleTheme');
```

### 模板引用（ref）

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue';
import type { ComponentPublicInstance } from 'vue';

// DOM 引用
const inputEl = ref<HTMLInputElement | null>(null);

// 组件引用（需要子组件 defineExpose 暴露方法）
const childRef = ref<InstanceType<typeof ChildComponent> | null>(null);

onMounted(() => {
  inputEl.value?.focus();
  childRef.value?.someMethod();
});
</script>

<template>
  <input ref="inputEl" type="text" />
  <ChildComponent ref="childRef" />
</template>
```

---

## 4.9 插槽（Slots）

```vue
<!-- 父组件使用 -->
<template>
  <Card>
    <template #header>
      <h2>卡片标题</h2>
    </template>
    
    <p>默认插槽内容</p>
    
    <template #footer="{ closeCard }">
      <button @click="closeCard">关闭</button>
    </template>
  </Card>
</template>

<!-- Card 组件实现 -->
<template>
  <div class="card">
    <div class="card-header">
      <slot name="header" />
    </div>
    <div class="card-body">
      <slot />  <!-- 默认插槽 -->
    </div>
    <div class="card-footer">
      <!-- 作用域插槽：向父组件传递数据 -->
      <slot name="footer" :closeCard="handleClose" />
    </div>
  </div>
</template>
```

---

## 4.10 内置组件

```vue
<template>
  <!-- Transition：单元素过渡动画 -->
  <Transition name="fade" mode="out-in">
    <component :is="currentView" :key="currentView" />
  </Transition>
  
  <!-- TransitionGroup：列表动画 -->
  <TransitionGroup name="list" tag="ul">
    <li v-for="item in items" :key="item.id">{{ item.name }}</li>
  </TransitionGroup>
  
  <!-- KeepAlive：缓存组件状态 -->
  <KeepAlive :include="['HomeView']" :max="10">
    <component :is="currentTab" />
  </KeepAlive>
  
  <!-- Teleport：将内容渲染到指定 DOM 位置 -->
  <Teleport to="body">
    <Modal v-if="showModal" @close="showModal = false" />
  </Teleport>
  
  <!-- Suspense：异步组件加载 -->
  <Suspense>
    <template #default>
      <AsyncComponent />
    </template>
    <template #fallback>
      <LoadingSpinner />
    </template>
  </Suspense>
</template>
```

---

## 4.11 自定义指令

```typescript
// 点击外部关闭下拉框
import type { Directive } from 'vue';

export const vClickOutside: Directive = {
  mounted(el, binding) {
    el._clickOutsideHandler = (event: MouseEvent) => {
      if (!el.contains(event.target as Node)) {
        binding.value(event);
      }
    };
    document.addEventListener('click', el._clickOutsideHandler);
  },
  unmounted(el) {
    document.removeEventListener('click', el._clickOutsideHandler);
    delete el._clickOutsideHandler;
  },
};

// 注册
app.directive('click-outside', vClickOutside);

// 使用
// <div v-click-outside="closeDropdown">...</div>
```

---

## 4.12 章节小结

| 知识点 | 重要程度 | 说明 |
|--------|---------|------|
| `<script setup>` + defineProps/Emit | ⭐⭐⭐⭐⭐ | 现代 Vue 3 写法标准 |
| ref + reactive + computed | ⭐⭐⭐⭐⭐ | 响应式核心 |
| watch + watchEffect | ⭐⭐⭐⭐⭐ | 副作用处理 |
| 模板语法（v-if/v-for/v-model） | ⭐⭐⭐⭐⭐ | 每天都用 |
| 生命周期 | ⭐⭐⭐⭐⭐ | 资源管理必备 |
| provide/inject | ⭐⭐⭐⭐ | 跨层级状态 |
| 插槽 | ⭐⭐⭐⭐ | 组件复用关键 |
| 自定义指令 | ⭐⭐⭐ | DOM 操作封装 |

> **下一步**：阅读 [第 05 章 - Vue 生态系统](05_vue_ecosystem.md)
