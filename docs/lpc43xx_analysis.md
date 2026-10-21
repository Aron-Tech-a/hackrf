# HackRF 中 LPC43xx ARM Cortex-M4 MCU 代码分析

本文档总结了这个项目中 LPC43xx ARM Cortex-M4 微控制器相关代码的位置、职责，以及它在 HackRF 中承担的功能。

## 1. MCU 在硬件中的位置

从硬件文档和代码来看，HackRF 里的主控制器是 LPC43xx 系列 ARM Cortex-M4 微控制器，通常是：

- HackRF One / Jawbreaker / rad1o 等平台：`LPC43xx ARM Cortex-M4` 作为主 MCU
- HackRF Pro：`ARM MCU + FPGA` 组合，其中 MCU 仍然是系统主控

关键证据：

- `docs/source/hardware_components.rst:53-60`
  - 明确列出：`LPC43xx ARM Cortex-M4 microcontroller`
- `host/libhackrf/src/hackrf.h:134-146`
  - 文档注释中写明：`This is the main processor of the unit. It's a multi-core ARM processor.`

也就是说，这个 MCU 不是“可选功能”，而是 HackRF 整体控制和协调的中心。

## 2. 代码入口位置

核心代码主要在：

- `firmware/hackrf_usb/hackrf_usb.c`
- `firmware/common/*.c`
- `firmware/hackrf_usb/sgpio_m0.s`

其中：

- `hackrf_usb.c`：M4 侧主程序入口，负责设备初始化和主状态机
- `common/*.c`：RF、时钟、GPIO、USB、SGPIO 等通用控制逻辑
- `sgpio_m0.s`：M0 协处理器代码，负责极其时序敏感的 SGPIO 交换

## 3. MCU 的核心职责

### 3.1 USB 主控制与 host 接口

`firmware/hackrf_usb/hackrf_usb.c` 是 MCU 的启动入口，里面有：

- `main(void)`
- `usb_vendor_request(...)`
- `radio_changed(...)`

它负责：

- 初始化 USB 设备
- 启动 M0 核心
- 读取/设置 RF 状态
- 处理来自 host 的 vendor requests
- 在主循环中不断执行 `transceiver_request`，切换 TX/RX/OFF 模式

也就是说，这个文件把主控 MCU 变成了一个完整的 SDR 控制器，USB host 发送命令后，由这段代码解释并驱动硬件。

### 3.2 RF 参数控制：频率、采样率、增益、模式

`firmware/common/radio.c` 是更偏“无线电状态管理”层：

- `radio_init()`
- `radio_reg_write()`
- `radio_update_direction()`
- `set_lo()` / `set_if()` / `set_afe_rate()`

它把配置参数统一存放在 `radio_t` 结构里，并在状态变化时更新：

- `TRANSCEIVER_MODE_OFF`
- `TRANSCEIVER_MODE_RX`
- `TRANSCEIVER_MODE_TX`
- `TRANSCEIVER_MODE_RX_SWEEP`

这层代码用于把 host 设定的参数转成具体硬件动作，例如：

- 频率设置：`RADIO_FREQUENCY_RF`
- 采样率：`RADIO_SAMPLE_RATE`
- 基带带宽：`RADIO_BB_BANDWIDTH_RX/TX`
- LNA / VGA / TX VGA 增益
- bias tee / 天线使能

因此 MCU 负责“控制 RF 链”，不是简单地传递数据。

### 3.3 SGPIO 高速数据通道

`firmware/common/sgpio.c` 负责把 LPC43xx 和 RF 前端 / CPLD / FPGA 之间的高速信号口配好。

主要功能：

- 配置 SGPIO 的 pinmux
- 设置方向：TX / RX
- 连接 ADC/DAC 和 CPLD/FPGA 之间的数据搬运
- 控制 trigger、capture、disable、q-invert 等信号

如代码中注释所示：

- SGPIO0~7：ADC/DAC 数据位
- SGPIO8：Clock Input
- SGPIO9：Capture Input
- SGPIO10：Disable Output
- SGPIO11：Direction Output

这说明 MCU 通过 SGPIO 不是在“普通 GPIO 上发数字”，而是在高速模式下进行 8-bit/16-bit 数据流交换。

### 3.4 M0 的极限时序处理

`firmware/hackrf_usb/sgpio_m0.s` 是 LPC43xx M0 核心实现，注释非常关键。

它解释了：

- M0 用于实现所有 timing-critical 的 SGPIO 使用
- 它负责在 32 bytes chunk 的基础上进行 RX/TX 交换
- 支持 5 个模式：IDLE / WAIT / RX / TX_START / TX_RUN
- 统计 underrun / overrun 和 longest shortfall
- 处理需要严格时序的高速 ADC/DAC 数据流

这段代码说明：

- M4 负责逻辑层和控制层
- M0 负责高实时高速数据搬运层
- MCU 架构中存在“主核 + 协处理器”分工

这是 HackRF 这类 SDR 设备中很典型的实现方式：把最严苛的时序放到专门的小核上。

## 4. MCU 还负责哪些外设控制

除了 RF 控制之外，LPC43xx 还驱动了一批外设：

### 4.1 时钟发生器

- `firmware/common/si5351c.c`
- `firmware/common/clock_gen.c`

控制 Si5351 生成各种时钟，例如：

- 采样时钟
- 参考时钟
- MCU 时钟
- RF 前端时钟

### 4.2 RF 前端器件

- `firmware/common/max283x.*`
- `firmware/common/rffc5071.*`
- `firmware/common/rf_path.*`

这些驱动负责：

- 控制频率合成器
- 选择收发方向
- 控制 LNA / VGA / mixer / filtering

### 4.3 电源和 GPIO

- `firmware/common/power.c`
- `firmware/common/pins.c`
- `firmware/common/platform_scu.*`

负责：

- 打开/关闭 RF 电源
- 使能不同电压 rails
- 配置引脚功能
- 控制 LED、按钮、trigger 等

### 4.4 SPI flash 与固件更新

- `firmware/common/rom_iap.c`
- `w25q80bv.c`
- USB vendor request 相关文件

这些代码支持：

- 从 SPI flash 启动
- DFU 更新
- 读取序列号、固件版本、板卡信息

## 5. HackRF Pro 中的额外角色：加载 FPGA bitstream

HackRF Pro 的 MCU 不只是控制 RF，还负责 FPGA 的右侧加载逻辑。

在 `firmware/hackrf_usb/hackrf_usb.c` 中，可见：

```c
#ifdef IS_PRALINE
if (IS_PRALINE) {
#if defined(DFU_MODE) || defined(RAM_MODE)
    selftest.fpga_image_load = SKIPPED;
    selftest.report.pass = false;
#else
    fpga_image_load(&fpga_loader, 0);
#endif
    delay_us(100);
    fpga_spi_selftest();
    fpga_sgpio_selftest();
}
#endif
```

这说明：

- MCU 负责启动时把 FPGA bitstream 从 flash/存储介质读取并加载到 FPGA
- FPGA 相关的数据链和 DSP 是由 MCU 触发并配置的

而这部分的 bitstream 和运行时控制逻辑，仓库中也明确说明：

- FPGA gateware 位于 `firmware/fpga/`
- 运行时加载在 `firmware/common/fpga_image.c`

因此 MCU 是整个系统控制平面的核心，FPGA 是算力加速层。

## 6. USB API 侧的 MCU 功能

在 `host/libhackrf/src/hackrf.h` 中，host 通过 USB vendor requests 调用 MCU 功能：

- `usb_vendor_request_set_transceiver_mode`
- `usb_vendor_request_set_freq`
- `usb_vendor_request_set_sample_rate_frac`
- `usb_vendor_request_write_max283x`
- `usb_vendor_request_write_si5351c`
- `usb_vendor_request_operacake_set_ports`
- `usb_vendor_request_set_fpga_bitstream`（Praline）
- `usb_vendor_request_read_partid_serialno`
- `usb_vendor_request_get_m0_state`

也就是说，用户侧的操作最终都落在 MCU 固件上，MCU 再转发给底层硬件。

## 7. 一句话总结

LPC43xx ARM Cortex-M4 在这个项目中实现的是：

- HackRF 的主控制器
- USB host 接口处理器
- RF 参数控制器
- SGPIO 数据搬运控制器
- 时钟和外设管理器
- 在 HackRF Pro 中还承担 FPGA bitstream 加载和配置控制

它不是“纯数学计算器”，而是整个 HackRF 设备的控制中枢和实时数据通道协调层。

## 8. 最值得继续读的源码顺序

如果你打算继续深入阅读这个 MCU 代码，建议按以下顺序：

1. `firmware/hackrf_usb/hackrf_usb.c`
2. `firmware/common/radio.c`
3. `firmware/common/sgpio.c`
4. `firmware/hackrf_usb/sgpio_m0.s`
5. `firmware/common/rf_path.c`
6. `firmware/common/clock_gen.c`
7. `firmware/common/si5351c.c`
8. `firmware/common/fpga_image.c`（Praline 额外逻辑）

这样能从“系统入口 -> RF 状态 -> 高速交换 -> 外设驱动 -> FPGA 加载”一路串起来。
