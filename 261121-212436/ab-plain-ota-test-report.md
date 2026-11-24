# A/B Plain OTA 真机测试报告（261121-212436 批次，devkit/NVMe）

- 日期：2026-11-23；设备：reComputer RK3576 DevKit（**NVMe**，plain loader，**A/B 味**，非加密）
- 镜像：26.11.0-trunk trixie cli，SEEED_EXTENSION_COMMIT=**92333196**（m-dev tip，与 recovery 批同 commit）
- 定位：A/B 味**首次在 devkit/NVMe 上跑**（此前 A/B 全在 industrial/mmc）；全程按 `.claude/ota-test-methods.md` A/B plain 节，重点观察分区变化
- 分区基线（5 分区，无 security；boot_slot=a，ab_boot_mode=filesystem）：

| 分区 | PARTLABEL | 大小 | 基线 UUID | 烧录态 |
|---|---|---|---|---|
| nvme0n1p1 | boot_a | 512M | fa9aaf09 | active（/boot） |
| nvme0n1p2 | boot_b | 512M | df09526d | 烧录态 |
| nvme0n1p3 | rootfs_a | 3.8G | 2a8820ff | active（/media/root-ro） |
| nvme0n1p4 | rootfs_b | 3.8G | 625c6d13 | 烧录态（21:56 建） |
| nvme0n1p5 | userdata | 110.6G | 018a74e4 | overlay upper |

- 状态基线：pristine（`Mode: unknown`/idle——state 无 OTA_MODE 属预期，首次 start 盖章；`/var/lib/armbian-ota/ota-state.env`）；installed-payload-id=2f9c0bcd

## 负向闸门（6/6 全中）

| # | 输入 | 结果 | 判定 |
|---|---|---|---|
| N1 | recovery 味 zst 包 | 元数据正常读出（**本机 zstd 在位**）→ `recovery OTA backend is not installed on this firmware (have: ab)` | ✓（zst 干净拒绝——industrial 误导性消息实为 zstd 缺失专属，此处复证） |
| N2 | 原始 AB_PART.img.xz | `tar: This does not look like a tar archive` → `metadata missing package.env` | ✓ |
| N3 | resolute gnome AB_PART 包 | `RELEASE mismatch: package=resolute, device=trixie` | ✓ |
| N4 | N3 + `--allow-release-change` | WARN 放行 → `IMAGE_VARIANT mismatch: desktop vs cli` 硬拒 | ✓ |
| N5 | crypt AB_PART 包（加密闸反向） | `encryption mismatch: package=yes, device=no` | ✓ |
| N6 | `rollback`（pristine） | `No OTA mode recorded... Start an OTA first` | ✓（A/B rollback 为 state 分发的用户命令） |

## D1 正向事务（双向 a→b / b→a，含同包重装幂等）

| 阶段 | 证据 | 判定 |
|---|---|---|
| 容量闸 ×2 | rootfs 2554176743B→p4/p3 4095737856B OK、boot 200779671B→p2/p1 OK | ✓ |
| staging | userdata overlay upper（`/media/root-rw/ota-ab/ota_work`）+ space gate | ✓ |
| 校验 | rootfs/boot .sha256 各过 | ✓ |
| 直写分区 | `Updating slot b: root=/dev/nvme0n1p4 boot=/dev/nvme0n1p2`（非 LUKS 分区直写路径） | ✓ |
| **mkfs 保 UUID** | p4/p3 重建（created 时间戳更新）但 **UUID 逐字节保留**（target.sh `-U ${old_uuid}`）——slot 寻址/引导配置不失效 | ✓ |
| boot 分区 | 原地更新不重格式化；新增 `payload-id` | ✓ |
| env 武装（原子性） | `boot_slot=b` 翻转、`ota_in_progress=1`、`slot_retry_left=3`、**`boot_success=a`（回滚锚点=旧确认槽）** | ✓ |
| 引导 | 串口：fsck+挂载 boot_b UUID → `armbian-ota-firstboot.service` 健康检查 | ✓ |
| 确认 | `AB OTA marked successful on slot b` → **boot_success=b 锚点转移**、retry 复位、Status=success | ✓✓ |
| b→a + 同包重装 | 无 INSTALLED_ID 闸（A/B 设计：重装写 inactive slot 天然幂等；state_mark_completed 不记 INSTALLED_ID）直过全链，slot a success | ✓ |

全程 ~73s/事务（NVMe + 本地包）。观察：state 落 `/var/lib/armbian-ota/ota-state.env`（userdata upper 持久跨 slot）。

## 中断注入组

| 测试 | 结果 | 判定 |
|---|---|---|
| SIGKILL @metadata/staging | env 完全未武装（A/B env 武装是最后一步）、state 不变、零副作用 | ✓ |
| SIGKILL @rootfs 解压中段 | 孤儿 xz/tar 继续完成（kill 只杀顶层 bash）、p4 半成品（下轮 mkfs 覆盖）、**env 仍未武装** | ✓ |
| 后续 start 自愈 | `Removing stale lock file` WARN + 重 staging | ✓ |
| **新发现：陈旧挂载无自愈** | 被杀事务遗留 `ab-root-mnt.*` 挂载目标 slot → 后续 start mkfs 拒绝（`is mounted; will not make a filesystem here!`）→ 事务干净失败（运行 slot 无损、env 未动）但 **OTA 卡死需人工 `umount` 或重启**；secure 侧有 stale-mapper 自愈（9111f326），plain 侧缺对应 stale-mount 自愈 | ⚠ 发现 |
| 解卡配方 | `umount .../ab-root-mnt.*`（单条）→ 重试 start 全链 success | ✓ |
| 并发锁 | `Another OTA process is running (PID: N)` + rc=1 | ✓ |

**发现处置**：~~stale-mount 无自愈~~ → **本轮已修复（46596614）+ 设备热补丁实测**，见下节补充轮。

## D4 switch-slot（boot-trial 语义）

| 输入 | 结果 | 判定 |
|---|---|---|
| `switch-slot b`（=当前槽） | `Already booting slot b`，零副作用 | ✓ |
| `switch-slot x` | `Usage: armbian-ota switch-slot [a|b]` | ✓ |
| `switch-slot a`（b→a） | 武装 trial（boot_slot=a、ota_in_progress=1、锚=b）→ 引导 → health → `marked successful on slot a` 锚点转移 | ✓ |

手动换槽不预标 target 成功（代码注释明示：未引导的槽不得 mark，坏槽无法回滚）。

## D3 目标 slot 不可引导（重试阶梯 + U-Boot 自动回滚）

武装事务（target=b）后藏匿 boot_b 的 boot.scr → 重启：

| 阶段 | 证据 | 判定 |
|---|---|---|
| 失败尝试 | 串口 `Scanning nvme 0:2...` → 无 boot.scr → 停 U-Boot 提示符；每次尝试 `Saving Environment to ENV_BLK`（`ab_preboot` 扣 slot_retry_left） | ✓ |
| 自动烧预算 | 板子在失败间**自复位循环**（无需人工/看门狗介入），预算 3→0 | ✓ |
| **U-Boot 层回滚** | `ab_preboot` 检出 ≤0 → boot_slot=boot_success=a、ota_in_progress=0、retry 复位、saveenv+reset | ✓✓ |
| Linux reconcile | `target slot b did not pass boot verification; running confirmed slot a` → **Status=rollback**、target 清空 | ✓ |

（journal 时间戳为 fake-hwclock 回放，偏差 ~45s，无害。）此后恢复 boot.scr 正常。

## 版本互换（history 261116 ↔ 当前批）

| 方向 | 证据 | 判定 |
|---|---|---|
| 降级 a→slot b（h116 包） | 内核 `#1 SMP Sun Nov 15 05:44:38`（原 Nov 20）——真换版本；slot b success | ✓ |
| 升回 b→slot a（当前包） | 内核回 `#1 SMP Fri Nov 20 15:55:18`；slot a success | ✓ |

## 分区变化终核（用户重点项）

| 项 | 基线 → 终态 | 判定 |
|---|---|---|
| GPT 分区表 | 5 分区（boot_a/boot_b/rootfs_a/rootfs_b/userdata）尺寸/LABEL/PARTLABEL/PARTUUID **零变化** | ✓ |
| 分区 UUID | 全部保留（mkfs `-U` 设计） | ✓ |
| slot 内容 | 双向互换 + 多轮重写；`tune2fs created` 时间戳如实更新 | ✓ |
| userdata | overlay upper + OTA staging（事务后清扫 ab-package.*；SIGKILL 残留两个空 `ab-root-mnt.*` 目录，无害） | ✓ |
| image-release | 版本互换后 RELEASE/COMMIT 读取正确（无 upper 遮蔽现象；版本核对仍建议 uname -v） | ✓ |

## N7 计划外闸门 + 中断注入补充轮（自愈修复验证）

| 测试 | 结果 | 判定 |
|---|---|---|
| N7：武装中（ready_to_boot 待重启验证）再 start | `Another AB OTA boot verification is still in progress` 拒绝 | ✓（计划外发现） |
| 自愈修复（46596614）·无孤儿路径 | `Detaching stale target-root mount` → mkfs → staged success | ✓ |
| 自愈修复·孤儿活跃路径 | kill@30% 立即重试 → Detaching 有界等待（~52s 孤儿写完，上限 120s）→ staged success rc=0 | ✓✓ |
| 修复后事务完整闭环 | 重启 → slot success（补丁经 userdata upper 跨 slot 生效） | ✓ |

**发现处置**：stale-mount 无自愈 → **已修复并提交 46596614**（`ab_reformat_target_root` mkfs 前 findmnt→umount，有界等待孤儿；与 9111f326 stale-mapper 自愈对齐）。设备热补丁实测两路径全绿。

## D2 真断电回滚（用户拔电配合，两档全闭环）

| 档 | 操作 | 结果 | 判定 |
|---|---|---|---|
| **多次拔**（耗尽档） | 试引导期间反复拔电 | 每次扣 1 预算 → 3 次耗尽 → **U-Boot 自动回滚锚 slot b** → `Status=rollback`、target 清空；`EXT4-fs (nvme0n1p5): recovery complete`（userdata 断电脏 → 自动恢复）；被切断的试引导 journal 损坏不留 boot 条目（符合预期）；**运行 rootfs/slot 分区零损伤**（fsck clean） | ✓✓ |
| **单次拔**（重试档） | 引导滚动时只拔一次，之后不动 | 扣 1 预算 → 下次启动**重试目标槽 slot a** → `orphan cleanup on readonly fs`（断电残留自清）→ health → `marked successful on slot a` → `Status=success` | ✓✓ |

断电证据链：前序系统干净关机（ramlog rsync+journald flush）→ 试引导中被硬切（无 shutdown 记录 + fs dirty/orphan）→ 阶梯自愈。两档均零人工介入、零数据损伤。

## 残余缺口

| 缺口 | 所需条件 |
|---|---|
| sd*/vd* 盘型 | A/B 味 + 此类盘 |

## 结论

A/B plain 首次在 devkit/NVMe 上手册全流程闭环：负向闸 6/6+N7（含 zst 在 zstd 机上的干净拒绝复证、武装中拒绝新 start）、D1 双向 + 同包幂等、中断组（**新发现 stale-mount 无自愈 → 当轮修复 46596614 并实测**）、D4 boot-trial、D3 失败阶梯全自动（U-Boot 层回滚 + reconcile 分类）、**D2 真断电两档**（单次拔=扣预算重试成功；多次拔=耗尽自动回滚，fs 自恢复零损伤）、版本互换双向、分区表全程零意外。与 industrial/mmc 侧行为一致（NVMe 速度 ~73s/事务）。**A/B plain 手册项全覆盖，残余仅 sd*/vd* 盘型。**
