# recovery-secure OTA 真机测试报告（批 261126-151330）

图例：✅ 通过 / 🚨 失败 / ⚠ 偏差（有解释）/ ⏸ 阻塞 / ⏹ 延后

- 通道：ssh rk3576（127.0.0.1:50003，重刷后清 host key+推公钥，已 enable ssh）；串口 office screen `USB0`（hardcopy/stuff）
- 设备/介质/loader：reComputer RK3576 DevKit / NVMe / SPI-NOR secure loader
- 镜像身份（/etc/armbian-image-release）：recomputer-rk3576-devkit / **trixie** / **cli** /
  26.11.0-trunk / SEEED_EXTENSION_COMMIT=**8dcba161**（busybug 21a80ff9 + mapper 提取修复 +
  b048f73f/4f70b1b1 raw itb 锚定链入像）/ BUILD_REPOSITORY_COMMIT=ef47b4b-dirty /
  IMAGE_UUID=ca87153e-…（批内硬停事故后重刷同镜像→6914491b，见总览注记）
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
| B-7 V3 FIT 武装态自愈 | ✅ | 单轮重放 FIT 精确复原（02b02d42） |
| B-7 V4 rootfs 武装态 | 🚨→发现 #2 | mapper 层元数据损坏不可自愈（mkfs 快路径 raw-FIT 布局跳过）；全梯耗尽 fail-stop shell；手工修复闭环 |
| B-7 V1 FIT 确认后损 | ✅ | fail-stop shell 精确命中 + 串口手工修复闭环（见下） |
| 2.5 无锚定 find_raw_boot_dev | ⏹ 不可现场构造 | FIT 内建 bootargs 恒含 bootdev token，env 层剥离无效；发现 #3 记录 |
| B-4 双坏自愈 | 🚨→发现 #4 | in-boot 重建静默 no-op（3× 复现）；chroot/运行态链路全通、脚本版本已核实为修复版；根因未定位，留下批 |
| B-5 收口 | ✅ | 解密锚启动盘 nvme、无外接盘、终态记录 |
| B-6 中断恢复 | ✅ | 断电落于 itb 写回后/applied 持久化前窄窗 → 单次上电幂等重放收敛（RETRY=2） |

批内事故注记：V4 调查中 B-4 双坏 env 与 p1 头损叠加构造失误 → 默认 bootcmd=boot_fit 无
OTA 回退路由 → U-Boot 硬停（CLI-less，软件通道全断）→ 用户 maskrom 重刷恢复（同 tag 同
commit，IMAGE_UUID ca87153e→6914491b，B-1 复核过、payload-id 不变）；教训：破坏性构造
不得叠加，双坏 env 测试须在 p1 健康前提下做，测后立即修复 env。

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
| backup 建段（本批不并 --no-backup） | ⚠→🚨发现 | backup f4c842b7 建段"成功"但 **boot.itb 只捕获 2560B**——见发现 #1 |
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

## B-7 V3（武装态 p1 FIT 损坏 → 自动修复）

E4 武装（`start --force --no-backup --allow-release-change rel`）后 `dd 1MB` 零进 p1 头（md5
02b02d42→7f91dcdc）→ reboot。

| 检查项 | 结果 | 说明 |
|---|---|---|
| rescue 路由+campaign 重放 | ✅ | 单轮 ~100s 收敛（含 normal boot）；apply 后 STATUS=success/RETRY=1 |
| FIT 重写 | ✅ | p1 前 74,322,432B md5=02b02d42 == 包内 boot.itb（精确复原） |
| 终态 | ✅ | in_progress=0/normal/syspw 净 |
| 串口证据 | ⚠ | 采集迟于 rescue 窗（本地循环逐轮 ssh office 在 reboot 期失败）；state/md5/ota.log 四源已证 |

## B-7 V4（武装态 rootfs 尾损坏）→ **发现 #2 + 全梯实测**

构造（依上批 V2 教训走 mapper 层）：E4 武装 → `dd 256KiB` 零进
`/dev/mapper/armbian-root` 尾部 + drop_caches + reboot -f。

| 检查项 | 结果 | 说明 |
|---|---|---|
| 损坏落盘 | ✅ | mapper 尾 512 扇区全零；命中 ext4 目录元数据（inode #34713 目录叶校验和坏） |
| 首启自愈/单轮重放 | 🚨 | **不可自愈**：apply 是 clean+extract 覆写旧 ext4，非 mkfs 重建（见发现 #2）→ tar 撞损坏目录块 rc=2 |
| 失败梯子（全程串口 `vcap-*.txt`） | ✅ | candidate attempt 1/2/3 全 `rootfs extract failed` → retry not exhausted → backup 切换（RESTORE_MODE=1）→ backup attempt 1-3 同样失败（同 fs）→ `state_mark_failed` → `RESCUE SHELL: no automatic path left: rootfs apply failed` 精确落壳 |
| 备份 itb 写回 | ✅（未触发） | rootfs apply 失败先于 itb 写回，p1 全程未动（02b02d42 保持）——发现 #1 的截断 FIT 未被实际写入 |
| fail-stop 语义 | ✅ | shell 只读不进一步损坏 rootfs；rootfs 安全（设计内） |
| 手工修复闭环 | ✅ | shell 内 `mkfs.ext4 -F -q -L armbi_root /dev/mapper/armbian-root` + state 重置 STATUS=prepared/RETRY=0/RESTORE_MODE=0 → reboot → campaign 重放单轮成功（干净 fs 上 extract 全链）→ STATUS=success/FIT md5 精确/payload marker 910438c2/零 I/O 残留/syspw 净 |
| 上批 V4 复盘 | ⚠ | 上批打 p4 裸分区（密文层）单轮自愈属侥幸（损坏未命中元数据/被页缓存回写掩盖）；本批 mapper 层构造证明"全量重写自愈"在 raw-FIT 布局无 fs 级保证 |

**发现 #2（设计缺口）**：`ota_reformat_rootfs` 的 mkfs 快路径 gated on `HAS_BOOT_PART=1`
（理由"/boot stays on rootfs"），raw-FIT secure recovery 布局（HAS_BOOT_PART=0）永远走
rm -rf clean+tar extract——但 raw-FIT 的引导在 p1 raw 分区，rootfs /boot 不承载引导链，
mkfs 完全安全。后果：武装态 rootfs **元数据**损坏 campaign 重放每轮必败（candidate 与
backup restore 同 fs 双双耗尽）→ 落 shell 须人工修复。plain recovery/AB 布局有独立
boot 分区走 mkfs 路径则真自愈。修复方向：mkfs 条件改为
`HAS_BOOT_PART=1 || boot_mode=raw-fit`（rootfs 侧 stamp 已有 raw-fit 标记可用）。

操作层教训（本批自身失误，供后续批参考）：
- 采集循环改 office 本地 hardcopy 轮转（`/tmp/vcap-*.txt` 批量取回），勿本地逐轮 ssh office
  （设备 reboot 期 office 通道间歇失败 → 迟采丢 U-Boot 段）
- screen `stuff` 多层引号会搅 sed 模式（本批 `^STATUS=` 变 `SSTATUS=` 首字母重复，落
  `unknown STATUS=` 二次 shell）；串口内文件编辑用无引号单命令逐行，改完先 cat 验证再 reboot
- 运行态 `reboot -f` 一次未生效（打印 "Rebooting." 但 journal/uptime 连续无重启，原因未明，
  待复现）；initramfs shell 内 `reboot -f` 正常。临时规避：必要处用 SysRq `echo b`

## B-7 V1（确认态 p1 FIT 损坏 → fail-stop shell + 手工修复）

确认态（in_progress=0/success）`dd 1MB` 零进 p1 头 → reboot（`v1-*.txt` 全程串口）。

| 检查项 | 结果 | 说明 |
|---|---|---|
| U-Boot normal 失败 | ✅ | `## Booting FIT Image FIT: No fit blob` / `FIT: No FIT image` |
| fallback 路由 | ✅ | bootcmd 顺延 `run ota_rescue_boot` → p2 rescue FIT 起 |
| fail-stop shell | ✅ | `RESCUE SHELL: Rescue entered with ota_in_progress=0 (normal boot failed or manual entry); fix and reboot manually (shell on ttyFIQ0)` 精确命中（~30s 到壳） |
| rootfs 安全 | ✅ | shell 只读不碰 rootfs |
| 手工修复闭环 | ✅ | 串口 `dd staged/boot.itb → p1`（conv=fsync）→ sha256sum -c boot.sha256 → reboot → U-Boot boot_fit 验签通过即修复证明 |
| 修复验证 | ✅ | p1 md5=02b02d42 复原；normal 健康启动；STATUS=success/in_progress=0 |

## 2.5 无锚定 find_raw_boot_dev（⏹ 不可现场构造）

- 构造尝试：两次改 `ota_rescue_boot_fit`（追加式剥 token / 整体替换 bootargs）均无效——
  rescue 内核 cmdline 恒含 `armbian.bootdev=nvme armbian.bootdevnum=0`，**来源是 p2 rescue
  FIT 内建 bootargs 配置**（U-Boot `boot_fit` 以 FIT config 覆盖 env bootargs，env 层剥离
  不可达）
- 因此 4f70b1b1 拒写分支与 b048f73f ROOT_DEV 回退锚在本镜像现场均不可触发；两次事务正常
  apply（STATUS=success/FIT md5 精确），env 原值已恢复
- **发现 #3（latent）**：b048f73f commit message 宣称 "find_raw_boot_dev then scopes the
  scan to that disk"，但实际只改了 payload.sh 的 guard（`fw_env_boot_disk "${ROOT_DEV}"`）；
  `find_raw_boot_dev` 内部仍 `fw_env_boot_disk`（cmdline-only、无锚参数）——无 token 场景下
  guard 过而扫描无锚，多盘时 itb 仍可能落错盘。当前 FIT 恒带 token 掩盖；修复：给
  find_raw_boot_dev 传锚参（与 guard 同源）

## B-4 双坏自愈（🚨 真失败 → 发现 #4，根因开放）

构造：双副本零化（8128/8064 各 64 扇区）→ reboot，三次独立复现（含重刷镜像后一次）。

| 检查项 | 结果 | 说明 |
|---|---|---|
| in-boot 重建 | 🚨 | 三次均**静默 no-op**：marker 行照打（脚本确在跑）、零 rebuild 行、双 blob 保持零 |
| fail-open 兜底 | ✅ | U-Boot 默认 env 引导 + 解密 scan order 照常进 normal（设计内） |
| 脚本版本核实 | ✅ | p1 FIT initramfs 解包（FDT 解析→ramdisk data-position 0x2d34e00/26.6MB）→ find-userdata.sh/zz-ota-normal-marker/fwenv.sh 与仓库 8dcba161 **逐字节一致**（21a80ff9/8dcba161 修复确已入像） |
| busybox 兼容 | ✅ | initramfs busybox 272 applets 含 mktemp/awk/ls 等；设备 busybox ls 有 owner 列（列布局无关提取正确） |
| 运行态链路 | ✅ | busybox PATH 忠实复现全链通：lower/raw_part(/dev/nvme0n1p4)/disk/write_config/crc_bad 全过 |
| chroot 真实环境复现 | ✅ | initramfs 树 chroot 进设备（bind 真实 /dev /proc /sys + root-ro/root-rw 挂载）：**全链至 `fw_env_write_routing_set 0 normal` 真写成功**（`env rebuilt` 行 + 落盘验证） |
| 根因 | ⏸ | chroot 与真实 boot 唯一差在 initramfs 运行时环境本身；静默退出点必在 guard 段（echo 之前），具体环节未定位——需 recovery 键进真实 initramfs shell 交互调试（本批被硬停事故阻断） |
| 运行态手工修复 | ✅ | 修复配方：`. fwenv.sh && . find-userdata.sh && fw_env_write_config /dev/nvme0n1 && OTA_RESCUE_ROOT_DEFAULT=/dev/mapper/armbian-root fw_env_write_routing_set 0 normal` → 路由集完整恢复（bootcmd 全链/in_progress=0/normal） |
| 主副本仍零 | ⚠ | libubootenv 双坏重建写 inactive 位（单副本落盘），同上批已知项；冗余读不受影响 |

**发现 #4**：8dcba161 镜像上 B-4 in-boot env 重建在真实 initramfs 中静默失效（上批定位的
busybug 已修、脚本已入像、chroot 可复现成功），实际 boot 仍 no-op——上批"boot 期自动重建
终验 ⏸ 待重建镜像"的验证结果为**不通过**。影响：双坏 env 后路由集不自动恢复，设备靠默认
env fail-open 可启动但 OTA 路由/`armbian.bootdev` token 缺失（rescue 引导路由依赖 env），
须手工修复（运行态配方见上）。遗留：根因定位需真实 initramfs shell（recovery 键路径）。

## B-6 Rescue apply 中断恢复（✅ 用户在场断电）

载体 `start --force --no-backup --allow-release-change rel.tar.zst`（E4 语义）。GO 后 ~80s
断电 + 5s 再上电（`b6b-*.txt` 全程串口）。

| 检查项 | 结果 | 说明 |
|---|---|---|
| 断电落点 | ⚠→更刁钻 | 落于 boot1 `boot.itb verified after write (sha256=dcab9846)`（kernel 77s）之后、`applied` 持久化之前的窄窗（解包窗 25–60s 已过）——itb 已写、rootfs 已写、状态未 commit |
| 幂等重放 | ✅ | 重上电单轮收敛：attempt 2/3（ROOTFS_TOUCHED=1 正确继承）→ clean → 全量重解 → applied → 回 normal；RETRY_COUNT=2 |
| 终态 | ✅ | STATUS=success / in_progress=0 / normal / FIT md5 02b02d42 精确 / payload marker 910438c2 / syspw 净 |
| 备份未被使用 | ✅ | --no-backup 语义，重放路径（非 backup restore） |
| armbianEnv 断电存续 | N/A | raw-FIT 形态结构性 N/A（布局无 armbianEnv.txt，同上批） |
| 时机记录 | ⚠ | 上轮 45s 指令早于实际解包窗（本镜像 60–105s）未落窗；此轮 80s 落于 apply 尾段——解包窗中段中断变体未单独覆盖（幂等语义已由尾段中断+V4 梯子双重覆盖） |

## B-5 收口（✅）

- 无克隆盘/SD 在场（lsblk sd|mmcblk 计 0；REC 全程未插）
- journal：decryption-disk `Boot disk from cmdline: /dev/nvme0n1` → LUKS p4/root+userdata mapper（启动盘锚定 ✓）
- 终态：STATUS=success / INSTALLED_ID=910438c2 / in_progress=0 / normal / env 路由集完整（B-4 修复后）





