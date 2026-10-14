# HackRF FPGA 源码分析

本文档总结了这个仓库中 FPGA 相关源码的位置、实现方式以及它和固件/host 侧的关系，方便快速理解 HackRF Pro 的 FPGA gateware 架构。

## 1. FPGA 代码实际所在位置

从仓库结构来看，FPGA 相关源码不在传统的 `src/` 下，而是在：

- `firmware/fpga/`
- `firmware/common/`（与 FPGA 相关的固件控制代码）
- `docs/source/gateware.rst`（说明文档）

关键文档说明：

- `docs/source/gateware.rst:7-14` 明确说明：
  - HackRF Pro 的 FPGA 是 iCE40UP5K
  - gateware 使用 Amaranth HDL 编写
  - 源码位于 `firmware/fpga/`

另外，`firmware/fpga/build.py` 会生成 FPGA bitstream，并打包成 `praline_fpga.bin`。这说明 FPGA 逻辑并非直接写在 C 里，而是在 Python + Amaranth 硬件描述语言中描述的。

## 2. 代码结构总览

### 2.1 顶层平台定义

- `firmware/fpga/board.py`

这个文件定义了 `PralinePlatform`，它继承自 `LatticeICE40Platform`，并声明使用的 FPGA 芯片是：

- `iCE40UP5K`
- package `SG48`

这里也定义了所有物理引脚映射，例如：

- `afe_clk`：ADC 时钟
- `fpga_clk`：FPGA 时钟
- `host_clk` / `host_data`：SGPIO 接口
- `spi`：SPI 端口
- `direction`, `disable`, `capture_en`：控制信号

这部分等于“把 FPGA 内部逻辑和板上实际硬件管脚绑定起来”。

### 2.2 顶层 gateware 入口

- `firmware/fpga/top/standard.py`
- `firmware/fpga/top/half_precision.py`
- `firmware/fpga/top/ext_precision_rx.py`
- `firmware/fpga/top/ext_precision_tx.py`

这些文件是不同 gateware 版本的主入口，分别对应：

- standard：标准模式
- half_precision：半精度模式
- ext_precision_rx：扩展精度 RX
- ext_precision_tx：扩展精度 TX

其中 `standard.py` 是最典型例子。它的 `Top` 类中会创建：

- `ClockDomainGenerator()`
- `MAX586xInterface(...)`
- `SGPIOInterface(...)`
- `SPIRegisterInterface(...)`
- 一组 DSP 模块（DC block、FIR、CIC、NCO 等）

这些模块通过 `connect()` 连接，构成完整的 RX/TX 数据处理链。

### 2.3 接口层

- `firmware/fpga/interface/max586x.py`
- `firmware/fpga/interface/sgpio.py`
- `firmware/fpga/interface/spi.py`

这几个模块负责桥接到外部接口：

#### `max586x.py`
- 连接 MAX586x ADC/DAC
- 使用 DDR input/output buffer 采样
- 处理 Q 通道极性反转等板级问题
- 把 ADC/DAC 的模拟接口转换成 FPGA 内部的 stream

#### `sgpio.py`
- 负责 MCU/Host 与 FPGA 之间的 SGPIO 通信
- 通行“同步/异步”时钟和数据字节流
- 处理 trigger、capture、PRBS 自检等功能
- 将 FPGA 内部采样流以字节流形式发给主处理器

#### `spi.py`
- 负责 FPGA 对外的 SPI 寄存器控制接口
- 支持可编地址寄存器读写
- 例如在 `standard.py` 中通过 `spi_regs.add_register(...)` 添加控制寄存器
- 这些寄存器控制：
  - RX DC blocker 开关
  - decimation ratio
  - quarter-shift
  - TX interpolation ratio
  - NCO phase / enable
  - trigger enable / PRBS enable

### 2.4 DSP 模块

- `firmware/fpga/dsp/`

这里是 FPGA 的“数字信号处理核心”，主要包括：

- `cic.py`：CIC 插值/抽取器
- `fir.py`：FIR 滤波器
- `dc_block.py`：直流偏置消除
- `quarter_shift.py`：fs/4 频移
- `nco.py`：数控振荡器（NCO）
- `mixer.py`：混频器
- `round.py`：舍入逻辑

这说明 HackRF 的 FPGA 并不是简单 io 翻译器，而是把大量数字信号处理能力前移到 FPGA 上了。

## 3. 标准 gateware 的设计思路

`standard.py` 是最值得阅读的入口。它构成了标准模式的数据链：

### RX 路径

```text
ADC -> DC Block -> Quarter Shift -> Half-band FIR decimator chain -> SGPIO output
```

代码中可以看到：

- `rx_chain` 中先接 `DCBlock`
- 然后 `QuarterShift`
- 然后多个 `HalfBandDecimator`
- 最后通过 `ClockConverter` 转到同步域，并送到 MCU 接口

### TX 路径

```text
SGPIO input -> ClockConverter -> Half-band FIR interpolator chain -> CIC interpolator -> DAC
```

处理逻辑中：

- 输入来自 MCU/host 的样本流
- 经过一组 FIR 插值器和 CIC 插值器
- 可配置 TX interpolation factor
- 最后送到 DAC

### 寄存器控制

`standard.py` 中定义了寄存器：

- `ctrl`，`rx_decim`，`rx_pstep`
- `tx_ctrl`，`tx_intrp`，`tx_pstep`

这些寄存器会通过 SPI 写入，并控制：

- 是否启用 trigger
- 是否启用 PRBS
- RX DC blocker 开关
- quarter shift 方向与幅度
- decimation ratio
- interpolation ratio
- TX NCO enable 与 phase step

也就是说：FPGA 的行为不是烧录后固定死的，而是通过寄存器动态配置的。

## 4. FPGA bitstream 是如何生成的

`firmware/fpga/build.py` 负责整个 bitstream 编译打包流程：

```python
fpga_images = {
    "0_standard": standard(),
    "1_halfprec": half_precision(),
    "2_extprec_rx": ext_precision_rx(),
    "3_extprec_tx": ext_precision_tx(),
}

for name, image in fpga_images.items():
    PralinePlatform().build(image, name=name)
```

这表示：

1. 先用 Amaranth 描述各个 top 模块
2. 通过 `PralinePlatform().build(...)` 生成比特流文件
3. 之后再压缩并写出到 `build/praline_fpga.bin`

压缩处理采用了 LZ4 block 的方法，文件头还有 bitstream 数量与偏移表，便于在 MCU 上按索引加载不同版本。

## 5. 固件是如何把 bitstream 加载到 FPGA 的

真正的“烧录/加载 FPGA”代码在 `firmware/common`。

### 5.1 `fpga_image.c`

- `firmware/common/fpga_image.c`

这个文件负责：

- 读取压缩后的 bitstream
- 解析偏移表和块长度
- LZ4 解压
- 调用 `ice40_spi_syscfg_program()` 完成 programming

核心代码：

- `fpga_image_load(...)`
- `fpga_image_read_block_cb(...)`

它会从 flash/loader 读取压缩块，并重复地提供给 FPGA programmer。

### 5.2 `ice40_spi.c`

- `firmware/common/ice40_spi.c`

这里是 FPGA SPI 编程接口的底层实现：

- 配置 SSP/SPI 端口
- 对 iCE40 进行 config mode
- 读写 FPGA 寄存器

### 5.3 `fpga.c`

- `firmware/common/fpga.c`

这是 FPGA 的“设备驱动层”，提供函数如：

- `fpga_set_trigger_enable()`
- `fpga_set_rx_dc_block_enable()`
- `fpga_set_rx_decimation_ratio()`
- `fpga_set_tx_interpolation_ratio()`
- `fpga_set_tx_nco_enable()`

本质上，它把 FPGA 寄存器看成配置接口，供固件和上层逻辑调用。

## 6. host 侧如何控制 FPGA

- `host/libhackrf/src/hackrf.c`

这里定义了：

- `HACKRF_VENDOR_REQUEST_FPGA_WRITE_REG`
- `HACKRF_VENDOR_REQUEST_FPGA_READ_REG`
- `HACKRF_VENDOR_REQUEST_SET_FPGA_BITSTREAM`

这说明 host 侧可通过 USB vendor requests：

- 读取 FPGA 寄存器
- 写入 FPGA 寄存器
- 切换 FPGA bitstream

所以 FPGA 在这个项目里是“可热切换、可配置”的硬件逻辑模块，而不是一次性固定逻辑。

## 7. 总结：这套 FPGA 实现的本质

HackRF 项目的 FPGA 设计可以概括成以下结构：

1. 硬件平台定义：`firmware/fpga/board.py`
2. gateware 顶层设计：`firmware/fpga/top/*.py`
3. 接口实现：`firmware/fpga/interface/*.py`
4. DSP 处理：`firmware/fpga/dsp/*.py`
5. bitstream 打包：`firmware/fpga/build.py`
6. 运行时 FPGA 编程：`firmware/common/fpga_image.c` + `ice40_spi.c`
7. host/FW 配置接口：`host/libhackrf/src/hackrf.c`

它的核心思想是：

- 把频谱处理、滤波、插值/抽取、NCO 等数字处理移到 FPGA 上
- 让 MCU 只负责控制和管理
- 通过 SPI / SGPIO 与 host/firmware 通信
- 用多种 bitstream 版本适配不同带宽/精度要求

## 8. 适合阅读的源码顺序

如果你想按顺序理解，建议先读：

1. `docs/source/gateware.rst`
2. `firmware/fpga/board.py`
3. `firmware/fpga/top/standard.py`
4. `firmware/fpga/interface/sgpio.py`
5. `firmware/fpga/interface/max586x.py`
6. `firmware/fpga/dsp/cic.py`
7. `firmware/fpga/build.py`
8. `firmware/common/fpga_image.c`
9. `firmware/common/fpga.c`

这样能从“硬件定义 -> 数据链 -> bitstream -> 运行时控制”一路串起来。

## 9. 一句话结论

这个项目里的 FPGA 代码核心位于 `firmware/fpga/`，采用 Amaranth HDL 实现 gateware；固件侧代码在 `firmware/common/` 负责把压缩 Bitstream 通过 SPI 下载到 iCE40 FPGA，并通过寄存器控制 FPGA 的接收/发送链和 DSP 参数。它是一个“硬件加速型 SDR 数字信号链”设计。
