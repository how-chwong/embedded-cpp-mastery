# 第 08 章：RTOS 与并发编程

> **学习目标**：掌握 FreeRTOS 任务、队列、信号量、互斥锁的使用，设计健壮的实时嵌入式系统。

---

## 目录

1. [RTOS 基础概念](#1-rtos-基础概念)
2. [FreeRTOS 简介与移植](#2-freertos-简介与移植)
3. [任务管理](#3-任务管理)
4. [队列（Queue）](#4-队列queue)
5. [信号量（Semaphore）](#5-信号量semaphore)
6. [互斥锁（Mutex）](#6-互斥锁mutex)
7. [事件标志组](#7-事件标志组)
8. [软件定时器](#8-软件定时器)
9. [内存管理](#9-内存管理)
10. [实战：多传感器数据采集系统](#10-实战多传感器数据采集系统)

---

## 1. RTOS 基础概念

### 1.1 为什么需要 RTOS

| 裸机（Super Loop） | RTOS |
|-------------------|------|
| 单一执行流 | 多任务并发 |
| 延时阻塞全部 | 延时只阻塞当前任务 |
| 难以响应多事件 | 优先级抢占 |
| 代码耦合度高 | 模块化，松耦合 |
| 实时性难保证 | 确定性调度 |

### 1.2 核心概念

| 概念 | 说明 |
|------|------|
| 任务（Task） | 独立的执行单元，有自己的栈 |
| 调度器（Scheduler） | 决定哪个任务运行 |
| 上下文切换 | 保存/恢复任务状态 |
| 优先级 | 数字越大（FreeRTOS）优先级越高 |
| 时间片 | 同优先级任务轮转的时间 |
| 临界区 | 不能被打断的代码段 |

### 1.3 调度算法

```
任务状态机：

           创建
            ↓
          就绪(Ready) ←──── 解除阻塞
            ↓  ↑
  调度器 → 运行(Running) → 阻塞(Blocked)
            ↓
          挂起(Suspended)
```

---

## 2. FreeRTOS 简介与移植

### 2.1 FreeRTOS 配置（FreeRTOSConfig.h）

```c
#define configUSE_PREEMPTION              1    // 抢占式调度
#define configCPU_CLOCK_HZ                168000000UL
#define configTICK_RATE_HZ                1000 // 1ms tick
#define configMAX_PRIORITIES              8
#define configMINIMAL_STACK_SIZE          128  // 字（words）
#define configTOTAL_HEAP_SIZE             (50 * 1024)  // 50KB 堆
#define configUSE_MUTEXES                 1
#define configUSE_COUNTING_SEMAPHORES     1
#define configUSE_TIMERS                  1
#define configSUPPORT_DYNAMIC_ALLOCATION  1
#define configUSE_TRACE_FACILITY          1
#define INCLUDE_vTaskDelay                1
#define INCLUDE_vTaskSuspend              1
#define INCLUDE_xTaskGetCurrentTaskHandle 1
```

### 2.2 STM32CubeMX 集成 FreeRTOS

在 CubeMX 中启用 FREERTOS（基于 CMSIS-RTOS v2）后，自动生成初始化代码。

---

## 3. 任务管理

### 3.1 创建任务

```c
#include "FreeRTOS.h"
#include "task.h"

// 任务函数原型
void led_task(void *pvParams);
void sensor_task(void *pvParams);

// 任务句柄
TaskHandle_t xLedTask;
TaskHandle_t xSensorTask;

int main(void) {
    HAL_Init();
    SystemClock_Config();

    // 创建任务
    xTaskCreate(
        led_task,         // 函数
        "LED",            // 名称（调试用）
        256,              // 栈大小（words）
        NULL,             // 参数
        1,                // 优先级（1=低）
        &xLedTask         // 句柄（可为 NULL）
    );

    xTaskCreate(
        sensor_task,
        "Sensor",
        512,
        NULL,
        3,               // 高优先级
        &xSensorTask
    );

    vTaskStartScheduler();  // 启动调度器，永不返回
    while (1);
}

// 任务函数（永不返回）
void led_task(void *pvParams) {
    (void)pvParams;
    while (1) {
        HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);
        vTaskDelay(pdMS_TO_TICKS(500));  // 非阻塞延时
    }
}
```

### 3.2 任务控制

```c
// 挂起/恢复
vTaskSuspend(xLedTask);
vTaskResume(xLedTask);

// 在中断中恢复任务（不能用 vTaskResume）
BaseType_t higher_prio_woken = pdFALSE;
vTaskResumeFromISR(xLedTask, &higher_prio_woken);
portYIELD_FROM_ISR(higher_prio_woken);

// 获取任务信息
UBaseType_t stack_free = uxTaskGetStackHighWaterMark(xLedTask);
printf("LED 任务剩余栈：%u words\n", stack_free);

// 删除任务
vTaskDelete(xLedTask);
```

### 3.3 空闲任务钩子（低功耗）

```c
void vApplicationIdleHook(void) {
    // 所有任务都在等待时执行，适合进入低功耗模式
    __WFI();  // 等待中断唤醒
}

// Tickless 空闲模式（更深层低功耗）
// 在 FreeRTOSConfig.h 中启用
#define configUSE_TICKLESS_IDLE 1
```

---

## 4. 队列（Queue）

```c
#include "queue.h"

// 定义消息类型
typedef struct {
    uint8_t sensor_id;
    float   value;
    uint32_t timestamp;
} SensorMsg;

QueueHandle_t xSensorQueue;

void queue_init(void) {
    // 创建队列：最多 10 个 SensorMsg
    xSensorQueue = xQueueCreate(10, sizeof(SensorMsg));
    configASSERT(xSensorQueue != NULL);
}

// 生产者任务（传感器读取）
void sensor_producer_task(void *pv) {
    SensorMsg msg;
    while (1) {
        msg.sensor_id = 0;
        msg.value     = read_temperature();
        msg.timestamp = HAL_GetTick();

        // 发送（等待最多 10ms）
        if (xQueueSend(xSensorQueue, &msg, pdMS_TO_TICKS(10)) != pdPASS) {
            // 队列满，处理错误
        }
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}

// 消费者任务（数据处理/上报）
void data_processor_task(void *pv) {
    SensorMsg msg;
    while (1) {
        // 阻塞等待消息（portMAX_DELAY = 永久等待）
        if (xQueueReceive(xSensorQueue, &msg, portMAX_DELAY) == pdPASS) {
            printf("[%lu] Sensor%d: %.2f\n",
                   msg.timestamp, msg.sensor_id, msg.value);
        }
    }
}

// 在中断中发送到队列
void ADC_IRQHandler(void) {
    SensorMsg msg = {.sensor_id=1, .value=read_adc()};
    BaseType_t woken = pdFALSE;
    xQueueSendFromISR(xSensorQueue, &msg, &woken);
    portYIELD_FROM_ISR(woken);
}
```

---

## 5. 信号量（Semaphore）

### 5.1 二值信号量（同步）

```c
#include "semphr.h"

SemaphoreHandle_t xDataReadySem;

// 初始化
xDataReadySem = xSemaphoreCreateBinary();

// 中断中"给出"信号量
void USART1_IRQHandler(void) {
    BaseType_t woken = pdFALSE;
    // 接收到完整帧
    xSemaphoreGiveFromISR(xDataReadySem, &woken);
    portYIELD_FROM_ISR(woken);
}

// 任务中"等待"信号量
void uart_process_task(void *pv) {
    while (1) {
        if (xSemaphoreTake(xDataReadySem, portMAX_DELAY) == pdPASS) {
            process_uart_frame();
        }
    }
}
```

### 5.2 计数信号量（资源池）

```c
// 控制最多 3 个并发 HTTP 连接
SemaphoreHandle_t xConnSem = xSemaphoreCreateCounting(3, 3);

void http_task(void *pv) {
    xSemaphoreTake(xConnSem, portMAX_DELAY);  // 获取连接槽
    // 发起 HTTP 请求...
    xSemaphoreGive(xConnSem);  // 释放连接槽
}
```

---

## 6. 互斥锁（Mutex）

```c
// 互斥锁：保护共享资源（比二值信号量多了优先级继承）
MutexHandle_t xUartMutex;

void uart_mutex_init(void) {
    xUartMutex = xSemaphoreCreateMutex();
}

// 线程安全的 UART 打印
void uart_printf(const char *fmt, ...) {
    char buf[128];
    va_list args;
    va_start(args, fmt);
    vsnprintf(buf, sizeof(buf), fmt, args);
    va_end(args);

    xSemaphoreTake(xUartMutex, portMAX_DELAY);
    HAL_UART_Transmit(&huart2, (uint8_t*)buf, strlen(buf), 100);
    xSemaphoreGive(xUartMutex);
}

// 递归互斥锁（同一任务可重入）
MutexHandle_t xRecMutex = xSemaphoreCreateRecursiveMutex();
xSemaphoreTakeRecursive(xRecMutex, portMAX_DELAY);
// 嵌套调用
xSemaphoreTakeRecursive(xRecMutex, portMAX_DELAY);
xSemaphoreGiveRecursive(xRecMutex);
xSemaphoreGiveRecursive(xRecMutex);
```

---

## 7. 事件标志组

```c
#include "event_groups.h"

#define EVT_SENSOR_READY   (1 << 0)
#define EVT_NETWORK_UP     (1 << 1)
#define EVT_CONFIG_LOADED  (1 << 2)
#define EVT_ALL_READY      (EVT_SENSOR_READY | EVT_NETWORK_UP | EVT_CONFIG_LOADED)

EventGroupHandle_t xSystemEvents;

void system_monitor_task(void *pv) {
    // 等待所有子系统就绪（AND 等待）
    xEventGroupWaitBits(
        xSystemEvents,
        EVT_ALL_READY,     // 等待的位
        pdTRUE,            // 退出时清除
        pdTRUE,            // AND（全部满足）
        portMAX_DELAY
    );
    printf("系统就绪，开始运行\n");
}

void sensor_init_task(void *pv) {
    init_sensors();
    xEventGroupSetBits(xSystemEvents, EVT_SENSOR_READY);
    vTaskDelete(NULL);
}
```

---

## 8. 软件定时器

```c
#include "timers.h"

TimerHandle_t xHeartbeatTimer;
TimerHandle_t xWatchdogTimer;

void heartbeat_callback(TimerHandle_t xTimer) {
    HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);
}

void watchdog_callback(TimerHandle_t xTimer) {
    // 系统超时，重启
    printf("Watchdog timeout! Resetting...\n");
    NVIC_SystemReset();
}

void timers_init(void) {
    // 心跳定时器：周期 500ms，自动重载
    xHeartbeatTimer = xTimerCreate("Heartbeat", pdMS_TO_TICKS(500),
                                    pdTRUE, 0, heartbeat_callback);
    xTimerStart(xHeartbeatTimer, 0);

    // 看门狗定时器：5秒无喂狗则重启
    xWatchdogTimer = xTimerCreate("Watchdog", pdMS_TO_TICKS(5000),
                                   pdFALSE, 0, watchdog_callback);
    xTimerStart(xWatchdogTimer, 0);
}

// 喂软件看门狗
void feed_watchdog(void) {
    xTimerReset(xWatchdogTimer, 0);
}
```

---

## 9. 内存管理

FreeRTOS 提供 5 种堆管理方案：

| 方案 | 分配 | 释放 | 说明 |
|------|------|------|------|
| heap_1 | ✅ | ❌ | 最简单，不支持释放 |
| heap_2 | ✅ | ✅ | 简单，有碎片 |
| heap_3 | 包装 malloc/free | — | 线程安全 malloc |
| heap_4 | ✅ | ✅ | 合并相邻空闲块 |
| heap_5 | ✅ | ✅ | 支持非连续内存 |

```c
// 推荐使用 heap_4
// 监控堆使用
size_t free_heap = xPortGetFreeHeapSize();
size_t min_heap  = xPortGetMinimumEverFreeHeapSize();
printf("堆：当前空闲 %zu B，历史最小 %zu B\n", free_heap, min_heap);

// 内存申请失败钩子
void vApplicationMallocFailedHook(void) {
    printf("FATAL: FreeRTOS malloc failed!\n");
    taskDISABLE_INTERRUPTS();
    while (1);
}
```

---

## 10. 实战：多传感器数据采集系统

```c
// 完整的 FreeRTOS 多任务架构示例

/* ─── 数据类型 ─── */
typedef struct {
    uint8_t  id;
    float    value;
    uint32_t ts_ms;
} SensorData;

/* ─── 全局对象 ─── */
static QueueHandle_t  xDataQueue;
static SemaphoreHandle_t xConsoleMutex;

/* ─── 工具函数 ─── */
void console_print(const char *fmt, ...) {
    char buf[128];
    va_list a; va_start(a, fmt); vsnprintf(buf, 128, fmt, a); va_end(a);
    xSemaphoreTake(xConsoleMutex, portMAX_DELAY);
    HAL_UART_Transmit(&huart2, (uint8_t*)buf, strlen(buf), 100);
    xSemaphoreGive(xConsoleMutex);
}

/* ─── 任务：温度采集（高优先级） ─── */
void task_temp_sensor(void *pv) {
    SensorData d = {.id = 0};
    while (1) {
        d.value = adc_read_temperature(0);
        d.ts_ms = HAL_GetTick();
        xQueueSend(xDataQueue, &d, pdMS_TO_TICKS(10));
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}

/* ─── 任务：湿度采集（高优先级） ─── */
void task_hum_sensor(void *pv) {
    SensorData d = {.id = 1};
    while (1) {
        float t, h;
        if (aht20_read(&hi2c1, &(AHT20_Data){t, h})) {
            d.value = h;
            d.ts_ms = HAL_GetTick();
            xQueueSend(xDataQueue, &d, pdMS_TO_TICKS(10));
        }
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

/* ─── 任务：数据处理与上报（中优先级） ─── */
void task_data_logger(void *pv) {
    SensorData d;
    while (1) {
        if (xQueueReceive(xDataQueue, &d, portMAX_DELAY) == pdPASS) {
            console_print("[%lu] Sensor%d = %.2f\n", d.ts_ms, d.id, d.value);
            // 也可写入 Flash / 上报 MQTT
        }
    }
}

/* ─── 任务：系统监控（低优先级） ─── */
void task_system_monitor(void *pv) {
    while (1) {
        console_print("堆: %zu B 空闲\n", xPortGetFreeHeapSize());
        vTaskDelay(pdMS_TO_TICKS(5000));
    }
}

/* ─── main ─── */
int main(void) {
    HAL_Init();
    SystemClock_Config();
    peripheral_init();

    xDataQueue    = xQueueCreate(20, sizeof(SensorData));
    xConsoleMutex = xSemaphoreCreateMutex();

    xTaskCreate(task_temp_sensor,    "TempSns", 256, NULL, 4, NULL);
    xTaskCreate(task_hum_sensor,     "HumSns",  512, NULL, 4, NULL);
    xTaskCreate(task_data_logger,    "Logger",  512, NULL, 3, NULL);
    xTaskCreate(task_system_monitor, "Monitor", 256, NULL, 1, NULL);

    vTaskStartScheduler();
    while (1);
}
```

---

**上一章**：[第 07 章：嵌入式硬件与外设编程 ←](07_embedded_hardware.md)  
**下一章**：[第 09 章：调试、测试与性能优化 →](09_debug_test_perf.md)
