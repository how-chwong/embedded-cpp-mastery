# 第 10 章：生产环境落地方案

> **学习目标**：掌握嵌入式项目从开发到量产的完整工程实践，包括 OTA、CI/CD、安全和可靠性设计。

---

## 目录

1. [工程项目结构](#1-工程项目结构)
2. [版本控制与分支策略](#2-版本控制与分支策略)
3. [持续集成 CI/CD](#3-持续集成-cicd)
4. [OTA 固件升级](#4-ota-固件升级)
5. [Bootloader 设计](#5-bootloader-设计)
6. [安全与加密](#6-安全与加密)
7. [可靠性设计](#7-可靠性设计)
8. [产品化配置管理](#8-产品化配置管理)
9. [量产测试与生产线工具](#9-量产测试与生产线工具)
10. [综合案例：IoT 智能传感节点](#10-综合案例iot-智能传感节点)

---

## 1. 工程项目结构

推荐的大型嵌入式项目目录结构：

```
project/
├── CMakeLists.txt
├── toolchain/
│   └── arm-none-eabi.cmake
├── config/
│   ├── FreeRTOSConfig.h
│   └── app_config.h
├── bsp/                    # 板级支持包
│   ├── include/
│   ├── src/
│   └── CMakeLists.txt
├── drivers/                # 外设驱动（可跨项目复用）
│   ├── aht20/
│   ├── w25q/
│   └── CMakeLists.txt
├── middleware/             # 中间件
│   ├── freertos/
│   ├── lwip/
│   ├── mbedtls/
│   └── CMakeLists.txt
├── app/                    # 应用层（业务逻辑）
│   ├── include/
│   ├── src/
│   └── CMakeLists.txt
├── tests/                  # 单元测试
│   ├── host/               # Host 运行的测试
│   └── target/             # 目标板测试
├── scripts/                # 构建、烧录、测试脚本
│   ├── flash.sh
│   └── gen_version.py
├── docs/                   # 文档
└── .github/
    └── workflows/
        └── ci.yml
```

---

## 2. 版本控制与分支策略

### 2.1 Git 分支模型（GitFlow 变体）

```
main          ●────────────────────────● (稳定发布)
               \                      /
release/v1.2   ●────────────────────●  (发布准备)
                \                  /
develop         ●──────────────────●   (集成分支)
               / \                /
feature/ota    ●                 /
feature/mqtt       ●────────────
```

### 2.2 语义化版本

```c
// version.h（由 CI 自动生成）
#define FW_VERSION_MAJOR  1
#define FW_VERSION_MINOR  2
#define FW_VERSION_PATCH  3
#define FW_VERSION_STR    "1.2.3"
#define FW_BUILD_DATE     "2024-01-15"
#define FW_GIT_HASH       "a1b2c3d"
#define FW_VERSION_INT    ((1 << 16) | (2 << 8) | 3)  // 0x010203
```

```python
# scripts/gen_version.py
import subprocess, datetime

git_hash = subprocess.check_output(['git', 'rev-parse', '--short', 'HEAD']).decode().strip()
build_date = datetime.datetime.now().strftime('%Y-%m-%d')

with open('config/version.h', 'w') as f:
    f.write(f'#define FW_GIT_HASH   "{git_hash}"\n')
    f.write(f'#define FW_BUILD_DATE "{build_date}"\n')
```

---

## 3. 持续集成 CI/CD

### 3.1 GitHub Actions 工作流

```yaml
# .github/workflows/ci.yml
name: Firmware CI

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive

      - name: 安装工具链
        run: |
          sudo apt-get update
          sudo apt-get install -y gcc-arm-none-eabi cmake ninja-build

      - name: 生成版本信息
        run: python3 scripts/gen_version.py

      - name: 配置 CMake
        run: |
          cmake -B build -G Ninja \
            -DCMAKE_TOOLCHAIN_FILE=toolchain/arm-none-eabi.cmake \
            -DCMAKE_BUILD_TYPE=Release

      - name: 编译
        run: cmake --build build

      - name: 检查固件大小
        run: |
          arm-none-eabi-size build/firmware.elf
          # 确保 Flash 不超限
          python3 scripts/check_size.py build/firmware.elf 512 192

      - name: 运行 Host 单元测试
        run: |
          cmake -B build_test -DBUILD_TESTS=ON
          cmake --build build_test
          cd build_test && ctest --output-on-failure

      - name: 上传构建产物
        uses: actions/upload-artifact@v4
        with:
          name: firmware
          path: |
            build/firmware.elf
            build/firmware.bin
            build/firmware.hex

  static-analysis:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Cppcheck
        run: |
          sudo apt-get install -y cppcheck
          cppcheck --enable=all --error-exitcode=1 \
                   --suppress=missingIncludeSystem \
                   -I include src/
```

### 3.2 固件大小检查脚本

```python
# scripts/check_size.py
import subprocess, sys

elf_file  = sys.argv[1]
flash_max = int(sys.argv[2]) * 1024  # KB → bytes
ram_max   = int(sys.argv[3]) * 1024

result = subprocess.check_output(['arm-none-eabi-size', elf_file]).decode()
# text + data = Flash; bss + data = RAM
lines = result.strip().split('\n')
parts = lines[1].split()
text, data, bss = int(parts[0]), int(parts[1]), int(parts[2])

flash_used = text + data
ram_used   = data + bss

print(f"Flash: {flash_used:,}/{flash_max:,} bytes ({flash_used/flash_max*100:.1f}%)")
print(f"RAM:   {ram_used:,}/{ram_max:,} bytes ({ram_used/ram_max*100:.1f}%)")

if flash_used > flash_max * 0.90:  # 超过 90% 告警
    print("⚠ Flash 使用超过 90%!")
    sys.exit(1)
```

---

## 4. OTA 固件升级

### 4.1 双 Bank 升级方案

```
Flash 布局（双 Bank OTA）：

0x08000000 ┌──────────────────┐
           │   Bootloader     │  32 KB（只读保护）
0x08008000 ├──────────────────┤
           │   App A（当前）   │  240 KB
0x08046000 ├──────────────────┤
           │   App B（备份）   │  240 KB
0x08084000 ├──────────────────┤
           │   配置/参数区      │  16 KB
0x08088000 └──────────────────┘
```

### 4.2 OTA 流程实现

```c
// ota.h
typedef enum {
    OTA_IDLE,
    OTA_DOWNLOADING,
    OTA_VERIFYING,
    OTA_READY,
    OTA_FAILED,
} OtaState;

typedef struct {
    uint32_t magic;      // 0x4F544155 "OTAU"
    uint32_t version;
    uint32_t size;
    uint32_t crc32;
    uint8_t  sha256[32];
} OtaHeader;

// ota.c
static OtaState ota_state = OTA_IDLE;
static uint32_t ota_write_addr;
static uint32_t ota_bytes_written;

bool ota_begin(const OtaHeader *hdr) {
    if (hdr->magic != 0x4F544155) return false;
    if (hdr->version <= get_current_version()) {
        // 拒绝降级（可选）
    }

    // 擦除 Bank B
    flash_erase_region(OTA_BANK_B_ADDR, hdr->size);
    ota_write_addr  = OTA_BANK_B_ADDR;
    ota_bytes_written = 0;
    ota_state = OTA_DOWNLOADING;
    return true;
}

bool ota_write_chunk(const uint8_t *data, size_t len) {
    if (ota_state != OTA_DOWNLOADING) return false;
    flash_write(ota_write_addr, data, len);
    ota_write_addr    += len;
    ota_bytes_written += len;
    return true;
}

bool ota_finalize(const OtaHeader *hdr) {
    // 验证 CRC32
    uint32_t crc = crc32_calc((uint8_t*)OTA_BANK_B_ADDR, hdr->size);
    if (crc != hdr->crc32) {
        ota_state = OTA_FAILED;
        return false;
    }

    // 写入 OTA 标志，通知 Bootloader 切换
    OtaFlag flag = {.magic = 0xA5A5, .bank = OTA_BANK_B};
    flash_write(OTA_FLAG_ADDR, &flag, sizeof(flag));
    ota_state = OTA_READY;
    return true;
}

void ota_reboot(void) {
    NVIC_SystemReset();
}
```

---

## 5. Bootloader 设计

```c
// bootloader/main.c
#define APP_A_ADDR  0x08008000
#define APP_B_ADDR  0x08046000
#define OTA_FLAG_ADDR 0x08084000

typedef void (*AppEntry)(void);

void boot_jump_to_app(uint32_t app_addr) {
    // 检查 SP 有效性（应在 RAM 范围内）
    uint32_t sp = *(volatile uint32_t*)app_addr;
    if ((sp < 0x20000000) || (sp > 0x20030000)) {
        printf("Invalid SP: 0x%08X\n", sp);
        return;
    }

    // 关闭所有中断
    __disable_irq();

    // 重置所有外设（关闭时钟，清除中断标志）
    HAL_RCC_DeInit();
    for (int i = 0; i < 8; i++) NVIC->ICER[i] = 0xFFFFFFFF;

    // 设置向量表
    SCB->VTOR = app_addr;

    // 设置栈指针和跳转
    __set_MSP(sp);
    uint32_t entry = *(volatile uint32_t*)(app_addr + 4);
    AppEntry jump = (AppEntry)entry;
    jump();
}

int main(void) {
    HAL_Init();
    SystemClock_Config();
    uart_init();

    printf("Bootloader v1.0 - Git: %s\n", FW_GIT_HASH);

    OtaFlag *flag = (OtaFlag*)OTA_FLAG_ADDR;

    if (flag->magic == 0xA5A5 && flag->bank == OTA_BANK_B) {
        printf("OTA: Trying Bank B...\n");

        // 验证 Bank B
        if (verify_firmware(APP_B_ADDR)) {
            // 清除 OTA 标志
            flash_erase_region(OTA_FLAG_ADDR, sizeof(OtaFlag));
            // 拷贝 B → A（或直接跳转 B）
            copy_bank_b_to_a();
            printf("OTA: Success! Booting new firmware\n");
        } else {
            printf("OTA: Verification failed, booting Bank A\n");
        }
    }

    printf("Booting application...\n");
    boot_jump_to_app(APP_A_ADDR);

    // 不应到达这里
    while (1);
}
```

---

## 6. 安全与加密

### 6.1 mbedTLS 集成（AES + SHA256）

```c
#include "mbedtls/aes.h"
#include "mbedtls/sha256.h"

// AES-256-CBC 加密
bool aes256_encrypt(const uint8_t *key, const uint8_t *iv,
                    const uint8_t *in, uint8_t *out, size_t len) {
    mbedtls_aes_context ctx;
    mbedtls_aes_init(&ctx);

    uint8_t iv_copy[16];
    memcpy(iv_copy, iv, 16);

    bool ok = false;
    if (mbedtls_aes_setkey_enc(&ctx, key, 256) == 0) {
        ok = (mbedtls_aes_crypt_cbc(&ctx, MBEDTLS_AES_ENCRYPT,
                                     len, iv_copy, in, out) == 0);
    }

    mbedtls_aes_free(&ctx);
    return ok;
}

// 计算固件 SHA-256 哈希（验证完整性）
void firmware_hash(const uint8_t *fw, size_t len, uint8_t hash[32]) {
    mbedtls_sha256_context ctx;
    mbedtls_sha256_init(&ctx);
    mbedtls_sha256_starts(&ctx, 0);  // 0 = SHA-256（非 224）
    mbedtls_sha256_update(&ctx, fw, len);
    mbedtls_sha256_finish(&ctx, hash);
    mbedtls_sha256_free(&ctx);
}
```

### 6.2 安全启动（Secure Boot）

```
1. Bootloader 中存储厂商公钥（OTP/Flash 写保护区）
2. 固件包含数字签名（ECDSA-256）
3. Bootloader 验证签名后才跳转应用
4. 拒绝未签名/签名无效的固件
```

### 6.3 Flash 读保护

```c
// STM32 RDP（Read Protection）Level 1
// 设置后 JTAG 无法读取 Flash，防止代码提取
// 注意：Level 2 不可逆！

void set_flash_protection(void) {
    FLASH_OBProgramInitTypeDef ob;
    HAL_FLASHEx_OBGetConfig(&ob);

    if (ob.RDPLevel != OB_RDP_LEVEL_1) {
        HAL_FLASH_Unlock();
        HAL_FLASH_OB_Unlock();
        ob.OptionType = OPTIONBYTE_RDP;
        ob.RDPLevel   = OB_RDP_LEVEL_1;
        HAL_FLASHEx_OBProgram(&ob);
        HAL_FLASH_OB_Launch();  // 触发复位生效
    }
}
```

---

## 7. 可靠性设计

### 7.1 看门狗定时器

```c
// 独立看门狗（IWDG）：由独立 LSI 时钟驱动，主时钟故障也能复位
IWDG_HandleTypeDef hiwdg;

void watchdog_init(uint32_t timeout_ms) {
    hiwdg.Instance       = IWDG;
    hiwdg.Init.Prescaler = IWDG_PRESCALER_256;   // 32kHz/256 = 125Hz
    hiwdg.Init.Reload    = timeout_ms * 125 / 1000 - 1;  // 最大 32s
    HAL_IWDG_Init(&hiwdg);
}

void watchdog_feed(void) {
    HAL_IWDG_Refresh(&hiwdg);
}

// 在主任务中定期喂狗
void main_task(void *pv) {
    while (1) {
        do_work();
        watchdog_feed();
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}
```

### 7.2 断言与防御性编程

```c
// 可配置的断言（生产环境记录日志而不崩溃）
#ifdef NDEBUG
#  define ASSERT(cond) \
    do { if (!(cond)) { log_error("ASSERT: %s:%d", __FILE__, __LINE__); } } while(0)
#else
#  define ASSERT(cond) assert(cond)
#endif

// 空指针检查宏
#define RETURN_IF_NULL(ptr, ret) \
    do { if ((ptr) == NULL) { LOG_E("NULL: "#ptr); return ret; } } while(0)

// 参数范围检查
#define CHECK_RANGE(val, lo, hi, ret) \
    do { if ((val)<(lo)||(val)>(hi)) { LOG_E("Range: %d", val); return ret; } } while(0)
```

### 7.3 ECC 内存（错误纠正）

对于关键应用（汽车、航空）：
- 使用带 ECC 的 SRAM（STM32H7）
- 软件 CRC 校验关键数据结构
- 三模冗余（TMR）关键计算

```c
// 关键数据双份存储 + CRC 校验
typedef struct {
    Config data;
    Config data_backup;
    uint32_t crc;
    uint32_t crc_backup;
} SafeConfig;

bool safe_config_read(SafeConfig *sc, Config *out) {
    uint32_t crc1 = crc32(&sc->data, sizeof(Config));
    uint32_t crc2 = crc32(&sc->data_backup, sizeof(Config));

    if (crc1 == sc->crc) {
        *out = sc->data;
        return true;
    }
    if (crc2 == sc->crc_backup) {
        *out = sc->data_backup;
        // 修复主副本
        sc->data = sc->data_backup;
        sc->crc  = sc->crc_backup;
        return true;
    }
    return false;  // 两份都损坏
}
```

---

## 8. 产品化配置管理

```c
// app_config.h - 编译期配置
#define CFG_HW_VERSION    2
#define CFG_UART_BAUD     115200
#define CFG_SENSOR_COUNT  4
#define CFG_OTA_ENABLED   1
#define CFG_LOG_LEVEL     LOG_INFO

// runtime_config.c - 运行时配置（存 Flash）
typedef struct {
    uint32_t magic;       // 0xCONF1234
    uint32_t version;
    char     wifi_ssid[32];
    char     wifi_pass[64];
    char     mqtt_broker[64];
    uint16_t mqtt_port;
    float    temp_offset;
    uint32_t report_interval_s;
    uint32_t crc32;
} RuntimeConfig;

// 提供 REST API 或串口命令修改配置
void config_set_via_serial(void) {
    char cmd[128];
    if (uart_readline(cmd, sizeof(cmd))) {
        if (strncmp(cmd, "SET SSID ", 9) == 0) {
            strncpy(g_config.wifi_ssid, cmd + 9, 31);
            config_save();
        }
        // ... 更多命令
    }
}
```

---

## 9. 量产测试与生产线工具

### 9.1 生产线烧录脚本

```python
#!/usr/bin/env python3
# production_flash.py

import subprocess, sys, json, datetime

FIRMWARE = "firmware_v1.2.3.hex"
OPENOCD  = "openocd"

def flash_device(serial_number):
    print(f"烧录设备 SN:{serial_number}...")

    # 1. 烧录固件
    result = subprocess.run([
        OPENOCD,
        "-f", "interface/stlink.cfg",
        "-f", "target/stm32f4x.cfg",
        "-c", f"program {FIRMWARE} verify reset exit"
    ], capture_output=True, text=True)

    if result.returncode != 0:
        print(f"烧录失败: {result.stderr}")
        return False

    # 2. 烧录序列号
    write_serial_number(serial_number)

    # 3. 功能测试（通过串口）
    if not run_production_test():
        return False

    # 4. 记录测试结果
    log_result(serial_number, "PASS")
    return True

def log_result(sn, status):
    with open("production_log.jsonl", "a") as f:
        json.dump({
            "sn": sn,
            "status": status,
            "firmware": FIRMWARE,
            "timestamp": datetime.datetime.now().isoformat()
        }, f)
        f.write("\n")
```

### 9.2 功能测试套件

```c
// test_suite.c（目标板上运行，生产测试模式）
#define TEST_PASS 0
#define TEST_FAIL 1

typedef struct {
    const char *name;
    int (*fn)(void);
} TestCase;

int test_led(void)     { /* 测试 LED 亮灭 */ return TEST_PASS; }
int test_uart(void)    { /* 回环测试 */ return TEST_PASS; }
int test_adc(void)     { /* 测量参考电压 */ return TEST_PASS; }
int test_flash(void)   { /* 读写 Flash */ return TEST_PASS; }
int test_eeprom(void)  { /* I2C EEPROM */ return TEST_PASS; }

const TestCase tests[] = {
    {"LED",    test_led},
    {"UART",   test_uart},
    {"ADC",    test_adc},
    {"Flash",  test_flash},
    {"EEPROM", test_eeprom},
};

void run_production_tests(void) {
    int pass = 0, fail = 0;
    for (size_t i = 0; i < ARRAY_SIZE(tests); i++) {
        int r = tests[i].fn();
        printf("%-10s: %s\n", tests[i].name, r == TEST_PASS ? "PASS" : "FAIL");
        r == TEST_PASS ? pass++ : fail++;
    }
    printf("结果: %d/%zu PASS\n", pass, ARRAY_SIZE(tests));
    printf(fail == 0 ? "PRODUCTION_PASS\r\n" : "PRODUCTION_FAIL\r\n");
}
```

---

## 10. 综合案例：IoT 智能传感节点

### 系统架构

```
┌─────────────────────────────────────────────────┐
│               STM32F411 @ 100MHz                 │
│                                                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │
│  │ AHT20    │  │ BMP280   │  │  W25Q64      │   │
│  │ 温湿度    │  │ 气压      │  │  Flash存储   │   │
│  └────I2C───┘  └────I2C───┘  └────SPI───────┘   │
│                                                   │
│  ┌──────────────────────────────────────────┐    │
│  │              FreeRTOS                    │    │
│  │  SensorTask  NetworkTask  LogTask        │    │
│  └──────────────────────────────────────────┘   │
│                                                   │
│  ┌──────────────────────────────────────────┐    │
│  │  ESP8266（AT 指令）→ MQTT → 云平台        │    │
│  └───────────UART────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

### 关键代码框架

```c
// main.c
int main(void) {
    bsp_init();
    
    // 读取运行时配置
    RuntimeConfig cfg;
    if (!config_load(&cfg)) config_default(&cfg);
    
    // 创建 FreeRTOS 对象
    xDataQueue    = xQueueCreate(20, sizeof(SensorData));
    xConsoleMutex = xSemaphoreCreateMutex();
    xSystemEvents = xEventGroupCreate();
    
    // 创建任务
    xTaskCreate(task_sensor,   "Sensor",  512, &cfg, 4, NULL);
    xTaskCreate(task_network,  "Network", 1024, &cfg, 3, NULL);
    xTaskCreate(task_logger,   "Logger",  512, NULL, 2, NULL);
    xTaskCreate(task_ota,      "OTA",     1024, &cfg, 2, NULL);
    xTaskCreate(task_watchdog, "WDT",     256, NULL, 5, NULL);  // 最高优先级
    
    vTaskStartScheduler();
    while (1);
}

// 看门狗任务：所有任务汇报心跳
#define WDT_SENSOR_BIT  (1<<0)
#define WDT_NETWORK_BIT (1<<1)
#define WDT_ALL_BITS    (WDT_SENSOR_BIT | WDT_NETWORK_BIT)

EventGroupHandle_t xWdtEvents;

void task_watchdog(void *pv) {
    while (1) {
        EventBits_t bits = xEventGroupWaitBits(
            xWdtEvents, WDT_ALL_BITS, pdTRUE, pdTRUE,
            pdMS_TO_TICKS(5000)  // 5 秒内所有任务必须汇报
        );
        if ((bits & WDT_ALL_BITS) == WDT_ALL_BITS) {
            watchdog_feed();  // 喂硬件看门狗
        } else {
            printf("任务超时！位=0x%lX，系统重启\n", bits);
            NVIC_SystemReset();
        }
    }
}
```

---

**上一章**：[第 09 章：调试、测试与性能优化 ←](09_debug_test_perf.md)  
**附录 A**：[C / C++ / C# 横向对比 →](appendix_a_comparison.md)
