# STM32-VCU Porting and Upgrade Guide

本指南详细说明了如何将当前基于 STM32F042 和 GCC/Makefile 构建系统的 CANable 固件，迁移到基于 Keil MDK-ARM 的工程，并支持 STM32F103 与 STM32G431 两种目标器件。文档以 STM32-VCU 应用场景为背景，覆盖底层驱动生成、外设配置、工程结构迁移以及验证策略。

## 1. 迁移概述

| 项目 | 现状 | 目标 |
| --- | --- | --- |
| 编译工具链 | GCC (arm-none-eabi) + Makefile | Keil MDK-ARM (ARMClang)
| 芯片平台 | STM32F042C6 | STM32F103Cx / STM32G431Cx
| 外设 | CAN、USB、LED (GPIO) | CAN、USB、LED (GPIO) — 由 STM32CubeMX 生成 HAL 驱动

迁移工作主要分为三部分：

1. **工具链切换**：在 Keil 中重建工程，移植现有源文件、启动文件与链接脚本配置。
2. **芯片差异适配**：根据目标 MCU 的时钟树、存储器布局以及外设差异调整代码和工程设置。
3. **外设重新配置**：使用 STM32CubeMX 配置 CAN、USB、LED，并导出 Keil 工程底层驱动。

## 2. 先决条件

- Keil MDK-ARM 5.38 或更新版本，并安装了 STM32F1 与 STM32G4 的 Device Family Pack (DFP)。
- STM32CubeMX 6.10 或更新版本，能够支持所选器件与外设中间件。
- 访问本仓库源代码，用于迁移业务逻辑（`inc/`, `src/`, `Drivers/`, `Middlewares/` 等目录）。
- 熟悉 CMSIS、STM32 HAL 库以及 Keil 工程结构。

## 3. 使用 STM32CubeMX 创建 Keil 工程

1. **创建工程**：
   - 打开 STM32CubeMX，创建新工程，选择目标 MCU：
     - `STM32F103CBTx`（或所选 F103 变体）。
     - `STM32G431CBUx`（或所选 G431 变体）。
   - 设置工程名称，例如 `stm32-vcu-f103` 或 `stm32-vcu-g431`，工具链选择 *MDK-ARM (uVision5)*。

2. **时钟配置**：
   - F103：如使用外部 8 MHz 晶振，配置 PLL 得到 72 MHz 系统频率；USB 需确保 48 MHz 时钟，若使用内部 HSI 则启用自动校准。
   - G431：使用 HSE 8 MHz 或内部 HSI，配置 PLL 得到 170 MHz 系统频率，同时配置 PLLQ 输出 48 MHz 给 USB。

3. **外设配置**：
   - **CAN**：
     - F103 使用 `CAN1`，启用普通模式；根据应用选择位速率（示例 1 Mbps），设置 `Prescaler`, `Time Seg1`, `Time Seg2` 和 `SJW`。
     - G431 使用 `FDCAN1`，若仅需经典 CAN 则关闭 FD 功能；同步外设时钟到 APB。
   - **USB 设备**：
     - 选择 `USB Device (FS)`，启用 `CDC` 类或自定义类，与现有固件协议一致。
     - 配置中断优先级，并确认 USB 使用的 GPIO（DM/DP）。
   - **LED/GPIO**：
     - 将指示灯连接的引脚配置为 `GPIO_Output`，设置默认状态和无上拉/下拉。
   - **调试接口**：保留 `SWD` 接口用于下载与调试。

4. **中间件与生成选项**：
   - 在 `Project Manager > Code Generator` 勾选：
     - *Generate peripheral initialization as a pair of .c/.h files per peripheral*。
     - *Keep User Code when re-generating*。
   - 确保勾选 *Generate under root*，以便将 `Drivers/`, `Core/` 等目录置于工程根目录。

5. **生成代码**：点击 `GENERATE CODE`，CubeMX 将生成带有 HAL 库、启动文件和链接脚本的 Keil 工程。

## 4. 整合现有业务逻辑

1. **导入源文件**：
   - 将本仓库的 `inc/` 与 `src/` 中业务逻辑文件复制到 `Core/Inc/` 与 `Core/Src/`，或在 Keil 工程中创建新的组并添加相应文件。
   - 若使用 `Drivers/` 或 `Middlewares/` 中的特定库（如 CAN USB 协议栈），在 Keil 工程中添加为独立分组并保持相对路径。

2. **移植配置宏**：
   - 对照 `inc/config.h` 以及原 Makefile 中的编译宏，在 Keil 工程的 `Options for Target > C/C++` 下添加对应的 `#define`。
   - 如果原工程区分内部/外部振荡器，可通过不同的 Keil target 或使用 `#ifdef` 在 `stm32_vcu_board.h` 中管理。

3. **启动文件和内存映射**：
   - CubeMX 默认提供适配目标芯片的启动文件与 scatter file，通常无需修改。
   - 若需要兼容原有 Bootloader（例如保留前 8 KB Flash），请在 `Options for Target > Linker > Use Memory Layout from Target Dialog` 取消勾选，并提供自定义 scatter file，将起始地址与大小调整为 Bootloader 预留后的地址区间。

4. **中断服务函数**：
   - 将原有 `stm32f0xx_it.c` 中的中断处理迁移到 CubeMX 生成的 `stm32f1xx_it.c` 或 `stm32g4xx_it.c`，保持函数签名一致。
   - 注意 FDCAN 与 CAN 的中断名称不同，例如 G431 的接收中断为 `FDCAN1_IT0_IRQHandler`，需对应修改。

5. **USB 堆栈适配**：
   - 若沿用 CDC 类，实现应移植到 `usbd_cdc_if.c` 的 `CDC_Receive_FS` 与发送接口。
   - 将原有串口协议解析逻辑（如 `usb_device.c`、`protocol.c`）接入 CubeMX 生成的回调。

## 5. 针对 STM32F103 的特殊注意事项

- **电源与时钟**：USB 需要 48 MHz 时钟，若使用外部晶振，启用 `PLLCLK` 分频；若使用 HSI，需启用自动校准并保证偏差 < 0.25%。
- **Flash 布局**：常见的 `STM32F103CB` 具有 128 KB Flash、20 KB RAM；根据固件大小设置合适的 `IROM1` 区间。
- **CAN 滤波器**：F1 系列的 CAN 滤波配置与 F0 略有不同，使用 `HAL_CAN_ConfigFilter()` 时需设置 `FilterScale = CAN_FILTERSCALE_32BIT`，并在初始化完成后调用 `HAL_CAN_Start()`。
- **USB 供电**：若 VCU 板使用自供电模式，需要在 `usb_pwr.c` 中确保 `VBUS` 检测引脚配置正确，或在 CubeMX 中禁用 VBUS 检测。

## 6. 针对 STM32G431 的特殊注意事项

- **FDCAN 差异**：G431 采用 FDCAN IP，CubeMX 会生成 `fdcan.c`；若仅需经典 CAN，设置 `FDCAN_FRAME_FORMAT_CLASSIC` 并禁用位速率切换。
- **Flash/RAM**：`STM32G431CB` 典型配置为 128 KB Flash、32 KB SRAM，确认链接脚本中 `RAM` 区域满足 USB 缓冲需求。
- **时钟保护**：G4 系列对 PLL 时钟要求更严格，建议启用 `HAL_RCCEx_PeriphCLKConfig()` 中的 FDCAN 与 USB 时钟；若使用内部时钟，需启用 `HSI48` 并通过 CRS 校准。
- **VDD 范围**：FDCAN 需 VDD ≥ 2.0 V，确保硬件供电满足要求并在 `SystemClock_Config()` 中启用对应电源调节。

## 7. 工程配置建议

- **目录结构**：
  ```
  stm32-vcu-f103/
  ├── Core/
  │   ├── Inc/
  │   └── Src/
  ├── Drivers/
  ├── Middlewares/
  ├── CANable/          ← 移植的协议与业务逻辑
  └── stm32-vcu.uvprojx
  ```
  将公共代码抽象为独立文件夹，便于在不同目标间复用。

- **多目标管理**：
  - 在 Keil 中使用 *Manage Project Items* 创建 `STM32F103` 与 `STM32G431` 两个 target。
  - 针对不同芯片设置独立的编译宏（例如 `TARGET_STM32F103`, `TARGET_STM32G431`）。
  - 在代码中使用条件编译区分差异化实现，例如 USB 时钟、CAN 初始化参数等。

- **优化与警告**：
  - 将编译优化设置为 `-O2` 或 `-Oz`（Keil 中的 `Level 2` / `Optimize for Size`）。
  - 打开所有警告，并将关键警告设为错误，以提前发现移植问题。

## 8. 测试与验证流程

1. **功能测试**：
   - 通过 USB CDC 向固件发送 `O`, `S8`, `T...` 等命令，验证协议兼容性。
   - 使用示波器或 CAN 分析仪检查 CAN 发送/接收。
   - 检查 LED 指示逻辑（上电自检、错误闪烁等）。

2. **回归验证**：
   - 使用 CubeMX 的 `Project > Generate Report` 导出配置清单，纳入版本管理。
   - 对比 GCC 版本与 Keil 版本的 CAN 时序，确认波形一致。
   - 在 Keil 中启用 `Event Recorder` 或 SWV 实时查看运行状态。

3. **生产烧录**：
   - 若保留 Bootloader，验证 `DFU` 或自定义协议是否兼容新固件。
   - 使用 `STM32CubeProgrammer` 测试直接下载 `.hex` 文件。

## 9. 版本管理与交付建议

- 将 CubeMX 生成的 `.ioc` 文件与 Keil 工程 (`.uvprojx`, `.uvoptx`) 纳入版本控制。
- 在仓库中新建分支（例如 `feature/keil-port`）存放 Keil 工程与 CubeMX 配置，保留原 GCC 工程以便回溯。
- 编写 README 或 `docs/CHANGELOG.md` 记录不同目标 MCU 的差异与测试覆盖情况。
- 建议在 CI 中引入 `uv2ol` 或 `uvproj2cmake` 等工具，或使用 Keil 命令行 `UV4.exe -b` 进行自动化构建。

## 10. 常见问题与排查

| 问题 | 可能原因 | 解决方案 |
| --- | --- | --- |
| USB 无法枚举 | 时钟未正确配置为 48 MHz | 检查 `RCC` 设置，确保启用 `PLLQ` 或 `HSI48` |
| CAN 无数据 | 滤波器屏蔽、波特率不匹配 | 调整 `HAL_CAN_ConfigFilter()`，确认波特率与总线一致 |
| 工程无法编译 | 头文件路径或宏缺失 | 在 Keil `Options for Target > C/C++` 中补全 Include 路径与预定义 |
| Bootloader 跳转失败 | 启动地址或栈指针错误 | 更新 scatter file，确保向量表偏移与 `VTOR` 设置正确 |

## 11. 后续工作

- 梳理代码中的 `TODO` 与 `FIXME`，在 Keil 工程中统一处理。
- 若未来引入 STM32G4 的 FDCAN FD 功能，需扩展协议栈以支持高数据速率。
- 将自动化测试脚本适配 Keil 产出的固件镜像，例如利用 `pyOCD` 或 `STLinkCLI` 进行批量烧录。

---

通过以上步骤，即可将 CANable 固件迁移到 Keil 环境，并针对 STM32F103 与 STM32G431 完成 STM32-VCU 的固件升级。建议在迁移过程中持续维护详细的配置说明与版本记录，以确保团队成员能够快速复现与迭代。
