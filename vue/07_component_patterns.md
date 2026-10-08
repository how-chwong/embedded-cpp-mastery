# 第 07 章：组件设计模式

> 目标：掌握 Vue 3 中常用的组件设计模式，写出高复用、低耦合、易维护的组件库与业务组件。

---

## 7.1 Composable（组合式函数）

Composable 是 Vue 3 最核心的复用模式，相当于 React Hooks，也类似 C# 中的扩展方法 + 状态封装。

### 基础 Composable 模式

```typescript
// composables/usePagination.ts
import { ref, computed } from 'vue';

interface PaginationOptions {
  initialPage?: number;
  initialPageSize?: number;
}

export function usePagination(options: PaginationOptions = {}) {
  const page = ref(options.initialPage ?? 1);
  const pageSize = ref(options.initialPageSize ?? 20);
  const total = ref(0);

  const totalPages = computed(() => Math.ceil(total.value / pageSize.value));
  const hasNextPage = computed(() => page.value < totalPages.value);
  const hasPrevPage = computed(() => page.value > 1);

  function nextPage() {
    if (hasNextPage.value) page.value++;
  }

  function prevPage() {
    if (hasPrevPage.value) page.value--;
  }

  function goToPage(p: number) {
    page.value = Math.max(1, Math.min(p, totalPages.value));
  }

  function reset() {
    page.value = 1;
  }

  return {
    page,
    pageSize,
    total,
    totalPages,
    hasNextPage,
    hasPrevPage,
    nextPage,
    prevPage,
    goToPage,
    reset,
  };
}
```

### 请求封装 Composable

```typescript
// composables/useRequest.ts
import { ref } from 'vue';

interface UseRequestOptions<T> {
  immediate?: boolean;
  onSuccess?: (data: T) => void;
  onError?: (error: Error) => void;
}

export function useRequest<T>(
  requestFn: (...args: any[]) => Promise<T>,
  options: UseRequestOptions<T> = {},
) {
  const data = ref<T | null>(null);
  const loading = ref(false);
  const error = ref<Error | null>(null);

  async function execute(...args: any[]) {
    loading.value = true;
    error.value = null;
    try {
      data.value = await requestFn(...args);
      options.onSuccess?.(data.value as T);
    } catch (e) {
      error.value = e as Error;
      options.onError?.(error.value);
    } finally {
      loading.value = false;
    }
  }

  if (options.immediate) {
    execute();
  }

  return { data, loading, error, execute };
}

// 使用
const { data: users, loading, execute: fetchUsers } = useRequest(
  () => userApi.getAll(),
  { immediate: true }
);
```

### 表单处理 Composable

```typescript
// composables/useForm.ts
import { reactive, ref } from 'vue';

type ValidationRule<T> = {
  required?: boolean;
  minLength?: number;
  maxLength?: number;
  pattern?: RegExp;
  validator?: (value: T) => string | true;
  message?: string;
};

type Rules<T extends Record<string, any>> = {
  [K in keyof T]?: ValidationRule<T[K]>[];
};

export function useForm<T extends Record<string, any>>(
  initialValues: T,
  rules: Rules<T> = {},
) {
  const form = reactive({ ...initialValues }) as T;
  const errors = reactive<Partial<Record<keyof T, string>>>({});
  const isDirty = ref(false);
  const isSubmitting = ref(false);

  function validateField(field: keyof T): boolean {
    const value = form[field];
    const fieldRules = rules[field] || [];

    for (const rule of fieldRules) {
      if (rule.required && !value) {
        errors[field] = rule.message || `${String(field)} 为必填项`;
        return false;
      }
      if (rule.minLength && String(value).length < rule.minLength) {
        errors[field] = rule.message || `最少 ${rule.minLength} 个字符`;
        return false;
      }
      if (rule.pattern && !rule.pattern.test(String(value))) {
        errors[field] = rule.message || '格式不正确';
        return false;
      }
      if (rule.validator) {
        const result = rule.validator(value);
        if (result !== true) {
          errors[field] = result;
          return false;
        }
      }
    }
    delete errors[field];
    return true;
  }

  function validate(): boolean {
    return (Object.keys(rules) as (keyof T)[]).every(validateField);
  }

  async function submit(submitFn: (values: T) => Promise<void>) {
    if (!validate()) return;
    isSubmitting.value = true;
    try {
      await submitFn({ ...form });
    } finally {
      isSubmitting.value = false;
    }
  }

  function reset() {
    Object.assign(form, initialValues);
    Object.keys(errors).forEach(k => delete errors[k as keyof T]);
    isDirty.value = false;
  }

  return { form, errors, isDirty, isSubmitting, validateField, validate, submit, reset };
}
```

---

## 7.2 无渲染组件（Renderless Components）

无渲染组件只提供逻辑，不渲染 DOM，通过作用域插槽将数据暴露给父组件，实现逻辑与UI完全解耦。

```vue
<!-- components/RenderlessDataTable.vue -->
<script setup lang="ts" generic="T extends Record<string, any>">
import { ref, computed } from 'vue';

interface Props {
  data: T[];
  columns: Array<{ key: keyof T; label: string; sortable?: boolean }>;
  pageSize?: number;
}

const props = withDefaults(defineProps<Props>(), { pageSize: 10 });

const page = ref(1);
const sortKey = ref<keyof T | null>(null);
const sortOrder = ref<'asc' | 'desc'>('asc');

const sorted = computed(() => {
  if (!sortKey.value) return [...props.data];
  return [...props.data].sort((a, b) => {
    const va = a[sortKey.value!];
    const vb = b[sortKey.value!];
    const cmp = va < vb ? -1 : va > vb ? 1 : 0;
    return sortOrder.value === 'asc' ? cmp : -cmp;
  });
});

const paginated = computed(() => {
  const start = (page.value - 1) * props.pageSize;
  return sorted.value.slice(start, start + props.pageSize);
});

function toggleSort(key: keyof T) {
  if (sortKey.value === key) {
    sortOrder.value = sortOrder.value === 'asc' ? 'desc' : 'asc';
  } else {
    sortKey.value = key;
    sortOrder.value = 'asc';
  }
}
</script>

<template>
  <!-- 通过作用域插槽暴露数据和方法 -->
  <slot
    :rows="paginated"
    :columns="columns"
    :page="page"
    :total="data.length"
    :pageSize="pageSize"
    :sortKey="sortKey"
    :sortOrder="sortOrder"
    :toggleSort="toggleSort"
    :setPage="(p: number) => (page = p)"
  />
</template>
```

```vue
<!-- 使用无渲染组件，自定义 UI -->
<template>
  <RenderlessDataTable :data="users" :columns="columns">
    <template #default="{ rows, columns, toggleSort, sortKey, sortOrder, page, total, pageSize, setPage }">
      <table>
        <thead>
          <tr>
            <th
              v-for="col in columns"
              :key="col.key"
              @click="col.sortable && toggleSort(col.key)"
            >
              {{ col.label }}
              <span v-if="sortKey === col.key">
                {{ sortOrder === 'asc' ? '▲' : '▼' }}
              </span>
            </th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="row in rows" :key="row.id">
            <td v-for="col in columns" :key="col.key">{{ row[col.key] }}</td>
          </tr>
        </tbody>
      </table>
      <Pagination :page="page" :total="total" :page-size="pageSize" @change="setPage" />
    </template>
  </RenderlessDataTable>
</template>
```

---

## 7.3 HOC 与组件组合

```typescript
// 使用 defineComponent + h 函数创建 HOC（高阶组件）
// 常用于：权限包装、日志记录、加载状态包装

import { defineComponent, h, ref } from 'vue';
import type { Component } from 'vue';

// 权限包装 HOC
function withPermission(WrappedComponent: Component, requiredPermission: string) {
  return defineComponent({
    name: `WithPermission(${(WrappedComponent as any).name})`,
    setup(props, { slots, attrs }) {
      const { permissions } = useAuthStore();
      const hasPermission = permissions.includes(requiredPermission);

      if (!hasPermission) {
        return () => h('div', { class: 'no-permission' }, '无权访问');
      }

      return () => h(WrappedComponent, attrs, slots);
    },
  });
}

const AdminButton = withPermission(BaseButton, 'admin:write');
```

---

## 7.4 动态组件与异步组件

```vue
<script setup lang="ts">
import { defineAsyncComponent, shallowRef } from 'vue';

// 动态组件
const currentTab = shallowRef('UserList');
const tabComponents = {
  UserList: () => import('@/components/UserList.vue'),
  UserForm: () => import('@/components/UserForm.vue'),
  UserStats: () => import('@/components/UserStats.vue'),
};

// 异步组件（带加载状态和错误处理）
const AsyncHeavyChart = defineAsyncComponent({
  loader: () => import('@/components/HeavyChart.vue'),
  loadingComponent: LoadingSpinner,
  errorComponent: ErrorMessage,
  delay: 200,      // 200ms 后显示 loading（避免闪烁）
  timeout: 3000,   // 3s 超时
});
</script>

<template>
  <!-- 动态组件 -->
  <KeepAlive>
    <component :is="tabComponents[currentTab]" />
  </KeepAlive>
  
  <!-- 异步组件配合 Suspense -->
  <Suspense>
    <AsyncHeavyChart />
    <template #fallback><LoadingSpinner /></template>
  </Suspense>
</template>
```

---

## 7.5 全局组件注册策略

```typescript
// plugins/components.ts（全局注册基础组件）
import type { App } from 'vue';

// 自动导入 components/base/ 下所有组件
const modules = import.meta.glob('@/components/base/*.vue', { eager: true });

export function registerComponents(app: App) {
  for (const path in modules) {
    const component = modules[path] as { default: any };
    const name = path.split('/').pop()!.replace('.vue', '');
    app.component(name, component.default);
  }
}

// main.ts
app.use(registerComponents);
```

---

## 7.6 插件系统

```typescript
// plugins/toast.ts
import type { App } from 'vue';
import ToastContainer from '@/components/ToastContainer.vue';

export interface ToastOptions {
  message: string;
  type?: 'success' | 'error' | 'warning' | 'info';
  duration?: number;
}

export interface ToastService {
  show(options: ToastOptions): void;
  success(message: string): void;
  error(message: string): void;
}

const ToastPlugin = {
  install(app: App) {
    // 创建容器
    const container = document.createElement('div');
    document.body.appendChild(container);
    
    const toastApp = createApp(ToastContainer);
    const instance = toastApp.mount(container) as any;
    
    // 提供全局服务
    const toast: ToastService = {
      show: (opts) => instance.show(opts),
      success: (msg) => instance.show({ message: msg, type: 'success' }),
      error: (msg) => instance.show({ message: msg, type: 'error' }),
    };
    
    app.provide('toast', toast);
    app.config.globalProperties.$toast = toast;
  },
};

export default ToastPlugin;

// 使用
const toast = inject<ToastService>('toast')!;
toast.success('保存成功');
```

---

## 7.7 表单组件设计规范

```vue
<!-- components/base/BaseInput.vue -->
<!-- 完整的受控表单组件，支持 v-model、验证、无障碍访问 -->
<script setup lang="ts">
defineOptions({ name: 'BaseInput' });

interface Props {
  modelValue: string | number;
  label?: string;
  placeholder?: string;
  type?: 'text' | 'email' | 'password' | 'number' | 'tel';
  error?: string;
  hint?: string;
  disabled?: boolean;
  required?: boolean;
  id?: string;
}

const props = withDefaults(defineProps<Props>(), {
  type: 'text',
});

const emit = defineEmits<{
  'update:modelValue': [value: string];
  blur: [event: FocusEvent];
}>();

const inputId = computed(() => props.id || `input-${Math.random().toString(36).slice(2)}`);
</script>

<template>
  <div class="form-field" :class="{ 'has-error': error, 'is-disabled': disabled }">
    <label v-if="label" :for="inputId" class="form-label">
      {{ label }}
      <span v-if="required" class="required-mark" aria-hidden="true">*</span>
    </label>
    
    <input
      :id="inputId"
      :type="type"
      :value="modelValue"
      :placeholder="placeholder"
      :disabled="disabled"
      :required="required"
      :aria-describedby="error ? `${inputId}-error` : undefined"
      :aria-invalid="!!error"
      class="form-input"
      @input="emit('update:modelValue', ($event.target as HTMLInputElement).value)"
      @blur="emit('blur', $event)"
    />
    
    <p v-if="error" :id="`${inputId}-error`" class="form-error" role="alert">
      {{ error }}
    </p>
    <p v-else-if="hint" class="form-hint">{{ hint }}</p>
  </div>
</template>
```

---

## 7.8 章节小结

| 模式 | 适用场景 | 复杂度 |
|------|---------|--------|
| Composable | 逻辑复用（请求、分页、表单） | ⭐⭐ |
| 无渲染组件 | 逻辑与 UI 完全分离 | ⭐⭐⭐ |
| 异步组件 | 大型组件按需加载 | ⭐⭐ |
| HOC | 横切关注点（权限、日志） | ⭐⭐⭐ |
| 插件 | 全局服务（Toast、Modal） | ⭐⭐⭐ |
| 动态组件 | Tab 切换、条件渲染 | ⭐⭐ |

> **下一步**：阅读 [第 08 章 - 性能优化](08_performance.md)
