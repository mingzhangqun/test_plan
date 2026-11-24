# Recovery Secure OTA 真机测试报告（261121-212436 批次）

- 日期：2026-11-23；设备：reComputer RK3576 DevKit（NVMe，SPI-NOR **secure loader**，recovery 味，LUKS）
- secure loader + secure OS 配对组合（对照此前 mismatch 测试：同对全通 / 跨 loader TEE 闸拦截）
- 方法：`.claude/ota-test-methods.md` 手册全流程

## 首启验证（flash 后第一次启动，串口）

| 检查点 | 证据 | 判定 |
|---|---|---|
| secure loader | banner `…-H9790-V82a0… (21:29:48)`（plain=H22d0/21:30:54）；`Cmd interface: disabled`；SPI FIT 含 **optee 条目** hash 校验过 | ✓ |
| BL32 链 | `BL31: Initializing BL32` → OP-TEE 3.13 → 内核 `optee: initialized driver`；fwver 带 `bl32-v1.08` | ✓ |
| OS FIT 验签 | 各子镜像 `Verifying Hash Integrity ... sha256,rsa2048:dev+ OK`（CONFIG_FIT_SIGNATURE） | ✓ |
| 自动解密 | security p3 marker `86da`（首启明文口令窗口）→ tee-supplicant → 口令取回 → root+userdata LUKS 双解锁 | ✓ |
| **SSKR 首启迁移** | 二次读取 p3 marker = **SSKR**（明文口令已进 TEE keybox，明文清除） | ✓ |
| 加密 overlayroot | upper=/dev/mapper/armbian-userdata（encrypted），/ 108G 只读 | ✓ |
| LUKS 一段式扩容 | `dm-1 resizing 505856→…→119939072`（977a40a6 路径复现） | ✓ |
| raw-FIT 布局 | `[seeed-mount-boot] no boot anchor recorded; skipping`、firstboot-hook skip——fs 组件按设计不适用 | ✓ |
| 状态 | `Mode: unknown / Status: idle`（pristine）；fw_env.config 双条目在位 | ✓ |

## 负向闸门（6/6 全中）

| 输入 | 结果 | 判定 |
|---|---|---|
| `armbian-ota rollback` | pristine 下报 "No OTA mode recorded… Start an OTA first" + RC=1 | ✓ |
| plain trixie 包（secure 设备） | `encryption mismatch: package=no, device=yes`（加密闸反向） | ✓ |
| AB_PART crypt 包 | `ab OTA backend is not installed (have: recovery)` | ✓ |
| 原始 .img.xz | 非 tar → `metadata missing package.env` | ✓ |
| resolute gnome-crypt 包 | `RELEASE mismatch: package=resolute, device=trixie` | ✓ |
| 同包 + `--allow-release-change` | WARN 放行 → `IMAGE_VARIANT mismatch: desktop vs cli` 硬拒 | ✓ |

## 正向事务与注入（全部 ssh 可达项完成）

| 测试项 | 证据 | 判定 |
|---|---|---|
| 并发锁 | `Another OTA process is running (PID)` / `Cannot acquire OTA lock` | ✓ |
| SIGINT @metadata | INT 延迟（跑到锁检查/后续阶段才退出），与 plain 侧观察一致 | 观察 |
| SIGKILL @中途 | 孤儿 tar 落 `.incoming`；后续 start `Removing stale lock file` + 重解压 | ✓ |
| **正向全事务** | space gate → staging（**manifest 签名验证 → 解密 rootfs.tar.zst.enc → SHA256 对账**，secure 专属链）→ backup 打包（b92fc52，LUKS lower）→ 武装 → 重启 raw-FIT Rescue 应用 → health → **success** | ✓✓ |
| INSTALLED_ID 闸 | 同包重装被拒（提示 --force） | ✓ |
| `--force --no-backup` | space gate `backup=0` + 无打包步骤 + 结构复用；提交 success | ✓ |
| fwenv partial fail-closed ×2 | 普通失败/bad-CRC 措辞均 rc=1、blob 逐字节不动 | ✓ |
| fwenv sanitize+poison 兜底 | WARN 进 sanitize → redundant bin 落 primary → poison 仅差 CRC 首字节 | ✓ |
| probe single 分支 | loop 挂 env-sanitize-single.bin → PROBE=single | ✓ |
| 意外实证：冗余副本兜底 | full-set 注入打坏 primary 后 fw_printenv 落到冗余副本（陈旧武装态复活）→ 人工 `fw_setenv ota_in_progress 0` 矫正——redundant scheme 兜底 + both-bad 重建必要性双证 | ✓（计划外） |
| 版本互换双向 | E1 降级 261121→261120（INSTALLED=8d8bf01、内核 11-19、旧 runtime 结构复用 backup≠candidate）；E2 旧 runtime 升回（跨版本 start、内核/INSTALLED 全回 26.1.172/4944936）；SSKR 全程保持 | ✓✓ |

## 断电 fallback（用户配合拔电 ×3）

| 阶段 | 证据 | 判定 |
|---|---|---|
| 武装 | candidate=8d8bf01（261120 降级）≠ backup=4944936（结构复用）——**backup≠candidate 真回滚配置** | ✓ |
| 断电 ×3 | 每次均切在 rootfs 解压 heartbeat 后；attempt 1→3 自动递增 | ✓ |
| 耗尽切换 | 第 4 次启动 `campaign 4944936… (RESTORE_MODE=1) attempt 1/3`——backup 战役自动接管 | ✓ |
| 备份战役抗断电 | 用户在 backup attempt 1 又多拔一次——backup 自带独立 3 次预算，重试无副作用 | ✓（计划外） |
| restore 路径 | raw-FIT 布局：mount LUKS root → `mkfs path skipped: no separate boot partition` → clean+extract | ✓ |
| 终态 | `success` + `Restored: yes (rolled back to backup 4944936)` + `Last error: retry budget exhausted`；内核回 261121（Nov 20）；env 解除 | ✓✓ |

小观察：失败候选 8d8bf01 滞留 store（GC 在下次 start 清扫，非缺陷）。

## backup=none 耗尽终态（F，含串口人工兜底）

| 阶段 | 证据 | 判定 |
|---|---|---|
| 真 backup=none 达成 | state 手术 pristine + `--no-backup` 异版本 → `Campaign prepared: backup=`（空） | ✓ |
| 毒化候选 | rm 解密后的 rootfs.tar.zst（保留 .enc）→ 3 次 attempt 均 `payload structure incomplete` 在破坏性写**之前**（ROOTFS_TOUCHED=0，rootfs 未动） | ✓ |
| 耗尽终态 | 无 backup 无切换 → `state_mark_failed` → RESCUE SHELL on ttyFIQ0 | ✓✓ |
| 操作员兜底 | 串口 `fw_setenv ota_in_progress 0; reboot -f` → rootfs 完好正常引导 | ✓ |
| failed 终态记录 | Status=failed、retry=3、rootfs_touched=0、Last error=structure incomplete、无 Restored | ✓ |
| failed→新 start 修复 | 新事务放弃失败事务 + 孤儿清扫 + 全新 backup 打包（7aa1674）→ 提交 success | ✓ |

## 残余缺口

| 缺口 | 所需条件 |
|---|---|
| 真跨 release 全量切换（trixie↔resolute 同 variant secure 包） | 缺 resolute cli crypt 包 |
| sd*/vd* security-part scan | A/B 味 + 此类盘 |

（secure 侧手册项全部覆盖；首启 fail-fast 提交 `515ef74d` 不在本批次镜像，下批次生效。）

## 结论

secure loader + secure OS 配对下，手册全流程真机闭环：负向闸 6/6、正向全事务（manifest 签名+解密链）、中断注入组、fwenv 故障注入（含冗余副本兜底意外实证）、版本互换双向、断电 fallback（**backup≠candidate 真回滚** + backup 战役自带独立重试预算）、backup=none 耗尽终态（串口人工兜底 + failed 后修复）。与 plain 侧行为一致，与 11-16 首轮 secure recovery 测试互为印证。残余 2 项需外部条件，不阻塞 m-dev。

## secure loader + plain OS 组合（loader/OS mismatch 矩阵闭环）

| log 证据 | 判定 |
|---|---|
| `Cmd interface: disabled` + `Unknown command 'source'` → `SCRIPT FAILED`（boot.scr 读入 11622B 但无法执行） | secure Kconfig 裁剪未验证启动命令生效 ✓ |
| 引导链走 plain env 路由（ota_normal/rescue 失败 → distro_bootcmd → scan /boot.scr） | secure loader 忠实执行外来 env，逐条命令被裁剪拦住 ✓ |
| `Unknown command 'pxe'`、`mtd_boot not defined`、UFS/mmc 扫描耗尽 | 所有兜底路径逐个耗尽 ✓ |
| `### ERROR ### Please RESET the board ###` | 干净停机，**零未验证代码执行**（无 kernel） ✓✓ |

四象限矩阵终态：plain/plain=正常；secure/secure=验签链正常；plain loader+secure OS=TEE 闸在解密层拦截（层 3）；secure loader+plain OS=**停在 loader 本身**（层 1，最干净）。不对称为有意设计（docs/06 §8）。此项即 D9-D11 中「loader 拒 plain OS」的实测闭环。
