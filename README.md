# STM32F103 + uC/OS-III + AHT20 实验工程

本项目是嵌入式系统实验的完整工程：把 uC/OS-III V3.08.02 移植到 STM32F103C8T6，建立三个周期任务，并在任务三中通过硬件 I2C 采集 AHT20 温湿度，从 USART1 输出。

> 说明：仓库中的代码已经用 Keil MDK 5.36 / ARMCC V5.06 完整重编译，两个 Target 均为 **0 Error(s), 0 Warning(s)**。由于整理环境没有连接实验板，报告中的串口图为“格式示意”，不是实机截图；上板后请按 `Docs/Lab_Report.md` 第 8 章和附录 A 的步骤补拍真实波形与串口截图。

## 1. 功能

- uC/OS-III V3.08.02 移植，1 kHz SysTick，PendSV 上下文切换。
- Task LED1：优先级 5，周期 1 s，翻转 PC13 板载 LED。
- Task LED2：优先级 6，周期 3 s，翻转 PB0 外接 LED。
- Task TASK3：优先级 7，周期 2 s。
  - `RTOS_Hello`：发送 `hello uc/OS! 欢迎来到RTOS多任务环境！`
  - `RTOS_AHT20`：采集 AHT20，发送温度与湿度。
- AHT20 驱动把“触发”和“读取”拆开，80 ms 转换等待用 `OSTimeDly()`，不阻塞 LED 任务。
- 两个 Keil Target 共用源码，通过 `APP_TASK3_MODE` 宏切换任务三行为。

## 2. 硬件连接

| 模块 | 引脚 | 接到 STM32F103C8T6 | 备注 |
|------|------|--------------------|------|
| LED1 | 阴极 / 阳极 | PC13 / 3.3 V（经 330 Ω） | 板载 LED，低电平亮 |
| LED2 | 阴极 / 阳极 | PB0 / 3.3 V（经 330 Ω） | 外接 LED，低电平亮 |
| AHT20 | VCC / GND | 3.3 V / GND | 不要接 5 V |
| AHT20 | SCL / SDA | PB8 / PB9 | I2C1 重映射，400 kHz |
| USB-TTL | RXD / TXD | PA9 / PA10 | 交叉连接 |
| ST-Link | SWDIO / SWCLK / GND / 3.3 V | PA13 / PA14 / GND / 3.3 V | SWD 下载调试 |

所有模块必须共地。AHT20 模块通常自带 4.7 kΩ 上拉；若使用长杜邦线，建议缩短到 20 cm 以内。

## 3. 目录结构

```text
RTOS_AHT20_Lab/
├─ Src/                     CubeMX 生成的 main.c、gpio.c、i2c.c、usart.c、stm32f1xx_it.c
├─ Inc/                     CubeMX 生成的头文件
├─ Drivers/                 STM32F1 HAL 与 CMSIS
├─ Middlewares/
│  ├─ uC-OS3/               uC/OS-III 内核、Cortex-M3 移植层、配置
│  ├─ uC-CPU/               uC/CPU
│  └─ uC-LIB/               uC/LIB
├─ User/
│  ├─ app.c / app.h         三个任务、任务创建、AHT20 上报
│  ├─ app_cfg.h             优先级、周期、栈大小、任务三模式
│  ├─ led.c / led.h         LED 控制
│  ├─ uart_port.c / .h      USART1 轮询发送
│  └─ aht20.c / aht20.h     AHT20 驱动
├─ MDK-ARM/
│  ├─ RTOS_AHT20_Lab.uvprojx   两个 Target 的 Keil 工程
│  ├─ startup_stm32f103xb.s    启动文件（PendSV 向量已指向 OS_CPU_PendSVHandler）
│  ├─ RTOS_AHT20/              编译输出（axf/hex/map）
│  └─ RTOS_Hello/              编译输出（axf/hex/map）
├─ Docs/
│  ├─ Lab_Report.md         实验报告 Markdown 源码
│  ├─ Lab_Report.pdf        实验报告 PDF
│  ├─ AI对话记录.md         与 AI 的完整学习记录
│  ├─ figures/              报告插图
│  └─ tools/                生成插图、Markdown 转 PDF 的脚本
├─ .vscode/settings.json    VSCode UTF-8 与 Markdown 设置
└─ RTOS_AHT20_Lab.ioc       STM32CubeMX 配置文件
```

## 4. 快速开始

### 4.1 用 Keil 编译

1. 打开 `MDK-ARM/RTOS_AHT20_Lab.uvprojx`。
2. 在 Target 下拉框选择：
   - `RTOS_AHT20`：任务三采集 AHT20（最终版本）。
   - `RTOS_Hello`：任务三发送 hello（任务一版本）。
3. 按 F7 编译；输出在 `MDK-ARM/<Target>/`。
4. 用 ST-Link 下载 `.hex` 或 `.axf`，复位运行。

### 4.2 用串口助手观察

- 波特率 115200，8 数据位，1 停止位，无校验，无流控。
- 编码选择 **UTF-8**。
- AHT20 版本每 2 s 输出一行，例如：

```text
[TASK3] AHT20: T= 25.36 C, RH= 48.72 %RH
```

- Hello 版本每 2 s 输出：

```text
hello uc/OS! 欢迎来到RTOS多任务环境！
```

### 4.3 用 STM32CubeMX 查看配置

打开 `RTOS_AHT20_Lab.ioc`，可以查看：

- HSE 8 MHz，PLL ×9，SYSCLK 72 MHz；
- APB1 /2，APB2 /1；
- USART1：PA9/PA10，115200-8-N-1；
- I2C1：PB8/PB9，400 kHz，Fast Mode；
- PC13 / PB0：推挽输出，标签 LED1 / LED2。

> 注意：CubeMX 重新生成代码时，`USER CODE` 段内的代码会保留，但 Keil 工程里手工添加的 uC/OS 文件组可能被重写。重新生成后请检查 `MDK-ARM/RTOS_AHT20_Lab.uvprojx` 的 Groups 和 Include Paths；本项目已经配置好，非必要不要重新生成。

## 5. 移植要点

| 文件 | 作用 |
|------|------|
| `Middlewares/uC-OS3/Cfg/os_cfg.h` | 关闭事件标志、内存、互斥量、队列、信号量、统计任务、软件定时器 |
| `Middlewares/uC-OS3/Cfg/os_cfg_app.h` | 1 kHz 节拍、任务栈、ISR 栈、空闲任务栈 |
| `Middlewares/uC-CPU/Cfg/cpu_cfg.h` | 小端、4 位 NVIC 优先级、内核感知边界 4 |
| `Middlewares/uC-OS3/Ports/ARM-Cortex-M/ARMv7-M/ARM/os_cpu.h` | M3 无 FPU，`OS_CPU_ARM_FP_EN=0` |
| `Src/stm32f1xx_it.c` | SysTick 同时服务 HAL 与 uC/OS-III |
| `MDK-ARM/startup_stm32f103xb.s` | PendSV 向量直接指向 `OS_CPU_PendSVHandler` |
| `User/app.c` | 三个任务与 `OSStart()` |

### 为什么 SysTick 可以共用

HAL 需要 1 ms 时基，uC/OS-III 需要 1 kHz 节拍，两者频率相同。`SysTick_Handler()` 先执行 `HAL_IncTick()`，再在 `OSRunning == OS_TRUE` 时执行 `OS_CPU_SysTickHandler()`。启动阶段 `OSRunning` 为 false，避免内核尚未启动就调用 `OSTimeTick()`。

### 为什么 PendSV 不放在 C 包装函数里

C 函数的序言会改写 `LR`（EXC_RETURN），uC/OS-III 的汇编上下文切换依赖进入异常时的 `LR`。因此本项目直接修改启动文件的向量表：

```asm
IMPORT  OS_CPU_PendSVHandler
DCD     OS_CPU_PendSVHandler
```

这样不依赖编译器是否把 C 调用优化成尾跳转，换优化等级也不会破坏上下文切换。

## 6. 编译结果

| Target | Code | RO-data | RW-data | ZI-data | 结果 |
|--------|------|---------|---------|---------|------|
| RTOS_AHT20 | 20046 B | 894 B | 56 B | 5104 B | 0 Error, 0 Warning |
| RTOS_Hello | 15832 B | 944 B | 56 B | 5104 B | 0 Error, 0 Warning |

Flash 约 21 KB，RAM 约 5.04 KB，均满足 STM32F103C8T6 的 64 KB / 20 KB 限制。

## 7. 文档

- `Docs/Lab_Report.md`：实验报告 Markdown 源码，可直接用 VSCode 预览。
- `Docs/Lab_Report.pdf`：由 Markdown 生成的 PDF，28 页左右。
- `Docs/实验报告.md`、`Docs/实验报告.pdf`：同上的中文文件名副本，便于提交学习通。
- `Docs/AI对话记录.md`：AI 辅助学习记录。
- `Docs/figures/`：系统框图、调度时序、I2C 帧、AHT20 流程、串口格式、编译结果。

重新生成 PDF：

```bat
D:\anaconda3\python.exe Docs\tools\md2tex.py
```

重新生成插图：

```bat
D:\anaconda3\python.exe Docs\tools\gen_figures.py
```

## 8. 说明与限制

- 本工程按 STM32F103C8T6 + HAL + Keil MDK 配置，使用的是 ARMCC 5.06；若改用 AC6，需要重新检查汇编语法与启动文件。
- 串口发送采用轮询，适合每 2 s 一帧的低速率场景；高吞吐量场景应改为 DMA + 信号量。
- AHT20 驱动未做 CRC 校验，仅检查忙标志并换算数据；如现场干扰较大，可增加 CRC8 校验和重试。
- 当前报告中的 LED 时序为理论值，串口图为格式示意；实机测试请用示波器/逻辑分析仪和 XCOM 补测。

## 9. 许可证

- `Middlewares/uC-OS3`、`uC-CPU`、`uC-LIB` 使用 Apache-2.0，详见各目录的 `LICENSE`。
- 本项目其余代码用于课程实验，可自由修改和提交。

## 10. 仓库地址

提交到 GitHub 后，把仓库地址填写在这里：

```text
https://github.com/<your-account>/<your-repo>
```
