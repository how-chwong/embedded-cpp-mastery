# 第 01 章：前端基础知识

> 目标：理解浏览器工作原理、掌握 HTML / CSS / JavaScript 核心概念，为后续的 Vue + TypeScript 学习打下坚实地基。

---

## 1.1 浏览器工作原理

### 关键流程

```
URL 输入
  → DNS 解析 → TCP 握手 → HTTP(S) 请求
  → 服务器返回 HTML
  → 浏览器解析 HTML → 构建 DOM 树
  → 解析 CSS → 构建 CSSOM 树
  → DOM + CSSOM → Render Tree
  → Layout（布局）→ Paint（绘制）→ Composite（合成）
```

### 与 C# 类比

| 概念 | C# / .NET | 浏览器前端 |
|------|-----------|-----------|
| 运行时 | CLR | V8 引擎 (Chrome) |
| 入口点 | `Main()` | `DOMContentLoaded` 事件 |
| 内存管理 | GC | V8 GC (分代回收) |
| 线程 | 多线程 | 单线程 + Event Loop |
| 异步 | `async/await` | `async/await` (相同！) |

### 重要概念：Event Loop

```javascript
// JavaScript 是单线程的，通过事件循环实现异步
console.log('1. 同步代码');

setTimeout(() => {
  console.log('3. 宏任务（setTimeout）');
}, 0);

Promise.resolve().then(() => {
  console.log('2. 微任务（Promise）');
});

// 输出顺序：1 → 2 → 3
// 微任务（Promise, queueMicrotask）优先于宏任务（setTimeout, setInterval）
```

---

## 1.2 HTML5 核心

### 语义化标签

```html
<!-- ❌ 旧写法：全是 div，无语义 -->
<div class="header">...</div>
<div class="nav">...</div>
<div class="content">...</div>

<!-- ✅ 语义化：利于 SEO、无障碍访问 -->
<header>
  <nav>
    <a href="/">首页</a>
    <a href="/about">关于</a>
  </nav>
</header>
<main>
  <article>
    <h1>文章标题</h1>
    <p>内容...</p>
  </article>
  <aside>侧边栏</aside>
</main>
<footer>版权信息</footer>
```

### 表单与数据属性

```html
<!-- 表单验证 -->
<form @submit.prevent="handleSubmit">
  <input 
    type="email" 
    required 
    minlength="5"
    placeholder="输入邮箱"
  />
  <input type="number" min="0" max="100" step="1" />
</form>

<!-- 自定义数据属性 data-* -->
<div data-user-id="123" data-role="admin">用户</div>
```

```javascript
const el = document.querySelector('[data-user-id]');
console.log(el.dataset.userId); // "123"
console.log(el.dataset.role);   // "admin"
```

---

## 1.3 CSS3 核心

### 盒模型

```css
/* 默认 box-sizing: content-box */
/* 设置宽高后，padding 和 border 会在外部增加 */

/* ✅ 推荐：border-box */
*, *::before, *::after {
  box-sizing: border-box;
}
/* 设置宽高后，padding 和 border 包含在内 */
```

### Flexbox（必须掌握）

```css
.container {
  display: flex;
  flex-direction: row;          /* 主轴方向：row | column */
  justify-content: center;      /* 主轴对齐 */
  align-items: center;          /* 交叉轴对齐 */
  gap: 16px;                    /* 子元素间距 */
  flex-wrap: wrap;              /* 换行 */
}

.item {
  flex: 1;          /* flex-grow: 1, flex-shrink: 1, flex-basis: 0 */
  flex: 0 0 200px;  /* 固定宽度，不拉伸不压缩 */
}
```

### Grid（复杂布局）

```css
.grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);     /* 3 等分列 */
  grid-template-columns: 200px 1fr 2fr;      /* 固定 + 弹性 */
  grid-template-rows: auto;
  gap: 20px;
}

/* 跨列/行 */
.header { grid-column: 1 / -1; } /* 横跨所有列 */
.sidebar { grid-row: 2 / 4; }    /* 占两行 */
```

### CSS 变量（Custom Properties）

```css
:root {
  --color-primary: #4f46e5;
  --color-text: #1f2937;
  --spacing-md: 16px;
  --border-radius: 8px;
}

.button {
  background: var(--color-primary);
  padding: var(--spacing-md);
  border-radius: var(--border-radius);
}

/* 动态主题切换 */
[data-theme="dark"] {
  --color-text: #f9fafb;
  --color-primary: #818cf8;
}
```

### 响应式设计

```css
/* 移动优先 */
.container {
  width: 100%;
  padding: 16px;
}

@media (min-width: 768px) {
  .container {
    max-width: 768px;
    margin: 0 auto;
  }
}

@media (min-width: 1024px) {
  .container {
    max-width: 1024px;
  }
}
```

---

## 1.4 JavaScript 核心（ES2020+）

### 变量与作用域

```javascript
// var：函数作用域，存在变量提升（避免使用）
// let：块级作用域，可重新赋值
// const：块级作用域，不可重新赋值（对象属性仍可修改）

const user = { name: 'Alice' };
user.name = 'Bob';    // ✅ 可以
user = {};            // ❌ 报错

// 暂时性死区（Temporal Dead Zone）
console.log(a); // ReferenceError
let a = 1;
```

### 解构赋值

```javascript
// 数组解构
const [first, second, ...rest] = [1, 2, 3, 4, 5];

// 对象解构
const { name, age = 18, address: { city } } = user;

// 函数参数解构
function greet({ name, role = 'user' }) {
  return `Hello ${name}, you are ${role}`;
}
```

### 展开运算符与剩余参数

```javascript
// 展开（浅拷贝）
const newArr = [...arr1, ...arr2];
const newObj = { ...obj1, ...obj2, extra: 'value' };

// 剩余参数
function sum(...numbers) {
  return numbers.reduce((acc, n) => acc + n, 0);
}
```

### 可选链与空值合并

```javascript
// 可选链 ?. 
const city = user?.address?.city;           // 不会抛出错误
const fn = obj?.method?.();                 // 方法调用
const val = arr?.[0];                       // 数组访问

// 空值合并 ?? （只对 null/undefined 起作用）
const name = user.name ?? '匿名';
// 区别于 ||（|| 对 0, '', false 也会取右侧）
const count = 0 ?? 'default';  // 0
const count2 = 0 || 'default'; // 'default'
```

### Promise 与 async/await

```javascript
// Promise 基础
function fetchUser(id) {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      if (id > 0) resolve({ id, name: 'Alice' });
      else reject(new Error('Invalid ID'));
    }, 1000);
  });
}

// async/await（推荐写法）
async function loadUserData(id) {
  try {
    const user = await fetchUser(id);
    const posts = await fetchPosts(user.id);
    return { user, posts };
  } catch (error) {
    console.error('加载失败:', error.message);
    throw error; // 继续向上抛出
  } finally {
    console.log('请求完成（无论成功失败）');
  }
}

// 并发请求
async function loadAll() {
  const [users, products] = await Promise.all([
    fetchUsers(),
    fetchProducts(),
  ]);
  return { users, products };
}

// 与 C# 对比
// C#:  Task<User> GetUser(int id) { ... }
//       var user = await GetUser(1);
// JS:  async function getUser(id) { ... }
//       const user = await getUser(1);
// 几乎一样！
```

### 模块系统（ESM）

```javascript
// 导出（named export）
export const PI = 3.14;
export function add(a, b) { return a + b; }
export class Calculator { ... }

// 默认导出
export default class App { ... }

// 导入
import App from './App.vue';
import { PI, add } from './math.js';
import * as MathUtils from './math.js';

// 动态导入（代码分割）
const module = await import('./heavy-module.js');
```

### 数组方法（高频使用）

```javascript
const users = [
  { id: 1, name: 'Alice', age: 25, active: true },
  { id: 2, name: 'Bob',   age: 30, active: false },
  { id: 3, name: 'Carol', age: 22, active: true },
];

// map：转换每个元素，返回新数组
const names = users.map(u => u.name); // ['Alice', 'Bob', 'Carol']

// filter：过滤，返回新数组
const active = users.filter(u => u.active);

// find：找到第一个匹配元素
const alice = users.find(u => u.name === 'Alice');

// findIndex：找到索引
const idx = users.findIndex(u => u.id === 2);

// some / every：判断
const hasAdmin = users.some(u => u.role === 'admin');
const allActive = users.every(u => u.active);

// reduce：聚合
const totalAge = users.reduce((sum, u) => sum + u.age, 0);

// flat / flatMap
const nested = [[1, 2], [3, 4]];
nested.flat();        // [1, 2, 3, 4]
nested.flatMap(x => x.map(n => n * 2)); // [2, 4, 6, 8]

// 链式调用
const result = users
  .filter(u => u.active)
  .map(u => ({ ...u, label: `${u.name}(${u.age})` }))
  .sort((a, b) => a.age - b.age);
```

---

## 1.5 DOM 操作（了解即可，Vue 会代劳大部分）

```javascript
// 查询
const el = document.querySelector('.my-class');
const els = document.querySelectorAll('li');

// 修改
el.textContent = '新文字';
el.innerHTML = '<strong>加粗</strong>';
el.classList.add('active');
el.classList.remove('active');
el.classList.toggle('active');

// 事件
el.addEventListener('click', (event) => {
  event.preventDefault(); // 阻止默认行为
  event.stopPropagation(); // 阻止冒泡
  console.log(event.target);
});

// 创建与插入
const div = document.createElement('div');
div.className = 'card';
document.body.appendChild(div);
```

---

## 1.6 常见面试与实战问题

### 防抖（Debounce）与节流（Throttle）

```javascript
// 防抖：最后一次触发后等待 delay 毫秒再执行（搜索框输入）
function debounce(fn, delay) {
  let timer;
  return function (...args) {
    clearTimeout(timer);
    timer = setTimeout(() => fn.apply(this, args), delay);
  };
}

// 节流：每 interval 毫秒最多执行一次（滚动事件）
function throttle(fn, interval) {
  let lastTime = 0;
  return function (...args) {
    const now = Date.now();
    if (now - lastTime >= interval) {
      lastTime = now;
      fn.apply(this, args);
    }
  };
}

// 实际使用（VueUse 库已封装）
import { useDebounceFn, useThrottleFn } from '@vueuse/core';
const debouncedSearch = useDebounceFn(search, 300);
```

### 深拷贝

```javascript
// 简单方法（不支持 Date, Function, undefined）
const clone = JSON.parse(JSON.stringify(obj));

// 结构化克隆（现代浏览器）
const clone2 = structuredClone(obj); // ✅ 推荐

// 递归深拷贝（手写）
function deepClone(value) {
  if (value === null || typeof value !== 'object') return value;
  if (Array.isArray(value)) return value.map(deepClone);
  return Object.fromEntries(
    Object.entries(value).map(([k, v]) => [k, deepClone(v)])
  );
}
```

---

## 1.7 章节小结

| 知识点 | 重要程度 | 备注 |
|--------|---------|------|
| 盒模型 + Flex + Grid | ⭐⭐⭐⭐⭐ | 每天都用 |
| async/await + Promise | ⭐⭐⭐⭐⭐ | 所有异步操作 |
| 解构 / 展开 / 可选链 | ⭐⭐⭐⭐⭐ | Vue 代码大量使用 |
| 数组方法（map/filter/reduce） | ⭐⭐⭐⭐⭐ | 数据处理核心 |
| Event Loop | ⭐⭐⭐⭐ | 面试必考 |
| DOM 操作 | ⭐⭐⭐ | Vue 代劳，了解即可 |
| 防抖 / 节流 | ⭐⭐⭐⭐ | 性能优化必备 |

> **下一步**：阅读 [第 02 章 - 开发工具链](02_dev_tools.md)
