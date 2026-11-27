# recovery-secure OTA 真机测试报告（批 261126-151330）

图例：✅ 通过 / ❌ 失败 / ⚠ 偏差（有解释）/ ⏸ 阻塞 / ⏹ 延后

- 通道：ssh rk3576（127.0.0.1:50003，重刷后清 host key+推公钥，已 enable ssh）；串口 office screen `USB0`（hardcopy/stuff）
- 设备/介质/loader：reComputer RK3576 DevKit / NVMe / SPI-NOR secure loader
- 镜像身份（/etc/armbian-image-release）：recomputer-rk3576-devkit / **trixie** / **cli** /
  26.11.0-trunk / SEEED_EXTENSION_COMMIT=**8dcba161**（busybug 21a80ff9 + mapper 提取修复 +
  b048f73f/4f70b1b1 raw itb 锚定链入像）/ BUILD_REPOSITORY_COMMIT=ef47b4b-dirty /
  IMAGE_UUID=ca87153e-…
- 形态自检：p1 boot + p2 recovery + p3 security + p4 rootfs(LUKS) + p5 userdata(LUKS) → recovery-secure；cmdline armbian.bootdev=nvme
- 烧录镜像 sha256 71815a9c…（261126-151330 trixie cli-crypt recovery）
- OTA 包：正包（secure rec trixie）sha256 24fc2d80…（PAYLOAD_ID=910438c2==装机 ID，同 ID
  重装语义）；素材 crafted history/261126-151330/（office 制包）
- 基线：boot 0 首启 2026-11-26 15:34 设备钟（慢 ~10h 已校正）；payload-id 基线 910438c2

## 结果总览

| 步骤 | 结果 | 说明 |
|---|---|---|
| B-1 首启 | ✅ | 见下 |
| B-2 Recovery OTA（--force --allow-release-change 合并 2.5 放行链） | ✅ | 全链 apply（60s 收敛）；itb md5 精确；health 自动 commit；**新发现 backup FIT 截断缺陷** |
| B-3 系列负例 | ✅ | 闸 7+staging 4+C 类 8+E 系列 4 全命中（见下） |
| B-4 双坏自愈 | | |
| B-5 收口 | | |
| B-6 中断恢复 | | |
| B-7 V1/V3/V4 | | |
| 2.5 补测 | | |

## B-1 首启（boot 0，journal 261126 15:34）

| 检查项 | 结果 | 说明 |
|---|---|---|
| 身份块 | ✅ | BOARD/RELEASE/IMAGE_VARIANT/VERSION/COMMIT 逐项一致（trixie/cli/8dcba161） |
| SSKR 迁移 | ✅ | journal `86da → No SSKR marker, reading raw passphrase → tee-supplicant started → Password successfully retrieved`；root/userdata 双 mapper ready |
| p3 SSKR marker | ✅ | 头 16B `53 53 4b 52 02`（SSKR v02，首启迁移落盘） |
| userdata resize | ✅ | exit 0；partition→LUKS（239878144 sectors）→ext4 三级单 boot（15:34:46→15:35:02）；userdata 114.4G 至盘尾 |
| fw_env.config | ✅ | 双条目 3 字段（0x3f8000/0x3f0000，父盘 /dev/nvme0n1） |
| init-uboot | N/A | AB 布局专属单元，REC 形态无此项 |
| payload-id 基线 | ✅ | 910438c266b1…（==正包 PAYLOAD_ID，同 ID 重装语义） |
| 首启串口窗口 | ⚠ | 烧录后自启错过；journal 全链补证（同上批口径） |

## B-2 Recovery OTA（载体 `start --force --allow-release-change rel.tar.zst`）

武装（01:48–01:51）：

| 检查项 | 结果 | 说明 |
|---|---|---|
| 2.5 放行链正向 | ✅ | `WARN Cross-release OTA allowed: package=bookworm, device=trixie` → 全链放行（上批待测项闭环） |
| 空间/容量双闸 | ✅ | avail 110.1G/need 6.35G；rootfs 2.56G→4.08G OK；boot 74.3M→512M OK |
| staging+签名链 | ✅ | extract+sha256 → manifest 签名 → LUKS stash 解密 → rootfs/boot checksum 全绿 |
| --force 重复闸 | ✅ | 同 ID（910438c2==装机）穿闸重装语义（B-3b --force 正向并入） |
| backup 建段（本批不并 --no-backup） | ⚠→❌发现 | backup f4c842b7 建段"成功"但 **boot.itb 只捕获 2560B**——见发现 #1 |
| 武装键 | ✅ | in_progress=1/boot_next=recovery；STATUS=prepared/ROOTFS_TOUCHED=0/RETRY=0 |

apply（reboot 后 rescue，01:52–01:53 全程 ~60s 收敛）：

| 检查项 | 结果 | 说明 |
|---|---|---|
| rescue 路由+campaign | ✅ | 一次上电全链：commit 日志 `OTA transaction committed INSTALLED_ID=910438c2`；RETRY_COUNT=1（attempt 计数） |
| itb 写回 | ✅ | p1 前 74,322,432B md5=02b02d42 == 包内 boot.itb（写回精确+尾部零填；BOOT_BYTES=74322497 含 padding 偏差 65B 非缺陷） |
| health 自动 commit | ✅ | `Backup rolled to 910438c2 (previous f4c842b7)` + STATUS=success；health.service Finished |
| boot_next 复位 | ✅ | =normal；in_progress=0 |
| /tmp/syspw | ✅ | 不存在 |
| mark-success 幂等 | ✅ | health 先行 commit 后 mark-success RC=1 `requires an active applied transaction`（=B-3d 预期语义） |
| BACKUP_ID 终态 | ⚠ | commit 后置 910438c2（installed 即 last-good 标记语义，同上批） |
| rescue 串口取证 | ⚠ | 采集脚本会话名误（usb0≠USB0）错过 apply 窗；state/ota.log/md5/health 四源已证，V3 重放补串口 |

**发现 #1（新缺陷）**：external-data FIT 下 backup 捕获截断。p1/package boot.itb FDT 头
totalsize=0x0a00（2560B）≠文件 74,322,432B（mkimage 外置 blob）；`recovery_create_backup`
按 `ota_fit_size_bytes`（读 totalsize）dd 只取 2560B → backup payload 的 boot.itb 缺全部
外置数据。backup boot.sha256 自洽+readback 同样只对 2560B → 校验形同虚设。影响：
campaign 耗尽后 backup restore 会向 p1 写残废 FIT（跨版本场景=头旧尾新混装，U-Boot hash
校验拒 → 砖，须重刷）；同 ID 场景字节恰好等价侥幸无害。主 apply 路径
`write_raw_boot_itb` 按 `file_size_bytes` 写全量，不受影响。上批全程 --no-backup 未暴露。
修复方向：external FIT 总长应取 image 节点 data-offset+data-size 最大值（或捕获
`BOOT_BYTES` 级全量）。

## B-3 / B-3b / B-3c / B-3d / B-3e / 2.5 负例（零重启窗，01:56–02:09）

素材：crafted history/261126-151330/（office 制包）+ 正包/软链 g5/g6/g7。闸序实证：
release/identity 闸先于 INSTALLED_ID 重复闸（rel 无 flag 先撞 RELEASE）；CLI 锁在
metadata 慢读**之前**（并发 start 任何包先撞锁）。

| 检查项 | 结果 | 说明 |
|---|---|---|
| B-3b 重复闸 | ✅ | `already installed (INSTALLED_ID); use --force` RC=1（正包无 flag） |
| G1 BOARD | ✅ | `BOARD mismatch: package=recomputer-rk3588-devkit` RC=1 |
| G2 RELEASE | ✅ | `RELEASE mismatch: package=bookworm …--allow-release-change` RC=1 |
| G3 放行后再拦 | ✅ | WARN `Cross-release OTA allowed` → `IMAGE_VARIANT mismatch: package=desktop, device=cli` RC=1 |
| G4 VARIANT | ✅ | `IMAGE_VARIANT mismatch` RC=1 |
| G5 加密方向 | ✅ | `encryption mismatch: package=no, device=yes` RC=1（plain trixie REC 正包） |
| G6 反向（AB 包） | ✅ | `ab OTA backend is not installed on this firmware (have: recovery)` RC=1 |
| G7 原始镜像 | ✅ | WARN tar 非归档 + `metadata missing package.env` RC=1 |
| 2.5 ambiguous 双 boot | ✅ | `contains both boot.itb and boot.tar.zst; refusing ambiguous boot payload` RC=1 |
| 2.5 篡改 payload | ✅ | `Decrypted payload SHA256 (fc76a7b1…) does not match signed manifest (18dec75…)` RC=1 |
| B-3 坏包 | ✅ | `KIND=full package has no boot payload (dual-payload assertion failed)` RC=1 |
| 2.5 容量闸 | ✅ | `payload=40000000000B, need=44000000000B (margin 110%)` RC=1 |
| 2.5 空间闸 | ✅ | `Insufficient OTA staging space: avail=108146981888, need=203792285226` RC=1（初版 99999999999 在 108G avail 下未触发，改制 200000000000 后命中） |
| C 类八例 | ✅ | 非 root/usage/未知选项（含 supported 列表）/switch-slot ab 拒/rollback not-user-command/mark-success 无事务 全 RC=1；reconcile RC=0（`Recovery mode reconciles on every boot… nothing to do`）；status 非 root RC=0 |
| env/staged 无损 | ✅ | 全程 ota_in_progress=0（B-3 段）→ 1（武装后不被扰动） |

B-3e：

| 检查项 | 结果 | 说明 |
|---|---|---|
| E2 并发锁 | ✅ | 事务中第二个 start `Another OTA process is running (PID: 9263)` RC=1 |
| E3 armed 重复 start | ✅ | 撞 INSTALLED_ID 闸 RC=1；STATUS=prepared/RETRY=0/in_progress=1 不扰动 |
| E4 --force 重武装 | ✅ | RC=0（同 ID 无 orphan 行=预期）；武装终态 prepared/1/recovery/RETRY=0 |
| E1 --no-backup | ✅ | start 日志零 backup 建段/复核段（仅 space 算式行 backup=0 与 campaign 行）；BACKUP_ID=910438c2 保留既有 last-good 非空（B-6 配对语义） |


