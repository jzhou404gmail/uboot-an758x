## 说明 / About

AN758x（Airoha AN7581 / AN7583）恢复引导，由本仓库 CI 从 tag `v1.0` 构建，
覆盖 12 个机型。基于上游 `pbs05/uboot-an758x` 的 `main`（`4b91e112`）。

本版本包含：

- 上游全部并行 NAND / SPI NAND 支持与修复，包括
  `W29N02KVSIAF`、`S34ML02G300TFI00`（ID `01:da:469500`）的 4 bit/512 B ECC，
  x8/x16 坏块标记读取，以及 FIP 归 UBI 时 BL2 跳过引导分区预读；
- 完善后的 `README.md` 与 `doc/an758x-notes.md` 工作记录；
- 修复：UNG00A 默认 env 不再携带并行 NAND 的 `mtdids`/`mtdparts`，
  重建 UBI 时不再出现 `Could not find a valid device for nand0`。

## 覆盖机型 / Targets

| 目标 | 设备 | SoC | 存储 |
| --- | --- | --- | --- |
| `hg5382a` | FiberHome HG5382A | AN7581 | 并行 NAND |
| `hg5585f-ct` | FiberHome HG5585F CT | AN7581 | 并行 NAND |
| `hg5585f-cu` | FiberHome HG5585F CU | AN7581 | 并行 NAND |
| `hm2004-du` | H3C HM2004-DU | AN7581 | SPI NAND |
| `xg2010g` | Gemtek XG2010G | AN7581 | SPI NAND |
| `xr1710g` | Gemtek XR1710G | AN7581 | SPI NAND |
| `zn504xg-d` | ZNXT ZN504XG-D | AN7581 | SPI NAND |
| `zn515xg-d` | ZNXT ZN515XG-D | AN7581 | SPI NAND |
| `ung00a` | UnionMan UNG00A | AN7581 | SPI NAND |
| `xg-040g-md` | Nokia XG-040G-MD | AN7581 | SPI NAND |
| `xg-040g-tf` | Nokia XG-040G-TF | AN7581 | SPI NAND |
| `xg-040g-mf` | Nokia XG-040G-MF | AN7583 | SPI NAND |

## 文件 / Files

每个机型 3 个文件 / Three files per target:

| 文件 / File | 用途 / Purpose |
| --- | --- |
| `*-preloader.bin` | 从 NAND `0x800` 偏移写入（前 `0x800` 保持擦除态） |
| `*-firstblock.bin` | 从 NAND `0x0` 整块写入；并行 NAND 机型的 128 KiB 首块镜像 |
| `*-bl31-u-boot.fip` | BL31 + U-Boot FIP，写入 UBI 的 `fip` 卷，或 BL2 请求 FIP 时经 XMODEM 发送 |

SPI NAND 机型没有 `*-firstblock.bin`，属于正常。

## 刷入 / 救砖顺序

1. 串口 115200 8N1，写 BL2：`*-firstblock.bin` 从 `0x0`，或 `*-preloader.bin` 从 `0x800`。
2. 上电。BL2 请求 FIP 时按 `x`，经 XMODEM 发送 `*-bl31-u-boot.fip`
   （NAND 上也可直接把 FIP 写进 UBI 的 `fip` 卷）。
3. 进入 U-Boot 后，上电约一秒内按住 Reset 进入网页恢复：板端自带 DHCP
   （`192.168.0.100`–`192.168.0.199`），网线直连电脑后打开 `http://192.168.0.1/`。
4. 首次安装：**重建 UBI** → 写入 U-Boot FIP → 恢复板级数据卷 →
   上传 sysupgrade 镜像 → **启动系统**。
5. 后续升级：直接 **刷写系统**。

## 注意 / Notes

- **ECC 强度必须三处一致**（U-Boot 的 `nand_ids.c`、BL2 的 TF-A 表、Linux 读到的
  参数页），不一致会出现 `ubi0 warning: ubi_io_read: error -74 (ECC error)`。
  详见 README 的“ECC strength has to agree on both sides / 两侧 ECC 强度必须一致”。
- 写 `*-firstblock.bin` 会整块重写第 0 块，连带擦掉第 0 页的 ONFI 参数页，
  Linux 之后改用 JEDEC 参数页，因此 U-Boot 声明的强度要与 Linux 从 JEDEC 取到的值一致。
- 重建 UBI 时打印的 `Skipping bad block at 0x...` 是正常现象：已标记的坏块会被擦除
  循环和 UBI 跳过。不要因此使用网页里的 **清除坏块标记**，它会擦掉整片 NAND
  （含出厂坏块标记和 BL2）。
- 本 Release 由 CI 构建；未配置 `AIROHA_SIGN_KEY` secret 时产物为未签名 FIP。
  需要签名时本地设置 `AIROHA_SIGN_KEY_PATH` 或 `AIROHA_SIGN_KEY`（RSA-4096 PEM）
  后重新构建。
- UNG00A 用户：`v1.0` 已修复重建 UBI 时的
  `Could not find a valid device for nand0`；若仍在使用旧镜像，可执行一次
  `setenv mtdids; setenv mtdparts; saveenv`。

## 参考 / References

- README：<https://github.com/jzhou404gmail/uboot-an758x/blob/main/README.md>
- 工作记录（现象、根因、上游状态、ECC 逐芯片核对）：
  <https://github.com/jzhou404gmail/uboot-an758x/blob/main/doc/an758x-notes.md>
- 上游项目：<https://github.com/pbs05/uboot-an758x>

**Full Changelog**:
<https://github.com/jzhou404gmail/uboot-an758x/compare/v2026.07-5-jzhou...v1.0>
