# recovery-secure OTA 真机测试报告（批 261126-090103，全覆盖轮）

图例：✅ 通过 / ❌ 失败 / ⚠ 偏差（有解释）/ ⏸ 阻塞 / ⏹ 延后

- 通道：ssh rk3576；串口 office screen `USB0`（hardcopy/stuff）
- 设备/介质/loader：reComputer RK3576 DevKit / NVMe / SPI-NOR secure loader
- 镜像身份（/etc/armbian-image-release）：recomputer-rk3576-devkit / **resolute** / **desktop**
  / 26.11.0-trunk / SEEED_EXTENSION_COMMIT=**08834155**（libyaml 修复含入）/
  BUILD_REPOSITORY_COMMIT=ef47b4b-dirty / IMAGE_UUID=ad013a55-…；
  镜像 sha256 cc8981d8…（261126-090103 resolute gnome-crypt_desktop recovery）
- 变体注记：本 tag 另含 trixie/cli 套件（后成）；烧录取先就绪的 resolute/desktop，
  形态级断言不受 release/variant 影响，闸面以本镜像身份为准重跑
- OTA 包 sha256：pkg-resolute 7a3a39ef…（本批主载体，PAYLOAD_ID=4279de3f==装机 ID，
  同 ID 重装语义）；pkg-trixie-secure 0356c0ff…（cap/space 闸改装基底）
- 基线：2026-11-26 首启 boot 0（09:21 设备钟，慢 ~2h 已校正）；payload-id 基线 4279de3f

## 结果总览

| 步骤 | 结果 | 说明 |
|---|---|---|
| B-1 首启 | ✅ | 身份块/SSKR 迁移/resize/fw_env.config 全过；首启串口窗口错过（journal 补证） |
| B-2 Recovery OTA（--force --no-backup 合并 E1） | ✅ | 全链 apply 无 libyaml 注入（rescue 真机闭环）；itb md5 精确；health 自动 commit |
| B-3 负例（坏包/AB 包） | ✅ | mode 闸先于身份闸；dual-payload 断言拒 |
| B-3b flags（重复闸/--force/--no-backup） | ✅ | 重复闸拒/--force 重装=B-2/--no-backup=E1 |
| B-3c 闸门负例 G1–G7 | ✅ | 七例全 RC=1 精确命中（G6 反向=AB 包） |
| B-3d 命令面 C 类 | ✅ | 八例全预期（reconcile RC=0 信息性、status 无 root 守卫） |
| B-3e 参数组合/重复执行 | ✅ | 并发锁/armed 重复不扰动/--force abandon 重武装 |
| B-4 双坏自愈 | ❌→✅(修复) | busybox ls 列布局缺陷（新发现）；修复 21a80ff9 + 忠实复现验证；boot 期终验待重建镜像 |
| B-5 收口 | 待测 | |
| B-6 中断恢复 | ⏹ | 远程中断不可达（FIQ/BREAK 均被 tty 吞）；物理断电待用户在场 |
| B-7 V1 FIT 确认后损 | ✅ | rescue shell fail-stop 精确命中 + 串口手工修复闭环 |
| B-7 V3 FIT 武装态自愈 | ✅ | 单轮重放全量重写（129.7s applied） |
| B-7 V4 rootfs 武装态自愈 | ✅ | 尾损 256KiB 全量重写自愈，零 I/O 残留 |
| B-7 V2 rootfs 确认后损 | ⏹ 不覆盖 | 用户裁定：commit 后介质级损坏非 OTA 契约（V3/V4 事务窗口自愈已验）；记已知限制一条 |
| 2.5 补测 | ✅×4 | ambiguous/篡改 payload/容量闸/空间闸全拒；放行链+无锚定待测 |

## B-1 首启

| 检查项 | 结果 | 说明 |
|---|---|---|
| 身份块 | ✅ | BOARD/RELEASE/IMAGE_VARIANT/VERSION/COMMIT 逐项一致（resolute/desktop/08834155） |
| SSKR 迁移 | ✅ | journal `86da → raw passphrase → tee-supplicant → retrieved`；p3 头 marker=SSKR 实证 |
| userdata resize | ✅ | exit 0；partition→LUKS→ext4 三级单 boot；userdata 116.9G(1k)blocks 盘尾 |
| fw_env.config | ✅ | 双条目 3 字段（0x3f8000/0x3f0000，父盘 /dev/nvme0n1） |
| init-uboot | N/A | AB 布局专属单元，REC 形态无此项 |
| 首启串口窗口 | ⚠ | 错过（烧录后自启）；journal 全链补证 |
| payload-id 基线 | ✅ | 4279de3f55a580337ac5ee65573d6a60f85043ec70bb30cc7963a7c41ee6fbe0 |

## B-2 Recovery OTA（首轮事务，载体 `start --force --no-backup pkg-resolute`）

武装（261126 12:11）：

| 检查项 | 结果 | 说明 |
|---|---|---|
| 空间/容量双闸 | ✅ | avail 107.3G/need 6.4G；rootfs 4.60G→7.15G 容量 OK；boot 97.4M→512M OK |
| staging+签名链 | ✅ | extract+sha256 → manifest 签名 → LUKS stash 解密 → rootfs/boot checksum 全绿 |
| --no-backup（E1） | ✅ | start 日志零 backup 建段/复核段；BACKUP_ID 空（fresh flash 无 last-good） |
| 重复闸/--force | ✅ | 同 ID 包 --force 穿闸重装语义（B-3b --force 正向并入本事务） |
| 武装键 | ✅ | in_progress=1/boot_next=recovery；STATUS=prepared/ROOTFS_TOUCHED=0/RETRY=0 |

apply（reboot 后 rescue 全程串口取证 `uart/uart-b6-*.txt`）：

| 检查项 | 结果 | 说明 |
|---|---|---|
| **libyaml 修复真机闭环** | ✅ | apply 全程**无 RAM 注入**：env 对账/flip 正常（对比 261126-063300 批每次 rescue 落 SHELL） |
| rescue 路由 | ✅ | cmdline `armbian.ota_role=recovery`；Boot disk from cmdline: /dev/nvme0n1 |
| campaign 重识别 | ✅ | `campaign 4279de3f attempt 1/3` → cleaning → extract（zstd+tar，~92s）→ applied |
| itb 写回 | ✅ | p1 前 97,365,504B md5=b3c853c3 == 包内 boot.itb（写回精确） |
| health 自动 commit | ✅ | `STATUS=success`；handing off to normal → health Finished（6.4s） |
| boot_next 复位 | ✅ | =normal（08834155 recovery_commit_success） |
| /tmp/syspw | ✅ | 不存在 |
| BACKUP_ID 终态 | ⚠ | commit 后置 4279de3f（installed 即 last-good 标记语义，非 backup 建段） |

注记：apply 完成行 `handing off to normal` 在串口缓冲内命中（本批未滚丢）；
payload-id 同 ID 重装语义不变（4279de3f）。远程中断尝试记录见 B-6 节。

## B-3 / B-3b / B-3c / B-3d / B-3e / 2.5 staging 负例（零重启窗，12:30–12:45）

素材：crafted/（office 制包：resolute 基底 sed package.env / 删成员 / 加假成员 /
篡改 .enc；trixie 基底身份改装做 cap/space）+ 软链 g5/g6/g7。全部带 `--force` 穿
INSTALLED_ID 重复闸（闸序发现复现：重复闸先于 extract 阶段断言）。

| 检查项 | 结果 | 说明 |
|---|---|---|
| B-3 AB 包 mode 闸 | ✅ | `ab OTA backend is not installed (have: recovery)` RC=1；mode 闸先于身份闸（trixie AB 包可直接用） |
| B-3 坏包 | ✅ | `KIND=full package has no boot payload (dual-payload assertion failed)` RC=1 |
| B-3b 重复闸 | ✅ | `already installed (INSTALLED_ID); use --force` RC=1 |
| G1 BOARD | ✅ | `BOARD mismatch: package=recomputer-rk3588-devkit` RC=1 |
| G2 RELEASE | ✅ | `RELEASE mismatch: package=bookworm …--allow-release-change` RC=1 |
| G3 放行后再拦 | ✅ | WARN `Cross-release OTA allowed` → `IMAGE_VARIANT mismatch` RC=1 |
| G4 VARIANT | ✅ | `IMAGE_VARIANT mismatch: package=cli, device=desktop` RC=1 |
| G5 加密方向 | ✅ | `encryption mismatch: package=no, device=yes` RC=1（plain resolute 正包） |
| G6 反向（AB 包） | ✅ | 同 B-3 第一例 |
| G7 原始镜像 | ✅ | `metadata missing package.env` RC=1（+tar WARN） |
| 2.5 ambiguous 双 boot | ✅ | `contains both boot.itb and boot.tar.zst; refusing ambiguous boot payload` RC=1 |
| 2.5 篡改 payload | ✅ | `Decrypted payload SHA256 (a9e657…) does not match signed manifest (4238b96…)` RC=1 |
| 2.5 容量闸 | ✅ | `payload=40000000000B … partition=7152336896B` RC=1 |
| 2.5 空间闸 | ✅ | `Insufficient OTA staging space … need=105834367679` RC=1 |
| C 类八例 | ✅ | 非 root/usage/未知选项/mark-success 无事务/switch-slot/rollback 全 RC=1；reconcile RC=0 信息性；status 非 root RC=0（只读设计） |
| env/staged 无损 | ✅ | 全程 ota_in_progress=0；staged payload 目录完整 |

B-3e（12:41–12:44）：

| 检查项 | 结果 | 说明 |
|---|---|---|
| E2 并发锁 | ✅ | 事务中第二个 start `Cannot acquire OTA lock` RC=1（实际措辞，计划原文为 PID 变体） |
| E3 armed 重复 start | ✅ | 撞 INSTALLED_ID 闸 RC=1，STATUS=prepared/CANDIDATE 不扰动 |
| E4 --force abandon | ✅ | RC=0 重武装；`Existing backup payload verified structurally`（懒 backup 复用）；无 orphan 行=同 ID 无孤儿（计划原文行仅异 ID 时出现） |
| E4 武装终态 | ✅ | in_progress=1/boot_next=recovery/RETRY=0 → 作 V3 载体 |

## B-7 V3（武装态 p1 FIT 损坏 → 自动修复）

E4 武装后 `dd 1MB` 零进 p1 头（md5 前后 b26b0780→待验）→ reboot。

| 检查项 | 结果 | 说明 |
|---|---|---|
| rescue 路由 | ✅ | 武装 env 直路由 p2 recovery FIT（`ota_role=recovery`+bootdev token） |
| campaign 重识别 | ✅ | `attempt 1/3 (RESTORE_MODE=0)` → clean → extract → `applied 4279de3f; handing off to normal`（129.7s） |
| 单轮收敛 | ✅ | 一次上电全链完成，回 normal health Finished |
| FIT 重写 | ✅ | p1 前 97,365,504B md5=b3c853c3 == 包内 boot.itb |
| 终态 | ✅ | STATUS=success / 0 / normal / syspw 净；RETRY_COUNT=1（attempt 计数信息项） |
| 串口证据 | ✅ | `uart/uart-v3-*.txt` |

## B-7 V4（武装态 rootfs 尾损坏 → 全量重写自愈）

重武装（--force）→ `dd 256KiB` 零进 p4 尾（sector 14001576 起，保 LUKS 头）→ reboot。

| 检查项 | 结果 | 说明 |
|---|---|---|
| 重放自愈 | ✅ | rescue 单轮 attempt 1/3 全量重写 rootfs（131.6s applied） |
| 终态 | ✅ | STATUS=success / 0 / normal / syspw 净 / FIT md5 精确 |
| I/O 残留 | ✅ | dmesg 零 `I/O error|EXT4-fs error`（重写彻底） |
| 串口证据 | ✅ | `uart/uart-v4-*.txt` |

## B-7 V1（确认后 p1 FIT 损坏 → fail-stop shell + 手工修复）

确认态（in_progress=0/success）`dd 1MB` 零进 p1 头 → reboot。

| 检查项 | 结果 | 说明 |
|---|---|---|
| U-Boot normal 失败 | ✅ | `FIT: No fit blob` / `FIT: No FIT image` |
| fallback 路由 | ✅ | bootcmd 顺延 `run ota_rescue_boot` → p2 FIT 起（ota_role=recovery） |
| fail-stop shell | ✅ | `OTA_RESCUE: Rescue entered with ota_in_progress=0 (normal boot failed or manual entry); fix and reboot manually (shell on ttyFIQ0)` 精确命中 |
| rootfs 安全 | ✅ | shell 只读不碰 rootfs（人工修复语义） |
| 手工修复闭环 | ✅ | 串口 `dd staged/boot.itb → p1`（conv=fsync）+ sha256sum 对 boot.sha256 全等（cf3760ff）→ reboot 回 normal |
| 修复验证 | ✅ | FIT 签名引导通过、normal 健康（U-Boot boot_fit 验签即修复证明） |
| 串口证据 | ✅ | `uart/uart-v1-*.txt`（含交互修复全程） |

## B-7 V2（确认后 rootfs 损坏）⏹ 用户裁定不覆盖（261126）

裁定：commit 后介质级损坏（bit rot/外部写）非 OTA 契约；事务窗口内损坏自愈由 V3/V4 覆盖且
全过；最坏情况恢复=maskrom 重刷，逃生通道存在。V6（三重故障）同理不测。

三次递进实验（保留为取证，证据 `uart/uart-v2*.txt`）：

| 尝试 | 破坏 | 结果 |
|---|---|---|
| 1 | p4 尾 512B 零（计划原配方） | ⚠ initramfs `fsck -a` 自愈，正常启动——配方不足以触发挂载失败 |
| 2 | 裸分区层全灭主+8 备超级块 | ⚠ 仍自愈——**层次陷阱**：裸 p4 写的是 LUKS 密文层，运行中 fs 页缓存会在 flush 时经 mapper 重写主 sb（密文层破坏可被页缓存回写覆盖） |
| 3 | **mapper 层**全灭 9 超级块 + drop_caches + 立即 `reboot -f`（跳过干净卸载） | ❌ 见下 |

尝试 3 实际结果（保留为**已知限制**记录，非待办）：

- fsck -a **部分修复**后挂载成功 → 运行时 ext4 illegal pblock 刷屏——系统带伤跛行而非 panic
- **已知限制**：health 只对 payload-id，无 fs 完整性感知（无 campaign+marker 匹配即过）；
  跛行态远程入口全灭（串口 login/sshd kex/FIQ+SysRq/bootdelay=0）——恢复=人工断电，
  落 initramfs shell 可串口救援、再跛行则 maskrom 重刷
- 设计预期"panic→重入 rescue shell"在 fsck 可修复的现实损坏下难触发（尝试 1/2 均自愈）

### V2 附带实证（操作层）

- rescue shell 手工修复后平凡 `reboot` 不触发重启（initramfs 信号语义），须 `reboot -f`
- LUKS 分层：ext4 破坏须走 mapper 层（明文层）；裸分区 dd=打密文，且会被运行中 fs 的
  flush 回写覆盖——构造 rootfs 损坏的标准手法=**mapper 层 dd + drop_caches + reboot -f**

## B-4 双坏自愈（❌ 真失败 → 修复 21a80ff9）

| 检查项 | 结果 | 说明 |
|---|---|---|
| 双坏 env 同 boot 重建 | ❌ | init-bottom 零输出，双 blob 零未重建（新缺陷，非 libyaml） |
| fail-open 兜底 | ✅ | 默认 env 引导 + scan order 解锁照常进 normal（设计内） |
| 断点定位 | ✅ | `find_user_data__raw_partition_of /dev/mapper/armbian-root` initramfs（busybox）返回 1 |
| 根因 | ✅ | busybox `ls -l` 5 列（无 owner/group/date）vs GNU 8 列；awk `$5:$6` 取到路径非 `254:0` → mapper 反查失败 → zz-ota-normal-marker 静默 return |
| 修复 | ✅ | find-userdata.sh maj:min 提取列布局无关（21a80ff9，本地未推） |
| 修复验证 | ✅ | p1 FIT 解包 initramfs + unshare/chroot busybox 忠实复现：全链至 `fw_env_write_routing_set 0/normal` 真写成功 |
| 重建终态+重启回归 | ✅ | 冗余副本 flags=01、路由集全量、`armbian.bootdev=nvme` token 回归、journal 零警告 |
| 主副本仍零 | ⚠ | libubootenv 双坏重建写 inactive 位（单副本落盘），下次写补齐；冗余读不受影响 |
| boot 期自动重建终验 | ⏸ | p1 FIT 签名不可原位补丁，待含 21a80ff9 的重建镜像 |

取证：`uart/uart-b4-{1..8}.txt`（坏 env boot 全程）、`uart/uart-b4-rb{,2}.txt`（重建后 boot）。
影响面：唯一 `ls -l` maj:min 解析点；rescue 侧 device.sh 同路径有 PARTLABEL fallback（历批 rescue
正常的原因）；runtime 为 GNU ls 不受影响。

## B-6 中断恢复 ⏹

- 载体武装+无中断完整 apply 已在 B-2 覆盖；**断电中断不可远程**：FIQ debugger 输入与
  serial BREAK+SysRq 均被 initramfs 控制台吞（仅回显）——断电须用户在场
- armbianEnv 增验：raw-FIT 形态结构性 N/A（布局无 armbianEnv.txt，捕获/合并链不启用）
- 待用户在场：武装 → 上电 → ~40s 断电于解包窗（resolute 4.6G rootfs 解包窗更宽，约 34–130s）
  → 再上电验证 <2min 重放收敛
