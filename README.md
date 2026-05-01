# yc
## 默认行为（每次对话自动遵守，无需用户重复）

1. 收到任务后，先查下方”模块索引”定位目标文件（不超过3秒）
2. 给用户”最小阅读列表”（≤5 个文件），确认后再开始改动
3. 不扫全仓、不读硬件映射全表、不读无关模块。除非用户明确要求

---

**STM32F407VET6 裸机车控**（CubeMX+GCC ARM+CMake）。核心三层：
1) **USART6 命令驱动**（文本协议）
2) **任务调度器+轻量状态机**（`period_ms` 控频）
3) **循迹 PID 执行层**（LINE/DUAL/CIRCLE）

核心入口 5 文件：`Core/Src/main.c` `Core/Src/stm32f4xx_it.c` `soft_user/src/app_scheduler.c` `soft_user/src/app_command.c` `soft_user/src/bsp_uart6_comm.c`

---

## 模块索引（按需读；每个.c 默认配套同名.h）

| 模块 | 主文件 | 关键符号/词 | 额外需读 |
|---|---|---|---|
| 启动/主循环 | `Core/Src/main.c` | `MX_*_Init`, `AppCommand_Poll`, `AppScheduler_Run` | — |
| 中断入口 | `Core/Src/stm32f4xx_it.c` | `HAL_UART_RxCpltCallback`, `HAL_TIM_PeriodElapsedCallback` | — |
| USART6通信 | `soft_user/src/bsp_uart6_comm.c` | ring buffer, `BspUart6Comm_ReadLine` | `Core/Src/usart.c` |
| 命令解析 | `soft_user/src/app_command.c` | `START/STOP/SET/STAT`, `TASK=`, `PERIOD=` | `app_task.h`, `app_scheduler.h` |
| 调度/状态机 | `soft_user/src/app_scheduler.c` | `AppScheduler_Start/Run/SetPeriod` | `app_task.h` |
| 任务元数据 | `soft_user/src/app_task.c` | `AppTask_Name`, `AppTask_StateName` | — |
| 电机执行 | `HARDWARE/Src/motor.c` | `Single_Motor`, `stop` | `Core/Src/tim.c` |
| PID执行 | `soft_user/src/pid.c` | `PID_Output`, `dual_PID_Output`, `circle_dual_PID_Output` | `huidu.h` |
| 旧路径脚本 | `soft_user/src/route.c` | `Route_RunActions`, `Route_RunTasks` | `motor.c`, `pid.c` |
| USART外设 | `Core/Src/usart.c` | `MX_USART6_UART_Init`, `HAL_UART_MspInit` | — |
| TIM外设 | `Core/Src/tim.c` | `MX_TIM6_Init`, `HAL_TIM_Base_Start_IT` | — |
| 构建入口 | `CMakeLists.txt` | `target_sources`, `add_subdirectory` | `cmake/.../CMakeLists.txt` |

---

## 任务边界（必须遵守）

- 主循环禁止阻塞（`HAL_Delay`、长 `while`）
- 中断只做轻量：收字节、置标志、tick++；重逻辑放主循环
- 新任务接入 `AppScheduler`，禁用 `zhixian()` 阻塞模式
- 无要求不新增抽象层、不大规模重命名、不扫全仓

---

## 关键运行时约束

- MCU: STM32F407VET6, SYSCLK 168MHz
- 电机 PWM: TIM3/TIM12 | 命令链路: USART6(PC6/PC7, 115200 8N1)
- 调度 tick: TIM6 1ms | 默认任务周期: 20ms(可调5~1000ms)

---

## USART6 协议

格式：`[mode,speed,time]\r\n`
- mode: `L`(LINE) / `D`(DUAL) / `C`(CIRCLE)
- speed: `0~65535`, time: `0~65535`ms

回包：`OK [a,b,c] TASK=...` | `ERR BAD_FRAME/BAD_MODE/START_FAIL`
兼容：`PING` → `OK PONG`

---

## TIM6 诊断（5 条快速检查）

1. `Core/Src/tim.c` — `MX_TIM6_Init()` 存在
2. `Core/Inc/tim.h` — 声明 `MX_TIM6_Init()` + `htim6`
3. `Core/Src/main.c` — 调用了 `MX_TIM6_Init()`
4. `main.c` — `HAL_TIM_Base_Start_IT(&htim6)` 在 init 之后
5. `stm32f4xx_it.c` — `if (htim->Instance == TIM6) AppScheduler_Tick1ms();`

5 条全满足 = 链路完整，无需改架构。

---

## CubeMX 再生成后回归检查

- `usart.c`: USART6 NVIC 是否启用
- `stm32f4xx_it.c`: `USART6_IRQHandler`、RxCplt 分支、TIM6 回调
- `main.c`: `AppScheduler_Init/BspUart6Comm_Init/AppCommand_Init` + 主循环轮询
- `CMakeLists.txt`: `app_*.c` 与 `bsp_uart6_comm.c` 是否在构建列表

---

## 硬件映射

仅在硬件改动时读全表；软件重构/命令逻辑/调度策略无需读。
