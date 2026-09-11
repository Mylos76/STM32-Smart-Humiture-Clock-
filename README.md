# 🕒 STM32-Smart-EnvTerminal
> **基于 STM32 与 FreeRTOS 的多功能智能环境感知终端（电子闹钟） | A Multifunctional Smart Environment Sensing & Alarm Terminal based on STM32F103 & FreeRTOS**

[![Platform](https://img.shields.io/badge/Platform-STM32F103C8T6-blue.svg)](https://www.st.com/)
[![RTOS](https://img.shields.io/badge/RTOS-FreeRTOS%20v10-green.svg)](https://www.freertos.org/)
[![Language](https://img.shields.io/badge/Language-C-orange.svg)](https://en.wikipedia.org/wiki/C_(programming_language))
[![Peripherals](https://img.shields.io/badge/Drivers-1--Wire%20%7C%20SPI%20%7C%20PWM%20%7C%20FSM-purple.svg)]()
[![Display](https://img.shields.io/badge/Display-Frame%20Buffer%20%7C%20TC5020EJ-yellow.svg)]()
[![License](https://img.shields.io/badge/License-MIT-brightgreen.svg)](LICENSE)

---

##  项目简介 (Overview)

本项目是一款基于 **STM32F103C8T6** 微控制器、搭载 **FreeRTOS** 实时操作系统的多传感器融合与智能交互桌面终端（多功能电子温湿度闹钟）。

系统深度融合了环境感知、高精度硬件时钟、点阵图形分层渲染、数字化音频语音播报以及声控/触摸人机交互机制。针对传统单片机裸机架构下外设繁杂、高频点阵扫屏与微秒级通信时序冲突的痛点，设计了基于 **FreeRTOS 的分级抢占式任务调度架构**，并在驱动层引入**显存映射（Frame Buffer）机制**与**嵌套状态机（FSM）**，实现了高内聚低耦合的系统级设计。

---

##  核心特性与技术亮点 (Key Highlights)

-  **FreeRTOS 实时多任务架构与内核对接**：
  - 划分按键扫描、点阵驱动、环境采集等多优先级独立任务，实现毫秒级交互响应（< 20ms）。
  - 在 `stm32f1xx_it.c` 中成功解决 **SysTick 双时基合并**（兼容 HAL 库与 FreeRTOS 调度），接管 `PendSV` 与 `SVC`。
  - 确立**单写入者、多读取者（Single-Writer, Multiple-Readers）**模式，杜绝变量并发修改冲突。
-  **显存映射 (Frame Buffer) 与时分扫屏技术**：
  - 针对 TC5020EJ 级联芯片控制 40 余个 LED 的硬件引脚复用限制，设计时分秒交替扫屏算法，利用**人眼视觉暂留性**（5ms 亮 / 5ms 灭循环）消除重影与闪烁。
  - 内存开辟 Frame Buffer 虚拟显存，业务层与物理移位驱动彻底解耦。
-  **精确微秒级单总线（1-Wire）时序解析**：
  - 深度解析 DHT11 单总线协议，精准捕获 85μs 低电平 + 87μs 高电平应答时序，通过数据上升沿延时采样判别逻辑“0”与“1”。
  - 加入**临界区保护（Critical Section）**防止内核调度破坏通信波形，并设计超时退出机制防止总线挂死。
-  **软硬件闭环音频控制系统 (NV020D)**：
  - 编写单线串行总线协议（占空比 1:3 与 3:1 编码），引入**二值信号量**互斥保护串口资源。
  - 实时检测物理 `PA3 (BUSY)` 引脚实现状态闭环，杜绝指令突发覆盖导致的芯片死机。
-  **低功耗智能化交互系统**：
  - 驻极体咪头声控触发外部中断（EXTI）快速唤醒背光，延时 10 秒自动休眠。
  - HK2301 电平翻转边沿检测，通过动态调节定时器 **PWM 占空比** 实现三级无极循环平滑调光。

---

##  软件任务架构与数据流 (Software Architecture)

```text
  +-------------------------------------------------------------------------------+
  |                                FreeRTOS 内核调度                               |
  +-------------------------------------------------------------------------------+
         | (Preemptive Scheduling / 抢占式调度)
         +------------------------+------------------------+
         | (High: Priority 3)     | (Med: Priority 2)      | (Low: Priority 1)
         v                        v                        v
  +--------------+         +--------------+         +--------------+
  |  Input Task  |         |   GUI Task   |         |  Sensor Task |
  | (按键/触摸)   |         | (点阵屏扫屏) |         | (温湿度/时钟)|
  +-------+------+         +-------+------+         +-------+------+
          |                        ^                        |
    (Queue/Event)                  | (Read-Only)      (Single-Writer)
          |                        |                        |
          v                        |                        v
  +--------------------------------+--------------------------------+
  |              全局数据段 / Frame Buffer 虚拟显存                 |
  +----------------------------------------------------------------+
```

### FreeRTOS 任务优先级规划表
| 任务名称 | 优先级级别 | 对应 API 优先级 | 职责描述与设计要求 |
| :--- | :--- | :--- | :--- |
| **`Input Task`** | **高优先级** | `configMAX_PRIORITIES - 1` | 实体按键检测、消抖、长短按判断与触摸检测，响应延迟 < 20ms |
| **`GUI Task`** | **中优先级** | `configMAX_PRIORITIES - 2` | TC5020EJ 显存数据串行移位刷新，固定 50Hz 扫屏，保证视觉连续性 |
| **`Sensor Task`** | **低优先级** | `tskIDLE_PRIORITY + 1` | 读取 DHT11 与 DS1302Z，环境数据变化慢，5s 采集一次，避免抢占 CPU |

---

##  硬件清单与引脚映射 (BOM & Pinout)

### 1. 核心物料清单 (BOM)
| 器件类型 | 芯片/器件型号 | 通信/驱动接口 | 核心作用与职责 |
| :--- | :--- | :--- | :--- |
| **主控芯片** | STM32F103C8T6 | ARM Cortex-M3 | 72MHz 主频，运行系统任务与外设驱动调度 |
| **实时时钟** | DS1302Z | 3线串行接口 (RST/IO/CLK) | 独立高精度硬件计时，断电时间保持与闰年补偿 |
| **温湿度传感器**| DHT11 | 单总线 (1-Wire) | 环境温度与湿度高精度数字式直读采集 |
| **LED 驱动芯片**| TC5020EJ (双片级联) | GPIO 模拟串行移位 | 控制 40 余个 LED 灯珠的分区时间/温湿度渲染 |
| **数字化音频** | NV020D | 单线/双线 串行协议 | 预录高保真音乐播放、闹钟响铃，支持 32 级音量调节 |
| **触摸检测** | HK2301 | 数字电平捕获 | 触摸按键检测，触发背光 PWM 调光 |
| **声控检测** | 驻极体咪头模块 | EXTI 外部中断 | 检测拍手/说话声响，低电平触发屏幕开启 |

### 2. MCU 物理引脚定义表
| 模块接口 | MCU 引脚 | 模式配置 | 硬件功能说明 |
| :--- | :--- | :--- | :--- |
| **时间设置** | `PB2` | `GPIO_Mode_IPU` | `time_set` 设置按键，按下接地低电平触发 |
| **数值调节** | `PB0` / `PB1` | `GPIO_Mode_IPU` | `up` (加) / `down` (减) 按键 |
| **闹钟控制** | `PA6` / `PA7` / `PA8` | `GPIO_Mode_IPU` | `alarm_set` (闹钟设置) / `alarm_en` (使能) / `alarm_5` (工作日) |
| **背光控制** | `PB10` / `PB11` | `GPIO_Mode_IPU` | `LED_on` 背光开关 / `light` 手动模式切换 |
| **PWM 输出** | `P13` (TIMx_CH) | `GPIO_Mode_AF_PP` | 输出可调占空比 PWM 驱动背光三级亮度 |
| **音频总线** | `PA3` / `PA4` / `PA5` | 输入浮空 / 推挽输出 | `PA3 (BUSY)` 监测，`PA4 (SCK)`，`PA5 (SDA)` 串行总线 |
| **温湿度引脚** | `DATA` (外设特定) | 推挽开漏双向切换 | DHT11 1-Wire 双向单总线通信 |
| **声控中断** | `MIC_IN` (EXTI) | 外部下降沿中断 | 咪头检测到声音输出低电平，触发 MCU 唤醒 |

---



---

