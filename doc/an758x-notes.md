# AN758x 调机工作记录 / Work log

本文档汇总 2026-09-20 ~ 2026-10-16 期间围绕 `pbs05/uboot-an758x` 派生仓库
所做的全部修改与调查：问题现象、根因、改动内容、上游状态、验证方式，以及
留给后续的可选项。README 只保留结论，细节在这里。

- 仓库：`uboot-an758x`（fork: `jzhou404gmail/uboot-an758x`）
- 涉及机型：FiberHome **HG5585F CU**（AN7581 + SkyHigh `S34ML02G300TFI00`）、
  FiberHome **HG5382A**（AN7581 + Winbond `W29N02KVSIAF`）
- 当前结论：**调机期间的全部代码改动都已进入上游 `main`，本地分支没有遗留
  的可提交差异**；本 fork 目前只有文档增量。

---

## 1. 时间线

| 日期 | 事件 |
| --- | --- |
| 09-20 | 从上游克隆（`434ff073`）。起因：HG5585F CU 想刷 U-Boot，需确认并行 NAND 支持情况 |
| 09-20 ~ 09-21 | 分析两张 NAND 识别表、准备交叉工具链、编译、与手上已编译镜像对比、用 RSA-4096 私钥重签 FIP |
| 09-21 ~ 09-22 | 定位并修复三个问题 → 分支 `fix-airoha-pnand-badblock-ubi-fip`，提交 `52a7476c`、`7df7e60a` |
| 09-23 | `S34ML02G300TFI00` 的 ECC 强度与 Linux 对齐 → 分支 `fix-airoha-pnand-boot-and-s34ml-ecc`，提交 `9c1650d2`；向上游提 **PR #9** |
| 09-22 / 09-23 | 上游并行出现等价修复（`cdca3873`、`ef8870ad`、`154ec96a`、`b5056eb6`、`9c5cb7f7`） |
| 09-24 | 上游合并 `d7d54582`（x16 坏块标记处理，**作者 jzhou404**） |
| 09-25 | 维护者 pbs05 关闭 PR #9：*“这些里面有一些已经修复，有合并冲突。我单独把你的 16bit nand 合并了”* |
| 09-30 | 上游 `main` 推进到 `4b91e112`（合并 PR #19） |
| 10-08 | 拉取上游最新；HG5382A + Winbond `W29N02KVSIAF` 变砖调查 → 结论：上游已支持该闪存，本地无需改代码；按上游 `4b91e112` 构建救砖镜像到 `output/hg5382a/` |
| 10-10 | 复核 Winbond 条目在已构建二进制中的实际取值 → 无 Winbond 相关的可提交改动 |
| 10-16 | 本轮：核对 ECC 一致性、完善 README、新增本文档 |

---

## 2. 三个修复

### 2.1 并行 NAND 坏块标记读取（x8 高字节未初始化）

- **提交**：`52a7476c` — *Fix parallel NAND bad block marker check for 8-bit buses*
- **现象**：`mtd erase ubi` / `nand bad` 把 `0x00000000` 起的连续块报成坏块，
  其中 0 号块正是 BootROM 刚刚加载 BL2 的块。
- **根因**：`airoha_nfc_block_bad()` 把 1 字节标记读进一个未初始化的 `u16`
  低字节，然后拿整个 `u16` 与 `0xFF` 比较。x8 传输只写低字节，高字节是栈上
  残留值，于是“好块/坏块”由无关数据决定。
- **危害不止统计**：`nand erase`/`nand write` 会跳过这些块，而 UBI 还会往它
  认为坏掉的 PEB 里写入真实的坏块标记，之后对 Linux 等其它读者也变成坏块。
- **改动**：只比较传输实际填写的字节（x8 比低字节，x16 比两个字），并把缓冲区
  初始化为 `0xffff`。
- **上游状态**：x8 部分由 `cdca3873`（09-22）以等价方式修复；x16 部分由
  `d7d54582`（09-24）合并，**该提交作者是你的 `jzhou404 <jzhou404@gmail.com>`**。
- **验证**：最终上游代码在功能上与本提交一致（差别仅是注释与 `(u8)` 写法）。

### 2.2 FIP 归 UBI 时 BL2 仍去预读引导分区

- **提交**：`7df7e60a` — *bl23: skip the boot partition FIP preload when UBI owns the FIP*
- **现象**：HG5585F CU 上 BL2 打印
  `ERROR: FIP preload read failed; entering XMODEM recovery`，而同一颗 BL2 随后
  从 UBI 的 `fip` 卷里能正常读出 FIP。
- **根因**：`bl2_plat_preload_setup()` 会从传统引导分区窗口（NAND 偏移 `0x800`，
  长度 `0x7f800`）预读 FIP，并把任何读失败当作致命错误转 XMODEM。该窗口只在
  “memmap 策略”下才是镜像来源（eMMC 用该策略，XMODEM 路径显式切换过去），
  NAND 上 FIP 位于 UBI 卷，预读结果和随后的 `plat_check_header()` 都不参与引导。
  窗口内任何一个合法读失败（擦除页、别的组件写过的页）都会中断一次本来可用的引导。
- **改动**：UBI 策略持有 FIP 时跳过预读与头检查，并在串口打印 FIP 来源；
  FIP 存在引导分区的机型不受影响。
- **上游状态**：已由 `9c5cb7f7` 及后续提交以等价方式实现。
  `git show 7df7e60a | git apply --check -R`（反向应用）**成功**，说明当前上游
  代码已包含该改动。

### 2.3 `S34ML02G300TFI00` 的 ECC 强度与 Linux 不一致

- **提交**：`9c1650d2` — *Use 4 bit/512B for S34ML02G300TFI00 to match Linux*
- **现象**：刷完固件启动，`ubi_attach()` 对每个已写 PEB 报
  `ubi0 warning: ubi_io_read: error -74 (ECC error)`，系统起不来。
- **根因**：同一颗芯片，U-Boot 侧声明 8 bit/512B，Linux 侧按 JEDEC 参数页取
  4 bit/512B。BCH 码字长度由强度决定，写读不一致就整片解不出来；BL2 与 U-Boot
  自己读回却“正常”，因为两边都是 8。
  另外：写 `*-firstblock.bin` 会整块重写第 0 块，擦掉第 0 页的 ONFI 参数页，
  Linux 只能改用 JEDEC 参数页（该家族为 4 bit/512B）。
- **改动**：U-Boot `nand_ids.c` 与 TF-A `parallel_nand_flash_table.c` 都改成
  4 bit/512B（TF-A 侧 `soc_ecc_ability` 8 → 4）。
- **上游状态**：已由 `154ec96a`（U-Boot，8 → 4）与 `b5056eb6`（TF-A，`4, 5, 8`
  → `1, 5, 4`）合并，生效强度与本提交相同（TF-A 的 `min_ecc_req` 保留 4 还是
  改成 1 不影响到实际使用的 `soc_ecc_ability`）。

### 2.4 上游去重核验方法

```sh
# 逐条检查本地提交是否已在上游：反向应用成功 == 上游已含同样改动
for c in 52a7476c 7df7e60a 9c1650d2; do
	git show "$c" | git apply --check -R - && echo "$c: 已在上游" || echo "$c: 上游为等价实现"
done
```

结果：`7df7e60a` 已在上游；`52a7476c`、`9c1650d2` 上游为等价实现（见上文逐条说明）。

---

## 3. PR #9 始末

- 提出：09-23，标题 *Fix parallel NAND boot and S34ML02G300TFI00 ECC strength*，
  含上述三个提交，base 为上游 `0f4146f4`。
- 关闭：09-25，维护者 pbs05：*“这些里面有一些已经修复，有合并冲突。我单独把你的
  16bit nand 合并了”*。合并结果即 `d7d54582`（保留你的作者信息）。
- 结论：不需要重开 PR；三个修复在上游都已具备。

---

## 4. HG5382A + Winbond `W29N02KVSIAF` 变砖调查（10-08 ~ 10-10）

- **起因**：HG5382A 刷机后变砖，闪存为 Winbond `W29N02KVSIAF`（数据手册放在本地
  `flash spec/W29N02KVSIAF.PDF`，未纳入版本库）。
- **代码支持结论（当前上游 `4b91e112`）**：
  - TF-A 表：`W29N02KVSIAF`，dev `0xda`、ext `0x069510`、2 Gb、2048 B 页 /
    128 B OOB、`min_ecc_req = 4`、`addr_cycle = 5`、`soc_ecc_ability = 4`。
  - U-Boot `nand_ids.c`：同芯片条目 `{0xef, 0xda, 0x10, 0x95, 0x06}`，
    2 KiB 页 / 256 MiB / 128 KiB 块 / 128 B OOB，`NAND_ECC_INFO(4, SZ_512)`。
  - 两侧一致，**上游已完整支持该闪存，本地无需任何改动**。
- **变砖原因（窗口期推断）**：上游 `4b7a0b6e`（09-21）先把 U-Boot 侧的
  `W29N02KV` 声明改成 4 bit/512B，而 TF-A 侧要到 `ef8870ad`（09-22）才同步为 4。
  用这个窗口内构建/发布的镜像刷机，两张表一为 8 一为 4，写入与读取的码字长度不同，
  就会出现读回失败/无法引导。用当前 `main` 构建的镜像不会再有这个问题。
- **救砖镜像（已构建，本地目录 `output/hg5382a/`，属于 git 忽略目录）**：

  | 文件 | 大小 | 用途 |
  | --- | --- | --- |
  | `an7581-fiberhome-hg5382a-preloader.bin` | 113664 B | 写到 NAND `0x800`（前 `0x800` 保持擦除态） |
  | `an7581-fiberhome-hg5382a-firstblock.bin` | 131072 B | 从 NAND `0x0` 整块写入（编程器/救砖） |
  | `an7581-fiberhome-hg5382a-bl31-u-boot.fip` | 330752 B | 写入 UBI `fip` 卷，或经 XMODEM 发送 |

- **二进制复核证据**：
  - `u-boot.bin` 中该条目解出 `ecc = 4 bit / 512B`；
  - `bl23`（LZMA 解压后）中该表项为 `ext = 10 95 06 00`、`timing = 0x40044326`、
    `min_ecc = 4`、`addr_cycle = 5`、`soc_ecc_ability = 4`。
  - 即 BL2 与 U-Boot 在同一镜像内一致。
- **PR 判定**：没有可提交的 Winbond 改动（本地 0 行 diff），当时已如实反馈。

---

## 5. ECC 一致性全面核对（10-16）

**规则**（见 README 同名小节）：

- U-Boot：声明值来自 `nand_ids.c`（无条目则来自 ONFI/JEDEC 参数页），
  `airoha_nfc_calc_ecc_strength()` 在 `{4, 6, 8, 10, 12, 14, 16}` 中取
  “≥ 声明值、且 ≤ OOB 预算 `(oobsize / steps - 8) * 8 / 13`”的最低档。
  控制器最低档是 4，所以**声明 `1` 实际生效为 `4`**。
- BL2：直接使用 TF-A 表项的 `soc_ecc_ability`。
- 两侧必须相同；否则一边写的页另一边解不出来（2.3 就是这一类问题）。

**方法**：解析 TF-A `PNAND_ID()` 21 条与 U-Boot `nand_flash_ids[]`，按上述选档
规则复算 U-Boot 生效强度，与 `soc_ecc_ability` 比对。

**结果**：

| 类别 | 数量 | 情况 |
| --- | --- | --- |
| 两张表都有 | 9 | 生效强度**全部一致**（`W29N02KVSIAF` 4、`W29N02KVSIAE` 8、`W29N02GV` 1→4、`S34ML02G300TFI00` 4、`S34ML02G2` 4、`S34ML01G1`/`S34ML02G1`/`S34ML04G1` 1→4、`TC58NVG1S3HTAI0` 8） |
| 仅 BL2 表有 | 11 | U-Boot 靠 ONFI/JEDEC 自动识别：`MX30LF1G18AC`、`MX30LF2G18AC`、`MX30LF1G08AA`、`F59L2G81A`、`F59L4G81A`、`W29N08GVSIAA`、`W29N08GVSIAD`、`MT29F01G08ABAEA`、`MT29F02G08ABAGA`、`MT29F08G08ABACA`、`TC58NVG4S0HTA20` |

**风险提示**：第二类芯片若参数页读不到（例如刷 `*-firstblock.bin` 后第 0 页
ONFI 参数页被擦掉），U-Boot 会退到“按 OOB 预算取最大档”。以
`MT29F02G08ABAGA`（2048/128，预算 `24 * 8 / 13 = 14`）为例，U-Boot 会取到
**14**，而 BL2 表写的是 **12**，两边就不一致。给这类芯片在 `nand_ids.c` 补条目
可以从根上避免（见第 7 节）。

---

## 6. 复现用的构建与刷机命令

本机工具链（`build/` 为 git 忽略目录，随用随下）：

```sh
export CROSS_COMPILE=/persistent/home/jzhou/work/uboot-an758x/build/host-tools/aarch64-linux-musl-cross/bin/aarch64-linux-musl-
export ARM32_CROSS_COMPILE=arm-none-eabi-
export MBEDTLS_DIR=/persistent/home/jzhou/work/uboot-an758x/build/host-tools/mbedtls-3.4.1
export BUILD_JOBS=12

./scripts/build-an758x.sh hg5382a        # 或 hg5585f-cu / hg5585f-ct / ...
```

可选签名（未提供密钥则产物为未签名 FIP）：

```sh
export AIROHA_SIGN_KEY_PATH=/path/to/rsa4096-root.pem
# 或 export AIROHA_SIGN_KEY="$(cat /path/to/rsa4096-root.pem)"
```

救砖顺序（串口 115200 8N1；网页 `192.168.0.1`）：
写 `*-firstblock.bin`（或 `*-preloader.bin`）→ 进 BL2 → XMODEM 发
`*-bl31-u-boot.fip` 或写入 UBI `fip` 卷 → 网页 **重建 UBI** → 写 U-Boot FIP →
恢复板级数据卷 → 刷 sysupgrade → **启动系统**。

---

## 7. 遗留事项与可选后续

1. **给 11 款“仅 BL2 有”的芯片补 U-Boot `nand_ids.c` 条目**：让两侧强度显式
   一致，避免参数页丢失时的退档风险。属于可提上游的改动。
2. **加一个一致性检查**（脚本或构建期断言）：解析两张表并断言生效强度相同，
   防止再出现 2.1~2.3 那种“一侧改了另一侧忘改”的窗口期问题。
3. 本地 `output/winbond-w29n02kv-review.md` 是 10-10 的复核记录（在 git 忽略
   目录内，未纳入版本库）。
4. `flash spec/W29N02KVSIAF.PDF` 为 2 MB 数据手册，未纳入版本库；如需长期保留
   建议单独存放或放入 `doc/`。

---

## 8. 相关上游提交索引

| 提交 | 作者 | 日期 | 说明 |
| --- | --- | --- | --- |
| `d7d54582` | **jzhou404** | 09-24 | mtd: nand: airoha: handle x16 bad-block markers（本仓库贡献） |
| `cdca3873` | 1261685372 | 09-22 | mtd: nand: airoha: fix 8-bit bad-block detection |
| `9c5cb7f7` | 1261685372 | 09-22 | tf-a: skip the FIP preload when UBI owns FIP |
| `4b7a0b6e` | 1261685372 | 09-21 | Fix W29N02KV ECC requirement（U-Boot 侧 8 → 4） |
| `ef8870ad` | 1261685372 | 09-22 | tf-a: 同步 U-Boot 表（SIAF `8 → 4`，并新增 SIAE 条目） |
| `154ec96a` | 1261685372 | 09-23 | S34ML02G300TFI00 ECC 降到 4 bit（U-Boot） |
| `b5056eb6` | 22p | 09-23 | 修正 S34ML02G300TFI00 在 TF-A 表中的 ECC 字段 |
| `4b91e112` | pbs05 | 09-30 | Merge PR #19（当前上游 `main`） |
