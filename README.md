# AN758x U-Boot

U-Boot for Airoha AN7581/AN7583 with a bilingual Web recovery interface.

面向 Airoha AN7581/AN7583 的 U-Boot，集成中英文 Web 恢复页面。

Hold Reset about one second after power-on to enter Web recovery.

上电约一秒后按住 Reset，即可进入 Web 恢复页面。

## Supported devices / 支持机型

| Target / 构建目标 | Device / 设备 | SoC | Storage / 存储 |
| --- | --- | --- | --- |
| `hg5382a` | FiberHome HG5382A | AN7581 | Parallel NAND |
| `hg5585f-ct` | FiberHome HG5585F CT | AN7581 | Parallel NAND |
| `hg5585f-cu` | FiberHome HG5585F CU | AN7581 | Parallel NAND |
| `hm2004-du` | H3C HM2004-DU | AN7581 | SPI NAND |
| `xg2010g` | Gemtek XG2010G | AN7581 | SPI NAND |
| `xr1710g` | Gemtek XR1710G | AN7581 | SPI NAND |
| `zn504xg-d` | ZNXT ZN504XG-D | AN7581 | SPI NAND |
| `zn515xg-d` | ZNXT ZN515XG-D | AN7581 | SPI NAND |
| `ung00a` | UnionMan UNG00A | AN7581 | SPI NAND |
| `xg-040g-md` | Nokia XG-040G-MD | AN7581 | SPI NAND |
| `xg-040g-tf` | Nokia XG-040G-TF | AN7581 | SPI NAND |
| `xg-040g-mf` | Nokia XG-040G-MF | AN7583 | SPI NAND |

## Verified on hardware / 真机实测

The two NAND chips below are not only entries in the tables: both boards were
flashed with U-Boot built from this tree, recovered from a half-brick and boot
normally — **tested on real hardware** /
下面两颗 NAND 不只是加了表项：两块板都用本仓库源码编译的 U-Boot 做过真机刷机，
从半砖状态救回并正常启动，**经过实测**。

| Board / 板子 | SoC | NAND | Full ID | Result / 结果 |
| --- | --- | --- | --- | --- |
| FiberHome HG5382A (`hg5382a`) | AN7581 | Winbond `W29N02KVSIAF` | `ef da 10 95 06` | Self-built U-Boot flashed, half-brick recovered, boots<br>自己编译的 U-Boot 刷入，半砖救回，正常启动 |
| FiberHome HG5585F CU (`hg5585f-cu`) | AN7581 | Skyhigh `S34ML02G300TFI00` | `01 da 00 95 46` (BL2 prints `01:da:469500`) | Self-built U-Boot flashed, half-brick recovered, boots<br>自己编译的 U-Boot 刷入，半砖救回，正常启动 |

Both chips use 4 bit / 512 B ECC in the U-Boot and BL2 tables, matching what
Linux reads from the JEDEC parameter page, so UBI attaches without
`-74` (ECC error) / 两颗芯片在 U-Boot 与 BL2 表中都是 4 bit / 512 B，与 Linux
从 JEDEC 参数页读到的值一致，UBI 挂载不再报 `-74`。

## Download / 下载

Prebuilt images for every target are attached to the
[latest release](https://github.com/jzhou404gmail/uboot-an758x/releases/latest) /
每个机型的预编译镜像见 [最新 Release](https://github.com/jzhou404gmail/uboot-an758x/releases/latest)。

Release images are built by CI from the tagged commit; they are signed only when
the repository has an `AIROHA_SIGN_KEY` secret. Build locally with
`AIROHA_SIGN_KEY_PATH` if your board verifies FIP signatures /
Release 镜像由 CI 从 tag 提交构建；只有仓库配置了 `AIROHA_SIGN_KEY` secret 时才签名。
若设备校验 FIP 签名，请按 [Build / 构建](#build--构建) 在本地带
`AIROHA_SIGN_KEY_PATH` 重新构建。

## Recovery entry / 进入恢复模式

The rescue bootloader is entered from the serial console or through the Web
interface. / 救砖引导有串口和网页两条入口。

- **Web recovery.** Hold Reset for about one second after power-on; U-Boot
  prints `Reset held: entering HTTP recovery`. The board runs a DHCP server
  (`192.168.0.100`–`192.168.0.199`) on its Ethernet port, so a directly
  connected PC can use DHCP; then open `http://192.168.0.1/`.<br>
  **网页恢复。** 上电约一秒后按住 Reset，串口打印
  `Reset held: entering HTTP recovery`。板端自带 DHCP 服务器
  （地址池 `192.168.0.100`–`192.168.0.199`），直连电脑用自动获取即可，
  随后打开 `http://192.168.0.1/`。
- **Serial console.** 115200 8N1, the U-Boot prompt is available at the same
  time; `httpd` starts the Web interface manually and `httpd dhcp` enables the
  address pool with it.<br>
  **串口控制台。** 115200 8N1，可同时使用 U-Boot 命令行；`httpd` 手动启动网
  页服务，`httpd dhcp` 连地址池一起启动。
- **XMODEM.** When BL2 cannot obtain a FIP it prints `entering XMODEM
  recovery`; press `x` and send `*-bl31-u-boot.fip` to boot U-Boot from RAM.<br>
  **XMODEM。** BL2 取不到 FIP 时打印 `entering XMODEM recovery`，按 `x` 并
  发送 `*-bl31-u-boot.fip`，从内存临时启动 U-Boot。

The Web page has three tabs / 网页分三个页签：

| Tab / 页签 | Functions / 功能 |
| --- | --- |
| Flash / 刷机 | Write BL2 (`*-preloader.bin`, `*-firstblock.bin`), rebuild UBI, restore board-data volumes, write U-Boot (`*-bl31-u-boot.fip`), install a sysupgrade image, boot the system, reboot<br>写入 BL2、重建 UBI、恢复板级数据卷、写入 U-Boot、刷写系统、启动系统、重启 |
| Backup / 备份 | Download an MTD partition or a UBI volume (main area only, bad blocks padded with `FF`, no OOB)<br>下载 MTD 分区或 UBI 卷（仅主数据区，坏块填 `FF`，不含 OOB） |
| Advanced / 高级 | RAM boot an initramfs, write a single partition, restore an entire-flash main-area image, clear all bad-block markers, edit environment variables, run an arbitrary command, return to serial<br>内存启动 initramfs、分区写入、恢复整片 Flash 主数据、清除坏块标记、环境变量、命令控制台、返回串口 |

The **Clear bad block markers / 清除坏块标记** action erases the whole NAND
chip, factory bad blocks included; it is a two-click confirmation and all data
on the chip is lost. / **清除坏块标记** 会擦掉整片 NAND（含出厂坏块标记），
需要两次点击确认，且会丢失全部数据。

## Build / 构建

```sh
export CROSS_COMPILE=/path/to/aarch64-linux-musl-
export ARM32_CROSS_COMPILE=/path/to/arm-none-eabi-
export MBEDTLS_DIR=/path/to/mbedtls-3.4.1
export BUILD_JOBS=$(nproc)

./scripts/build-an758x.sh hg5585f-ct
```
Artifacts are written to `output/<target>/` / 产物位于 `output/<target>/`

Optional environment / 可选环境变量:

| Variable / 变量 | Meaning / 说明 |
| --- | --- |
| `AIROHA_SIGN_KEY_PATH` | RSA-4096 PEM private key file; signs both FIPs<br>RSA-4096 PEM 私钥文件，用于签名两个 FIP |
| `AIROHA_SIGN_KEY` | The same key as PEM contents<br>同样的私钥内容（PEM 文本） |
| `BUILD_ROOT`, `OUTPUT_ROOT` | Build workspace and output directory<br>构建目录与产物目录 |

Without a key the FIPs stay unsigned / 未提供密钥时产物为未签名 FIP。

Each parallel-NAND target also gets `*-firstblock.bin`, a 128 KiB first
eraseblock with the preloader FIP at offset `0x800`, ready for a raw NAND
programmer / 并行 NAND 机型额外产出 `*-firstblock.bin`：128 KiB 的首个擦除块，
preloader FIP 位于 `0x800` 偏移，可直接用编程器写入。

CI builds every target on push and attaches them to a release for tags / CI 在
每次推送时构建全部机型，打 tag 时自动发布 Release。

### Build every target / 构建全部机型

```sh
for b in hg5382a hg5585f-ct hg5585f-cu hm2004-du xg2010g xr1710g \
         zn504xg-d zn515xg-d ung00a xg-040g-md xg-040g-tf xg-040g-mf; do
	./scripts/build-an758x.sh "$b" || break
done
```

Each target builds out of tree under `build/<target>/` and writes its artifacts
to `output/<target>/`, so targets can also be built in parallel from separate
shells / 每个机型在 `build/<target>/` 下独立构建、产物写入 `output/<target>/`，
也可以在多个终端里并行构建。

`xg-040g-mf` is the only AN7583 target; the other eleven are AN7581 /
`xg-040g-mf` 是唯一 AN7583 机型，其余 11 个都是 AN7581。

## Installation / 刷入

### Automatic / 自动刷机

Run [AN758x-Stock2UBI](https://github.com/pbs05/an758x-stock2ubi) from the
stock system. Back up the needed partitions, then upload `*-preloader.bin` or
`*-firstblock.bin` together with
`*-bl31-u-boot.fip`. The tool writes the recovery bootloader and reboots.<br>
在原厂系统中运行 [AN758x-Stock2UBI](https://github.com/pbs05/an758x-stock2ubi)，
备份需要保留的分区，然后上传 `*-preloader.bin` 或 `*-firstblock.bin`，以及
`*-bl31-u-boot.fip`。工具写入恢复引导后自动重启。

### Manual / 手动刷机

1. Back up every MTD partition and UBI volume.<br>
   备份全部 MTD 分区和 UBI 卷。
2. Write one BL2 image at the beginning of NAND:<br>
   在 NAND 开头写入以下一种 BL2 镜像：

   - Write `*-firstblock.bin` from NAND offset `0x0`.<br>
     从 NAND `0x0` 偏移写入 `*-firstblock.bin`。
   - Leave the first `0x800` bytes erased and write `*-preloader.bin` from
     NAND offset `0x800`.<br>
     保持前 `0x800` 字节为擦除态，从 NAND `0x800` 偏移写入
     `*-preloader.bin`。
3. Boot the device. When BL2 requests a FIP, press `x` and send
   `*-bl31-u-boot.fip` through XMODEM.<br>
   启动设备；BL2 请求 FIP 时按 `x`，通过 XMODEM 发送
   `*-bl31-u-boot.fip`，临时进入 U-Boot。

With either method, connect Ethernet and open `http://192.168.0.1/` (the first
boot may take about one minute). For the first installation, select
**Rebuild UBI / 重建 UBI**, then write `*-bl31-u-boot.fip`, restore the board-data
volumes, upload the sysupgrade image, and select **Boot system / 启动系统**.<br>
通过以上任一方法进入 U-Boot 后，连接网线并打开 `http://192.168.0.1/`（首次启动
可能需要约一分钟）。首次安装选择 **重建 UBI**，随后写入 `*-bl31-u-boot.fip`、
恢复板级数据卷、上传 sysupgrade 镜像，最后选择 **启动系统**。

Subsequent upgrades use **Install system / 刷写系统** directly.

后续升级直接使用 **刷写系统**。

### Unbrick / 救砖

A board whose BL2, FIP or UBI was damaged is recovered with the same images,
without opening the case when the boot ROM still runs / 只要 BootROM 还能启动，
就可用同一套镜像在原位救回，无需拆机上编程器：

1. Enter the Web interface by holding Reset, or reach BL2 over XMODEM.<br>
   按住 Reset 进入网页，或用 XMODEM 进入 BL2。
2. Write `*-firstblock.bin` (or `*-preloader.bin`) at the beginning of NAND and
   `*-bl31-u-boot.fip` afterwards; on NAND the FIP goes into the UBI `fip`
   volume, so **Rebuild UBI / 重建 UBI** first if the volume table is gone.<br>
   写入 NAND 开头的 BL2 镜像，再写入 U-Boot FIP；NAND 上 FIP 存在 UBI 的
   `fip` 卷里，若卷表已丢失，先执行 **重建 UBI**。
3. If `nand bad` lists blocks that should be good, or the system fails to
   attach UBI with `error -74 (ECC error)`, read
   [Troubleshooting / 故障排查](#troubleshooting--故障排查) and the notes in
   [doc/an758x-notes.md](doc/an758x-notes.md) before erasing anything else.<br>
   若 `nand bad` 报出一批本应正常的块，或挂载 UBI 报
   `error -74 (ECC error)`，请先看下面的故障排查与
   [doc/an758x-notes.md](doc/an758x-notes.md)，再考虑擦除。

Writing `*-firstblock.bin` rewrites eraseblock 0 in full, which erases the
ONFI parameter page on page 0. Linux then falls back to the chip's JEDEC
parameter page, so the ECC strength U-Boot declares for that chip must match
what Linux picks from JEDEC / 写入 `*-firstblock.bin` 会整块重写第 0 块，连带
擦掉第 0 页的 ONFI 参数页；Linux 随后改用芯片的 JEDEC 参数页，因此 U-Boot
为该芯片声明的 ECC 强度必须与 Linux 从 JEDEC 得到的值一致。

## Parallel NAND support / 并行 NAND 支持

Parallel-NAND boards (`hg5382a`, `hg5585f-ct`, `hg5585f-cu`) are identified
twice: BL2 uses
`tf-a/plat/ecnt/common/drivers/flash/parallel_nand_flash_table.c`, U-Boot uses
`drivers/mtd/nand/raw/nand_ids.c` / 并行 NAND 机型有两张识别表：BL2 用 TF-A 的
`parallel_nand_flash_table.c`，U-Boot 用 `nand_ids.c`。

Chips listed in both tables / 两张表都有的芯片:

| Chip / 芯片 | Density | Page / OOB | U-Boot declared (`nand_ids.c`) | U-Boot effective | BL2 `soc_ecc_ability` |
| --- | --- | --- | --- | --- | --- |
| `W29N02KVSIAF` | 2 Gb | 2048 / 128 | 4 bit / 512 B | 4 | 4 |
| `W29N02KVSIAE` | 2 Gb | 2048 / 128 | 8 bit / 512 B | 8 | 8 |
| `W29N02GV` | 2 Gb | 2048 / 64 | 1 bit / 512 B | 4 | 4 |
| `S34ML02G300TFI00` | 2 Gb | 2048 / 128 | 4 bit / 512 B | 4 | 4 |
| `S34ML02G2` | 2 Gb | 2048 / 128 | 4 bit / 512 B | 4 | 4 |
| `S34ML01G1` | 1 Gb | 2048 / 64 | 1 bit / 512 B | 4 | 4 |
| `S34ML02G1` | 2 Gb | 2048 / 64 | 1 bit / 512 B | 4 | 4 |
| `S34ML04G1` | 4 Gb | 2048 / 64 | 1 bit / 512 B | 4 | 4 |
| `TC58NVG1S3HTAI0` | 2 Gb | 2048 / 128 | 8 bit / 512 B | 8 | 8 |

Chips known to BL2 but absent from `nand_ids.c`; U-Boot identifies them from
their ONFI or JEDEC parameter page instead / 只在 BL2 表里、U-Boot 靠 ONFI 或
JEDEC 参数页识别的芯片: `MX30LF1G18AC`, `MX30LF2G18AC`, `MX30LF1G08AA`,
`F59L2G81A`, `F59L4G81A`, `W29N08GVSIAA`, `W29N08GVSIAD`, `MT29F01G08ABAEA`,
`MT29F02G08ABAGA`, `MT29F08G08ABACA`, `TC58NVG4S0HTA20`.

### ECC strength has to agree on both sides / 两侧 ECC 强度必须一致

The NAND controller writes a BCH codeword whose length is fixed by the ECC
strength in use, so a page written with one strength cannot be decoded with
another / 控制器写入的 BCH 码字长度由 ECC 强度决定，一边写的页另一边按不同强度
解不出来：

- **U-Boot** takes the declaration from `nand_ids.c` (or the ONFI/JEDEC
  parameter page for chips without an entry) and then
  `airoha_nfc_calc_ecc_strength()` picks the lowest value in
  `{4, 6, 8, 10, 12, 14, 16}` that is at least the declaration and still fits
  the OOB budget `(oobsize / steps - 8) * 8 / 13`. A declaration of `1` is
  therefore the same as `4`, the lowest step the controller supports.<br>
  **U-Boot** 用 `nand_ids.c` 的声明值（无条目时用 ONFI/JEDEC 参数页），再由
  `airoha_nfc_calc_ecc_strength()` 在 `{4, 6, 8, 10, 12, 14, 16}` 中取
  “不小于声明值、且不超过 OOB 预算 `(oobsize / steps - 8) * 8 / 13`”的最低档；
  因此声明 `1` 实际等于 `4`（控制器最低档）。
- **BL2** uses `soc_ecc_ability`, the last field of the chip's `PNAND_ID()`
  entry, directly / **BL2** 直接用表项最后一个字段 `soc_ecc_ability`。
- **Linux** derives it from the parameter pages of the chip itself / **Linux**
  由芯片自身的参数页决定。

To add a chip, add it to both tables with the same effective strength, and keep
the declaration in line with the JEDEC parameter page Linux will read / 新增芯片
时两张表都要加，生效强度必须相同，声明值要与 Linux 读到的 JEDEC 参数页一致。

## Troubleshooting / 故障排查

| Symptom / 现象 | Cause and fix / 原因与处理 |
| --- | --- |
| BL2 prints `FIP preload read failed; entering XMODEM recovery` although the UBI volume holds a good FIP<br>BL2 报 FIP preload 失败，尽管 UBI 里的 FIP 完好 | BL2 was reading the legacy boot-partition window at offset `0x800` and treated one unreadable page as fatal. Current BL2 skips that preload when the UBI policy owns the FIP; update to a current build<br>旧 BL2 会去读 `0x800` 处的传统引导分区窗口，任何一个页读失败就判死。当前版本在 FIP 归 UBI 管时跳过该预读，请更新镜像 |
| `nand bad` lists blocks `0x0` onwards, including the block BL2 booted from<br>`nand bad` 从 0 号块开始报出大量坏块 | Older builds compared an uninitialised high byte of the marker word and reported good blocks as bad. Fixed upstream; on an older build `nand scrub -y <off> <size>` clears the false markers, at the cost of erasing that range<br>旧版本把标记字未初始化的高字节一起比较，把好块判成坏块。已在上游修复；旧镜像上可用 `nand scrub -y` 清掉误标，但该范围内的数据会被一并擦除 |
| `ubi0 warning: ubi_io_read: error -74 (ECC error)` on every written PEB after Linux attaches<br>Linux 挂载后每个已写 PEB 都报 `error -74 (ECC error)` | U-Boot and Linux (and BL2) disagree on the ECC strength for that chip; see [ECC strength has to agree on both sides](#ecc-strength-has-to-agree-on-both-sides--两侧-ecc-强度必须一致)<br>U-Boot 与 Linux/BL2 对该芯片的 ECC 强度不一致，见上一节 |
| BL2 identifies the NAND and then waits in XMODEM forever<br>BL2 认出 NAND 后一直停在 XMODEM | The FIP is neither in the `fip` volume nor in the boot partition: rebuild UBI and write `*-bl31-u-boot.fip`, or send it over XMODEM<br>FIP 既不在 `fip` 卷也不在引导分区：重建 UBI 后写入 `*-bl31-u-boot.fip`，或用 XMODEM 发送 |
| UNG00A prints `Could not find a valid device for nand0` while rebuilding UBI<br>UNG00A 重建 UBI 时打印 `Could not find a valid device for nand0` | Fixed in `v1.0`. The board env inherited the parallel-NAND `mtdids`/`mtdparts` (`nand0`), but UNG00A is SPI NAND (`spi-nand0`) and its `bl2`/`ubi` partitions come from the DTS. Update to `v1.0` or later, or run `setenv mtdids; setenv mtdparts; saveenv` once<br>`v1.0` 已修复。该机型 env 误带了并行 NAND 的 `nand0` 表项，而 UNG00A 是 SPI NAND（`spi-nand0`），`bl2`/`ubi` 分区来自 DTS。请更新到 `v1.0` 及以后，或在命令行执行一次 `setenv mtdids; setenv mtdparts; saveenv` |
| `"spi-nand0" partitions still in use, can't delete them` while rebuilding UBI<br>重建 UBI 时打印 `"spi-nand0" partitions still in use...` | Informational: the rebuild probes MTD before detaching UBI, so the existing partitions are kept and reused; the rebuild continues<br>提示性信息：重建时先探测 MTD 再 detach UBI，删除被拒后直接复用现有分区，重建会继续 |
| `Skipping bad block at 0x...` while rebuilding UBI<br>重建 UBI 时打印一串 `Skipping bad block` | Normal: the erase loop and UBI both skip blocks already marked bad. Do not use **Clear bad block markers** for this; it erases the whole chip, BL2 included<br>正常：擦除循环和 UBI 都会跳过已标记的坏块。不要因此使用 **清除坏块标记**，它会擦掉整片（含 BL2） |
| Web page unreachable<br>网页打不开 | Check the link LED and that the PC uses DHCP or `192.168.0.2/24`; the interface is only started in recovery, and `ipaddr`/`netmask` can be overridden with `httpd_ipaddr`/`httpd_netmask`<br>检查网口灯与电脑是否自动获取 IP 或设为 `192.168.0.2/24`；恢复模式才会启动网口，地址可用 `httpd_ipaddr`/`httpd_netmask` 覆盖 |

## Changes in this repository / 本仓库改动

Most bring-up changes are in upstream `main`; this fork adds the documentation
below and one board-env fix that is not upstream yet /
大部分调机改动已进入上游 `main`；本仓库额外增加下面的文档，以及一处尚未上游的
板级 env 修复：

| Work / 工作 | Status / 状态 |
| --- | --- |
| Bad-block marker read on x8/x16 NAND (`airoha_nfc_block_bad`)<br>x8/x16 NAND 坏块标记读取 | Merged: `cdca3873` (x8) and `d7d54582` (x16, authored by this fork)<br>已合并：`cdca3873`（x8）与 `d7d54582`（x16，作者为本仓库） |
| BL2 skips the boot-partition FIP preload when UBI owns the FIP<br>FIP 归 UBI 时 BL2 跳过引导分区预读 | Merged as `9c5cb7f7` and follow-ups<br>已由 `9c5cb7f7` 等提交合并 |
| `S34ML02G300TFI00` ECC lowered to 4 bit / 512 B in both tables<br>两张表的 `S34ML02G300TFI00` 统一为 4 bit / 512 B | Merged as `154ec96a` and `b5056eb6`<br>已由 `154ec96a`、`b5056eb6` 合并 |
| UNG00A default env no longer carries the parallel-NAND `nand0` `mtdids`/`mtdparts`<br>UNG00A 默认 env 去掉并行 NAND 的 `nand0` 表项 | **This fork only** (`v1.0`)<br>**仅本仓库**（`v1.0`） |
| `v1.0` release: CI-built images for all 12 targets<br>`v1.0` 发布：CI 构建的全部 12 个机型镜像 | [Releases](https://github.com/jzhou404gmail/uboot-an758x/releases) |
| Local work log, symptoms and verification<br>本地工作记录、现象与验证 | [doc/an758x-notes.md](doc/an758x-notes.md) |

## TF-A sources / TF-A 来源

- [Ansuel/atf-airoha](https://github.com/Ansuel/atf-airoha)
- [mkshevetskiy/atf-airoha](https://github.com/mkshevetskiy/atf-airoha)
- [Yuzhii0718/atf-airoha](https://github.com/Yuzhii0718/atf-airoha)
