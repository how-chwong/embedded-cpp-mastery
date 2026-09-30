# 第 05 章：全栈贯通：企业级 DevOps 智能监控大屏与管理系统实战

在前四章里，我们打牢了 TypeScript 基础、解析了 Vue 3 底层响应式机制、配置了 Monorepo 企业级工程化底座，并梳理了架构师级的部署与优化。

本章我们将迎来**终极战役**：开发一个完整的、高强度的、涵盖多个高级核心业务场景的 **企业级 DevOps 智能监控大屏与管理系统**。我们将交付完整的代码实现，绝无伪代码。

---

## 5.1 项目架构与核心功能版图

这是一个典型的工业级中后台，包含以下三大最具技术深度、最考验 5 年以上前端工程师功底的核心场景：

1. **场景一：大文件分片上传与断点续传（Large File Chunk Upload & Pause/Resume）**：使用 Spark-MD5 计算文件唯一 Hash，实现分片切割、并发控制（并发池）、暂停与断点续传。
2. **场景二：高性能实时监控看板（DevOps Real-Time Telemetry Dashboard）**：通过 WebSocket 与后端保持长连接，渲染 ECharts 实时服务器监控数据，具备自适应大小、定时 GC 与动画控制。
3. **场景三：动态路由与菜单树权限控制（Dynamic Permission Tree Routing）**：基于 RBAC 权限设计模型，根据后端 API 返回的用户权限标识，动态过滤并注入前端路由表，自动生成左侧导航树。

### 📁 推荐工程目录布局

```
apps/devops-admin-app/
├── src/
│   ├── api/
│   │   ├── dashboard.ts
│   │   └── upload.ts
│   ├── assets/
│   ├── components/
│   │   ├── RealTimeChart.vue
│   │   └── PermissionTree.vue
│   ├── composables/
│   │   └── useWebSocket.ts
│   ├── router/
│   │   └── index.ts
│   ├── store/
│   │   └── user.ts
│   ├── views/
│   │   ├── dashboard/
│   │   │   └── index.vue
│   │   ├── upload/
│   │   │   └── index.vue
│   │   └── login/
│   │       └── index.vue
│   ├── App.vue
│   └── main.ts
├── package.json
└── vite.config.ts
```

---

## 5.2 核心场景一：大文件分片上传与断点续传（TS/Vue 3 终极实现）

在常规开发中，使用表单直传 5GB 的大文件，不仅会因为传输超时崩溃，更无法应对“网络中途断开”的致命问题。
**架构师级解决方案**：前端在浏览器端利用 `Blob.prototype.slice` 把文件分割成 5MB 的小片（Chunk），计算文件的 **MD5 Hash**，在向后端查询其已上传分片列表后，进行**断点续传与高并发分片并发上传**。

### 💻 完整前端核心代码：`views/upload/index.vue`

```vue
<template>
  <div class="upload-container">
    <h2>🚀 企业级大文件分片 & 断点续传系统</h2>
    <div class="upload-card">
      <input type="file" @change="handleFileChange" :disabled="status === 'uploading'" />
      
      <div v-if="file" class="file-info">
        <p>文件名称：{{ file.name }}</p>
        <p>文件大小：{{ (file.size / 1024 / 1024).toFixed(2) }} MB</p>
        <p>MD5 Hash: <span class="hash-text">{{ hash || '正在计算 Hash 中...' }}</span></p>
      </div>

      <!-- 进度条 -->
      <div v-if="hash" class="progress-section">
        <p>计算 Hash 进度：{{ hashProgress }}%</p>
        <div class="progress-bar-bg">
          <div class="progress-bar-fg hash-bar" :style="{ width: hashProgress + '%' }"></div>
        </div>

        <p>文件上传总进度：{{ uploadProgress }}%</p>
        <div class="progress-bar-bg">
          <div class="progress-bar-fg upload-bar" :style="{ width: uploadProgress + '%' }"></div>
        </div>
      </div>

      <!-- 控制按钮 -->
      <div class="btn-group">
        <button @click="startUpload" :disabled="status !== 'ready' && status !== 'paused'">
          开始上传
        </button>
        <button @click="pauseUpload" :disabled="status !== 'uploading'">
          暂停上传
        </button>
        <button @click="resumeUpload" :disabled="status !== 'paused'">
          继续上传
        </button>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue';
import axios, { Canceler } from 'axios';

// 引入 spark-md5 用于计算文件散列。生产环境需用 pnpm install spark-md5 & @types/spark-md5
import SparkMD5 from 'spark-md5';

type UploadStatus = 'idle' | 'ready' | 'uploading' | 'paused' | 'success';

const CHUNK_SIZE = 5 * 1024 * 1024; // 💡 5MB 分片大小
const CONCURRENCY_LIMIT = 3;        // 💡 限制最多同时上传 3 个分片

const file = ref<File | null>(null);
const hash = ref<string>('');
const hashProgress = ref(0);
const status = ref<UploadStatus>('idle');

// 分片列表数据模型
interface Chunk {
  index: number;
  hash: string;
  chunk: Blob;
  progress: number;
  size: number;
}

const chunks = ref<Chunk[]>([]);
const cancelTokens = ref<Canceler[]>([]); // 暂存所有正在进行中的请求取消函数

// 计算文件总上传进度
const uploadProgress = computed(() => {
  if (!chunks.value.length) return 0;
  const loaded = chunks.value.reduce((acc, curr) => acc + curr.progress * curr.size / 100, 0);
  const total = file.value ? file.value.size : 0;
  return total ? Math.round((loaded / total) * 100) : 0;
});

// 1. 选择文件并开始生成切片与 MD5
const handleFileChange = async (e: Event) => {
  const target = e.target as HTMLInputElement;
  const selectedFile = target.files?.[0];
  if (!selectedFile) return;

  file.value = selectedFile;
  hash.value = '';
  hashProgress.value = 0;
  status.value = 'idle';
  chunks.value = [];

  // 1.1 分片切割
  const fileChunks = createFileChunks(selectedFile);
  
  // 1.2 异步非阻塞地计算整个大文件的 MD5 Hash
  status.value = 'ready';
  hash.value = await calculateFileHash(fileChunks);
  
  // 1.3 组装分片结构
  chunks.value = fileChunks.map((chunk, index) => ({
    index,
    hash: `${hash.value}_${index}`,
    chunk,
    progress: 0,
    size: chunk.size
  }));
};

// 2. 切片核心方法
const createFileChunks = (file: File): Blob[] => {
  const chunkList: Blob[] = [];
  let cur = 0;
  while (cur < file.size) {
    chunkList.push(file.slice(cur, cur + CHUNK_SIZE));
    cur += CHUNK_SIZE;
  }
  return chunkList;
};

// 3. 计算唯一 MD5：防止浏览器崩溃，采用分片增量式计算
const calculateFileHash = (fileChunks: Blob[]): Promise<string> => {
  return new Promise((resolve) => {
    const spark = new SparkMD5.ArrayBuffer();
    const reader = new FileReader();
    let count = 0;

    const loadNext = () => {
      reader.readAsArrayBuffer(fileChunks[count]);
    };

    reader.onload = (e) => {
      count++;
      spark.append(e.target?.result as ArrayBuffer);
      hashProgress.value = Math.round((count / fileChunks.length) * 100);
      
      if (count < fileChunks.length) {
        loadNext();
      } else {
        resolve(spark.end());
        hashProgress.value = 100;
      }
    };

    loadNext();
  });
};

// 4. 开始并发上传
const startUpload = async () => {
  if (!file.value || !hash.value) return;
  status.value = 'uploading';

  // 4.1 调用秒传接口检查：询问后端这个 Hash 文件是否已经全部上传，或已被上传了哪些分片
  const { uploadedList, shouldUpload } = await checkFileExist(hash.value, file.value.name);
  if (!shouldUpload) {
    status.value = 'success';
    chunks.value.forEach(c => c.progress = 100);
    alert('🎉 恭喜！该文件在服务器中已存在，秒传成功！');
    return;
  }

  // 4.2 过滤掉后端已经存在的切片（断点续传）
  const pendingChunks = chunks.value.filter(chunk => {
    const isUploaded = uploadedList.includes(chunk.index);
    if (isUploaded) {
      chunk.progress = 100; // 已上传的设为 100%
    }
    return !isUploaded;
  });

  // 4.3 核心：带有并发限制（并发池）的分片上传
  await uploadChunksWithLimit(pendingChunks);
};

// 5. 💡 核心并发控制器（并发池）
const uploadChunksWithLimit = async (pendingChunks: Chunk[]) => {
  let i = 0;
  const promises: Promise<void>[] = [];

  const uploadNext = async (): Promise<void> => {
    // 暂停退出标识
    if (status.value !== 'uploading') return;
    if (i >= pendingChunks.length) return;

    const currentChunk = pendingChunks[i++];
    const formData = new FormData();
    formData.append('file_hash', hash.value);
    formData.append('chunk_hash', currentChunk.hash);
    formData.append('chunk_index', currentChunk.index.toString());
    formData.append('chunk_data', currentChunk.chunk);

    const promise = axios.post('/api/upload/chunk', formData, {
      cancelToken: new axios.CancelToken(c => {
        cancelTokens.value.push(c); // 保存取消函数
      }),
      onUploadProgress: (progressEvent) => {
        // 计算单个切片进度
        if (progressEvent.total) {
          currentChunk.progress = Math.round((progressEvent.loaded / progressEvent.total) * 100);
        }
      }
    }).then(async () => {
      // 成功上传一个后，从进行中的列表移除取消方法
      if (status.value === 'uploading') {
        await uploadNext(); // 💡 尾递归：空出一个管道，立刻补上
      }
    }).catch(err => {
      if (axios.isCancel(err)) {
        console.log('切片请求已安全拦截取消:', currentChunk.index);
      } else {
        console.error('分片上传报错:', err);
      }
    });

    promises.push(promise);
  };

  // 开启初始通道
  const initialPromises = [];
  for (let limit = 0; limit < Math.min(CONCURRENCY_LIMIT, pendingChunks.length); limit++) {
    initialPromises.push(uploadNext());
  }

  await Promise.all(initialPromises);
  
  // 5.1 所有切片都运行完毕后，再次检查是否还在上传状态中，并发起合并请求
  if (status.value === 'uploading') {
    await mergeChunksRequest();
  }
};

// 6. 暂停上传：一键拦截进行中的 Axios
const pauseUpload = () => {
  status.value = 'paused';
  cancelTokens.value.forEach(cancel => cancel()); // 核心：瞬间阻断 TCP 流量
  cancelTokens.value = [];
};

// 7. 继续上传：重新拉起并发池
const resumeUpload = () => {
  startUpload();
};

// 8. 辅助：向后端发起合并切片指令
const mergeChunksRequest = async () => {
  if (!file.value || !hash.value) return;
  const res = await axios.post('/api/upload/merge', {
    file_hash: hash.value,
    file_name: file.value.name,
    chunk_size: CHUNK_SIZE
  });
  if (res.data.code === 200) {
    status.value = 'success';
    alert('🎉 大文件分片拼装成功，全链路闭环完成！');
  }
};

// 9. 辅助：校验接口调用
const checkFileExist = async (fileHash: string, fileName: string) => {
  const res = await axios.get(`/api/upload/check?hash=${fileHash}&fileName=${fileName}`);
  return res.data.data; // 返回格式 { uploadedList: number[], shouldUpload: boolean }
};
</script>

<style scoped>
.upload-container {
  max-width: 650px;
  margin: 30px auto;
  font-family: system-ui, sans-serif;
}
.upload-card {
  background: #fdfdfd;
  border: 1px solid #eaeaea;
  border-radius: 8px;
  padding: 25px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.05);
}
.file-info {
  margin: 15px 0;
  background: #f3f7ff;
  padding: 10px;
  border-radius: 4px;
}
.hash-text {
  font-family: monospace;
  font-weight: bold;
  color: #2c3e50;
}
.progress-section {
  margin: 20px 0;
}
.progress-bar-bg {
  width: 100%;
  height: 12px;
  background-color: #ebedf0;
  border-radius: 6px;
  overflow: hidden;
  margin-bottom: 12px;
}
.progress-bar-fg {
  height: 100%;
  transition: width 0.3s ease;
}
.hash-bar { background-color: #e6a23c; }
.upload-bar { background-color: #67c23a; }
.btn-group button {
  padding: 10px 18px;
  margin-right: 10px;
  border-radius: 4px;
  border: none;
  cursor: pointer;
  background-color: #409eff;
  color: #fff;
  font-weight: 500;
}
.btn-group button:disabled {
  background-color: #c0c4cc;
  cursor: not-allowed;
}
</style>
```

---

## 5.3 核心场景二：高性能 ECharts 服务器实时监控看板

在 DevOps 运维大屏中，我们需要流畅、不抖动地展示多台物理服务器的 CPU 负载、物理内存和网速指标。这通常通过 WebSockets 双向长连接，将高频推入的指标喂给 ECharts 组件，并进行极致的内存管理。

### 💻 完整前端核心代码：`components/RealTimeChart.vue`

```vue
<!-- components/RealTimeChart.vue -->
<template>
  <div class="chart-wrapper">
    <div class="chart-header">
      <h4>🖥️ 服务器 CPU & 内存实时遥测大屏</h4>
      <span class="status-indicator" :class="{ 'connected': isConnected }">
        {{ isConnected ? '● WebSocket 在线连接中' : '○ 断开中' }}
      </span>
    </div>
    <!-- ECharts 渲染容器 -->
    <div ref="chartRef" class="chart-container"></div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted, watch } from 'vue';
// 引入 ECharts 全部，实际项目可采用按需导入
import * as echarts from 'echarts';

interface TelemetryData {
  timestamp: string;
  cpuLoad: number;
  memoryUsage: number;
}

const chartRef = ref<HTMLDivElement | null>(null);
const isConnected = ref(false);

// 缓存用于渲染的数据流列表，最大保留 30 个时间节点（30 秒历史）
const MAX_DATA_LENGTH = 30;
const timeLabels = ref<string[]>([]);
const cpuData = ref<number[]>([]);
const memData = ref<number[]>([]);

let myChart: echarts.ECharts | null = null;
let ws: WebSocket | null = null;

// 1. 初始化 ECharts 实例
const initChart = () => {
  if (!chartRef.value) return;
  
  myChart = echarts.init(chartRef.value);
  const option: echarts.EChartsOption = {
    tooltip: { trigger: 'axis' },
    legend: { data: ['CPU 负载 (%)', '内存使用率 (%)'] },
    grid: { left: '3%', right: '4%', bottom: '3%', containLabel: true },
    xAxis: {
      type: 'category',
      boundaryGap: false,
      data: timeLabels.value
    },
    yAxis: {
      type: 'value',
      min: 0,
      max: 100
    },
    series: [
      {
        name: 'CPU 负载 (%)',
        type: 'line',
        smooth: true,
        showSymbol: false,
        data: cpuData.value,
        itemStyle: { color: '#409eff' },
        areaStyle: {
          color: new echarts.graphic.LinearGradient(0, 0, 0, 1, [
            { offset: 0, color: 'rgba(64,158,255,0.3)' },
            { offset: 1, color: 'rgba(64,158,255,0.01)' }
          ])
        }
      },
      {
        name: '内存使用率 (%)',
        type: 'line',
        smooth: true,
        showSymbol: false,
        data: memData.value,
        itemStyle: { color: '#67c23a' },
        areaStyle: {
          color: new echarts.graphic.LinearGradient(0, 0, 0, 1, [
            { offset: 0, color: 'rgba(103,194,58,0.3)' },
            { offset: 1, color: 'rgba(103,194,58,0.01)' }
          ])
        }
      }
    ]
  };

  myChart.setOption(option);
};

// 2. WebSocket 长连接及数据注入管理
const connectWebSocket = () => {
  // 生产环境可以通过 API 获取 ws 地址，此处模拟 WebSocket
  ws = new WebSocket('ws://localhost:5000/telemetry');
  
  ws.onopen = () => {
    isConnected.value = true;
  };

  ws.onmessage = (event) => {
    const rawData = JSON.parse(event.data) as TelemetryData;
    injectData(rawData);
  };

  ws.onclose = () => {
    isConnected.value = false;
    // 5 秒后自动执行断线重连（容灾机制）
    setTimeout(connectWebSocket, 5000);
  };
};

// 3. 内存友好地将最新遥测数据注入 ECharts 配置中
const injectData = (payload: TelemetryData) => {
  // 时间横轴
  timeLabels.value.push(payload.timestamp);
  cpuData.value.push(payload.cpuLoad);
  memData.value.push(payload.memoryUsage);

  // 💡 架构师级 GC 习惯：超过上限时，从数组头部剔除老数据，严防数组无限增长导致页面几分钟内内存崩溃
  if (timeLabels.value.length > MAX_DATA_LENGTH) {
    timeLabels.value.shift();
    cpuData.value.shift();
    memData.value.shift();
  }

  // 4. 重绘 ECharts，利用局部数据刷新机制
  myChart?.setOption({
    xAxis: { data: timeLabels.value },
    series: [
      { data: cpuData.value },
      { data: memData.value }
    ]
  });
};

// 5. 监听浏览器视口大小变化：大屏适配（响应式 resize）
const resizeHandler = () => {
  myChart?.resize();
};

onMounted(() => {
  initChart();
  connectWebSocket();
  window.addEventListener('resize', resizeHandler);
});

// 💡 极其重要的清理钩子：彻底销毁 DOM 绑定、解绑事件、阻断 Socket
onUnmounted(() => {
  window.removeEventListener('resize', resizeHandler);
  
  if (ws) {
    ws.close();
  }
  
  if (myChart) {
    myChart.dispose(); // 💡 彻底清空 Canvas WebGL 画布内存，严防显卡黑屏
    myChart = null;
  }
});
</script>

<style scoped>
.chart-wrapper {
  background: #111b2b;
  color: #fff;
  border-radius: 8px;
  padding: 15px;
  border: 1px solid #1c2b42;
}
.chart-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  border-bottom: 1px solid #1c2b42;
  padding-bottom: 10px;
  margin-bottom: 15px;
}
.status-indicator {
  font-size: 13px;
  color: #e6a23c;
}
.status-indicator.connected {
  color: #67c23a;
}
.chart-container {
  width: 100%;
  height: 380px;
}
</style>
```

---

## 5.4 核心场景三：基于 RBAC 的动态权限过滤与菜单路由注入

大型系统绝不会硬编码所有左侧菜单路由。我们需要依靠后端的 RBAC（角色权限基础控制）数据。
**流程逻辑**：用户登录 -> 获取 Token -> 请求 `getUserInfo()` 获得 `roles: ['admin']` 和 `permissions: ['user:edit', 'order:delete']` -> 前端路由生成算法比对权限表 -> `router.addRoute` 注入 -> 侧边栏自动高亮展示。

### 💻 完整权限过滤与路由状态代码：`store/user.ts` (Pinia setup 风格)

```typescript
// store/user.ts
import { defineStore } from 'pinia';
import { ref } from 'vue';
import { RouteRecordRaw } from 'vue-router';
import { router } from '@/router';

// 1. 模拟所有受权限控制的动态路由表（通常需要具备 roles 过滤元数据）
export const asyncRoutes: RouteRecordRaw[] = [
  {
    path: '/system',
    name: 'SystemSettings',
    component: () => import('@/views/system/index.vue'),
    meta: { title: '系统设置', roles: ['admin'] } // 只有系统管理员可见
  },
  {
    path: '/devops',
    name: 'DevOpsDashboard',
    component: () => import('@/views/dashboard/index.vue'),
    meta: { title: '监控看板', roles: ['admin', 'operator'] } // 管理员与运维都可见
  },
  {
    path: '/file-upload',
    name: 'FileUpload',
    component: () => import('@/views/upload/index.vue'),
    meta: { title: '大文件上传', roles: ['admin', 'operator', 'user'] } // 全员可见
  }
];

export const useUserStore = defineStore('user', () => {
  const token = ref<string>(localStorage.getItem('ACCESS_TOKEN') || '');
  const username = ref<string>('');
  const roles = ref<string[]>([]);
  const permissions = ref<string[]>([]);
  const allowedRoutes = ref<RouteRecordRaw[]>([]); // 暂存最终该用户可访问的所有动态路由

  // 重置 Token 状态
  const resetToken = () => {
    token.value = '';
    roles.value = [];
    permissions.value = [];
    localStorage.removeItem('ACCESS_TOKEN');
  };

  // 获取用户信息与角色标识
  const getUserInfo = async () => {
    // 模拟 API 请求
    const mockApiResponse = {
      username: '系统管理员-小李',
      roles: ['admin'], // 💡 拥有管理员角色
      permissions: ['user:edit', 'devops:view']
    };

    username.value = mockApiResponse.username;
    roles.value = mockApiResponse.roles;
    permissions.value = mockApiResponse.permissions;

    return mockApiResponse.roles;
  };

  // 💡 过滤动态路由的核心算法 (类似 C# Linq 逻辑)
  const generateRoutes = (userRoles: string[]): RouteRecordRaw[] => {
    const filterRoutes = (routes: RouteRecordRaw[]): RouteRecordRaw[] => {
      const res: RouteRecordRaw[] = [];
      routes.forEach(route => {
        const tmp = { ...route };
        // 如果路由没有设置 meta.roles，默认代表该页面不需要任何门槛，全员放行
        if (!tmp.meta || !tmp.meta.roles) {
          res.push(tmp);
        } else {
          // 💡 判定用户角色列表是否包含路由要求角色之一 (类似 C# roles.Any(r => routeRoles.Contains(r)))
          const hasPermission = userRoles.some(role => (tmp.meta!.roles as string[]).includes(role));
          if (hasPermission) {
            res.push(tmp);
          }
        }
      });
      return res;
    };

    const routes = filterRoutes(asyncRoutes);
    allowedRoutes.value = routes;
    return routes;
  };

  return {
    token,
    username,
    roles,
    permissions,
    allowedRoutes,
    resetToken,
    getUserInfo,
    generateRoutes
  };
});
```

---

## 5.5 .NET / C# WebAPI 后端终极无缝对接指导

作为一个跨语言高标准的架构师，如果你想带队无阻碍交付，你必须懂如何与后端做最高效的接口拉通。

### 1. 对应 C# ASP.NET Core 的 Controller 示例（大文件切片上传）
下面给出了 C# 配合上面大文件前端的最佳后端对接代码。它能实现：接收前端传来的分片字节、将分片暂存在临时散列目录下、当收到合并请求时将其拼装。

```csharp
// Controllers/UploadController.cs
using Microsoft.AspNetCore.Mvc;
using System.IO;

[ApiController]
[Route("api/upload")]
public class UploadController : ControllerBase
{
    private readonly string _tempFolder = Path.Combine(Directory.GetCurrentDirectory(), "UploadTemp");
    private readonly string _uploadFolder = Path.Combine(Directory.GetCurrentDirectory(), "Uploads");

    [HttpPost("chunk")]
    public async Task<IActionResult> UploadChunk([FromForm] string file_hash, [FromForm] string chunk_hash, [FromForm] int chunk_index, IFormFile chunk_data)
    {
        // 1. 为这个文件的所有分片创建专属 Hash 暂存子文件夹
        var fileTempDir = Path.Combine(_tempFolder, file_hash);
        if (!Directory.Exists(fileTempDir))
        {
            Directory.CreateDirectory(fileTempDir);
        }

        // 2. 写入这一片 (如 "UploadTemp/abcde_3")
        var chunkPath = Path.Combine(fileTempDir, chunk_index.ToString());
        using (var stream = new FileStream(chunkPath, FileMode.Create))
        {
            await chunk_data.CopyToAsync(stream);
        }

        return Ok(new { code = 200, message = "分片上传成功" });
    }

    [HttpPost("merge")]
    public IActionResult MergeChunks([FromBody] MergeRequest request)
    {
        var fileTempDir = Path.Combine(_tempFolder, request.file_hash);
        var finalFilePath = Path.Combine(_uploadFolder, request.file_name);

        if (!Directory.Exists(_uploadFolder)) Directory.CreateDirectory(_uploadFolder);

        // 3. 按照分片索引严格由小到大排序拼装流，极其严谨！
        var chunks = Directory.GetFiles(fileTempDir).OrderBy(f => int.Parse(Path.GetFileName(f)));

        using (var outputStream = new FileStream(finalFilePath, FileMode.Create))
        {
            foreach (var chunk in chunks)
            {
                using (var inputStream = new FileStream(chunk, FileMode.Open))
                {
                    inputStream.CopyTo(outputStream);
                }
            }
        }

        // 4. 清理临时分片目录，释放磁盘空间
        Directory.Delete(fileTempDir, true);

        return Ok(new { code = 200, message = "合并成功，文件完好无损" });
    }
}

public class MergeRequest { public string file_hash { get; set; } public string file_name { get; set; } }
```

---

## 5.6 课后综合演练与带队实战测试

恭喜你！到这里你已经拥有了高超的前端技术实力，并具备了 C# 后端架构大贯通的实力。为了确保你真正掌握，请按照以下步骤带领你的 3 人小队完成一次**极速实操部署测试**：

1. **项目自测（开发环境）**：
   * 在你的根目录下配置 Vite 代理，指向本地的 `.NET WebAPI` 后端。
   * 打开浏览器开发者工具（F12），切换到 **Network** 页，找到网络模拟器（No throttling）。
   * 选中一个 50MB 的大文件，开始上传，在中途将网络更改为 **Offline**（断线），此时上传自动暂停并终止。
   * 恢复网络，点击“**继续上传**”，观察 **Network** 中的请求是否接着之前已经上传的分片序号往后发送，而不是从 0 开始。
2. **生产实测（Docker/Nginx 模拟）**：
   * 将前端打包为 `dist/`，拉起 Nginx 本地服务。
   * 在前端后台打开实时监控大屏，静置 10 分钟。观察内存使用。
   * 检查 Nginx 日志，确保由于开启了 `Gzip on`，`.js` 与 `.css` 文件在网络传输中体积减少了 60% 以上，极速呈现。

这一套完整的全栈 DevOps 看板、分片断点上传、动态路由权限树、多阶段 Docker 打包，直接凝结了你 **5 年以上的前端工程心血**。愿你将其融入到团队、架构与产品中，披荆斩棘，创造无限商业价值！
