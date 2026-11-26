# OTA 真机通用测试计划（{plain,secure} × {A/B,recovery}）

- 前置：ssh 可达（或串口）、root、fw_printenv/fw_setenv 在 PATH
- 用法：每批选形态 F ∈ {plain, secure}，跑 Session AB(F) + REC(F)；换形态另起一批
- 命令均按 NVMe 系统盘写死 `/dev/nvme0n1`；换启动介质须自行映射（手册 mmcblkX 偏移/扇区数不变）

---

# 一、准备

## 1.1 素材预备（host 侧）

- 构建两镜像同 commit；**REC 配异 VERSION 包**（payload-id 翻转才有意义；同版包只能配 B-3b 的
  --force 场景）；AB 同 VERSION 可行（无 version 闸）。payload-id v2 内容寻址（R/B 哈希+域 tag，
  VERSION 不进哈希）——同 VERSION 异内容也翻转，配包仍按异 VERSION
- 坏包 ×1：解包删 boot 成员 → 重打包
- USB 克隆盘：dd 同镜像（两 Session 复用，间期 host 重 dd）
- 本形态 + 异形态 loader（loaders/ 相对路径）；scp 通道、串口 screen 会话就绪
- 记录镜像与包 sha256 入报告

## 1.2 包投放与 /mnt/imgs 挂载

office 反向隧道 sshfs → 设备 `/mnt/imgs`，包放 `history/<批号>/`：

```sh
ssh office 'SSH_AUTH_SOCK=/run/user/1000/gcr/ssh ~/upgrade_tool/mnt-imgs.sh root@recomputer-rk3576-devkit.lan -p root'
```

gcr socket 是 office 唯一持钥 agent；设备密码 root/root；脚本装 sshfs、建 2200 隧道并挂载
（~50MB/s）；设备上不落密钥。

## 1.3 烧录后形态自检（用户刷完告知后执行）

```sh
cat /etc/armbian-image-release        # 身份块，核对 SEEED_EXTENSION_COMMIT
lsblk -o NAME,TYPE,PARTLABEL /dev/nvme0n1
blkid | grep -E "PARTLABEL|LUKS"
cat /proc/cmdline
fw_printenv | grep -E "boot_slot|ab_boot_mode"
```

判定：`boot_a/boot_b` → A/B；仅 `boot` → recovery。`security` 或 TYPE=crypt → secure；专用
`recovery` 分区（raw 无 fs）→ secure REC；boot 为 ext4 `armbi_boot` → plain REC。
核对后生成报告骨架 `<批号>/<form>-ota-test-report.md`（格式见公共「报告」节）。

## 1.4 前置确认（每 Session 首启；不过则中止）

`cat /etc/armbian-image-release` 逐项核对入报告：

- BOARD / RELEASE / IMAGE_VARIANT / VERSION：与镜像、OTA 包三方一致
- **SEEED_EXTENSION_COMMIT == 被测 commit**，不符先烧新镜像
- BUILD_REPOSITORY_COMMIT（-dirty 注明）、IMAGE_UUID

辅证 `uname -v`：事务后 image-release 可能被 upper 遮蔽显旧值（预期非失败），前置只在首启做。

## 1.5 串口监控配方

- 每步一抓：boot 后 25s 内存档 + 按预期行 grep；成功启动的 U-Boot 段须 reboot 后 5-15s 连抓，
  失败启动（硬停）天然全保
- 读（不抢 Attached）：`ssh office 'screen -S usb0 -X hardcopy -h /tmp/uart.txt && tail -N /tmp/uart.txt'`
- 写：`ssh office 'screen -S USB0 -X stuff "命令\n"'`——换行必须真实传入（外层 ssh 双引号内
  写 `$'…\n'` 最稳；`$(printf "x\n")` 会吞换行）
- 用户查看：`screen -x usb0`；industrial 板另接 1a86 会话
- ssh 不可达段（A-9 halt 后、B-1 首启、B-2 apply 全程、RESCUE SHELL 交互）均走串口

| 步骤 | 串口预期行 |
|---|---|
| boot 类 | `security partition resolved: <盘>p3`、`SSKR`、`mapper ready`、`overlayroot: configured` |
| A-4 | journal `persistent env unreadable` + 串口正常解锁链 |
| A-7 事务 reboot | `Found LUKS device: …p5` |
| A-9 | `loader/OS mismatch` + `panic`（运行时串出自 armbian-build 侧补丁，以实测为准） |
| B-2 | `OTA done, rebooting` / rescue apply 行 |

## 1.6 重挂 /mnt/imgs 与远程恢复

- **reboot 后 mount 不复活**：跑事务前重挂（命令幂等；50002 有僵尸转发会话先 kill 设备侧 sshd pid）
- **重刷后第一件事 `systemctl enable ssh`**（镜像出厂未 enable，首启可达是暂态）；隧道活用密码
  root/root（sshpass），不通走串口；enable 后推本机公钥，host key 必变先
  `ssh-keygen -R '[127.0.0.1]:50003'` + accept-new
- ssh 失联三坑（顺序 1→3→2）：userdata 510M 两段式扩容（cryptsetup resize+resize2fs）、
  50002 僵尸会话、openssh disabled（串口 enable）

## 1.7 成本预算（每形态一对 Session）

| 项 | 次数 | 构成 |
|---|---|---|
| 全盘烧录 | 2 | AB-F、REC-F |
| loader 烧录 | 1 | §7 换异形态（SPI-NOR） |
| ssh 重启 | ~12 | §1×2、§2a×1、§3×1、§6×1、§2b×2 + C/F 系列 |
| 重挂 /mnt/imgs | ~7 | 跟随 reboot；同窗打包免重启项可压 |
| 人工断电 | 3 窗 + 短断电 | W1 烧 AB、W3 烧 REC；F1 坏槽硬停 ~4 次；B-6 apply 中断 1 次 |

## 1.8 执行顺序（每批编排）

- §1 只能占烧录后首启，放最前；零重启负例穿插消化
- 同一 ssh 活窗打包所有"免重启且需 /mnt/imgs"的步骤（A-5/A-5b/A-5c）——reboot 杀挂载，散窗 = 每窗重挂
- **A-4 先于 A-7**：双坏重建丢槽记忆（boot_success 重置为探测值），A-7 成功态会被 A-4 清掉
- §2a 先于 §3：自愈刚重建的健康 env，才能可信判定"无 rebuilding 警告"
- §4 真事务 kill 制造残留，重跑即 §3 本体，并与锁、§5 克隆盘合并省 1 次重启
- §7 收尾 AB：halt 后断电窗口直滚 REC 烧录（含写回本形态 loader），零额外恢复

---

# 二、公共（两布局通用）

## 2.1 形态差异表（未列项两形态通用）

| 项 | plain | secure |
|---|---|---|
| 镜像 | 无 LUKS | rootfs 全 LUKS |
| security 分区 | 无 | 有（PARTLABEL=security） |
| Rescue 引导 | boot 分区 ext4 `/recovery/` + booti | 专用 recovery 分区 raw FIT（boot_fit） |
| OTA 包 boot 成员 | boot.tar.<fmt> | boot.itb |
| 首启额外观察 | — | AB: journal SSKR 迁移；REC: 串口明文口令窗口 |
| A-7 残留记录 | findmnt armbi_ | + cryptsetup status armbian-ota-root-<slot> |
| §5 克隆盘锚定 | 分区/PARTLABEL 解析 | + security 分区 / LUKS 容器 |
| §7 失配方向 | 换 **secure** loader | 换 **plain** loader |

## 2.2 判据与工具

- **每步 UART 取证（强制）**：每步收尾（boot 后 25s 内、事务后、异常时）抓 `usb0` 存档
  `.claude/test_plan/<批号>/uart/<步骤>-<HHMMSS>.txt`，按该步预期行 grep；journal 与串口双源互证。
  缓冲 ~1000 行、**迟抓即丢**；成功启动的 U-Boot 段须 reboot 后 5-15s 内连抓
- **取事务 RC 勿接管道**：`cmd | tail` 的 `$?` 是 tail 的——输出重定向到文件再取 RC
- env 几何：主 `0x3f8000`、冗余 `0x3f0000`，各 `0x8000`（64 扇区）；冗余头 CRC(4)+flags(1)，flags=1
- 坏 env 判据（勿看退出码）：`fw_printenv 2>&1 >/dev/null | grep -qiE "bad crc|cannot read
  environment|failed to read environment" && echo BAD || echo OK`
- blob 头校验（5 字节头，第 5 字节 = 01）：
  `dd if=/dev/nvme0n1 bs=512 skip=8128 count=1 status=none | od -A d -t x1 -N 8`（主）
  `dd if=/dev/nvme0n1 bs=512 skip=8064 count=1 status=none | od -A d -t x1 -N 8`（冗余）

## 2.3 闸门负例 G1–G7（两形态、两布局通用；换树必重跑）

gate 在签名校验前生效，篡改 `package.env` 即触发。制包（office）：解包正包 → sed 改字段 →
`tar -cf` 落 `history/<批号>/gates/`；plain/raw 例软链现成文件。判据：RC≠0 + env 不触碰。

| # | 包 | 预期 |
|---|---|---|
| G1 | BOARD 改异板 | `BOARD mismatch` |
| G2 | RELEASE+VARIANT 双改 | `RELEASE mismatch (…--allow-release-change)` |
| G3 | G2 + `--allow-release-change` | WARN 放行 release → `IMAGE_VARIANT mismatch`（双改防误入真事务） |
| G4 | 仅 VARIANT 改 desktop | `IMAGE_VARIANT mismatch` |
| G5 | plain 正包 | `encryption mismatch: package=no, device=yes` |
| G6 | 异布局正包 | `<对侧> OTA backend is not installed (have: <本侧>)` |
| G7 | 原始 .img.xz | `metadata missing package.env` |

执行位：**ab 侧**随 A-5 同窗跑 G1–G5+G7（G5 用 plain AB 包）；**recovery 侧** B-3c
（G5 用 plain REC 包、G7 raw 镜像、G6 反向即 B-3 的 AB 包例）。

无素材未覆盖：容量/空间闸拒绝分支、payload magic mismatch、双 boot 成员 ambiguous、zst 协商拒、
加密反向（plain 设备收加密包）。

## 2.4 失败处置

- 合并事务失败：先清残留（`umount -R` + [仅 secure] `cryptsetup luksClose armbian-ota-root-*`）
  再单项重跑；各项可独立复测
- **rescue initramfs fw_* 不可用**（RESCUE SHELL 硬停）：panic shell 从已解锁 rootfs 注入缺失库
  （`mount /dev/mapper/armbian-root /r && cp /r/usr/lib/…/*.so* /usr/lib/…`）→ 卸 /r、
  /mnt/userdata → `sh /scripts/init-premount/99-ota-apply` 重跑
- §7 未按预期（明文挂载/继续启动）→ 立即断电，保存串口全文
- 失败时附：`fw_printenv`、`/etc/fw_env.config`、双偏移 od 头 16 字节、`journalctl -b -u armbian-ota-*`

## 2.5 未覆盖项补测方法（素材就绪后按下表执行）

| 项 | 前置素材 | 操作 | 预期 |
|---|---|---|---|
| `p?` optional-p 启动盘侧分支 | 启动盘为 sd/vd 的板（NVMe 启动盘走不到该正则前缀） | 常规事务跑一次 | 分区解析命中启动盘侧 `p?[0-9]` 正则 |
| 真跨 release 全量切换 | 同变体异 release 包对（现有矩阵无此对） | `start --allow-release-change <跨 release 包>` | release WARN 放行 → 全链事务成功 |
| itb-on-ext4 混装闸 | 加密 ext4-boot 镜像 + itb 包（现有包先被加密闸拦） | `start <itb 包>`；武装后进 Rescue | start 拒配对；campaign_fail `boot.itb payload but device exposes an ext4 boot partition` |
| find_raw_boot_dev 无锚定 | 手工剥离 cmdline 的 `armbian.bootdev` token | 武装后触发 itb 写回 | 拒写 + campaign 失败（fail-closed，不落无锚 dd） |
| rescue boot 分区挂载失败 | 损坏 boot 分区 superblock | 武装 → reboot 进 Rescue | fail-campaign（不 skip，停在设计分支） |

## 2.6 报告

每类镜像独立一份（形态自检判定哪类生成哪类，互不混写）：`ab-plain`/`ab-secure`/
`recovery-plain`/`recovery-secure` 四选二，落 `.claude/test_plan/<批号>/<form>-ota-test-report.md`，
**头部必含镜像身份块**：

```
- 设备/介质/loader：reComputer RK3576 DevKit / NVMe / SPI-NOR loader
- 镜像身份（/etc/armbian-image-release）：BOARD / RELEASE / IMAGE_VARIANT / VERSION /
  SEEED_EXTENSION_REPO + SEEED_EXTENSION_COMMIT / BUILD_REPOSITORY_COMMIT / IMAGE_UUID
- 烧录镜像 sha256 / OTA 包 sha256；烧录时间与 boot 计数基线
```

**报告条目规则**：

- **逐项列出**：每个测试步骤的每个检查项独立成行（不合并、不省略，含偏差/阻塞项），结果用
  ✅ 通过 / ❌ 失败（⚠ 偏差有解释 / ⏸ 阻塞注明出路）
- **描述精简**：每行说明一句因果即止——关键数值（sha/md5/RC/状态值）保留，journal/串口关键行
  原文引用进证据列或注记行，禁止成段 prose
- 节结构：命令一行 → 检查项表（检查项 / 结果 / 说明）→ 取证注记（仅偏差与豁免时）
- **收尾清理**：测试完成后清冗余——已复验闭环的偏差项从原节移除（保留复验处记录）、陈旧表述
  （指向已变更的 commit/状态）、重复证据与过程性中间态；报告只留终态有效内容

## 2.7 覆盖矩阵（手册项 → 执行位索引）

| 手册项 | 执行位 |
|---|---|
| §1 A/B 首启 init-uboot | A-1–A-3 |
| §2a A/B 双坏自愈 | A-4 |
| §2b REC 双坏自愈 | B-4 |
| §3 正常事务 | A-7（并入 §4） |
| §3b 并发锁 / 负例 | A-5、A-7 锁；B-3 复测 |
| §4 陈旧挂载自愈 | A-7（kill 真事务） |
| §5 克隆盘锚定 | A-6–A-8、B-2/B-4 |
| §6 payload-id 翻转顺序 | B-2 |
| §7 loader/OS 失配硬停 | A-9（方向按形态差异表） |
| 闸门负例 G1–G7 | 公共 2.3；A-5b/B-3c 执行；B-3 补 G6 反向 |
| 命令面 C1–C7 | A-5c；B-3d REC 侧；B-3b 补 --force/--no-backup |
| 参数组合与重复执行 | B-3e |
| 失败回落 + 坏槽修复 F1 | A-5d |
| REC 中断恢复 | B-6 |
| 双干扰源三盘锚定 | A-6b |
| 首启 userdata resize | A-1、B-1；A-2/A-3 安静复跑 |
| 双坏 env 引导内重建 | A-4、B-4 |
| itb 写回 + 目标锚定闸 | B-2 硬断言 |
| pre-OTA armbianEnv 断电存续 | B-6 增验 |

---

# 三、recovery 布局（Session REC(F)）

### B-0 🔌 W3（紧接 A-9 断电）：NVMe 烧 REC-F；loader 写回 F 形态（dd 同款配方，ssh 不通则随烧录物理写）→ 上电

### B-1 首启

前置确认（1.4）+ resize 四查同 A-1（recovery 布局 userdata 同为末分区）。
[仅 secure] 串口观察首启**明文口令窗口**（86da → tee-supplicant → SSKR 迁移，仅首启可见）。
payload-id 基线：`cat /usr/share/armbian-ota/installed-payload-id`（normal 态 boot 分区不挂载，
boot 分区 payload-id 仅 apply 时可见）。插克隆盘（USB）。

### B-2 Recovery OTA（§6 + §5 recovery 面）

```sh
/usr/sbin/armbian-ota start pkg.tar.gz
fw_printenv ota_in_progress ota_boot_next    # =1 / =recovery
reboot                                       # 进 Rescue apply（ssh 不可达，串口观察）；完成自动回 normal
```

硬断言：apply 前 installed-payload-id=旧值；回 normal 后 == status CANDIDATE_ID → mark-success；
[仅 secure] p1 FIT md5 == 包内 boot.itb（HAS_BOOT_ITB 无条件写回，新 rootfs+旧 FIT 是致命组合）；
settle 后 ota_boot_next=normal；串口记录 payload-id 行相对位置（应最晚——health 不在 boot 文件
半新半旧时提前放行）。观察项：backup raw-FIT 分支含分区回读校验（journal 记录即可）。
**全程拔 SD 克隆盘**（SD 在位劫持引导，rescue 路由失败）；USB 克隆可留（rkimg 扫描不含 USB）。

### B-3 负例（B-2 回 normal 后先重挂）

```sh
/usr/sbin/armbian-ota start bad.tar.gz     # 同款 extract 后断言拒
/usr/sbin/armbian-ota start <AB 包>        # G6 反向：require_mode_backend 拒（have: recovery）
```

### B-3b recovery 专属 flags

- `--force`：装过同 ID 包后再 `start` 同包 → INSTALLED_ID 重复闸拒；加 --force 放行重装
- `--no-backup`：无 backup 建段/复核段；**BACKUP_ID 保留既有 last-good 非空**——与 B-6 配对

### B-3c 闸门负例（G 套件 REC 侧执行位）

按公共 2.3 制包与判定；素材：g5 = plain REC 正包软链、g7 = raw 镜像软链、G6 反向 = AB 包（B-3 第二例）。
每例 metadata 阶段秒级拒；判据 RC=1 + env 不触碰。

### B-3d 命令面轻负例（C 类 REC 侧）

非 root start（`su -s /bin/sh -c "…" nobody`，守卫先于包检查）/ 无参与坏命令 usage / start 未知
选项 / mark-success 无事务——均 RC=1。命令级语义：switch-slot → ab 拒 RC=1、rollback →
"not a user command" RC=1、reconcile → "nothing to do" **RC=0（信息性成功，设计语义）**；
status 无 root 守卫为只读设计。

### B-3e 参数组合与重复执行

| # | 操作 | 预期 |
|---|---|---|
| E1 | `start --no-backup <异 ID 包>` | 武装且 start 日志零 backup 行（grep 计数 = 0） |
| E2 | 事务中并发第二个 `start` | `Another OTA process is running (PID:…)` RC=1，env/state 不动 |
| E3 | armed 态同 ID 重复 `start` | 重复闸先于 abandon，armed 态不被扰动 |
| E4 | `start --force <异 ID 包>` 穿闸 | `Removing orphan payload directory: …` → 新 campaign 重武装 |
| E5 | 三 flag 并用 / mark-success 正向 | 低价值 / 结构性走不到（health auto-confirm 同函数），不测 |

### B-4 双坏自愈（§2b）

```sh
dd if=/dev/zero of=/dev/nvme0n1 bs=512 seek=8128 count=64 conv=notrunc
dd if=/dev/zero of=/dev/nvme0n1 bs=512 seek=8064 count=64 conv=notrunc
sync && reboot
```

判定：串口/initramfs 见 `armbian-ota: env unreadable; rebuilding routing set on <disk>`，
**同 boot 内**完成（fail-open：失败仅 `WARNING env rebuild failed; continuing boot` 不砖）；
双坏时有武装事务（STATUS=prepared|applied）改走 `env unreadable; re-arming for Rescue (STATUS=…)`，
事务不丢；重启后路由集+bootdev token 完整、直接武装可跑（原死锁已解除）。
注记：坏 env 首启解密仍走 scan order（勿插克隆盘）；loader 默认 env 不带路由（init-bottom 重建
已消解其影响）。

### B-5 收口

拔克隆盘；[仅 secure] 串口/journal 确认 decryption-disk 的 LUKS 容器取启动盘；终态记录入报告。

### B-6 Rescue apply 中断恢复（需一次人工断电）

载体 `start --no-backup --force`；事务前 armbianEnv 加 `myoverlay=1`；reboot 后断电于 rootfs
解包窗（上电 ~20s 进 apply、解包窗 ~28-66s、apply 全程 ~74s）。

判定：

- 一次上电 <2 min 收敛：rescue 路由 → campaign 重识别 → clean + 从头重解 → applied → 自动回
  normal——**候选幂等重放**（非备份恢复，备份在场未被使用；arm 期任意点断电安全）
- `myoverlay=1` 重放后仍在（捕获副本落 userdata/ota-recovery/staging，boot 分区捕获优先、
  rootfs /boot 兜底）——断电后 `ls` staging 副本在手前不 reboot
- 武装前置：cmdline `armbian.bootdev=nvme` 最稳；`fw_env_boot_disk` 亦接受 mapper 锚；env 异常
  先按 B-4 恢复链修 env 再武装
- 串口存档中断前 + 重放全程各一份

旧镜像兜底（新镜像 B-4 自愈后不需要）：`fw_setenv -f /etc/u-boot-initial-env <k> <v>` 只重建
最小集（无路由/token，不能武装）；完整恢复须镜像 blob 级——office
`xz -dc <img>.xz | dd bs=512 skip=8064 count=128` → 推设备 `dd of=/dev/nvme0n1 bs=512 seek=8064
conv=notrunc` → reboot → 武装。

---

# 四、ab 布局（Session AB(F)）

### A-0 🔌 W1：关机 → NVMe 烧 AB-F + SPI-NOR 写 F loader → 上电

### A-1 首启（前置确认 + §1 init-uboot 全链）

```sh
cat /etc/armbian-image-release
journalctl -b | grep -iE "SSKR|tee-supplicant"    # [仅 secure] SSKR 迁移，仅首启可见
systemctl status armbian-ota-init-uboot.service
journalctl -b -u armbian-ota-init-uboot | grep -E "Installed|unreadable"
cat /etc/fw_env.config                             # 双条目 3 字段，父盘 /dev/nvme0n1
fw_printenv | head
```

init-uboot 已并入 common 运行时：journal 之外另落 `/var/log/armbian-ota/ota.log`（双源取证）。
通过：config 双条目 3 字段、boot_slot/分区变量读回正确、正常进 slot。

userdata resize 四查：

```sh
systemctl status armbian-resize-userdata.service   # active (exited)，非循环
journalctl -b -u armbian-resize-userdata | grep resize-userdata
lsblk -o NAME,SIZE,TYPE,PARTLABEL | grep -i userdata   # 扩至盘尾
df -h /media/root-rw
```

[仅 secure] journal 须含 `resizing LUKS mapping arbian-userdata`（分区+LUKS+ext4 单 boot 三级）；
exit 75 循环后收敛算过但记录次数；plain 不应见任何 LUKS 行。
通过：exit 0、容量≈盘尾、无 "not the last partition"/"refusing to shrink"/"unsupported userdata filesystem"。

### A-2/A-3 重启×2（§1 安静通过）

```sh
reboot    # ×2，每次 boot 后：
journalctl -b -u armbian-ota-init-uboot --no-pager | tail -3
journalctl -b -u armbian-resize-userdata --no-pager | tail -2   # already uses available space 快退
```

通过：每 boot 服务都启动（无 Condition 跳过）、健康快速退出、无 "unreadable"、boot 耗时无明显增加。

### A-4 双坏自愈（§2a）

```sh
sync
dd if=/dev/zero of=/dev/nvme0n1 bs=512 seek=8128 count=64 conv=notrunc   # 0x3f8000
dd if=/dev/zero of=/dev/nvme0n1 bs=512 seek=8064 count=64 conv=notrunc   # 0x3f0000
sync && reboot
```

```sh
journalctl -b -u armbian-ota-init-uboot | grep -E "unreadable|Installed"
fw_printenv | grep -E "boot_slot|boot_success"
```

通过：一次重启内自愈、slot/boot_success 与运行槽一致、无手工干预、config 仍双条目、两副本
blob 头 flags=01。增验（0d8c9f34）：journal 见 `persistent env unreadable; re-running full init
(fw_setenv -f rebuild)`；双坏时有武装事务（ready_to_boot/boot_verifying）则重建槽取 state 的
CURRENT_SLOT——先 switch-slot 武装再零化可验 armed 槽不丢。
[仅 secure] 重建在 rootfs 起：env 双坏时引导期解密若先失败则到不了重建点——实测确认首启收敛，
勿插克隆盘。

### A-5 负例（零重启；A-4 后先重挂 /mnt/imgs）

```sh
/usr/sbin/armbian-ota start bad.tar.gz   # extract 后拒（断言跑在解压目录上，~73s）
fw_printenv -n ota_in_progress           # 未触碰
```

报错含 `dual-payload assertion failed`（no boot payload）。

### A-5b 闸门负例（G 套件 ab 侧执行位）

按公共 2.3 制包与判定，随 A-5 同窗执行（G5 用 plain AB 包）；判定与判据同公共节。

### A-5c 命令面 C 系列

| # | 操作 | 预期 |
|---|---|---|
| C1 | `switch-slot <t>` 双向 | armed：boot_slot=t、**boot_success 保持锚槽**、ota_in_progress=1、retry=3；reboot 后健康自动确认 |
| C2 | 守卫组 | 事务中 switch-slot 拒；目标=当前槽 `Already booting`；坏参 usage 拒 |
| C3 | `rollback` 无事务 | `nothing to rollback` RC=0 |
| C4 | `rollback` 真路径 | env 复位回锚槽 + state=rollback + **自动 reboot**（新守卫：已确认槽不再被 rollback，仅验正常路径） |
| C5 | `reconcile` 显式 | 手动 `fw_setenv ota_in_progress 0`（合法模拟断电）→ `target slot … did not pass boot verification`、state=rollback、TARGET 清空；须手工恢复 boot_slot 锚 |
| C6/C7 | usage / 非 root | usage 文本；`must be run as root`（守卫先于包检查） |

未覆盖：--force/--no-backup（recovery 专属 → B-3b）；reconcile 失败分支；start 未知选项（trivial）。

### A-5d F1 失败自动回落

| 步骤 | 预期 |
|---|---|
| dd 1MB 零进 boot_b 头 → `switch-slot b` 武装（retry=3）→ reboot | 每次尝试：`Saving Environment … done` → `FIT: No fit blob` → `### ERROR ### Please RESET`（硬停无 CLI，**人工断电驱动扣减**，3 次耗尽后第 4 次自动回落） |
| retry 耗尽 | ab_preboot 自动：boot_slot=boot_success、ota_in_progress=0、retry=max → 锚槽启动 |
| 锚槽落地 | reconcile 记 U-Boot-rollback：`target slot … did not pass boot verification`，state=rollback |
| 修复 | 重跑 start 重写坏槽（p2+rootfs）→ reboot 候选槽健康确认；可与 A-7 牺牲事务合并省一次全量 |

U-Boot 阶段 ssh 不可达，全程串口；扣减次数记人工断电数入报告。F2（坏 rootfs 变体）留后续。

### A-6 插克隆盘（§5）

USB 克隆盘插入；blkid 双份 PARTLABEL（[仅 secure] `grep -c security` = 2；[仅 plain] 分区双份无 security）。

### A-6b 双干扰源（USB + SD 同场）

SD 克隆盘（可启动）+ USB 克隆盘 → 三盘并存（nvme+sd+mmcblk）：

| 检查点 | 预期 |
|---|---|
| PARTLABEL | blkid 计 3× |
| mmcblk 锚定分支 | `ota_blkid_first_on_disk` 的 mmcblk 分支覆盖；resize/ab 解析仍落 nvme |
| U-Boot 引导选择 | SD 在场 → `armbian.bootdev=mmc`（**SD 抢 NVMe = RK 既定行为**）；拔 SD 回 nvme |
| 时机 | 与 A-9 同轮 reboot 可双验（换 loader 前插好 SD） |

源码级结论：引导选择在 C 级 `boot_devtype_init`（只看卡在不在，不看可引导性），rkimgtest 不参与；
U-Boot env 跟随引导盘——SD 引导读 SD env（镜像默认），NVMe 武装事务被搁置。**REC session 全程拔 SD**
（A/B 的 USB 克隆无此问题）。成本：dd ~10G → SD 约 10-15 分钟 + 插卡。

### A-7 合并事务（§4 残留自愈 + §3b 锁 + §3 + §5）

```sh
# 1. 牺牲事务：后台起，target 挂载出现后杀（制造残留）
/usr/sbin/armbian-ota start pkg.tar.gz &         # 记精确 PID
until findmnt | grep -q armbian-ota-root-; do sleep 5; done   # [仅 plain] 改盯 findmnt|grep armbi_
sleep 30 && kill -9 <PID>
# 勿 pkill/pgrep -f（匹配自身命令行误杀 ssh）；孤儿写者留下为预期
findmnt | grep armbi_                            # 记录残留挂载
cryptsetup status armbian-ota-root-<slot>        # [仅 secure] 记录残留 mapper

# 2. 真事务：前台重跑
/usr/sbin/armbian-ota start pkg.tar.gz           # [仅 secure] Stale LUKS mapper … closing → Detaching stale mount
# 并发终端：start pkg2 → Another OTA process is running（锁）
fw_printenv -n ota_in_progress                   # =1，无 WARNING "rebuilding env"

reboot                                           # 候选槽启动
/usr/sbin/armbian-ota status && /usr/sbin/armbian-ota mark-success
fw_printenv -n ota_in_progress                   # =0
cat /var/lib/armbian-ota/ota-state.env           # 键齐全（单次原子写）
systemctl status armbian-ota-firstboot.service
ls -l /boot/initrd.img*                          # 无悬空软链
ls /tmp/syspw 2>&1                               # [仅 secure] 须不存在
```

通过：arm 两键写入零失败、读回一致、候选槽健康自动确认（firstboot 先于手动，mark-success 幂等
兜底）；并发 start RC≠0 且不碰 env/state；Detaching 拆干净后 mkfs 正常；无悬空 initrd 软链；
[仅 secure] syspw 不留（含杀事务后复跑）。ota_boot_next AB 语义为空，勿当失败。

### A-8 §5 AB 收口

journal 确认分区解析落 `/dev/nvme0n1p*`（[仅 secure] security 路径 ota_get_security_part；
[仅 plain] boot/rootfs PARTLABEL 路径），非克隆盘；拔盘。

### A-9 §7 失配硬停（软件化，无需 maskrom）

```sh
# 先重挂 /mnt/imgs：
dd if=/mnt/imgs/<批号>/<异形态>/rkspi_loader.img of=/dev/mtdblock0 bs=64k conv=fsync status=none
sync; md5sum /dev/mtdblock0 /mnt/imgs/<批号>/<异形态>/rkspi_loader.img   # 16M 写 ~5 分钟，读回防半写
reboot    # 串口观察硬停（ssh 不再回来——闸门生效的反证）
```

预期（按形态/盘状态）：

- F=secure（plain loader）+ SSKR 已迁移盘：`SSKR marker found → Error: … requires working
  OP-TEE and keybox_app` → cryptroot 手动口令提示（二进制 keyfile 不可输）→ 事实死锁、rootfs 锁定 ✓；
  内核佐证 `optee: api uid mismatch`
- F=secure + 86da 新鲜盘（raw-FIT 引导）：FIT 自带 bl32 可放行 raw passphrase、完整启动
  （**86da 闸对 raw-FIT 引导存在缺口**，记录）
- SD 在场 U-Boot 选 SD（`Bootdev(atags): mmc 1`）；A-9 须拔 SD/USB
- F=plain（secure loader）：`booti: raw Image boot is disabled on verified-boot builds`

通过（统一）：rootfs 保持锁定或启动停止、无明文 fallback、不回退挂明文 rootfs、不走 overlayroot
RW fallback。观察后人工断电。
