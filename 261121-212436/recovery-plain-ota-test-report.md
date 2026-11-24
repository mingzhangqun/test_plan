# Recovery Plain OTA 真机复测报告（261121-212436 批次）

- 日期：2026-11-21 晚 – 11-23；设备：reComputer RK3576 DevKit（NVMe，SPI-NOR loader，recovery 味）
- 本轮 = 上轮 commit 段失败（rescue-default-env patch 缺失 + env bad CRC → health 死循环）后，SPI 重烧复测 + ssh 补测

## 版本确认（均为最新）

| 项 | 设备证据 | 本地核对 |
|---|---|---|
| SPI loader | banner `Nov 21 2026 - 21:30:54`，`Verified-boot: 0`（plain） | 92333196 落地 21:22 → 构建 21:24 → uboot 编译 21:30:54，时间链吻合 |
| env 补丁 | 全程无 bad CRC；`Saving Environment → redundant nvme(0)` 成功 | rescue-default-env patch 已在产物里，上轮缺口①②证据消失 |
| OS | `SEEED_EXTENSION_COMMIT=92333196…`、`BUILD_REPOSITORY_COMMIT=1ce67c2-dirty` | m-dev tip 92333196（clean）；armbian-build pre-release HEAD 1ce67c259 |

## 负向闸门（5/5 全中）

| 输入 | 结果 | 判定 |
|---|---|---|
| secure crypt OTA 包（plain 设备） | `encryption mismatch: package=yes, device=no` | ✓ |
| AB_PART 包（recovery 固件） | `ab OTA backend is not installed (have: recovery)` | ✓ |
| 原始 .img.xz | 非 tar → `metadata missing package.env` | ✓（输入本非 OTA 包） |
| resolute GNOME 包（trixie 设备） | `RELEASE mismatch… --allow-release-change` | ✓（codename 回退闸生效） |
| resolute 包 + `--allow-release-change` | WARN 放行 → `IMAGE_VARIANT mismatch: desktop vs cli` 硬拒 | ✓（variant 闸 plain 首测） |

## 正向事务全链路

| 阶段 | 证据 | 判定 |
|---|---|---|
| start 闸 | space gate（110G ≥ 6.2G）+ 双 capacity gate | ✓ |
| staging | 解压 + 全量 sha256（ada4c9e0…） | ✓ |
| backup | last-good 6433bf8d（1.1G）创建 | ✓ |
| env 武装 | `ota_in_progress=1`、`ota_boot_next=recovery` 写 NVMe 冗余 env；status=prepared（backend v0.8） | ✓ |
| 重启 1 路由 | U-Boot 按 env 引导 rescue（直读 Image/uInitrd/dtb，无 boot.scr），cmdline `ota_role=recovery` | ✓（上轮路由缺口收口） |
| initramfs 事务 | root/boot 探测（nvme0n1p2/p1）；attempt 1/3；mkfs 重建保留 label+UUID；双 payload 解压；marker 校验；armbianEnv.txt/fstab 补 rootdev=UUID；overlay upper 清理；handoff | ✓ |
| 重启 2 正常 | `boot mode: normal` + env 保存；boot.scr 正常执行；systemd 全绿；overlayroot=device；`armbian-ota-health` Finished | ✓（上轮 health 死循环不复现） |
| 终态 | status=success，installed=ada4c9e0，`ota_in_progress=0`；GC 清 backup 只剩 installed | ✓ |

## 断电 fallback 测试（--force 重装 + 恢复期三次断电）

| 阶段 | 证据 | 判定 |
|---|---|---|
| INSTALLED_ID 闸 | 同版本重装无 `--force` 被拒 | ✓ |
| `--force` 准备 | 复用 staged payload；`Existing backup payload verified structurally` | ✓ |
| 断电 ×3（解压中） | attempt 1→2→3 递增；`ROOTFS_TOUCHED=1` 跨启动持久；每次 ext4 journal recovery + 重新 mkfs | ✓ |
| 第 4 次启动 | `retry budget exhausted after 3/3 attempts` → 切 backup 战役（RESTORE_MODE=1）→ `OTA done, rebooting` | ✓ |
| 第 5 次启动（restore） | attempt 1/3、RESTORE_MODE=1、ROOTFS_TOUCHED=0；全链应用 → handoff | ✓ |
| 终态 | `success` + `Restored: yes` + `Last error: retry budget exhausted`；`ota_in_progress=0`；GC 只剩 installed | ✓ |

语义备注：backup==candidate（`--force` 重装）→ 回滚到同版本；异版本 backup 路径 industrial D7 已覆盖。

## ssh 补测轮（2026-11-23）

| 测试项 | 证据 | 判定 |
|---|---|---|
| `armbian-ota rollback` | 硬拒绝文案（v0.8 §15.4）+ RC=1 | ✓ |
| 并发锁守卫（计划外） | `Another OTA process is running (PID: N) / Cannot acquire OTA lock` | ✓ |
| SIGKILL @metadata/staging 边界 | 无孤儿、staging 无残留、env 未武装、状态零污染 | ✓ |
| SIGKILL 陈旧锁恢复 | 遗留锁文件 → 下次 start `Removing stale lock file` WARN 清除 | ✓ |
| SIGKILL @解压中途 | 孤儿 tar/zstd 跑完落 `.incoming` → `rm -rf .incoming` 重解压 | ✓ |
| SIGINT @metadata | INT 被延迟，跑到锁检查才退出——忠实 ^C 依旧无法复现（非缺陷） | 观察 |
| `--no-backup` | space gate 计 `backup=0`；事务提交 success | ✓ |
| `--allow-release-change` | `Cross-release OTA allowed: package=resolute, device=trixie` WARN 放行 → IMAGE_VARIANT 闸硬拒 | ✓ |
| fwenv both-bad rebuild（新 bin 实现） | 双 CRC 毒化（`Cannot read environment, using default`）→ arm 时 `CRC bad…rebuilding full routing set` WARN → `env-sanitize-redundant.bin` 重建（rescue env 含完整默认变量）+ 回读 → 重建路由 env 实战引导 → success | ✓✓ |

杂项：tar "future" 时间戳=无 RTC+未 NTP 时钟偏差；`ethaddr` 报错=vendor 噪声；exhausted 启动 `WARN: umount /mnt/root failed`=清理未挂载路径，均无害。

## 故障注入补测轮（2026-11-23 续，ssh 沙盒）

| 测试项 | 注入手段 | 结果 | 判定 |
|---|---|---|---|
| partial fail-closed（普通失败） | PATH 注入假 `fw_setenv`（exit 1）+ 调 `fw_env_set_multi`（非 full-set） | rc=1、stderr 透传、env blob 逐字节不动 | ✓ |
| partial fail-closed（bad-CRC 措辞） | stub 改报 `Bad CRC` | 同样 rc=1、blob 不动——partial 永不 sanitize | ✓ |
| sanitize-retry 失败兜底 | full-set + 必败 stub | WARN 进 sanitize → `env-sanitize-redundant.bin` dd 入 primary（双条目 config 正确选 redundant bin）→ 重试失败 → `fw_env_poison_crc` 翻 CRC 首字节，blob 与 bin 仅差 byte 1、冗余副本不动、rc=1 | ✓✓ |
| probe redundant 分支 + NUL residue | 真盘：零区 `fw_env_redund_area_written`=false、真 env 区=true | NUL/0xFF 剥离判定正确 | ✓ |
| probe single 分支 + write_config 单条 | loop 挂 `env-sanitize-single.bin`（合法单格式 blob） | probe=single；config 只写一条 | ✓ |
| 机制澄清 | — | 冗余格式 blob 单条目读必 CRC 失配（CRC 覆盖 flags 后数据）→ 这正是 probe 判 redundant 的原理；单副本"bad"读数是预期非故障 | — |

设备终态：env 经真 `fw_setenv` 冗余写重建（primary 恢复有效+活跃），`in_progress=0`，双条目读干净。

## store 清空 + `--no-backup` 全周期（2026-11-23 终轮）

| 阶段 | 证据 | 判定 |
|---|---|---|
| 清空 store 后 start | space gate `backup=0`（不预留）；无 "Creating last-good backup" 步骤（跳过创建/重打包）；从包内全新 staging | ✓ |
| BACKUP_ID 语义 | 仍 = installed（同版本重装回退天然保留）——`--no-backup` 只省创建，不破回退 | ✓（语义澄清） |
| 提交 | 重启 → success、`in_progress=0`、store 重建为 installed 单条 | ✓ |

「backup=none 无回退」分支需不同版本 cli 包触发（同版本重装不适用），归残余。

## 版本互换与 backup=none 失败终态（2026-11-23 终轮，用 /mnt/imgs/history/261120-045308 包）

**E：版本互换双向（payload edf60fc4 ↔ ada4c9e，内核/运行时随换）**

| 阶段 | 证据 | 判定 |
|---|---|---|
| E1 降级 261121→261120 | backup≠candidate 结构复用（无重打包）；跨版本 apply；应用后内核=11-19 旧内核、runtime 无 sanitize bins、INSTALLED=edf60fc4 | ✓ |
| E2 旧 runtime 升回 | 261120 代码 start 新包成功（跨版本 start）；提交后内核/runtime/INSTALLED 全回 261121 | ✓ |
| 观测怪癖 | image-release 显示 92333196——upper 里陈旧文件遮蔽新 lower（OTA 按设计只清 lib/modules/overlayroot.conf，其余 upper 保留）；也解释跨 OTA ssh host key 稳定 | 非缺陷 |

**F：backup=none 耗尽终态（状态重置手术 + 毒化候选 + 串口人工兜底）**

| 阶段 | 证据 | 判定 |
|---|---|---|
| 真 backup=none 达成 | state 重置为 pristine + `start --no-backup` 异版本包 → `Campaign prepared: backup=`（空）、Installed/backup 均 none | ✓ |
| 毒化候选 | rm 候选 rootfs.tar.zst → 三次 attempt 均 `payload structure incomplete` **在 mkfs 之前**（ROOTFS_TOUCHED=0，rootfs 全程未动） | ✓ |
| 耗尽终态 | 无 backup 无切换 → `state_mark_failed` → `RESCUE SHELL: no automatic path left` + shell on ttyFIQ0 | ✓✓ |
| 操作员兜底 | 串口 `fw_setenv ota_in_progress 0; reboot -f` → rootfs 完好正常引导（fsck clean） | ✓ |
| failed 终态记录 | Status=failed、retry=3、rootfs_touched=0、Last error=structure incomplete、无 Restored | ✓ |
| failed→新 start 修复 | 新事务放弃失败事务 + 孤儿清扫 + 全新 backup 打包（1b2914≠candidate，重打包分支复覆盖）→ 提交 success | ✓ |

## plain loader + secure OS 组合测试（G，2026-11-23 续，串口实测）

| 观测 | 判定 |
|---|---|
| FIT 在 plain loader 引导成功（hash 链校验过，无 C 守卫拒绝） | **按 4d679b1e 设计**——plain 侧 do_bootm/do_boot_fit 守卫被有意移除（保 rescue fallback 链；plain 契约=未验证层级），0c185134 的 C 闸方案已废弃 |
| env stub 未拦（`PLAIN: raw FIT boot refused` 仅覆盖 plain 自产 env；secure 镜像 prefill env 带真 boot_fit 路由被执行） | 预期，文档明示的 accepted trade-off |
| `raw passphrase path requires OP-TEE (no /dev/tee*)` → 手动口令提示，secure OS 无法完成启动 | **TEE 闸兜底实测生效** ✓ |
| 附带覆盖：security partition marker 解析（86da→p3）、secure 5 分区布局、GPT 备份头错位告警（cosmetic） | ✓ |

## 残余缺口

| 缺口 | 所需条件 |
|---|---|
| 真跨 release 全量切换（trixie↔resolute） | 缺 resolute cli 包（现有 resolute 包为 GNOME，被 IMAGE_VARIANT 闸硬拒） |
| sd*/vd* security-part scan | A/B 味 + 此类盘（本设备无）；mmc 锚定 industrial 已覆盖 |

（原「plain U-Boot 拒 raw FIT 需串口验证」已由 G 项销掉：结论=按设计不拒，TEE 闸兜底有效。）

坑位备注：`pkill/pgrep -f` 会匹配自身命令行（两次误杀 ssh 会话）——远端清理用精确 PID。

## 结论

四闸门 + recovery plain OTA 全事务真机闭环，上轮路由级缺口确认修复；断电 fallback 阶梯、fwenv both-bad rebuild（新 bin 实现）、故障注入组、版本互换双向、backup=none 耗尽终态（含串口人工兜底与 failed 后修复）在 devkit/NVMe 全部复现，与 industrial 行为一致。剩余缺口 3 项均需外部条件，不阻塞 m-dev。
