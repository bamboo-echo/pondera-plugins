---
name: stm32-init-review
description: 审查 STM32（HAL/LL）初始化代码：时钟树与外设时钟使能、GPIO 模式选择、NVIC 优先级分组与抢占、DMA 配置、中断回调与返回值检查。当用户要 review CubeMX 生成代码、排查外设不工作或中断互相打断时使用。
runas: inline
---

# STM32 外设初始化审查

## 何时使用

- CubeMX 生成代码改动后，外设行为异常。
- 串口/ADC/SPI 偶发丢数据，怀疑中断优先级或 DMA 配置。
- 需要在上板前做一次初始化静态审查。

不负责 PCB 与时序电气问题；KiCad 网表检查用 `kicad-netlist-check`。

## 执行步骤

1. 找齐初始化链路：`SystemClock_Config` → `MX_GPIO_Init` → `MX_<外设>_Init` → `HAL_<外设>_MspInit`（时钟与 NVIC 使能通常在这里）。
2. 逐项核对时钟：HSE/HSI 源、PLL 倍频分频、总线频率（HCLK/APB1/APB2）、外设时钟使能（`__HAL_RCC_USART1_CLK_ENABLE()`）。
3. 核对 GPIO：模式（推挽/开漏/复用）、上下拉、速度、复用号是否与外设引脚匹配。
4. 核对中断：`HAL_NVIC_SetPriorityGrouping`、每个 IRQ 的抢占/子优先级、是否与 RTOS 的 `configMAX_SYSCALL_INTERRUPT_PRIORITY` 冲突。
5. 核对 DMA：通道与请求映射、循环/单次模式、缓冲区大小与对齐、`HAL_DMA_Start_IT` 返回值。
6. 核对错误处理：每个 `HAL_*_Init` 返回值是否检查；中断回调里是否调用了阻塞 API。
7. 输出问题清单与最小修复。

## 检查清单

- `HAL_Init()` 之前调用外设初始化 → 依赖的 tick 未就绪。
- `SystemClock_Config` 里 PLL 参数与实际主频不符，但注释写的是目标值。
- 外设时钟未使能就配置寄存器 → 写入无效，读回全 0。
- GPIO 复用模式写成 `GPIO_MODE_OUTPUT_PP` 却用于 USART TX → 无输出。
- `GPIO_InitStruct.Pin` 用 `GPIO_PIN_0` 但结构体只 Init 一次多引脚 → 只有最后一个生效（需按引脚分组）。
- NVIC 优先级数值越大优先级越低，混用「数值大=高优先级」的直觉 → 抢占顺序错。
- 中断回调里调用 `HAL_Delay` / 阻塞式 `HAL_UART_Transmit` → 死等或看门狗复位。
- `HAL_UART_Receive_IT` 未重新挂起，只收到第一帧。
- DMA 缓冲区在栈上定义（局部数组）→ 函数返回后数据被覆盖。
- `HAL_ADC_Start_DMA` 前未 `HAL_ADCEx_Calibration_Start` → 读数偏移。
- 未处理 `HAL_ERROR` / `HAL_BUSY` / `HAL_TIMEOUT` 分支。
- 共享变量在中断与主循环间读写未加 `volatile` 或临界区。

## 代码级判定（示例）

```c
/* 反例：结构体只初始化一次，多引脚只有最后一个生效 */
GPIO_InitStruct.Pin = GPIO_PIN_0 | GPIO_PIN_1;
HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

/* 正例：按引脚/模式分组，或逐个调用 */
GPIO_InitStruct.Pin = GPIO_PIN_0;
HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
GPIO_InitStruct.Pin = GPIO_PIN_1;
HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

/* 中断回调里禁止阻塞；只置标志，主循环处理 */
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart) {
    g_rx_flag = 1;                 /* volatile uint8_t */
    HAL_UART_Receive_IT(huart, g_buf, sizeof(g_buf));  /* 重新挂起 */
}
```

## 判定规则

- 时钟配置与目标主频不一致 → 阻断（所有时序外设都会偏）。
- 中断优先级与 RTOS 冲突 → 阻断（可能 HardFault）。
- DMA 缓冲区位于栈 → 阻断。
- 只收到首帧 / 只发首帧 → 优先级次于上，但必须修。
- 返回值未检查、`volatile` 缺失 → 警告。
- 无法静态确定的（时序裕量、电气匹配）→ 标「需示波器/逻辑分析仪实测」。

## 输出格式

```markdown
## STM32 初始化审查

**芯片/系列**：<...>　**工具链**：<CubeMX 版本 / HAL 版本>
**目标主频**：<...>　**实际配置**：<...>

### 阻断
| # | 位置 | 问题 | 后果 | 修复 |

### 警告
| # | 位置 | 问题 | 修复 |

### 需实测
- [ ] <...>
```

## 纪律

- 不修改用户工程；只给定位与最小修复。
- 不臆测芯片型号与封装；不知道就问。
- 静态无法确认的时序问题必须标「需实测」，不写成结论。
