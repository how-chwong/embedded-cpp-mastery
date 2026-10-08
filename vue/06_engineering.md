# 第 06 章：工程化与代码规范

> 目标：建立一套标准化的项目工程体系，包括统一的目录结构、代码规范、单元测试和 CI/CD 流程，确保团队协作顺畅、代码质量可控。

---

## 6.1 推荐项目结构

```
my-vue-app/
├── .github/
│   └── workflows/
│       └── ci.yml            # GitHub Actions CI
├── .husky/                   # Git hooks
├── .vscode/                  # 编辑器配置
│   ├── settings.json
│   └── extensions.json
├── public/
│   └── favicon.ico
├── src/
│   ├── main.ts               # 应用入口
│   ├── App.vue               # 根组件
│   ├── assets/               # 静态资源
│   │   ├── images/
│   │   └── styles/
│   │       ├── variables.css # CSS 变量
│   │       ├── reset.css     # 重置样式
│   │       └── main.css      # 全局样式
│   ├── components/           # 通用组件
│   │   ├── base/             # 基础组件（Button、Input、Modal）
│   │   └── business/         # 业务组件（跨页面复用）
│   ├── composables/          # 可复用 Composable 函数
│   │   ├── useAuth.ts
│   │   ├── usePagination.ts
│   │   └── useRequest.ts
│   ├── layouts/              # 布局组件
│   │   ├── DefaultLayout.vue
│   │   └── AdminLayout.vue
│   ├── router/               # 路由
│   │   ├── index.ts
│   │   └── guards.ts
│   ├── services/             # API 服务层
│   │   ├── http.ts           # Axios 实例
│   │   └── api/
│   │       ├── user.ts
│   │       └── auth.ts
│   ├── stores/               # Pinia Store
│   │   ├── auth.ts
│   │   └── user.ts
│   ├── types/                # 全局类型定义
│   │   ├── index.ts          # 业务类型
│   │   ├── api.ts            # API 请求/响应类型
│   │   └── env.d.ts          # 环境变量类型
│   ├── utils/                # 工具函数
│   │   ├── format.ts         # 格式化
│   │   ├── validate.ts       # 验证
│   │   └── storage.ts        # 存储封装
│   └── views/                # 页面组件
│       ├── HomeView.vue
│       ├── auth/
│       │   └── LoginView.vue
│       └── user/
│           ├── ListView.vue
│           └── DetailView.vue
├── tests/                    # 测试文件
│   ├── unit/
│   └── e2e/
├── .env
├── .env.development
├── .env.production
├── .eslintrc.js
├── .prettierrc
├── tsconfig.json
├── vite.config.ts
└── package.json
```

---

## 6.2 命名规范

### 文件命名

```
# 组件文件：PascalCase
UserCard.vue
BaseButton.vue
AdminLayout.vue

# 非组件文件：camelCase
useAuth.ts
userStore.ts
formatDate.ts

# 视图文件（与路由对应）：PascalCase + View 后缀
HomeView.vue
UserListView.vue
UserDetailView.vue

# 类型文件：camelCase
user.types.ts    # 或直接放 types/index.ts
```

### 组件命名规则

```typescript
// ✅ 多单词组件名（避免与 HTML 标签冲突）
defineOptions({ name: 'UserCard' }); // 不能是 'User'

// ✅ 基础组件加 Base 前缀
// BaseButton.vue、BaseInput.vue、BaseModal.vue

// ✅ 单例组件加 The 前缀（每页只出现一次）
// TheHeader.vue、TheFooter.vue、TheSidebar.vue

// ✅ 业务组件紧密耦合用父组件名前缀
// UserList.vue / UserListItem.vue / UserListItemButton.vue
```

### 变量命名规范

```typescript
// ✅ 响应式变量：直观名词
const userName = ref('');
const userList = ref<User[]>([]);
const isLoading = ref(false);    // Boolean: is/has/can 前缀
const hasError = ref(false);

// ✅ Composable 函数：use 前缀
function useAuth() { ... }
function usePagination() { ... }

// ✅ 事件处理函数：handle 前缀
function handleSubmit() { ... }
function handleUserClick(user: User) { ... }

// ✅ API 函数：动词前缀
function fetchUsers() { ... }
function createUser() { ... }
function updateUser() { ... }
function deleteUser() { ... }
```

---

## 6.3 组件代码规范

### 标准组件模板

```vue
<script setup lang="ts">
/**
 * UserCard 组件
 * 展示用户卡片信息，支持编辑和删除操作
 */

// 1. 导入声明（按顺序：Vue → 第三方 → 本地）
import { ref, computed, onMounted } from 'vue';
import { ElMessage } from 'element-plus';
import { userApi } from '@/services/api/user';
import type { User } from '@/types';

// 2. 组件选项（名称、继承等）
defineOptions({
  name: 'UserCard',
  inheritAttrs: false,
});

// 3. Props
interface Props {
  user: User;
  editable?: boolean;
}
const props = withDefaults(defineProps<Props>(), {
  editable: false,
});

// 4. Emits
const emit = defineEmits<{
  edit: [user: User];
  delete: [id: number];
}>();

// 5. 响应式状态（按关联性分组）
const isExpanded = ref(false);
const isDeleting = ref(false);

// 6. 计算属性
const cardClass = computed(() => ({
  'user-card': true,
  'user-card--expanded': isExpanded.value,
  'user-card--inactive': !props.user.active,
}));

// 7. 方法
async function handleDelete() {
  if (!confirm(`确认删除 ${props.user.name}？`)) return;
  
  isDeleting.value = true;
  try {
    await userApi.delete(props.user.id);
    emit('delete', props.user.id);
    ElMessage.success('删除成功');
  } catch {
    ElMessage.error('删除失败');
  } finally {
    isDeleting.value = false;
  }
}

// 8. 生命周期
onMounted(() => {
  // 初始化逻辑
});
</script>

<template>
  <div v-bind="$attrs" :class="cardClass">
    <!-- ... -->
  </div>
</template>

<style scoped>
/* 仅使用类选择器，避免样式污染 */
.user-card {
  /* ... */
}
</style>
```

---

## 6.4 单元测试（Vitest）

### 配置

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';
import vue from '@vitejs/plugin-vue';
import { fileURLToPath, URL } from 'node:url';

export default defineConfig({
  plugins: [vue()],
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: ['./tests/setup.ts'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'lcov'],
      exclude: ['node_modules/', 'tests/'],
    },
  },
  resolve: {
    alias: { '@': fileURLToPath(new URL('./src', import.meta.url)) },
  },
});
```

```typescript
// tests/setup.ts
import '@testing-library/jest-dom';
```

### 测试 Composable

```typescript
// tests/unit/composables/usePagination.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import { usePagination } from '@/composables/usePagination';

describe('usePagination', () => {
  it('初始状态正确', () => {
    const { page, pageSize, total } = usePagination();
    expect(page.value).toBe(1);
    expect(pageSize.value).toBe(20);
    expect(total.value).toBe(0);
  });

  it('翻页正确更新', () => {
    const { page, nextPage, prevPage } = usePagination();
    nextPage();
    expect(page.value).toBe(2);
    prevPage();
    expect(page.value).toBe(1);
    prevPage(); // 不能低于 1
    expect(page.value).toBe(1);
  });
});
```

### 测试组件

```typescript
// tests/unit/components/UserCard.test.ts
import { describe, it, expect, vi } from 'vitest';
import { mount } from '@vue/test-utils';
import UserCard from '@/components/UserCard.vue';

const mockUser = {
  id: 1,
  name: 'Alice',
  email: 'alice@example.com',
  active: true,
};

describe('UserCard', () => {
  it('正确渲染用户信息', () => {
    const wrapper = mount(UserCard, {
      props: { user: mockUser },
    });
    expect(wrapper.text()).toContain('Alice');
    expect(wrapper.text()).toContain('alice@example.com');
  });

  it('点击编辑按钮触发 edit 事件', async () => {
    const wrapper = mount(UserCard, {
      props: { user: mockUser, editable: true },
    });
    await wrapper.find('[data-testid="edit-btn"]').trigger('click');
    expect(wrapper.emitted('edit')?.[0]).toEqual([mockUser]);
  });

  it('删除前需要确认', async () => {
    const confirmSpy = vi.spyOn(window, 'confirm').mockReturnValue(false);
    const wrapper = mount(UserCard, {
      props: { user: mockUser, editable: true },
    });
    await wrapper.find('[data-testid="delete-btn"]').trigger('click');
    expect(confirmSpy).toHaveBeenCalled();
    // 取消确认，不触发 delete 事件
    expect(wrapper.emitted('delete')).toBeUndefined();
  });
});
```

### 测试 Store

```typescript
// tests/unit/stores/user.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest';
import { setActivePinia, createPinia } from 'pinia';
import { useUserStore } from '@/stores/user';
import { userApi } from '@/services/api/user';

vi.mock('@/services/api/user');

describe('useUserStore', () => {
  beforeEach(() => {
    setActivePinia(createPinia());
  });

  it('fetchUsers 成功时更新 users', async () => {
    const mockUsers = [{ id: 1, name: 'Alice', active: true }];
    vi.mocked(userApi.getAll).mockResolvedValue({ items: mockUsers, total: 1 });

    const store = useUserStore();
    await store.fetchUsers();

    expect(store.users).toEqual(mockUsers);
    expect(store.loading).toBe(false);
  });
});
```

---

## 6.5 CI/CD（GitHub Actions）

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: pnpm/action-setup@v3
        with:
          version: 8
      
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'
      
      - name: 安装依赖
        run: pnpm install --frozen-lockfile
      
      - name: 类型检查
        run: pnpm vue-tsc --noEmit
      
      - name: Lint
        run: pnpm lint
      
      - name: 单元测试
        run: pnpm test --coverage
      
      - name: 构建
        run: pnpm build

  deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      # ... 部署步骤
```

---

## 6.6 代码审查清单（Code Review Checklist）

### 提交前自查

```markdown
## 功能
- [ ] 功能是否按需求正确实现
- [ ] 边界情况是否处理（空数据、网络错误、超时）
- [ ] 是否有合适的加载状态和错误提示

## 代码质量
- [ ] 无 TypeScript 错误或 any 滥用
- [ ] 变量/函数命名清晰
- [ ] 无重复代码（DRY 原则）
- [ ] 复杂逻辑有注释

## 性能
- [ ] 列表渲染有 key
- [ ] 大列表是否需要虚拟滚动
- [ ] 图片是否懒加载
- [ ] 是否有不必要的重渲染

## 安全
- [ ] 用户输入是否做了验证
- [ ] 无 v-html 使用不受信任的内容
- [ ] 敏感信息不暴露在前端

## 测试
- [ ] 关键逻辑是否有单元测试
- [ ] 测试覆盖正常路径和异常路径
```

---

## 6.7 章节小结

| 实践 | 重要程度 | 说明 |
|------|---------|------|
| 统一目录结构 | ⭐⭐⭐⭐⭐ | 团队协作基础 |
| 命名规范 | ⭐⭐⭐⭐⭐ | 可读性关键 |
| ESLint + Prettier | ⭐⭐⭐⭐⭐ | 自动化质量保障 |
| 单元测试 | ⭐⭐⭐⭐ | 重构安全保障 |
| CI/CD | ⭐⭐⭐⭐ | 自动化部署 |
| Code Review | ⭐⭐⭐⭐ | 团队代码质量 |

> **下一步**：阅读 [第 07 章 - 组件设计模式](07_component_patterns.md)
