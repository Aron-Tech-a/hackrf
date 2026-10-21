# FPGA / CPLD 固件存储位置：MCU Flash 还是外部 Flash？

结论先说：

- 这个项目里，MCU 运行代码本身是存放在外部 SPI Flash 中的，不是 MCU 内部 Flash。
- FPGA 的 bitstream（HackRF Pro / Praline 用到的 iCE40 配置镜像）也不是单独放在 MCU 内部 Flash，而是被打包进 firmware image，并存放在同一颗外部 SPI Flash 中。
- CPLD 的 bitstream（老 HackRF One/旧版设计）更像是“配置映像”，通常通过 JTAG / 自启动编程逻辑来加载；在新版固件中，MCU 也会在启动时自动将 CPLD bitstream 载入——但这个 bitstream 也不是 MCU 内部 Flash，而是嵌入 firmware image 之中，最终由 MCU 读取并加载。

一句话：

- MCU 的 firmware：放在外部 SPI Flash
- FPGA bitstream：也放在外部 SPI Flash（并作为 firmware image 的一部分）
- CPLD bitstream：老平台要通过 JTAG/配置逻辑加载，现代固件中通常也是由 MCU 从 firmware image 中取出并加载，不是 MCU 内部 Flash

## 1. 关键证据：MCU 默认从 SPI flash 启动

官方文档明确写了：

> "HackRF devices ship with firmware on the SPI flash memory."

参考：

- `docs/source/updating_firmware.rst:7`
- `docs/source/updating_firmware.rst:49-50`

并且这里还有一句非常关键：

> "By default, HackRF boots from SPI flash memory (SPIFI)."

这说明 LPC43xx 的启动代码并不是从 MCU 自己的内部 flash 直接启动，而是从外部 SPI flash 启动。也就是说，MCU 自己的程序镜像是存放在外部 SPI flash 中的。

## 2. FPGA bitstream 也在外部 SPI flash 里

关键代码在：

- `firmware/hackrf_usb/hackrf_usb.c`
- `firmware/common/fpga_image.c`
- `firmware/hackrf-common.cmake`
- `firmware/fpga/build.py`

### 2.1 代码中直接把 FPGA bitstream 当作 firmware 的一部分读取

`firmware/hackrf_usb/hackrf_usb.c` 中有：

```c
extern uint32_t _binary_fpga_bin_start;

void fpga_loader_setup(void)
{
	w25q80bv_setup(&spi_flash);
}

void fpga_loader_read(uint32_t addr, uint32_t size, uint8_t* buf)
{
	w25q80bv_read(&spi_flash, addr, size, buf);
}

struct fpga_loader_t fpga_loader = {
	.start_addr = (uint32_t) &_binary_fpga_bin_start,
	.setup = fpga_loader_setup,
	.read = fpga_loader_read,
	.in_buffer = lz4_in_buf,
	.out_buffer = lz4_out_buf,
};
```

这里的意思非常明确：

- `fpga_loader` 的数据源不是 MCU 内部存储器，而是 `w25q80bv_read(&spi_flash, ...)`
- 也就是说，FPGA 的 bitstream 是从外部 SPI flash 读取出来的
- `start_addr = (uint32_t) &_binary_fpga_bin_start` 表示这个 bitstream 被链接进了 firmware image 里，最终仍然作为 firmware 的一部分存放在 SPI flash 里

### 2.2 FPGA image 的读取逻辑

`firmware/common/fpga_image.c` 中：

```c
bool fpga_image_load(struct fpga_loader_t* loader, unsigned int index)
{
	if (loader->setup != NULL)
		loader->setup();

	uint32_t addr = loader->start_addr;
	uint32_t num_bitstreams, bitstream_offset;
	loader->read(addr, 4, (uint8_t*) &num_bitstreams);
	if (index >= num_bitstreams)
		return false;
	loader->read(addr + 4 * (index + 1), 4, (uint8_t*) &bitstream_offset);

	ice40_spi_target_init(&ice40);
	struct fpga_image_read_ctx fpga_image_ctx = {
		.loader = loader,
		.addr = loader->start_addr + bitstream_offset,
	};
	const bool success = ice40_spi_syscfg_program(
		&ice40,
		loader->out_buffer,
		fpga_image_read_block_cb,
		&fpga_image_ctx);
```

这段代码说明：

- MCU 先从 `loader->start_addr` 读取 bitstream 表头
- 再从指定偏移中拿到 FPGA 配置数据
- 最后用 `ice40_spi_syscfg_program()` 通过 SPI 发送给 FPGA

这里的 `loader->read` 具体就是 `w25q80bv_read()`，也就是从 SPI flash 读取。

### 2.3 生成时也会打包成 bin 文件

`firmware/hackrf-common.cmake` 里有：

```cmake
SET(PATH_PRALINE_FPGA_BIN ${PATH_HACKRF_FIRMWARE}/fpga/build/praline_fpga.bin)
```

而 `firmware/hackrf_usb/CMakeLists.txt` 中又会把它复制到 firmware bundle 里：

```cmake
COMMAND ${CMAKE_COMMAND} -E copy ${PATH_PRALINE_FPGA_BIN} "fpga.bin"
```

说明 FPGA bitstream 是作为工程产物被打包进固件文件中的，而不是存放在 MCU 内部 flash。

## 3. 这和 CPLD 是什么关系？

这里要分两种情况看：

### 3.1 旧版 HackRF One / CPLD 时代

`host/libhackrf/src/hackrf.h` 里对 CPLD 的说明很明确：

> "The Spartan II CPLD inside the HackRF One devices could also be reconfigured in the past, but in newer firmwares, the ARM MCU automatically reconfigures it on startup with a bitstream baked into the firmware image..."

也就是说：

- 过去 CPLD 可以单独重配置
- 但新版 firmware 中，MCU 会在启动时自动重新加载 CPLD bitstream
- 这个 bitstream 是"baked into the firmware image"，也就是固件镜像中的一部分

这说明即使是 CPLD，也不在 MCU 内部 flash；它的配置信息还是通过 firmware image / 外部 SPI flash 作为来源。

### 3.2 代码里确实有 JTAG 载入 CPLD 的逻辑

`firmware/common/cpld_jtag.c` 中：

```c
bool cpld_jtag_sram_load(jtag_t* const jtag)
{
	cpld_jtag_take(jtag);
	cpld_xc2c64a_jtag_sram_write(jtag, &cpld_hackrf_program_sram);
	const bool success = cpld_xc2c64a_jtag_sram_verify(
		jtag,
		&cpld_hackrf_program_sram,
		&cpld_hackrf_verify);
	cpld_jtag_release(jtag);
	return success;
}
```

这个函数是通过 JTAG 把 CPLD bitstream 写进 CPLD 的 SRAM/配置逻辑。

但它并不等于说“CPLD bitstream 存在 MCU 内部 flash”。

它更准确的说法是：

- CPLD 的配置数据可能来自固件镜像、外部 flash，或者在老版设计中通过工具单独 JTAG 程序化
- 总体上仍然不是 MCU 自身内部 Flash 的存储模型

## 4. 所以最终结论

### 4.1 MCU 自己的 firmware

它存放在外部 SPI flash（SPIFI）中，MCU 在启动时从那儿执行代码。官方文档明确写了：

- `docs/source/updating_firmware.rst:7`
- `docs/source/updating_firmware.rst:49-50`

### 4.2 FPGA bitstream

它也不是存放在 MCU 内部 Flash，而是：

- 由 `firmware/fpga/build.py` 生成
- 作为 fpga.bin / `praline_fpga.bin` 被打包进 firmware
- 在运行时由 MCU 通过 `w25q80bv_read()` 从外部 SPI flash 读取
- 再通过 `ice40_spi_syscfg_program()` 编程到 iCE40 FPGA

### 4.3 CPLD bitstream

- 老平台：通过 JTAG / CPLD 配置逻辑写入
- 新版 firmware：MCU 在启动时自动加载，且这个 bitstream 是固件镜像中的一部分
- 无论哪种情况，都不是 MCU 的内部 flash

## 5. 一张最简图

```text
[Host PC]
   |
   | update firmware / flash.bin
   v
[External SPI Flash on HackRF board]
   |-- MCU firmware image (LPC43xx code)
   |-- FPGA bitstream image (Praline/iCE40)
   |-- optional CPLD bitstream payload (legacy / auto-load)
   v
[LPC43xx MCU]
   | boot from SPIFI
   | reads FPGA/CPLD image from external SPI flash
   +--> configures FPGA via SPI
   +--> configures CPLD via JTAG / startup auto-load
```

## 6. 最终一句话

“FPGA/CPLD 的配置固件，不在 MCU 内部 flash，而是在板载外部 SPI flash 里；MCU 启动时从这里读取并下载到 FPGA/CPLD。”

因此，最准确的理解是：

- MCU：运行代码来自外部 SPI flash
- FPGA/CPLD：配置数据也来自外部 SPI flash（或通过固件镜像中嵌入的 bitstream 载入），不是 MCU 内部存储器
