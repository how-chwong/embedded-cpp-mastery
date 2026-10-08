# 第 09 章：调试与排错经验

> 目标：掌握前端调试工具和方法，积累常见错误的排查经验，提高解决复杂问题的能力。

---

## 9.1 浏览器 DevTools

### Elements 面板

```
Elements 面板用途：
1. 查看/修改 DOM 结构
2. 查看/修改 CSS 样式（实时生效）
3. 查看计算后的样式（Computed）
4. 检查元素盒模型（Box Model）
5. 事件监听器（Event Listeners）

实用技巧：
- 右键元素 → "Break on" → DOM 变化时自动断点
- $0 在 Console 中代表当前选中的元素
- 强制元素状态：:hover / :focus / :active
```

### Console 面板

```javascript
// 基础输出
console.log('信息');
console.warn('警告');
console.error('错误');

// 分组
console.group('用户数据');
console.log('姓名:', user.name);
console.log('年龄:', user.age);
console.groupEnd();

// 表格（展示数组对象）
console.table(users);

// 计时
console.time('fetchUsers');
await fetchUsers();
console.timeEnd('fetchUsers'); // fetchUsers: 245ms

// 断言（条件失败时报错）
console.assert(users.length > 0, '用户列表不能为空');

// 打印带标签（颜色区分）
console.log('%c[AUTH]%c 用户已登录', 'color: green; font-weight: bold', 'color: inherit');
```

### Network 面板

```
排查 HTTP 问题：
1. 查看请求状态码（200/401/403/404/500）
2. 查看请求/响应 Headers（Token 是否传递）
3. 查看请求体（Request Payload）
4. 查看响应体（Response）
5. 查看请求耗时（Timing）
6. 过滤 XHR/Fetch 请求

常见问题：
- 请求未发出 → 检查代码逻辑
- 401 → Token 未传或过期
- 403 → 权限不足
- 404 → URL 错误
- 500 → 后端错误（看响应体）
- CORS 报错 → 查看 Console 错误，检查后端 CORS 配置
```

### Performance 面板

```
录制运行时性能：
1. 点击 Record，操作页面，点击 Stop
2. 查看 Flame Chart（火焰图）
3. 找出长任务（Long Tasks > 50ms）
4. 查看 Layout/Paint 频率
5. 识别 JavaScript 执行瓶颈

常见性能问题识别：
- 大量红色标记 → 帧率下降
- 长黄色条 → JS 主线程阻塞
- 频繁 Layout → 样式频繁变动（触发回流）
```

---

## 9.2 Vue DevTools

```bash
# 安装 Chrome 扩展：Vue.js devtools
# 或使用独立版：pnpm add -D @vue/devtools
```

```
Vue DevTools 功能：
1. Components 面板：
   - 查看组件树
   - 实时查看/修改 Props、Data、Computed
   - 追踪 Inject/Provide

2. Pinia 面板：
   - 查看所有 Store 状态
   - 时间旅行调试（回放状态变化）
   - 手动触发 Action

3. Router 面板：
   - 查看路由历史
   - 查看当前路由参数

4. Timeline 面板：
   - 追踪事件时序
   - 查看性能标记
```

---

## 9.3 常见错误排查手册

### 响应式失效

```typescript
// ❌ 问题：修改数据但视图不更新
const state = reactive({ user: null });
state.user = { name: 'Alice' }; // ✅ 这样是响应式的

// ❌ 直接替换 reactive 对象（失去响应性）
let config = reactive({ theme: 'light' });
config = reactive({ theme: 'dark' }); // ❌ config 变量不再是响应式

// ✅ 修改属性
config.theme = 'dark';
// 或使用 Object.assign
Object.assign(config, { theme: 'dark' });

// ❌ 解构 reactive 失去响应性
const { theme } = config; // theme 是普通值
// ✅ 使用 toRefs
const { theme } = toRefs(config); // theme 是 Ref

// ❌ ref 忘记 .value
const count = ref(0);
count++; // ❌ 这修改的是 count 这个引用，不是值
count.value++; // ✅
```

### 组件不更新

```typescript
// 原因 1：数组/对象更新方式错误
const list = ref([1, 2, 3]);
list.value[0] = 99;         // ✅ Vue 3 Proxy 可以检测到
list.value.length = 0;      // ✅ 也能检测到

// 原因 2：使用了非响应式数据
// ❌ 从 reactive 解构出来的值
const { items } = store; // items 不是响应式
// ✅ 使用 storeToRefs
const { items } = storeToRefs(store);

// 原因 3：Props 传入了相同引用的对象但内容变了
// 确保传入新引用
this.users = [...this.users]; // 替换数组
this.user = { ...this.user }; // 替换对象
```

### 生命周期问题

```typescript
// ❌ 在 setup 顶层 await 导致后续代码不执行
const data = await fetchData(); // ❌ 会使后续响应式代码失效
// ✅ 在函数中 await
async function load() {
  const data = await fetchData();
}
onMounted(load);

// ❌ 在组件卸载后修改响应式数据（内存泄漏）
onMounted(() => {
  const timer = setInterval(() => {
    count.value++; // 组件卸载后仍在修改
  }, 1000);
  // ✅ 必须清理
  onUnmounted(() => clearInterval(timer));
});
```

### 路由问题

```typescript
// ❌ 同路由参数变化时组件不重新加载
// /user/1 → /user/2，组件不刷新
// ✅ 监听路由参数变化
watch(() => route.params.id, (newId) => {
  loadUser(newId);
}, { immediate: true });

// 或者给路由视图加 key
<RouterView :key="route.fullPath" />

// ❌ 在路由守卫中忘记调用 next()
router.beforeEach((to, from, next) => {
  if (condition) {
    doSomething();
    // 忘记 next()，导致路由卡住
  }
  next(); // ✅ 必须调用
});
```

### TypeScript 常见错误

```typescript
// 错误：Object is possibly 'null'
const name = user.name; // ❌ user 可能是 null
// ✅ 可选链
const name = user?.name;
// ✅ 非空断言（确认不为 null）
const name = user!.name;
// ✅ 提前判断
if (user) { const name = user.name; }

// 错误：Property 'x' does not exist on type 'Y'
// 通常意味着类型定义不正确或 API 返回数据与类型不匹配
// 检查：1. 接口定义 2. API 实际返回 3. 是否需要类型断言

// 错误：Type 'string | undefined' is not assignable to type 'string'
function greet(name?: string) {
  const greeting = `Hello, ${name}`; // ❌ name 可能是 undefined
  const greeting2 = `Hello, ${name ?? 'Guest'}`; // ✅
}
```

---

## 9.4 网络请求排查

```typescript
// 统一错误日志（在 Axios 拦截器中）
http.interceptors.response.use(
  null,
  (error) => {
    // 详细日志
    console.error('请求失败', {
      url: error.config?.url,
      method: error.config?.method,
      status: error.response?.status,
      data: error.response?.data,
      message: error.message,
    });
    return Promise.reject(error);
  },
);

// CORS 错误处理
// 错误信息：Access to XMLHttpRequest at 'xxx' from origin 'yyy' has been blocked by CORS policy
// 原因：后端未设置 CORS 头
// 解决：
// 1. 开发时：在 vite.config.ts 配置 proxy
// 2. 生产时：后端添加 Access-Control-Allow-Origin 头
```

---

## 9.5 CSS 排查技巧

```css
/* 调试布局：给所有元素加边框 */
* { outline: 1px solid red !important; }

/* 检查 z-index 问题 */
/* z-index 只在 position 非 static 的元素上生效 */
.modal { position: fixed; z-index: 1000; }

/* 检查 overflow 隐藏问题 */
/* 父元素 overflow: hidden 会剪切子元素 */
/* 解决 fixed 定位被剪切：移除父元素的 overflow:hidden 或使用 Teleport */

/* Flexbox 调试 */
/* 子元素为什么没撑开？检查：
   1. 父元素是否有 height
   2. 是否有 align-items: stretch（默认）
   3. flex-grow 是否设置 */
```

---

## 9.6 错误监控与上报

```typescript
// 全局错误捕获
// main.ts
app.config.errorHandler = (error, instance, info) => {
  reportError({ error, component: instance?.$options.name, info });
};

// Promise 未捕获错误
window.addEventListener('unhandledrejection', (event) => {
  reportError({ error: event.reason, type: 'unhandledRejection' });
});

// 与 Sentry 集成
import * as Sentry from '@sentry/vue';

Sentry.init({
  app,
  dsn: import.meta.env.VITE_SENTRY_DSN,
  environment: import.meta.env.MODE,
  integrations: [
    Sentry.browserTracingIntegration({ router }),
    Sentry.replayIntegration(),
  ],
  tracesSampleRate: 0.1,
});
```

---

## 9.7 常见 Bug 案例集

### 案例 1：搜索框输入 ABC 只搜到了 C

```typescript
// 原因：未做防抖，每次按键都触发请求，请求顺序不一致
// ❌
watch(searchQuery, async (query) => {
  results.value = await search(query);
});

// ✅ 防抖 + 取消旧请求
let abortController: AbortController | null = null;

const debouncedSearch = useDebounceFn(async (query: string) => {
  abortController?.abort();
  abortController = new AbortController();
  try {
    results.value = await search(query, { signal: abortController.signal });
  } catch (e) {
    if (e instanceof DOMException && e.name === 'AbortError') return;
    throw e;
  }
}, 300);
```

### 案例 2：表单提交多次

```typescript
// 原因：用户快速多次点击提交按钮
// ✅ 禁用按钮 + loading 状态
const isSubmitting = ref(false);

async function handleSubmit() {
  if (isSubmitting.value) return;
  isSubmitting.value = true;
  try {
    await submitForm();
  } finally {
    isSubmitting.value = false;
  }
}
```

### 案例 3：登录后跳转到正确页面

```typescript
// 在路由守卫中记录目标路由
router.beforeEach((to, from, next) => {
  if (to.meta.requiresAuth && !isLoggedIn) {
    next({ name: 'Login', query: { redirect: to.fullPath } });
  } else {
    next();
  }
});

// 登录成功后读取 redirect
async function handleLogin() {
  await login(form);
  const redirect = route.query.redirect as string;
  router.push(redirect || '/');
}
```

### 案例 4：组件销毁后内存泄漏

```typescript
// ❌ 常见泄漏场景
onMounted(() => {
  // 1. 未清理的事件监听
  document.addEventListener('keydown', handler);
  
  // 2. 未清理的定时器
  setInterval(tick, 1000);
  
  // 3. 未关闭的 WebSocket
  const ws = new WebSocket(url);
  
  // 4. 未取消的动画帧
  requestAnimationFrame(animate);
});

// ✅ 在 onUnmounted 中清理
const cleanup: Array<() => void> = [];

onMounted(() => {
  document.addEventListener('keydown', handler);
  cleanup.push(() => document.removeEventListener('keydown', handler));
  
  const timer = setInterval(tick, 1000);
  cleanup.push(() => clearInterval(timer));
});

onUnmounted(() => cleanup.forEach(fn => fn()));

// 或者使用 VueUse 的自动清理
useEventListener(document, 'keydown', handler); // 自动在 unmounted 时移除
```

---

## 9.8 调试技巧汇总

```typescript
// 1. 在模板中直接打印状态（临时调试）
// <pre>{{ JSON.stringify(state, null, 2) }}</pre>

// 2. 条件断点（DevTools 中右键行号 → Add conditional breakpoint）
// 条件：userId === 'problematic-id'

// 3. debugger 语句
function suspiciousFunction() {
  debugger; // 暂停执行，打开 DevTools 自动停在此处
  return complexCalculation();
}

// 4. 追踪响应式变化
watch(suspiciousRef, (newVal, oldVal) => {
  console.trace(`${suspiciousRef.value} changed:`, oldVal, '→', newVal);
}, { deep: true });

// 5. 性能标记
performance.mark('operation-start');
doExpensiveWork();
performance.mark('operation-end');
performance.measure('operation', 'operation-start', 'operation-end');
```

---

## 9.9 章节小结

| 技能 | 重要程度 | 说明 |
|------|---------|------|
| Chrome DevTools | ⭐⭐⭐⭐⭐ | 必须熟练 |
| Vue DevTools | ⭐⭐⭐⭐⭐ | 调试 Vue 专用 |
| 响应式问题排查 | ⭐⭐⭐⭐⭐ | 最常见问题类型 |
| 网络请求排查 | ⭐⭐⭐⭐ | 前后端联调必备 |
| 错误监控 | ⭐⭐⭐⭐ | 生产环境必需 |
| 内存泄漏排查 | ⭐⭐⭐⭐ | 长期运行应用 |

> **下一步**：阅读 [第 10 章 - 团队协作与项目管理](10_team_and_project.md)
