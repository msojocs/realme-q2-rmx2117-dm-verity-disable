# realme RMX2117 (narzo 30 Pro 5G) dm-verity 开机警告修复记录

日期：2026-10-31  
主机：一台 Linux PC（USB 2.0 口，500mA）  
设备：realme narzo 30 Pro 5G / RMX2117，Android 10（`RMX2117_11_A.15`），MTK MT6853（天玑 800U），Magisk Kitsune `R6687BB53`（27001），bootloader 已解锁（orange），recovery 里 SELinux permissive，adb root。

> 说明：本文已脱敏，可公开发布。**请勿把二进制镜像备份（boot/vbmeta/seccfg/misc/expdb/lk 等）或完整聊天/工具日志一起公开**——它们包含设备绑定数据/日志，可能含序列号、MAC、账号等。

---

## 1. 结论

- **已修复**：开机时 `dm-verity corruption / Your device is corrupt / Press power button to continue / Or, device will power off in %ds` 警告已消失，手机可以**不按电源键自动启动**（已用“完全不碰手机”的重启验证，30 秒进入系统）。
- 还剩 **orange state / device is unlocked** 提示：这是解锁 bootloader 固有的警告，通常几秒后自动继续；要彻底去掉只能恢复原厂固件并上锁（会失去 root，且可能清数据）。
- 本次**保留了 Magisk/root**。
- 根因不是电源键硬件，也不是普通 bootloop：是 **MTK seccfg 分区里的 `dm_verity_state = 1 (DM_VERITY_GENERAL_ERROR)`**，由之前用 mtkclient 解锁 bootloader 时写错字段留下；LK 每次启动读到该状态就弹出 dm-verity 警告，5 秒内不按电源键就自动关机，再开机重复，看起来像无限重启。

---

## 2. 症状

开机时很快出现：

```
Your device is corrupt.
It can't be trusted and may not work properly.
Press power button to continue.
Or, device will power off in 5s
```

按电源键后出现：

```
Orange State
Your device has been unlocked and can't be trusted
Your device will boot in 5 seconds
```

然后进入系统。若不按电源键，5 秒后电量/电源状态会关机，再次开机重复。

补充：手机放在 PC 的 USB2 口上，只有 500mA；电池 5 年、SOH 85%、满充 4064mAh（标称约 5000mAh）。电量耗尽后会进入 MTK kpoc 关机充电模式（USB 显示 `0e8d:20ff`，屏幕为充电动画），此时按电源键是正常的“退出关机充电并开机”，与 dm-verity 警告是两件事。

---

## 3. 关键证据

### 3.1 LK 字符串（`lk` 分区）

`lk.img` 中含：

```
dm-verity corruption
Your device is corrupt.
It can't be trusted and may not work properly.
Press power button to continue.
Or, device will power off in %ds
```

以及 `Orange State / Your device has been unlocked and can't be trusted`。所以第一段是 **LK 的 dm-verity 警告**，第二段是解锁的 orange 警告。

### 3.2 LK 日志（expdb）

```
[avb] dm_status:1 avb_flag:0x3 hashree_error_mode: 0
Loading vbmeta struct from partition 'vbmeta'
ERROR: Hash does not match!
ERROR: vbmeta: Error verifying vbmeta image: HASH_MISMATCH
DEBUG: vbmeta: VERIFICATION_DISABLED bit is set.
...
[avb] avb_ret = 0 hashtree_error_mode:0
[avb] ret = 0
boot state: orange
```

### 3.3 seccfg（根因）

`seccfg` V4 头部（60 字节）字段：

| 偏移 | 字段 | 修复前 | 修复后 |
|---|---|---|---|
| 0x00 | magic `MMMM` | 0x4D4D4D4D | 同 |
| 0x04 | version | 4 | 4 |
| 0x08 | size | 0x3C | 0x3C |
| 0x0C | `lock_state` | 3（unlocked） | 3 |
| **0x10** | **`dm_verity_state`** | **1 = DM_VERITY_GENERAL_ERROR** | **0 = OK** |
| 0x14 | `sboot_runtime` | 0 | 0 |
| 0x18 | endflag `EEEE` | 0x45454545 | 同 |
| 0x1C+ | 32 字节硬件校验和 | — | LK 重写 |

MTK 的 `verity_state` 枚举：`DM_VERITY_STATUS_OK = 0`，`DM_VERITY_GENERAL_ERROR = 1`。mtkclient 解锁时把 `LKCS_UNLOCK=1` 误写进了 offset 16，导致 dm-verity 错误状态持久化。

---

## 4. 实际修复步骤

> 以下按实际执行顺序记录；最终生效的是 seccfg 被清 0。中间对 vbmeta/boot 的修改也保留在备份中，便于理解/回退。

### 4.1 修改 boot 镜像内置 vbmeta 标志

当前 boot 是 Magisk 打过补丁的镜像，内置 vbmeta `flags=0x0`（保留 AVB），与主 vbmeta 的 `flags=0x3` 不一致。

在手机上用 Magisk 自带的 `magiskboot` 重新打包，设置 `PATCHVBMETAFLAG=true`：

```sh
cd /data/local/tmp/avbpatch
magiskboot unpack boot_current.img
PATCHVBMETAFLAG=true magiskboot repack boot_current.img new-boot.img
```

结果：`new-boot.img` 内置 vbmeta `flags=0x3`（disable verity + disable verification），大小仍为 32MiB。已写入 `boot` 分区并 md5 校验。

### 4.2 重算主 vbmeta 内部 hash

主 `vbmeta` 原本已是 `flags=0x3`，但改动 flags 后内部 hash 未重算，LK 报 `HASH_MISMATCH`。AVB 算法：`SHA256(vbmeta_header(256B) + aux_data_block)`，结果写入 auth block 的 `hash_offset` 处。

已对 `vbmeta_current.img` 重算并写入 `vbmeta` 分区，得到 `vbmeta_rehashed.img`；写入后分区 md5 一致。

### 4.3 修改 vbmeta_system / vbmeta_vendor

`vbmeta_system`、`vbmeta_vendor` 原为 `flags=0x0`（verity/verification 开启）。已把头部 flags 改为 `0x3` 并写入两个分区（`vbmeta_system_patched.img`、`vbmeta_vendor_patched.img`）。  
注意：这两个镜像的内部 hash 未重算（LK 当前未对它们报错），如果未来出现新的 HASH_MISMATCH，可按 4.2 的方法补算。

### 4.4 触发 LK 清除 dm_verity_state（关键）

根据 MTK 的 `clear_dm_verity_status()` 机制，在 `misc` 分区偏移 **4096** 处写入非零值，让 LK 走 OTA-result 清理路径：

```sh
# 4 字节 01 00 00 00，写到 misc + 4096
dd if=/sdcard/misc_ota.bin of=/dev/block/by-name/misc bs=1 seek=4096 conv=notrunc
sync
```

重启后：

- `seccfg` offset 0x10 从 **1 → 0**；
- `misc` offset 4096 被 LK 清回 0；
- LK 用硬件校验和正确写回 seccfg；
- 之后连续重启（包括“不碰手机”的干净重启）都不再出现 dm-verity 警告。

> 说明：expdb 中能看到 `[LK] check_ota_result = 0` / `ota-fail` 的旧日志，说明 4 字节标记不一定被当成有效 OTA 结果；但 seccfg 确实被清零，misc 标记也被清掉。可能与 4.2 重算 vbmeta hash 后 LK 的 AVB 结果变干净、再触发 `clear_dm_verity_status()` 有关。无论具体触发点，最终状态已验证。

---

## 5. 验证

- `seccfg` offset 0x0C..0x13：`03 00 00 00 00 00 00 00`（lock_state=3，dm_verity_state=0）。
- 修复前后对比：差异只有 offset 0x10（1→0）和 32 字节硬件校验和区域（`cmp -l` 验证），说明是 LK 正确重写了该状态。
- `misc` offset 4096：全 0。
- 不按任何键 `adb reboot`，30 秒自动进入系统，`sys.boot.reason=reboot,adb`。
- 验证确认：**dm-verity 警告消失**，只剩 orange 解锁提示。

---

## 6. 备份清单

### 主机：`~/rmx2117-backup/`

| 文件 | 说明 |
|---|---|
| `boot_current.img` | 最初备份的 boot（Magisk 已打补丁，内置 vbmeta flags=0，AVB 保留）。警告状态下的 boot。**仅应急回退用**。 |
| `new-boot.img` | 已写入的 boot：Magisk ramdisk + 内置 vbmeta flags=0x3。内部 hash 未重算，但当前可正常启动。 |
| `vbmeta_current.img` | 最初的主 vbmeta（flags=0x3，内部 hash 不匹配，LK 曾报 HASH_MISMATCH）。 |
| `vbmeta_rehashed.img` | 已写入的主 vbmeta（flags=0x3，内部 hash 已重算）。 |
| `vbmeta_system_current.img` | 原始 vbmeta_system（flags=0，hash 正确）。 |
| `vbmeta_system_patched.img` | 已写入的 vbmeta_system（flags=0x3，内部 hash 未重算）。 |
| `vbmeta_vendor_current.img` | 原始 vbmeta_vendor（flags=0，hash 正确）。 |
| `vbmeta_vendor_patched.img` | 已写入的 vbmeta_vendor（flags=0x3，内部 hash 未重算）。 |
| `seccfg.img` | **修复前**的 seccfg（dm_verity_state=1）。**不要恢复**，否则警告会回来。 |
| `misc.img` | 原始 misc（全 0）。 |
| `expdb*.img` | 各阶段 LK/AVB 日志。 |
| `boot_para.img` | boot_para 备份（全 0）。 |
| `seccfg_fixed.img` | **修复后**的 seccfg（dm_verity_state=0）。与旧 `seccfg.img` 的差异只有 offset 0x10（1→0）和硬件校验和区域。 |

### 手机：`/sdcard/`

- `boot_current.img`、`new-boot.img`
- `vbmeta_current.img`、`vbmeta_rehashed.img`
- `vbmeta_system_current.img`、`vbmeta_system_patched.img`
- `vbmeta_vendor_current.img`、`vbmeta_vendor_patched.img`
- `misc_ota.bin`（4 字节 `01 00 00 00`）
- `seccfg_fixed.img`（修复后 seccfg，dm_verity_state=0，同机恢复可用）
- `recovery-backup/vbmeta_system.img`、`recovery-backup/vbmeta_vendor.img`（修复前分区备份）
- `/data/local/tmp/` 下还有 `expdb*.img`、`boot_para.img`、`misc*.img`、`seccfg*.img` 等临时 dump（可清理）。

---

## 7. 回退 / 应急

- **不要恢复 `seccfg.img`**（dm_verity_state=1），否则 dm-verity 警告会重新出现。
- 如果 dm-verity 警告再次出现，且确认 seccfg offset 0x10 又变成 1，可在 recovery 里写回**修复后**的 `seccfg_fixed.img`（同机备份，硬件校验和有效）：
  ```sh
  dd if=/sdcard/seccfg_fixed.img of=/dev/block/by-name/seccfg bs=4096 conv=fsync
  sync
  ```
- 若 boot/vbmeta 改坏导致不进系统：
  - 进入 **recovery**（电源+音量键菜单，选 Recovery）；recovery 的 adb 是 **root**，且 `/sdcard`、`/dev/block/by-name/*` 可读写。
  - 用 `dd` 写回 `boot_current.img` / 原始 vbmeta 分区，或从 `/sdcard/recovery-backup/` 恢复。
  - 注意：恢复原始 vbmeta_system/vendor（flags=0，hash 正确）通常安全；恢复原始 boot 会回到“需要按电源键”的状态。
- **fastboot 在这台机器上目前进不去**（`adb reboot bootloader` / `reboot bootloader` 都回系统；key combo 也只到 recovery 菜单）。不要依赖 fastboot 回退；recovery root + dd 是当前可用的恢复路径。
- **不要上锁 bootloader**：在 boot/vbmeta 已修改的情况下 `fastboot flashing lock` 会验证失败，可能变砖。要上锁必须先用完整原厂固件恢复所有分区。

---

## 8. 其他注意事项

### 8.1 orange 解锁提示

现在只剩 `Orange State / Your device has been unlocked and can't be trusted`，通常 5 秒后自动继续，不需要按键。要彻底去掉：

1. 备份数据；
2. 刷回完整原厂固件（RMX2117 对应版本）；
3. 恢复 stock boot/vbmeta/system 后 `fastboot flashing lock` / `oem lock`；
4. 代价：失去 Magisk/root，可能清数据。  
如果还需要 root，就保留 orange 提示。

### 8.2 电池 / 供电（与本次警告独立的另一个问题）

- 电池 5 年，SOH 85%，满充 4064mAh；PC 的 USB2 口最大 500mA，实测充电电流约 0.3–0.4A，25% 充满要 10 小时以上；亮屏/Wi-Fi/GMS 时可能入不敷出。
- 电量耗尽会关机并进入 **kpoc 关机充电**（USB `0e8d:20ff`，屏幕充电动画）；此时按电源键退出 kpoc 是正常操作，不是 bootloop。
- 建议：用 5V/2A+ 墙充或带电源的 USB Hub，至少 USB3（900mA）；屏幕保持熄灭；电量 40% 以上再长时间使用；若在好充电器上仍 20%+ 自动关机/循环，换电池。
- 如果手机需要长期挂在 PC 上：可开 **Wi-Fi ADB**（主机与手机在同一局域网可达），把手机插到墙充，数据走 Wi-Fi。

### 8.3 USB 模式 / adb 掉线

- 多次重启后手机会枚举成 `22d9:2046`（MIDI），导致 adb 不可用。请在开发者选项 → **默认 USB 配置** 改为 **文件传输 / File Transfer**，或从 USB 通知里选文件传输。
- 若仍需要稳定 adb，可考虑 Wi-Fi ADB（`adb tcpip 5555` + `adb connect <手机IP>:5555`）。

### 8.4 Magisk 自动救砖模块

- 已安装 `Automatic_brick_rescue`（v221204）：连续快速重启两次会禁用所有模块并重启；排查 bootloop 时先临时 `touch /data/adb/modules/Automatic_brick_rescue/disable`，避免干扰。
- 本次修复过程中它没有触发（无 disable 文件、无救砖日志）。

### 8.5 OTA / 系统更新

- 不要在有 root/修改过 boot/vbmeta 的状态下直接 OTA；OTA 可能重新启用 verity、重置 seccfg 或导致无法开机。
- 如必须升级，先在原厂固件上完成，再重新 root，并重新检查 seccfg 的 `dm_verity_state`。

---

## 9. 相关命令速查

```sh
# 读 seccfg 状态（root）
adb shell "su -c 'dd if=/dev/block/by-name/seccfg bs=1 skip=12 count=8 | od -An -tx1'"

# 读 misc OTA 区域
adb shell "su -c 'dd if=/dev/block/by-name/misc bs=1 skip=4096 count=8 | od -An -tx1'"

# 备份分区（root）
adb shell "su -c 'cat /dev/block/by-name/vbmeta > /data/local/tmp/vbmeta.img'"
adb pull /data/local/tmp/vbmeta.img .

# 写回分区（root；务必确认镜像与分区大小一致）
adb shell "su -c 'dd if=/sdcard/xxx.img of=/dev/block/by-name/xxx bs=4096 conv=fsync; sync'"

# 查看 LK/AVB 日志
adb shell "su -c 'cat /dev/block/by-name/expdb > /data/local/tmp/expdb.img'"
adb pull /data/local/tmp/expdb.img .
strings -n 4 expdb.img | grep -iE '\[avb\]|dm_status|HASH_MISMATCH|boot state'
```

---

## 10. 参考

- MTK seccfg 分区与 dm-verity error（offset 16 = dm_verity_state）：  
  https://zhuanlan.zhihu.com/p/538441745
- mtkclient（USB ID / seccfg / 解锁）：  
  https://github.com/bkerler/mtkclient
- Android Verified Boot（dm-verity warning 流程）：  
  https://source.android.google.cn/docs/security/features/verifiedboot/boot-flow
- realme RMX2117 root/解锁/mtk 教程：  
  https://xdaforums.com/t/root-rmx2117-realme-narzo-30-pro.4394087/
- realme 官方定时开关机（kpoc 之外的无人在场开机兜底）：  
  https://www.realme.com/cn/support/kw/doc/2046181
