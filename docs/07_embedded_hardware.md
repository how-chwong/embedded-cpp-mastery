# 第 07 章：嵌入式硬件与外设编程

> **学习目标**：掌握 GPIO、UART、SPI、I2C、ADC、DMA、PWM 等常见外设的编程方法。

---

## 目录

1. [GPIO 编程](#1-gpio-编程)
2. [UART 串口通信](#2-uart-串口通信)
3. [SPI 总线](#3-spi-总线)
4. [I2C 总线](#4-i2c-总线)
5. [ADC 与 DAC](#5-adc-与-dac)
6. [DMA（直接内存访问）](#6-dma直接内存访问)
7. [PWM 与电机控制](#7-pwm-与电机控制)
8. [CAN 总线](#8-can-总线)
9. [Flash 存储编程](#9-flash-存储编程)
10. [实战：多协议传感器驱动](#10-实战多协议传感器驱动)

---

## 1. GPIO 编程

### 1.1 GPIO 模式

| 模式 | 说明 | 典型用途 |
|------|------|---------|
| 输入浮空 | 无上下拉 | 外接上下拉电阻 |
| 输入上拉 | 内部上拉到 VCC | 按键（低有效） |
| 输入下拉 | 内部下拉到 GND | 按键（高有效） |
| 推挽输出 | 强驱动高低 | LED、片选信号 |
| 开漏输出 | 只能拉低 | I2C 总线、线与 |
| 复用功能 | 外设控制引脚 | UART/SPI/I2C |
| 模拟输入 | 连接 ADC | 模拟传感器 |

### 1.2 STM32 HAL GPIO 示例

```c
#include "stm32f4xx_hal.h"

// 初始化 LED（PA5，推挽输出）
void gpio_init(void) {
    __HAL_RCC_GPIOA_CLK_ENABLE();

    GPIO_InitTypeDef cfg = {
        .Pin   = GPIO_PIN_5,
        .Mode  = GPIO_MODE_OUTPUT_PP,
        .Pull  = GPIO_NOPULL,
        .Speed = GPIO_SPEED_FREQ_LOW,
    };
    HAL_GPIO_Init(GPIOA, &cfg);
}

// 初始化按键（PC13，输入下拉）
void button_init(void) {
    __HAL_RCC_GPIOC_CLK_ENABLE();

    GPIO_InitTypeDef cfg = {
        .Pin  = GPIO_PIN_13,
        .Mode = GPIO_MODE_INPUT,
        .Pull = GPIO_PULLDOWN,
    };
    HAL_GPIO_Init(GPIOC, &cfg);
}

// 外部中断按键
void exti_button_init(void) {
    __HAL_RCC_GPIOC_CLK_ENABLE();

    GPIO_InitTypeDef cfg = {
        .Pin  = GPIO_PIN_13,
        .Mode = GPIO_MODE_IT_FALLING,  // 下降沿触发
        .Pull = GPIO_PULLUP,
    };
    HAL_GPIO_Init(GPIOC, &cfg);

    HAL_NVIC_SetPriority(EXTI15_10_IRQn, 2, 0);
    HAL_NVIC_EnableIRQ(EXTI15_10_IRQn);
}

void EXTI15_10_IRQHandler(void) {
    HAL_GPIO_EXTI_IRQHandler(GPIO_PIN_13);
}

void HAL_GPIO_EXTI_Callback(uint16_t pin) {
    if (pin == GPIO_PIN_13) {
        HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);  // 翻转 LED
    }
}
```

### 1.3 按键消抖

```c
#define DEBOUNCE_MS  20

typedef struct {
    GPIO_TypeDef *port;
    uint16_t      pin;
    uint32_t      last_change_ms;
    bool          state;
    bool          prev_raw;
} Button;

bool button_read(Button *btn) {
    bool raw = (HAL_GPIO_ReadPin(btn->port, btn->pin) == GPIO_PIN_SET);
    if (raw != btn->prev_raw) {
        btn->last_change_ms = HAL_GetTick();
        btn->prev_raw = raw;
    }
    if ((HAL_GetTick() - btn->last_change_ms) > DEBOUNCE_MS) {
        btn->state = raw;
    }
    return btn->state;
}
```

---

## 2. UART 串口通信

### 2.1 UART 基础参数

```c
// 波特率、数据位、停止位、校验位
// 常用：115200, 8N1（8 数据位，无校验，1 停止位）
UART_HandleTypeDef huart2 = {
    .Instance        = USART2,
    .Init = {
        .BaudRate    = 115200,
        .WordLength  = UART_WORDLENGTH_8B,
        .StopBits    = UART_STOPBITS_1,
        .Parity      = UART_PARITY_NONE,
        .Mode        = UART_MODE_TX_RX,
        .HwFlowCtl   = UART_HWCONTROL_NONE,
    },
};
HAL_UART_Init(&huart2);
```

### 2.2 轮询、中断、DMA 三种模式

```c
uint8_t tx_buf[] = "Hello UART!\r\n";
uint8_t rx_buf[64];

// 1. 轮询（阻塞）
HAL_UART_Transmit(&huart2, tx_buf, sizeof(tx_buf), 100);
HAL_UART_Receive(&huart2, rx_buf, sizeof(rx_buf), 1000);

// 2. 中断（非阻塞）
HAL_UART_Receive_IT(&huart2, rx_buf, 1);  // 每次接收 1 字节

void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart) {
    if (huart->Instance == USART2) {
        process_byte(rx_buf[0]);
        HAL_UART_Receive_IT(&huart2, rx_buf, 1);  // 重启接收
    }
}

// 3. DMA（最高效，CPU 无需参与）
HAL_UART_Receive_DMA(&huart2, rx_buf, sizeof(rx_buf));
```

### 2.3 环形缓冲区（UART 接收最佳实践）

```c
#define RING_BUF_SIZE  256

typedef struct {
    uint8_t  buf[RING_BUF_SIZE];
    uint16_t head;
    uint16_t tail;
} RingBuf;

static RingBuf uart_rx_ring;

void ring_push(RingBuf *rb, uint8_t byte) {
    uint16_t next = (rb->head + 1) % RING_BUF_SIZE;
    if (next != rb->tail) {  // 非满
        rb->buf[rb->head] = byte;
        rb->head = next;
    }
}

bool ring_pop(RingBuf *rb, uint8_t *byte) {
    if (rb->head == rb->tail) return false;  // 空
    *byte = rb->buf[rb->tail];
    rb->tail = (rb->tail + 1) % RING_BUF_SIZE;
    return true;
}

// 在中断中推入
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *h) {
    ring_push(&uart_rx_ring, rx_char);
    HAL_UART_Receive_IT(h, &rx_char, 1);
}
```

### 2.4 Modbus RTU 协议（工业 UART 应用）

```c
#define MODBUS_READ_HOLDING  0x03

uint8_t modbus_build_request(uint8_t *buf, uint8_t dev_addr,
                              uint16_t reg_addr, uint16_t count) {
    buf[0] = dev_addr;
    buf[1] = MODBUS_READ_HOLDING;
    buf[2] = reg_addr >> 8;
    buf[3] = reg_addr & 0xFF;
    buf[4] = count >> 8;
    buf[5] = count & 0xFF;
    uint16_t crc = modbus_crc16(buf, 6);
    buf[6] = crc & 0xFF;
    buf[7] = crc >> 8;
    return 8;
}
```

---

## 3. SPI 总线

### 3.1 SPI 基础

```
主机(Master)         从机(Slave)
SCLK  ─────────────→ SCLK   时钟（由主机产生）
MOSI  ─────────────→ MOSI   主发从收
MISO  ←───────────── MISO   主收从发
CS    ─────────────→ CS     片选（低有效）
```

### 3.2 SPI 配置与使用

```c
SPI_HandleTypeDef hspi1 = {
    .Instance               = SPI1,
    .Init = {
        .Mode               = SPI_MODE_MASTER,
        .Direction          = SPI_DIRECTION_2LINES,
        .DataSize           = SPI_DATASIZE_8BIT,
        .CLKPolarity        = SPI_POLARITY_LOW,   // CPOL=0
        .CLKPhase           = SPI_PHASE_1EDGE,    // CPHA=0（Mode 0）
        .NSS                = SPI_NSS_SOFT,
        .BaudRatePrescaler  = SPI_BAUDRATEPRESCALER_16,  // fclk/16
        .FirstBit           = SPI_FIRSTBIT_MSB,
    },
};
HAL_SPI_Init(&hspi1);

// 读写字节
uint8_t spi_transfer(uint8_t tx) {
    uint8_t rx;
    HAL_SPI_TransmitReceive(&hspi1, &tx, &rx, 1, 10);
    return rx;
}

// 读取 W25QXX Flash ID
uint32_t w25q_read_id(void) {
    HAL_GPIO_WritePin(CS_PORT, CS_PIN, GPIO_PIN_RESET);  // CS 低
    spi_transfer(0x9F);  // 发送读 ID 命令
    uint32_t id  = (uint32_t)spi_transfer(0xFF) << 16;
    id |= (uint32_t)spi_transfer(0xFF) << 8;
    id |=  (uint32_t)spi_transfer(0xFF);
    HAL_GPIO_WritePin(CS_PORT, CS_PIN, GPIO_PIN_SET);   // CS 高
    return id;  // 如 0xEF4017（W25Q64）
}
```

---

## 4. I2C 总线

### 4.1 I2C 基础

```
主机          从机1（地址 0x48）   从机2（地址 0x50）
SDA ─────────────────────────────────────── （漏极开路）
SCL ─────────────────────────────────────── （漏极开路）
```

- 两线：SDA（数据）+ SCL（时钟）
- 多主多从（地址区分从机）
- 标准 100 kHz / 快速 400 kHz / 高速 1 MHz

### 4.2 读写 AHT20 温湿度传感器

```c
#define AHT20_ADDR    0x38
#define AHT20_MEASURE 0xAC

typedef struct {
    float temperature;
    float humidity;
} AHT20_Data;

bool aht20_read(I2C_HandleTypeDef *hi2c, AHT20_Data *out) {
    // 发送测量命令
    uint8_t cmd[3] = {AHT20_MEASURE, 0x33, 0x00};
    if (HAL_I2C_Master_Transmit(hi2c, AHT20_ADDR << 1,
                                 cmd, 3, 10) != HAL_OK)
        return false;

    HAL_Delay(80);  // 等待测量完成

    // 读取 6 字节数据
    uint8_t data[6];
    if (HAL_I2C_Master_Receive(hi2c, AHT20_ADDR << 1,
                                data, 6, 10) != HAL_OK)
        return false;

    if (data[0] & 0x80) return false;  // 忙标志

    // 解析
    uint32_t hum_raw  = ((uint32_t)data[1] << 12) |
                        ((uint32_t)data[2] << 4)  |
                        (data[3] >> 4);
    uint32_t temp_raw = ((uint32_t)(data[3] & 0x0F) << 16) |
                        ((uint32_t)data[4] << 8) |
                        data[5];

    out->humidity    = (float)hum_raw  / 1048576.0f * 100.0f;
    out->temperature = (float)temp_raw / 1048576.0f * 200.0f - 50.0f;
    return true;
}
```

---

## 5. ADC 与 DAC

### 5.1 ADC 基础

```c
// STM32 ADC 12 位，0~3.3V 输入
// 分辨率：3.3V / 4096 ≈ 0.8 mV/LSB

ADC_HandleTypeDef hadc1;

void adc_init(void) {
    hadc1.Instance = ADC1;
    hadc1.Init.Resolution = ADC_RESOLUTION_12B;
    hadc1.Init.ScanConvMode = DISABLE;
    hadc1.Init.ContinuousConvMode = DISABLE;
    HAL_ADC_Init(&hadc1);

    ADC_ChannelConfTypeDef ch = {
        .Channel = ADC_CHANNEL_0,  // PA0
        .Rank    = 1,
        .SamplingTime = ADC_SAMPLETIME_480CYCLES,
    };
    HAL_ADC_ConfigChannel(&hadc1, &ch);
}

float adc_read_voltage(void) {
    HAL_ADC_Start(&hadc1);
    HAL_ADC_PollForConversion(&hadc1, 10);
    uint32_t raw = HAL_ADC_GetValue(&hadc1);
    HAL_ADC_Stop(&hadc1);
    return raw * 3.3f / 4095.0f;
}

// NTC 温度计算
float adc_to_temperature(uint16_t raw) {
    float voltage = raw * 3.3f / 4095.0f;
    float r_ntc = 10000.0f * voltage / (3.3f - voltage);  // 10K 分压
    // Steinhart-Hart 简化公式
    float lnr = logf(r_ntc / 10000.0f);
    return 1.0f / (1.0f/298.15f + lnr/3950.0f) - 273.15f;
}
```

### 5.2 DMA ADC（多通道高速采样）

```c
#define ADC_CHANNELS  4
uint16_t adc_buf[ADC_CHANNELS];

// 配置 DMA 循环模式，自动填充 adc_buf
HAL_ADC_Start_DMA(&hadc1, (uint32_t*)adc_buf, ADC_CHANNELS);
// 之后 adc_buf 始终保持最新值，无需 CPU 干预
```

---

## 6. DMA（直接内存访问）

```c
// DMA 传输：外设 → 内存，不占用 CPU
// 典型用途：UART/SPI/I2C 接收、ADC 采样、内存搬运

// 配置 DMA 搬运内存（memcpy 加速）
DMA_HandleTypeDef hdma;
hdma.Instance = DMA2_Stream0;
hdma.Init = {
    .Channel             = DMA_CHANNEL_0,
    .Direction           = DMA_MEMORY_TO_MEMORY,
    .PeriphInc           = DMA_PINC_ENABLE,
    .MemInc              = DMA_MINC_ENABLE,
    .PeriphDataAlignment = DMA_PDATAALIGN_WORD,
    .MemDataAlignment    = DMA_MDATAALIGN_WORD,
    .Mode                = DMA_NORMAL,
    .Priority            = DMA_PRIORITY_HIGH,
};
HAL_DMA_Init(&hdma);

uint32_t src[256], dst[256];
HAL_DMA_Start_IT(&hdma, (uint32_t)src, (uint32_t)dst, 256);
// DMA 传输完成后触发中断

void DMA2_Stream0_IRQHandler(void) {
    HAL_DMA_IRQHandler(&hdma);
}

void HAL_DMA_XferCpltCallback(DMA_HandleTypeDef *h) {
    // DMA 传输完成
}
```

---

## 7. PWM 与电机控制

### 7.1 PWM 基础

```c
// TIM3 CH1 输出 PWM，控制 LED 亮度或电机速度
TIM_HandleTypeDef htim3;

void pwm_init(uint32_t freq_hz, uint32_t duty_percent) {
    uint32_t period = 168000000 / (1 * freq_hz) - 1;  // 168MHz 时钟

    htim3.Instance = TIM3;
    htim3.Init = {
        .Prescaler = 0,
        .CounterMode = TIM_COUNTERMODE_UP,
        .Period = period,
        .ClockDivision = TIM_CLOCKDIVISION_DIV1,
    };
    HAL_TIM_PWM_Init(&htim3);

    TIM_OC_InitTypeDef oc = {
        .OCMode     = TIM_OCMODE_PWM1,
        .Pulse      = period * duty_percent / 100,
        .OCPolarity = TIM_OCPOLARITY_HIGH,
    };
    HAL_TIM_PWM_ConfigChannel(&htim3, &oc, TIM_CHANNEL_1);
    HAL_TIM_PWM_Start(&htim3, TIM_CHANNEL_1);
}

// 动态修改占空比（0~100）
void pwm_set_duty(uint32_t duty_percent) {
    uint32_t pulse = (htim3.Init.Period + 1) * duty_percent / 100;
    __HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_1, pulse);
}
```

### 7.2 直流电机 PID 控制

```c
typedef struct {
    float kp, ki, kd;
    float integral;
    float prev_error;
    float output_min, output_max;
} PID;

float pid_compute(PID *pid, float setpoint, float measured, float dt) {
    float error    = setpoint - measured;
    pid->integral += error * dt;
    // 积分限幅（防饱和）
    if (pid->integral > 100.0f) pid->integral = 100.0f;
    if (pid->integral < -100.0f) pid->integral = -100.0f;

    float derivative = (error - pid->prev_error) / dt;
    pid->prev_error  = error;

    float output = pid->kp * error +
                   pid->ki * pid->integral +
                   pid->kd * derivative;

    // 输出限幅
    if (output > pid->output_max) output = pid->output_max;
    if (output < pid->output_min) output = pid->output_min;
    return output;
}
```

---

## 8. CAN 总线

```c
// CAN 2.0B，最高 1 Mbit/s
CAN_HandleTypeDef hcan1;

void can_init(void) {
    hcan1.Instance = CAN1;
    hcan1.Init = {
        .Prescaler = 3,       // APB1/3 = 14 MHz
        .Mode      = CAN_MODE_NORMAL,
        .SyncJumpWidth = CAN_SJW_1TQ,
        .TimeSeg1 = CAN_BS1_13TQ,
        .TimeSeg2 = CAN_BS2_2TQ,
        // 波特率 = 14MHz / (1+13+2) = 875 kbps
    };
    HAL_CAN_Init(&hcan1);
    HAL_CAN_Start(&hcan1);
}

// 发送 CAN 帧
bool can_send(uint32_t id, uint8_t *data, uint8_t len) {
    CAN_TxHeaderTypeDef hdr = {
        .StdId = id,
        .IDE   = CAN_ID_STD,
        .RTR   = CAN_RTR_DATA,
        .DLC   = len,
    };
    uint32_t mailbox;
    return HAL_CAN_AddTxMessage(&hcan1, &hdr, data, &mailbox) == HAL_OK;
}
```

---

## 9. Flash 存储编程

```c
// 内部 Flash 读写（存储配置参数）
#include "stm32f4xx_hal_flash.h"

#define CONFIG_FLASH_SECTOR  FLASH_SECTOR_7  // 最后扇区，128KB
#define CONFIG_FLASH_ADDR    0x08060000UL

typedef struct {
    uint32_t magic;       // 0xDEADBEEF，验证有效性
    float    calibration;
    uint32_t crc32;
} Config;

bool flash_write_config(const Config *cfg) {
    HAL_FLASH_Unlock();

    FLASH_EraseInitTypeDef erase = {
        .TypeErase    = FLASH_TYPEERASE_SECTORS,
        .Sector       = CONFIG_FLASH_SECTOR,
        .NbSectors    = 1,
        .VoltageRange = FLASH_VOLTAGE_RANGE_3,
    };
    uint32_t error;
    if (HAL_FLASHEx_Erase(&erase, &error) != HAL_OK) {
        HAL_FLASH_Lock();
        return false;
    }

    uint64_t *p = (uint64_t*)cfg;
    for (size_t i = 0; i < sizeof(Config)/8; i++) {
        HAL_FLASH_Program(FLASH_TYPEPROGRAM_DOUBLEWORD,
                          CONFIG_FLASH_ADDR + i * 8, p[i]);
    }

    HAL_FLASH_Lock();
    return true;
}

const Config *flash_read_config(void) {
    return (const Config*)CONFIG_FLASH_ADDR;
}
```

---

## 10. 实战：多协议传感器驱动

```c
// 统一传感器驱动接口（C 风格多态）
typedef struct SensorDriver {
    const char *name;
    bool (*init)(struct SensorDriver *self);
    bool (*read)(struct SensorDriver *self, float *temp, float *hum);
    void (*deinit)(struct SensorDriver *self);
    void *priv;  // 私有数据（I2C 句柄、配置等）
} SensorDriver;

// AHT20（I2C）实现
static bool aht20_init(SensorDriver *self) {
    I2C_HandleTypeDef *hi2c = self->priv;
    uint8_t init_cmd[] = {0xBE, 0x08, 0x00};
    return HAL_I2C_Master_Transmit(hi2c, 0x70, init_cmd, 3, 10) == HAL_OK;
}

static bool aht20_read_data(SensorDriver *self, float *temp, float *hum) {
    I2C_HandleTypeDef *hi2c = self->priv;
    return aht20_read(hi2c, &(AHT20_Data){*temp, *hum});
}

SensorDriver aht20_drv = {
    .name   = "AHT20",
    .init   = aht20_init,
    .read   = aht20_read_data,
    .deinit = NULL,
    .priv   = &hi2c1,
};

// 使用
void sensor_task(SensorDriver *drv) {
    if (!drv->init(drv)) { printf("Init failed\n"); return; }

    float t, h;
    while (1) {
        if (drv->read(drv, &t, &h)) {
            printf("[%s] T=%.1f°C H=%.1f%%\n", drv->name, t, h);
        }
        HAL_Delay(1000);
    }
}
```

---

**上一章**：[第 06 章：嵌入式系统基础 ←](06_embedded_fundamentals.md)  
**下一章**：[第 08 章：RTOS 与并发编程 →](08_rtos.md)
