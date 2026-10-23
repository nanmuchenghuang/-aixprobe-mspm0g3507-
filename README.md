# 使用 AixProbe V821 调试 TI MSPM0G3507 最小系统板

本文给出一套可执行的准备和连接流程，目标是用 AixProbe V821 通过 SWD 调试或烧录一块**没有板载 XDS110 的 MSPM0G3507 最小系统板**。

> 适用范围：AixProbe V821 + MSPM0G3507 最小系统板。LP-MSPM0G3507 LaunchPad 的 J101/J103 处理只放在文末附录，不属于本文主流程。
>
> 验证状态：本文根据 AixProbe V821 资料、原理图和 AixProbe 随附的 OpenOCD 源码整理；OpenOCD 目标配置和器件表已核对，尚未在实物上完成烧录验证。首次使用时请先按文末检查项逐项确认。

> 本次实物调试的故障定位与恢复记录见：[问题.md](问题.md)。

## 0. 结论和需要准备的东西

AixProbe V821 的调试接口芯片是 CH347F，AixProbe 随附的 OpenOCD 源码包含：

- `ch347` 适配器驱动；
- `target/ti/mspm0.cfg`；
- MSPM0 系列 Flash 驱动；
- MSPM0G3507 器件 ID 条目。

因此软件方向是可行的，但不能直接套用 AixProbe 资料包里的 `mspm0-launchpad.cfg`，因为那个文件面向板载 XDS110，并默认使用 10 MHz。

准备清单：

- [ ] AixProbe V821 已能正常启动，UI 可以连接；
- [ ] AixProbe 的 I/O 电压选择为 3.3 V；
- [ ] 一根 AixProbe CN1 2x13 转最小系统板 SWD 排针的转接线；
- [ ] 目标板独立供电，且与 AixProbe 共地；
- [ ] 目标板 SWD 接口至少包含 SWDIO、SWCLK、GND，最好同时引出 NRST；
- [ ] AixProbe OpenOCD 使用自定义 CH347F 配置；
- [ ] 一份带符号和 DWARF 信息的 `.elf` 或 TI `.out`；
- [ ] 一套 `arm-none-eabi-gdb/readelf/nm/objdump/addr2line` 工具；
- [ ] MSPM0G3507 的 SVD 文件，或至少准备好 SDK 寄存器头文件；
- [ ] MSPM0 的 SWD 没有启用密码保护，或已经有办法解锁。

## 1. 硬件准备

### 1.1 设置 AixProbe 的 I/O 电平

MSPM0G3507 使用 3.3 V I/O，不是 5 V 容忍器件。

按 AixProbe V821 原理图，板上存在 3V3、2V8、1V8 电平选择。首次使用请把 AixProbe 的 I/O 电压设置为 **3.3 V**，不要设置为 1.8 V 或 2.8 V。

接线前先确认：

- [ ] AixProbe 和目标板都断电；
- [ ] AixProbe 电平选择为 3.3 V；
- [ ] 目标板 3.3 V 供电正常；
- [ ] AixProbe 与目标板已经共地。

### 1.2 CN1 到最小系统板 SWD 的接线

AixProbe V821 的 CN1 关键引脚：

| CN1 引脚 | 信号 | 用途 |
|---|---|---|
| 1 | `+5V` | 默认不要接目标板 |
| 2 | `JTAG_TMS / SWDIO` | SWD 数据 |
| 3 | `CH347F_VIO` | 电平参考，首次不要直接接目标板 |
| 4 | `JTAG_TCK / SWCLK` | SWD 时钟 |
| 5 | `GND` | 地 |
| 10 | `NRESET` | 目标复位 |
| 21 | `3V3` | 板载 3.3 V，默认不要接目标板 |

最小系统板通常只把 SWD 信号引到排针，没有板载调试器。请先找到板子上的丝印，再按信号名接线，不要按排针位置猜：

| AixProbe CN1 | 最小系统板 SWD 排针 |
|---|---|
| CN1-2 `SWDIO` | `SWDIO` |
| CN1-4 `SWCLK` | `SWCLK` |
| CN1-5 `GND` | `GND` |
| CN1-10 `NRESET` | `NRST` / `RESET`，如果有 |

如果板子的 SWD 排针是常见的 5 针布局，通常还会有一个 `3V3` 引脚。这个 3.3 V 用作目标板供电或电压参考，**不要把它和 AixProbe 的 CN1-1 `+5V`、CN1-3 `CH347F_VIO` 或 CN1-21 `3V3` 直接并联**。目标板请独立供电，只连接 GND、SWDIO、SWCLK 和 NRESET。

如果最小系统板只有 4 针 SWD 排针（GND、SWDIO、SWCLK、3V3）而没有 `NRST`：

- 基础连接和下载有时仍然可以工作；
- `mspm0_mass_erase`、`mspm0_factory_reset` 和锁定恢复可能失败；
- 需要手动按复位键，或者把 MCU 的 nRST 引脚单独引出来。

重要警告：

- **不要把 CN1-1 `+5V` 接到最小系统板的 3.3 V 引脚。**
- 不要把 CN1-3 `CH347F_VIO` 或 CN1-21 `3V3` 与目标板的 3.3 V 电源直接并联。两套电源同时驱动可能造成倒灌。
- 如果 UI 提示缺少目标电压参考，再按 AixProbe 的具体说明处理 `VTREF/VIO`，不要凭猜测接电源。
- SWD 线尽量短，建议不超过 10 到 15 cm。
- 先连接地线，再连接信号线；不要在带电状态下热插拔。

### 1.3 没有板载 XDS110 时不需要 J101/J103

最小系统板没有板载 XDS110，因此不需要处理 LaunchPad 的 J101 隔离跳线，也没有两个调试器争用 SWD 的问题。只要最小系统板本身有正常的 3.3 V 供电，并按上面的信号名接好 SWD 即可。

如果以后换成 LP-MSPM0G3507 LaunchPad，再参考文末附录处理 J101/J103；不要把 LaunchPad 的步骤套到最小系统板上。

### 1.4 电源和复位

- MSPM0G3507 目标板建议独立供电。
- AixProbe 由自己的 USB-C 或电源供电。
- AixProbe 和目标板必须共地。
- 建议连接 `NRESET`。普通下载有时可以不接，但 mass erase、factory reset 和锁定芯片恢复通常需要目标 nRST。
- 如果目标程序很快关闭 SWD 引脚或进入低功耗，使用“连接时复位”的方式：保持 nRST 为低，启动连接，再释放复位。

### 1.5 串口是可选项

如果还需要串口日志，AixProbe CN1 提供调试 UART：

| CN1 引脚 | 信号 |
|---|---|
| 20 | `DEBUG_UART0_TX` |
| 22 | `DEBUG_UART0_RX` |

使用 3.3 V TTL 电平，并将目标 TX 接 AixProbe RX、目标 RX 接 AixProbe TX，同时共地。不要把 RS-232 电平直接接入目标板。

## 2. AixProbe 软件准备

### 2.1 启动和连接 AixProbe

1. 启动 AixProbe-UI。
2. 确认 PC 能通过 USB 网络、局域网或 SSH 访问 AixProbe。
3. 确认 AixProbe 的 OpenOCD 服务可以启动。
4. 确认 OpenOCD 能识别 CH347F。

AixProbe 的 OpenOCD 通常运行在 AixProbe 设备上，PC 侧一般不需要再安装 CH347 驱动。PC 主要负责 UI、SSH 和 GDB 客户端。

### 2.2 使用 CH347F 的 OpenOCD 配置

不要直接使用资料包中的 `tcl/board/ti/mspm0-launchpad.cfg`。该文件默认：

```tcl
source [find interface/xds110.cfg]
adapter speed 10000
```

这两个设置面向 XDS110，不适用于 AixProbe 的 CH347F。

使用仓库中的配置文件：

[configs/aixprobe-mspm0g3507.cfg](configs/aixprobe-mspm0g3507.cfg)

核心内容：

```tcl
adapter driver ch347
transport select swd
adapter speed 500
source [find target/ti/mspm0.cfg]
```

如果机器上连接了多个 CH347 设备，再加入：

```tcl
ch347 vid_pid 0x1a86 0x55de
```

如果 AixProbe UI 要求填写 OpenOCD 命令文件，可以将该配置作为一个独立文件加载；如果它要求填写 `/etc/ailink/openocd_cmd`，请先备份原文件，再按 UI 支持的格式合并。

### 2.3 AixProbe 固件的注意事项

资料包中包含 AixProbe 自身固件和系统镜像，不要把它们当作 MSPM0G3507 的应用固件。

- `板子固件/AixProbe_V821.img` 是 AixProbe 自身固件。
- `update-aixprobe-2026042512.img` 是 AixProbe 的系统更新镜像。
- MSPM0G3507 的应用固件应通过 AixProbe 的 OpenOCD/调试流程烧录。

如果 AixProbe 当前固件没有 MSPM0 目标配置或 CH347F 驱动，再按厂商说明更新系统。设备已经正常工作时不要为了“换配置”而刷写整机系统镜像。

## 3. MSPM0G3507 工程准备

### 3.1 建议先使用最小工程

第一次连接建议使用 TI SDK 的 `empty` 或 GPIO 点灯工程，不要直接用复杂电机控制或 RTOS 工程。这样可以先确认：

- SWD 能连接；
- CPU 能 halt；
- Flash 能识别；
- 程序能下载并复位运行。

确认链路正常后，再切换到正式工程。

### 3.2 生成可用于 AI 调试的构建产物

至少准备：

- [ ] `.elf` 或 TI `.out`；
- [ ] `.map`；
- [ ] 源代码目录；
- [ ] `ti_msp_dl_config.c` 和 `ti_msp_dl_config.h`；
- [ ] 项目 `.syscfg`；
- [ ] 链接脚本 `mspm0g3507.cmd` 或 `mspm0g3507.lds`；
- [ ] 编译时使用的 SDK 和工具链版本。

编译要求：

- 使用 `-g` 生成 DWARF 调试信息；
- 不要使用 `-s` 或 strip 掉符号；
- 保留函数名、全局变量和源码行号；
- 如果使用优化，建议先用 `-O0` 或 `-Og`，确认稳定后再改。

只有 `.hex` 或 `.bin` 可以做烧录，但不足以让 AI 按函数名设断点或把崩溃地址映射回源码行。

### 3.3 准备 GNU Arm 工具链

AixProbe 的 ELF 工作流依赖以下工具：

- `arm-none-eabi-gdb`
- `arm-none-eabi-readelf`
- `arm-none-eabi-nm`
- `arm-none-eabi-objdump`
- `arm-none-eabi-addr2line`

安装 GNU Arm Embedded Toolchain，或把已有的工具链 `bin` 目录加入 PATH。

AixProbe-UI 的配置示例中有一个 GDB 路径占位值，通常不是当前机器的真实路径。请将它改成实际存在的位置，例如：

```text
@gdb_path:C:\Tools\arm-gnu-toolchain\bin\arm-none-eabi-gdb.exe
@gdb_port:3333
```

验证命令：

```powershell
arm-none-eabi-gdb --version
arm-none-eabi-readelf --version
arm-none-eabi-addr2line --version
```

如果使用 TI Arm Clang 生成 ELF，也可以调试，但 AI 端的符号工具路径需要相应调整；使用 GNU Arm 工具链可以避免前缀不一致的问题。

## 4. SVD 和 AI 芯片识别准备

### 4.1 放置 MSPM0G3507 SVD

AixProbe 资料包没有附带 `MSPM0G3507.svd`，而 AixProbe 的通用芯片参考表也没有完整的 TI MSPM0 条目。为了让 AI 正确解析 GPIO、UART、ADC、SysTick 等外设寄存器，建议准备：

```text
aixprobe/svd/ti/MSPM0G3507.svd
```

SVD 可以从 TI 的 MSPM0 CMSIS Pack、SysConfig 安装目录或 Keil 器件包中获取。若暂时没有 SVD，可以把 SDK 中的 `hw_*.h` 寄存器头文件提供给 AI，但寄存器解释的准确性和便利性会下降。

### 4.2 首次连接时明确告诉 AI

AixProbe 连接后应得到：

```text
arch: cortex_m
target_name: mspm0x.cpu
```

建议在首次提示中写明：

```text
目标芯片是 TI MSPM0G3507，Cortex-M0+，只支持 SWD。
OpenOCD target_name 是 mspm0x.cpu，使用 target/ti/mspm0.cfg。
不要套用 STM32、GD32 或通用 cortex_m 配置。
请优先使用 MSPM0G3507.svd 解析外设寄存器。
```

如果 AI 只返回 CPUID 或显示未知 Cortex-M0+，不要让它自行猜测芯片型号；直接给出目标芯片和 SVD 路径。

## 5. 第一次连接流程

### 5.1 断电接线

1. 断开 AixProbe 和目标板电源。
2. 连接 CN1-5 到目标 GND。
3. 连接 CN1-2 到目标 `SWDIO`。
4. 连接 CN1-4 到目标 `SWCLK`。
5. 连接 CN1-10 到目标 `NRST` / `RESET`，如果板子引出了该信号。
6. 检查没有把 +5 V 接到目标板。

### 5.2 上电并检查电压

1. 目标板独立上电。
2. 测量目标 3.3 V 和 GND。
3. 确认 AixProbe I/O 选择为 3.3 V。
4. 最小系统板没有板载 XDS110，不需要处理 J101 隔离跳线。

### 5.3 启动 OpenOCD 并连接

1. 启动 AixProbe 的 OpenOCD，加载 `configs/aixprobe-mspm0g3507.cfg`。
2. 在 AI 调试端执行 `connect()`。
3. 确认返回 `target_name: mspm0x.cpu`。
4. 执行 `halt()`。
5. 读取 `0xE000ED00`，确认 CPUID 为 Cortex-M0+。
6. 读取 MSPM0 器件 ID/用户 ID，确认不是其他器件。
7. 执行 Flash 探测，确认 MAIN Flash 被识别。

建议先读取以下地址：

```text
0xE000ED00  CPUID
0x41C40004  MSPM0 器件 ID/工厂信息
```

具体 ID 解码应以 MSPM0G3507 数据手册和 OpenOCD 日志为准。

### 5.4 烧录最小工程

连接和 Flash 探测成功后，先烧录一个简单工程：

```text
program firmware.elf verify reset
```

如果 UI 使用图形化烧录，确认：

- 烧录文件是 ELF 或正确的 HEX/BIN；
- BIN 的加载地址是 MSPM0 MAIN Flash 的 `0x00000000`；
- 不勾选 NONMAIN 擦除；
- 不使用 factory reset 作为普通烧录选项。

## 6. Flash 和恢复操作

MSPM0 的 OpenOCD 目标配置通常提供三个 Flash bank：

| Bank | 区域 | 用途 |
|---|---|---|
| 0 | MAIN | 用户应用 |
| 1 | NONMAIN | 启动、BSL、SWD 安全策略 |
| 2 | DATA | 可选数据 Flash |

普通应用烧录只操作 MAIN。NONMAIN 只在明确知道自己在改什么时操作。

- `mspm0_mass_erase`：只擦 MAIN，通常用于恢复应用。
- `mspm0_factory_reset`：擦 MAIN 并把 NONMAIN 恢复默认，属于恢复操作，不是普通的“全片擦除”。
- Factory Reset 过程中如果 NonMain 配置损坏，设备可能无法启动，严重时不可恢复。

## 7. 已知限制和排查

### 7.1 SWD 密码保护

AixProbe 随附的 MSPM0 target 配置注明：当前不支持 SWD 密码认证。

如果芯片在 NONMAIN 中启用了 128 位 SWD 密码：

- AixProbe 不能直接输入密码解锁；
- 需要 TI CCS、TI Factory Reset GUI 或其他支持密码认证的工具；
- 也可能需要更换未加密芯片或恢复到默认 NONMAIN。

开发阶段建议保持默认配置：SWD 使能、无密码。

### 7.2 CH347F 的 SWD 速度

CH347 系列在 SWD AP 读写和长时间操作上存在已知时序问题。建议：

- 初始速度设为 500 kHz；
- 连接稳定后再尝试 1 MHz；
- 不要直接使用 XDS110 配置中的 10 MHz；
- 出现 `parity error`、`DAP sticky error`、`SECAP timeout` 时，降到 250 kHz 或 100 kHz；
- 缩短 SWD 线，确认共地，并保留 AixProbe 板上的串联电阻。

AixProbe V821 的 SWD 信号已有串联电阻设计，但仍应避免长线和松动杜邦线。

### 7.3 常见故障

| 现象 | 优先检查 |
|---|---|
| 找不到 CH347F | AixProbe 固件、USB/网络连接、OpenOCD 日志 |
| 连接后立即断开 | SWD 线过长、共地不良、速度过高 |
| DAP parity/sticky error | 降到 500 kHz 以下，检查 SWDIO/SWCLK |
| 能 halt 但不能擦 Flash | 目标是否运行在 MAIN、Flash 保护、目标电压 |
| SECAP timeout | nRST 是否连接、DSSM 是否被 NONMAIN 禁用 |
| 提示密码或拒绝访问 | NONMAIN 的 SWD 密码策略 |
| AI 只识别到未知 Cortex-M0+ | 未加载 MSPM0 target 配置或 SVD |
| 能下载但不能源码调试 | ELF 被 strip、缺少 `-g`、GDB 路径错误 |

## 8. 推荐的上手顺序

1. 只接 SWDIO、SWCLK、GND；如果板子引出 NRST，一并接上。
2. 目标板独立供电，确认 3.3 V。
3. AixProbe 电平设为 3.3 V。
4. 加载 CH347F + MSPM0G3507 OpenOCD 配置。
5. 连接并确认 `target_name: mspm0x.cpu`。
6. halt、读 CPUID、读 MSPM0 器件 ID。
7. 探测 MAIN Flash。
8. 烧录 TI `empty` 或点灯工程。
9. 再烧录正式工程 ELF。
10. 最后再接串口、示波器或其他外设。

## 9. 参考配置和源码核对点

- [OpenOCD CH347F 配置](configs/aixprobe-mspm0g3507.cfg)
- AixProbe OpenOCD 源码中的 `tcl/target/ti/mspm0.cfg`
- AixProbe OpenOCD 源码中的 `src/flash/nor/mspm0.c`
- AixProbe OpenOCD 源码中的 `tcl/board/ti/mspm0-launchpad.cfg`
- AixProbe V821 原理图中的 CN1、CH347F 和电平选择

其中 `mspm0-launchpad.cfg` 只适合板载 XDS110，不应当直接用于 AixProbe 的 CH347F。

## 附录：LP-MSPM0G3507 LaunchPad（不适用于最小系统板）

如果以后改用 LP-MSPM0G3507 LaunchPad，J103 是标准 10 针 Arm Cortex 调试口：

| J103 引脚 | 信号 |
|---|---|
| 1 | `VTREF` |
| 2 | `SWDIO/TMS` |
| 3 | `GND` |
| 4 | `SWCLK/TCK` |
| 5 | `GND` |
| 6 | `SWO/TDO` |
| 7 | `KEY` |
| 8 | `TDI` |
| 9 | `GNDDetect` |
| 10 | `NRESET` |

此时 AixProbe 的接线为：

| AixProbe CN1 | LP-MSPM0G3507 J103 |
|---|---|
| CN1-2 `SWDIO` | J103-2 |
| CN1-4 `SWCLK` | J103-4 |
| CN1-5 `GND` | J103-3、J103-5 或 J103-9 |
| CN1-10 `NRESET` | J103-10 |

并且需要按 TI LaunchPad 用户指南移除 J101 隔离排针上的相关跳线，避免板载 XDS110 和 AixProbe 同时驱动 SWD。不要把 CN1-1 `+5V` 接到 J103-1；J103-1 是 `VTREF`，通常不是 5 V。
