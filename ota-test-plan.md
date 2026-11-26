# OTA 真机通用测试计划（{plain,secure} × {A/B,recovery}）

- 基线：统一冗余 env（PR#42）+ PR45/46/47 + dev 线合入 main（PR50–54：261125 跨形态重构收敛 + 261125/261126
  修复批——双坏 env 引导内重建、itb 写回链、armbianEnv 断电存续、keybox 清理等）；被测版本以
  `SEEED_EXTENSION_COMMIT` 核对为准
- 前置：目标板 ssh 可达（或串口）、root、fw_printenv/fw_setenv 在 PATH
- 用法：每批选定形态 F ∈ {plain, secure}，跑 Session AB(F) + Session REC(F)（同形态配对）；换形态另起一批，互不依赖
- 设备/rig：reComputer RK3576 DevKit / NVMe 系统盘 / loader 在 SPI-NOR；手册 `mmcblkX` → `/dev/nvme0n1`，偏移/扇区数不变

## 形态差异表（未列项两形态通用）

| 项 | plain | secure |
|---|---|---|
| 镜像 | 无 LUKS | crypt（rootfs 全 LUKS） |
| security 分区 | 无 | 有（PARTLABEL=security） |
| Rescue 引导 | boot 分区 ext4 `/recovery/` + booti | 专用 recovery 分区 raw FIT（boot_fit） |
| OTA 包 boot 成员 | boot.tar.<fmt> | boot.itb |
| 首启额外观察 | — | AB: journal SSKR 迁移 86da→SSKR；REC: 串口明文口令窗口 |
| A-7 残留记录 | findmnt armbi_ | + cryptsetup status armbian-ota-root-<slot> |
| §5 克隆盘锚定 | 分区/PARTLABEL 解析 | + security 分区 / LUKS 容器（decryption-disk） |
| §7 失配方向 | 换 **secure** loader | 换 **plain** loader |

## 判据与工具（各步骤通用）

- **每步 UART 取证（强制）**：每步执行后（尤其每次 boot 后）立即抓 `usb0` 缓冲存档并核对预期行——
  journal 与串口双源互证，防 journal 丢失/轮转掩盖（volatile journal 会滚）。存
  `.claude/test_plan/<批号>/uart/<步骤号>-<HHMMSS>.txt`，按该步预期 grep（见串口配方节关键行表）。
  缓冲 ~1000 行、U-Boot 段会滚出，**迟抓即丢**，boot 完成 25s 内抓
- env 几何：主副本 `0x3f8000`、冗余副本 `0x3f0000`，各 `0x8000`（64 扇区）；冗余头 CRC(4)+flags(1)，flags=1
- 坏 env 判据（`ab_env_healthy` 语义，勿看退出码）：

```sh
fw_printenv 2>&1 >/dev/null | grep -qiE "bad crc|cannot read environment|failed to read environment" && echo BAD || echo OK
```

- 校验盘上 blob 头（应见 5 字节头，第 5 字节 = 01）：

```sh
dd if=/dev/nvme0n1 bs=512 skip=8128 count=1 status=none | od -A d -t x1 -N 8   # 0x3f8000
dd if=/dev/nvme0n1 bs=512 skip=8064 count=1 status=none | od -A d -t x1 -N 8   # 0x3f0000
```

## 成本预算（每形态一对 Session）

| 项 | 次数 | 构成 |
|---|---|---|
| 全盘烧录 | 2 | AB-F、REC-F 各一 |
| 仅 loader 烧录 | 1 | §7 换异形态 loader（SPI-NOR） |
| ssh 重启 | ~12 | §1×2、§2a×1、§3×1、§6×1、§2b×2 + C/F 系列新增（261124 批 AB 段实测 12） |
| 重挂 /mnt/imgs | ~7 | 跟随 reboot；同窗打包免重启项可压（261124 批散窗实测 6 次仅 AB 段） |
| 人工断电 | 3 窗口 + F1/B-6 短断电 | W1 烧 AB、W3 烧 REC（A-9 软件化后无独立 W2，硬停后断电直滚 W3）；F1 坏槽硬停 ~4 次快速断电；B-6 apply 中断 1 次 |

## 覆盖矩阵

| 手册项 | 执行位 |
|---|---|
| §1 A/B 首启 init-uboot | A-1–A-3 |
| §2a A/B 双坏自愈 + PR45 增验 | A-4 |
| §2b Recovery 双坏自愈 | B-4 |
| §3 正常事务 | A-7（并入 §4 事务） |
| §3b 并发锁 / 负例 | A-5 负例、A-7 锁；B-3 recovery 复测 |
| §4 陈旧挂载自愈 | A-7（kill 真事务制造残留） |
| §5 克隆盘锚定 | A-6–A-8、B-2/B-4（锚定面按形态差异表） |
| §6 payload-id 翻转顺序 | B-2 |
| §7 loader/OS 失配硬停 | A-9（窗口 W2，方向按形态差异表） |
| 闸门负例 G1–G7 | A-5b（B-3 补 G6 反向） |
| 命令面 C1–C7 | A-5c（B-3b 补 --force/--no-backup） |
| 失败自动回落 + 坏槽修复 F1 | A-5d |
| recovery 侧中断恢复（B-6/F1 对应物） | B-6（窗口 W4） |
| 双干扰源三盘锚定 + 引导选择 | A-6b |
| 首启 userdata resize（6bfacffe：secure 单 boot 分区+LUKS+ext4） | A-1、B-1 四查；A-2/A-3 安静复跑 |
| 双坏 env 引导内重建（0d8c9f34+809c0676+a5c6d1c8） | A-4、B-4（B-4 判定口径已按新实现重写） |
| itb 写回 + 目标锚定闸（e03211e9/4f70b1b1/b048f73f） | B-2 硬断言；混装/无锚负例未覆盖 |
| pre-OTA armbianEnv 断电存续（c9200e89） | B-6 增验 |

## 排序理由

- §1 只能占烧录后首启，放最前；负例/锁等零重启项穿插消化
- **同一 ssh 活窗打包所有"免重启且需 /mnt/imgs"的步骤**（A-5 负例 + A-5b G 系列 + A-5c C5/C4），
  reboot 会杀挂载，散窗执行 = 每窗重挂（261124 批实测散窗代价 ~6 次重挂）
- **A-4 必须在 A-7 之前**：双坏重建丢槽记忆（boot_success 被重置为探测值），A-7 事务成功态会被
  A-4 清掉（261124 批反序补跑实证）
- §2a 先于 §3：自愈刚重建的健康 env，才能可信判定 §3"无 rebuilding 警告"
- §4 用真事务 kill 制造残留（比合成 bind mount 更贴原始事故），重跑的同一事务就是 §3 本体，并与 §3b 锁、§5 克隆盘合并 → 省 1 次重启
- §7 收尾 Session AB：halt 后必断电，断电窗口直接滚进 Session REC 烧录（含写回本形态 loader），零额外恢复

## 预备（host 侧，不占设备）

- 构建两镜像：AB-F、REC-F，同 commit；**REC 配 VERSION 互异包**（payload-id 才翻转，§6 才有意义；
  同版包只能配 B-3b 的 --force 重装场景——261124 批 office 现货 REC 包即同版，须另配异版包或明确降级）；
  **AB 同 VERSION 可行**（无 version 闸，261124 批实证）；boot 成员名按形态差异表。
  payload-id 已改内容寻址 v2（85d70694：R/B 哈希+域 tag，BOARD/RELEASE/KERNEL/VERSION 不进哈希输入）——
  同 VERSION 异内容也翻转；配包策略不变，仍按异 VERSION
- 坏包 ×1：解包删 boot 成员 → 重打包
- USB 克隆盘：dd 同镜像（两 Session 可复用一块盘，Session 间在 host 重 dd）
- 本形态 + 异形态 loader 文件（loaders/ 相对路径）；scp 通道就绪、串口 screen 会话按下节起好
- OTA 包投放：office 镜像目录反向隧道 sshfs 挂到设备 `/mnt/imgs`，包放 `history/<批号>/`，
  设备端直接 `start /mnt/imgs/history/<批号>/pkg.tar.gz`。挂载命令（Claude 经 `ssh office` 执行，幂等）：

```sh
ssh office 'SSH_AUTH_SOCK=/run/user/1000/gcr/ssh ~/upgrade_tool/mnt-imgs.sh root@recomputer-rk3576-devkit.lan -p root'
```

  gcr socket 是 office 上唯一持钥的 agent（`gpg-agent.ssh` 为空）；设备密码 root/root；脚本自动
  装 sshfs、建 2200 隧道 + agent 转发并挂 `/mnt/imgs` → office `~/upgrade_tool/imgs`（实测 ~50MB/s，
  设备与 office 同 LAN）。设备上不落任何密钥（office 自身公钥进 office authorized_keys，标记 mnt-imgs）。
- 记录烧录镜像与 OTA 包的 sha256（host 侧，入报告）

## 烧录后形态自检（用户手动刷完并告知后，Claude 执行）

```sh
cat /etc/armbian-image-release        # 身份块（含 SEEED_EXTENSION_COMMIT 核对被测版本）
lsblk -o NAME,TYPE,PARTLABEL /dev/nvme0n1
blkid | grep -E "PARTLABEL|LUKS"
cat /proc/cmdline                     # armbian.ota_role=recovery / distro bootargs
fw_printenv | grep -E "boot_slot|ab_boot_mode"
```

判定：`boot_a/boot_b` → A/B 布局，仅 `boot` → recovery 布局；`security` PARTLABEL 或 lsblk
`TYPE=crypt` → secure 形态；专用 `recovery` 分区（raw 无 fs）→ secure recovery，boot 为
ext4 `armbi_boot` → plain recovery。身份块与烧录镜像 sha256 三方核对后即生成报告骨架
`.claude/test_plan/<批号>/<form>-ota-test-report.md`：身份块节 + 每个测试步骤一节
（`## A-4 双坏自愈` 等），节内固定五行：命令 / 预期 / 通过标准 / 证据（待填，贴 journal、
串口、命令输出）/ 判定（待填 ✓✗）。后续逐步贴输出，填一节判一节，失败节按失败处置单项重跑。

## 重挂 /mnt/imgs 与远程恢复

- **每次 reboot 后 mount 不复活**：跑事务前先重挂（重跑上方挂载命令，幂等；50002 有僵尸转发会话时先 kill 设备侧 sshd pid 再挂）
- **重刷后第一件事：`systemctl enable ssh`**（镜像出厂未 enable；首启 ssh 可达只是暂态——
  261124 批实测用户漏 enable 时重新上电即失联）。路径：隧道若活用密码 root/root（sshpass）连入；
  隧道不通走串口 root/root 登录。enable 后顺手推本机公钥（免后续密码），host key 必变须先
  `ssh-keygen -R '[127.0.0.1]:50003'` + accept-new

ssh 仍失联按三坑处置（顺序 1→3→2）：userdata 510M 两段式扩容（cryptsetup resize+resize2fs）、
50002 僵尸转发会话（kill 设备侧 sshd pid 等重连）、openssh disabled（串口 enable）。

## 串口监控配方（Claude 远程读写的 screen 会话）

- **执行规则：每步一抓**。Claude 在每个测试步骤收尾（boot 后 25s 内、事务后、异常时）跑一次取证据：
  存档 + 按该步"预期行"grep 判定，报告证据节贴串口行（与 journal 互证）
- office 上有常驻会话 `usb0`（devkit UART，CP2105 适配器，用户 Attached 查看）；`industrial` 会话另接 1a86
- Claude 读（含回滚缓冲）：`ssh office 'screen -S usb0 -X hardcopy -h /tmp/uart.txt && tail -N /tmp/uart.txt'`
  （hardcopy 不抢 Attached 终端；缓冲 ~1000 行，约一次 initramfs→login 全程。**成功启动抓 U-Boot 段须在
  reboot 后 ~5-15s 窗口内连抓**，否则被内核输出滚掉；失败启动（硬停）天然全保）
- 写（发命令进 shell）：`ssh office 'screen -S usb0 -X stuff "<命令>\n"'`
- 用户人工查看：`screen -x usb0`
- 各步预期行（grep 目标）：boot 类 = `security partition resolved: <启动盘>p3`、`SSKR`、`mapper ready`、
  `overlayroot: configured`；A-4 = 坏 env 后 `persistent env unreadable`（journal）+ 正常解锁链（串口）；
  A-7/A/B 事务 reboot = 候选槽解锁（`Found LUKS device: …p5`）；A-9 = `loader/OS mismatch` + `panic`；
  B-2 = `OTA done, rebooting` / rescue apply 行
- A-9（halt 后 ssh 不可达）、B-1 首启观察、B-2 Rescue apply 全程、RESCUE SHELL 交互/解武均走此通道

## 前置确认（每 Session 首启后第一步；不通过则中止该 Session）

```sh
cat /etc/armbian-image-release
```

逐项核对并记入报告：

- `BOARD` / `RELEASE` / `IMAGE_VARIANT` / `VERSION`：与烧录镜像、OTA 包三方一致
- `SEEED_EXTENSION_REPO` + `SEEED_EXTENSION_COMMIT`：**必须等于被测 commit**，不符先烧新镜像再测
- `BUILD_REPOSITORY_COMMIT`（带 `-dirty` 需注明）、`IMAGE_UUID`

版本核对辅以 `uname -v`：事务后 image-release 可能被 upper 陈旧文件遮蔽显示旧值
（261121 批 E1 实证，属预期非失败），前置确认只在首启做。

---

## Session AB(F)：A/B-F

### A-0 🔌 人工断电窗口 W1
关机 → NVMe 全盘烧 AB-F 镜像 + SPI-NOR 写 F 形态 loader → 上电。

### A-1 首启（前置确认 + §1 init-uboot 全链）

```sh
cat /etc/armbian-image-release     # § 前置确认：逐项核对，含 SEEED_EXTENSION_COMMIT
journalctl -b | grep -iE "SSKR|tee-supplicant"   # [仅 secure] 首启 SSKR 迁移（仅首启可观察）
systemctl status armbian-ota-init-uboot.service
journalctl -b -u armbian-ota-init-uboot | grep -E "Installed|unreadable"
cat /etc/fw_env.config        # 双条目 3 字段（无 erase 列），设备为父盘 /dev/nvme0n1
fw_printenv | head
```

init-uboot 已并入 common 运行时（891a2c1a）：journal 之外日志另落 `/var/log/armbian-ota/ota.log`，
uboot_initialized 标记迁至 `/var/lib/armbian-ota/`——取证时可双源。

通过标准（env 部分）：config 双条目 3 字段、boot_slot/分区变量读回正确、系统正常启动进 slot。

首启 userdata resize（烧录后必测；镜像出厂 510M → 扩到盘尾）：

```sh
systemctl status armbian-resize-userdata.service   # active (exited)，非 failed/restarting 循环
journalctl -b -u armbian-resize-userdata | grep resize-userdata
lsblk -o NAME,SIZE,TYPE,PARTLABEL | grep -i userdata   # 分区已扩至盘尾（对比盘总容量）
df -h /media/root-rw                               # overlay upper 已是扩容后容量（两形态通用）
```

[仅 secure] journal 须含 `resizing LUKS mapping arbian-userdata`——分区+LUKS+ext4 三级**单 boot** 完成（6bfacffe，旧两段式已废）。
[仅 secure] exit 75 循环后收敛也算过，但须记录重试次数；plain 不应见任何 LUKS 行。

通过标准：service exit 0、userdata 容量 ≈ 盘尾剩余、无 "not the last partition" / "refusing to shrink" /
"unsupported userdata filesystem"。

### A-2/A-3 重启×2（§1 安静通过）

```sh
reboot                        # 两次；每次 boot 后看
journalctl -b -u armbian-ota-init-uboot --no-pager | tail -3
journalctl -b -u armbian-resize-userdata --no-pager | tail -2   # 安静复跑：already uses available space → resize2fs 快退 exit 0
```

通过标准：每 boot 服务都启动（无 Condition 跳过）、健康时快速退出、无 "unreadable"、
boot 耗时无明显增加（早退 = 一次 fw_printenv）。

### A-4 双坏自愈（§2a + PR45 增验）

```sh
sync
dd if=/dev/zero of=/dev/nvme0n1 bs=512 seek=8128 count=64 conv=notrunc   # 0x3f8000
dd if=/dev/zero of=/dev/nvme0n1 bs=512 seek=8064 count=64 conv=notrunc   # 0x3f0000
sync && reboot
```

重启后验证：

```sh
journalctl -b -u armbian-ota-init-uboot | grep -E "unreadable|Installed"
fw_printenv | grep -E "boot_slot|boot_success"    # slot 正确
# 判据与工具的坏 env 判据 → OK；双偏移 od 头均合法（另一副本可为零/旧）
```

通过标准：一次重启内自愈完成、slot/boot_success 与实际运行 slot 一致、无手工干预；
config 仍双条目（未误判单副本）、两副本 blob 头 flags=01（PR45 增验收口）。

261125 增验（0d8c9f34）：重建槽位状态感知（ab_env_rebuild_slot）——journal 须见
`persistent env unreadable; re-running full init (fw_setenv -f rebuild)`；若双坏时有武装事务
（state ready_to_boot/boot_verifying），重建槽取 state 的 CURRENT_SLOT 而非运行槽——双坏前先
switch-slot 武装再零化 env，可验证 armed 槽不丢。
[仅 secure] 遗留待拍板（261125）：重建发生在 rootfs 起（init-uboot），env 双坏时引导期解密链
若先失败则到不了重建点、无硬件层兜底——实测确认首启解密是否收敛（scan order 下勿插克隆盘）。

### A-5 负例（§3b 前半，零重启；A-4 刚 reboot，先重挂 /mnt/imgs）

```sh
/usr/sbin/armbian-ota start bad.tar.gz
# 拒绝在 extract 之后（断言跑在解压目录上，分钟级非秒级，261121 批事务 ~73s 量级）
# 报错含 "dual-payload assertion failed"（no boot payload）
fw_printenv -n ota_in_progress   # 未被触碰（仍 0/unset）
```

### A-5b 闸门负例套件（G1–G7，两形态通用；换树必重跑——gate 代码重构频繁）

gate 全在签名校验**之前**生效，故篡改 `package.env` 即可触发，无需伪造签名。制包（office 侧）：
解包正包 → `sed` 改 package.env 字段 → `tar -cf`（免重压缩）→ 落 `history/<批号>/gates/`；
plain/REC/raw 三例直接用现成文件软链。每例收口判据：RC≠0 + `ota_in_progress` 不触碰。

| # | 包 | 预期拒绝点 |
|---|---|---|
| G1 | board.tar（BOARD 改异板） | `BOARD mismatch` |
| G2 | release-var.tar（RELEASE+VARIANT 双改，无 flag） | `RELEASE mismatch (…--allow-release-change)` |
| G3 | 同上 + `--allow-release-change` | WARN 放行 release → `IMAGE_VARIANT mismatch`（防误入真事务：双改确保再被拦） |
| G4 | variant.tar（仅 VARIANT 改 desktop） | `IMAGE_VARIANT mismatch`（单因） |
| G5 | plain AB 正包（xz） | `encryption mismatch: package=no, device=yes` |
| G6 | REC zst 正包 | `mode mismatch`（recovery backend 没装/have: ab） |
| G7 | 原始 .img.xz | `metadata missing package.env` |

未覆盖（本 rig 无素材）：容量/空间闸**拒绝**分支（需超尺寸包或缩分区）、payload magic mismatch、
双 boot 成员 ambiguous、zst 协商拒绝、加密反向闸（plain 设备收加密包，留 plain session）。

### A-5c 命令覆盖套件（C 系列；A/B 命令面，两形态中 ab 部分通用）

CLI 公开命令 `start/status/switch-slot` + 内部 `mark-success/rollback/reconcile` 的非事务路径。
每步按"每步 UART 取证"规则抓 `usb0` 存档。

| # | 操作 | 预期 |
|---|---|---|
| C1 | `switch-slot <t>` 双向 | armed：boot_slot=t、**boot_success 保持锚槽**、ota_in_progress=1、retry=3、state ready_to_boot；reboot 后健康自动确认 boot_success=t（不预标成功，坏槽可回） |
| C2 | 守卫组 | 事务中 switch-slot 拒绝；目标=当前槽 `Already booting`；坏参 usage 拒绝 |
| C3 | `rollback` 无事务 | `nothing to rollback` RC=0 |
| C4 | `rollback` 真路径 | armed 事务 → rollback → env 复位（boot_slot=boot_success 锚槽、ota_in_progress=0）+ state=rollback + **自动 reboot** 回锚槽。261125 新守卫（c9e9999b）：env 已确认槽（boot_success=运行槽）不再被 rollback——触发需状态文件写失败，难低成本构造，真机仅验正常路径无回归 |
| C5 | `reconcile` 显式 | armed 后手动 `fw_setenv ota_in_progress 0`（代码注释认定的合法模拟：power cut/manual）→ reconcile 记 U-Boot-rollback 分支：`target slot … did not pass boot verification; running confirmed slot …`、state=rollback、TARGET 清空；随后**须手工恢复 boot_slot 锚**防静默换槽 |
| C6/C7 | usage / 非 root | usage 文本、`must be run as root`（root 守卫先于包检查） |

未覆盖：`--force`/`--no-backup`（recovery 专属→B 系列）；reconcile 失败分支（需 current==target 且
未确认的场景）；start 未知选项拒绝（trivial）。

### A-5d 升级后启动失败自动回落（F1）

| 步骤 | 预期 |
|---|---|
| 破坏 boot_b（`dd` 1MB 零进 p2 头）→ `switch-slot b` 武装（retry=3）→ reboot | 每次尝试：串口先见 `Saving Environment … Writing to redundant nvme… done`（扣减落盘）→ `FIT: No fit blob` → `### ERROR ### Please RESET`（**硬停无 CLI、不自动 reset——须人工断电驱动扣减**，起始 3 则断电 3 次后第 4 次自动回落） |
| retry 耗尽 | ab_preboot 自动：`boot_slot=boot_success`、`ota_in_progress=0`、`retry=max`、saveenv、reset → 锚槽正常启动 |
| 锚槽落地 | reconcile 记 U-Boot-rollback：`target slot … did not pass boot verification; running confirmed slot …`，state=rollback |
| 修复 | 重跑 OTA start 重写坏槽（p2+rootfs）→ reboot 候选槽健康确认（验坏槽修复路径）。**可与 A-7 牺牲事务合并**：F1 回落后直接进 A-6/A-7，牺牲事务本身就是重写坏槽——省一次全量事务 |

U-Boot 阶段 ssh 不可达，全程串口取证；扣减次数无法从 env 终态回读（回落后 retry 复位 max），
记录人工断电次数入报告。内核起但健康失败的变体（F2：损坏 rootfs 而非 boot）留后续。

### A-6 插克隆盘（§5）

USB 克隆盘插入；`blkid` 应见双份 PARTLABEL（[仅 secure] `grep -c security` 计双份；
[仅 plain] 计 boot_a/rootfs_a 等分区双份，无 security）。

### A-6b 双干扰源（USB + SD 同场，261124 批新增）

SD 克隆盘（dd 同镜像，可启动）+ USB 克隆盘同时在场——三盘型并存（nvme+sd+mmcblk）：

| 检查点 | 预期 |
|---|---|
| 三份 PARTLABEL | blkid 计 3×（nvme/sda/mmcblkXpY 各一套） |
| **mmcblk 锚定分支** | `get_partition_number`/`p?[0-9]`/`ab_blkid_first_on_disk` 的 mmcblk 分支首次真机覆盖；resize/ab 解析仍落 nvme |
| **U-Boot 引导设备选择** | SD 在场 reboot → `armbian.bootdev=mmc`（**SD 抢走 NVMe = RK 既定行为，261124 实测**，记录在案即可）；拔 SD 后 bootdev=nvme |
| 时机 | 若与 A-9 同轮 reboot：mtd 换 loader 前插好 SD，一次 reboot 双验（引导选择 + mismatch 硬停） |

**261124 批源码级结论（A-6b 后续）**：U-Boot 引导设备选择在 C 级 `boot_devtype_init`（atag `mmc 1`
→ `bootdev_init` 只看卡在不在、不看可引导性），**rkimgtest 根本不参与**——SD 卡物理在位 = 永远从
SD 引导，清 idblock/FIT 头无效。**U-Boot env 跟随引导盘**（bootdisk 0x3f8000）：SD 引导读 SD env
（镜像默认）、NVMe 引导读 NVMe env（武装态）——SD 在场时 NVMe 的武装事务被无限搁置。
克隆 SD 盘在 recovery session 全程**必须拔除**（A/B session 的 USB 克隆无此问题）。

成本：office dd ~10G → SD 约 10-15 分钟 + 一次物理插卡。

### A-7 合并事务（§4 + §3b 锁 + §3 + §5）

```sh
# 1. 牺牲事务：后台起，等 target 挂载出现后杀掉（制造残留）
/usr/sbin/armbian-ota start pkg.tar.gz &     # 记 start 的精确 PID
until findmnt | grep -q armbian-ota-root-; do sleep 5; done   # [仅 secure]（plain 无 LUKS mapper，改盯 findmnt | grep armbi_）
sleep 30 && kill -9 <精确PID>
# 勿 pkill/pgrep -f（261121 批实证会匹配自身命令行、误杀 ssh 会话）
# kill 只杀顶层 bash；孤儿写者留下为预期——正是 Detaching 有界等待要验的场景
findmnt | grep armbi_                         # 记录残留挂载（两形态通用）
cryptsetup status armbian-ota-root-<slot>     # [仅 secure] 记录残留 mapper

# 2. 真事务：前台重跑
/usr/sbin/armbian-ota start pkg.tar.gz
# 期望：[仅 secure] Stale LUKS mapper ... closing →（两形态通用）Detaching stale target-root mount（§4）
# 事务期间另一终端并发：/usr/sbin/armbian-ota start pkg2 → Another OTA process is running（§3b 锁）
fw_printenv -n ota_in_progress               # =1，无 WARNING "rebuilding env"（§3）
# ota_boot_next 仅 recovery 有值；AB 语义为空（261124 实证），勿当失败

reboot                                        # 候选 slot 启动（双盘在场，仍从启动盘引导/解锁）
/usr/sbin/armbian-ota status && /usr/sbin/armbian-ota mark-success
fw_printenv -n ota_in_progress               # =0，flags 清除
cat /var/lib/armbian-ota/ota-state.env       # OTA_MODE/STATUS/TARGET_SLOT 键齐全（单次原子写）
systemctl status armbian-ota-firstboot.service
ls -l /boot/initrd.img* 2>/dev/null          # 无 dangling initrd.img/initrd.img.old 软链（77870bd9）
ls /tmp/syspw 2>&1                           # [仅 secure] 须不存在（keybox 各失败/成功路径均清理，78e7ff29/a0cce75f）
```

通过标准：arm 两键写入即成功（坏 CRC 重试不触发）、读回一致、事务全程 env 写入零失败、
**候选槽健康检查自动确认**（firstboot，261124 批实证先于手动），`mark-success` 为幂等兜底，
状态与 env 收敛；并发第二个 start 退出码非 0 且不碰 env/state；
Detaching 拆干净（含嵌套）后 mkfs 正常（残留写者 120s 拆不掉才失败，负例可不做）；
/boot 无悬空 initrd 软链；[仅 secure] /tmp/syspw 不留存（含杀事务后的失败路径复跑）。

### A-8 §5 AB 收口

journal 确认分区解析选中 `/dev/nvme0n1p*`（[仅 secure] security 分区路径 ab_get_security_part；
[仅 plain] boot/rootfs PARTLABEL 路径），均非克隆盘；拔克隆盘。

### A-9（§7，方向按形态）—— **软件化，无需断电/maskrom**

ssh 在线时直接换 loader（设备 mtdblock0 16M，SPI-NOR 运行期不占用，热写安全）：

```sh
# 先重挂 /mnt/imgs，然后：
dd if=/mnt/imgs/<批号>/<异形态>/rkspi_loader.img of=/dev/mtdblock0 bs=64k conv=fsync status=none
sync; md5sum /dev/mtdblock0 /mnt/imgs/<批号>/<异形态>/rkspi_loader.img   # 读回比对
reboot   # 串口观察硬停（ssh 不会再回来——闸门生效的反证）
```

261124 批实证：16M 写入 ~5 分钟（mtdblock 慢），读回 md5 比对防半写。上电/重启后串口预期（261124 批实测修正）：

- **F=secure（plain loader）+ SSKR 已迁移盘**：`SSKR marker found → Error: SSKR marker requires working
  OP-TEE and keybox_app` → **落到 cryptroot 手动口令提示**（非 panic——panic 只在 86da 分支）；口令为
  二进制 keyfile 控制台不可输入 → 事实死锁、rootfs 保持锁定 ✓。内核侧佐证 `optee: api uid mismatch`
- **F=secure（plain loader）+ 86da 新鲜盘（raw-FIT 引导）**：FIT 自带 bl32 可使 keybox_ready 通过 →
  raw passphrase 放行、系统完整启动（**发现：86da 闸对 raw-FIT 引导存在缺口**，且 mmc/nvme 引导路径
  对 FIT 内 TEE 处理有差异——SD 引导时 /dev/tee0 存在、NVMe 引导时 api uid mismatch）
- **引导设备选择**：可启动 SD 在场时 U-Boot 选 SD（`Bootdev(atags): mmc 1`）——RK boot ROM 既定
  行为（SD 最高优先），非缺陷；A-9 须拔 SD/USB 才能测到 NVMe 场景
- F=plain（secure loader，AB）预期：`booti: raw Image boot is disabled on verified-boot builds`
  （secure 侧 booti lockdown；distro_bootcmd/Rescue fs 路径为已知残余缺口）

通过标准（两方向统一语义）：rootfs 保持锁定或启动停止、无明文 fallback、不得回退挂明文 rootfs、
不得走到 overlayroot RW fallback。观察完 → **人工断电**（设备锁死/halt，ssh 不可达）。

---

## Session REC(F)：recovery-F

### B-0 🔌 人工断电窗口 W3（紧接 W2，设备已在断电态——A-9 硬停后人工断电）
NVMe 全盘烧 REC-F 镜像；loader 写回 F 形态用同款 dd 配方（若设备已能 ssh；否则随 NVMe 烧录一并物理写）→ 上电。

### B-1 首启

前置确认（同 A-1：`cat /etc/armbian-image-release` 逐项核对）。
userdata resize 验证同 A-1（service/journal/lsblk/df 四查 + [仅 secure] LUKS 单 boot 行；recovery 布局
userdata 同为末分区，`/media/root-rw` 同为 overlay upper）。
[仅 secure] 串口观察首启**明文口令窗口**（security marker 86da → tee-supplicant → SSKR 迁移，
仅首启可见）；健康进入 normal（decryption-disk 正常解锁）。
记录 payload-id 基线（**normal 态 boot 分区不挂载**——rootfs 侧镜像为准：
`cat /usr/share/armbian-ota/installed-payload-id`；boot 分区上的 `payload-id` 仅 apply 时挂载可见）；
插克隆盘。

### B-2 Recovery OTA（§6 + §5 recovery 面）

```sh
/usr/sbin/armbian-ota start pkg.tar.gz
fw_printenv ota_in_progress ota_boot_next    # =1 / =recovery
reboot                                        # 进 Rescue apply（期间 ssh 不可达，观察走串口）
# [仅 secure] Rescue 走专用 recovery 分区 raw FIT（boot_fit）；[仅 plain] 走 boot 分区 ext4 /recovery/ + booti
# apply 完成后设备自动 reboot 回 normal（串口 "OTA done, rebooting"，手动重启预算不变）
# 硬断言（前后）：apply 前 installed-payload-id=旧值；回 normal 后 == status 的 CANDIDATE_ID → mark-success
# [仅 secure] itb 写回硬断言（e03211e9/b048f73f）：回 normal 后 raw boot 分区 FIT md5 == 包内 boot.itb
#   ——itb 写回曾被折叠进 DO_BOOT_OTA=0 区域（新 rootfs+旧 FIT 恰是致命组合），现按 HAS_BOOT_ITB 无条件写
# 串口日志若含 boot 文件交换序，记录 payload-id 行的相对位置（应最晚）
# 克隆盘在场：armbi_boot/rootfs（[仅 secure] +LUKS 容器）解析均落启动盘（§5）
# settle 后：fw_printenv ota_boot_next → =normal（0d8c9f34 收口复位；不再永远留 recovery 读作 pending OTA）
```

通过标准：payload-id 归入 critical 类最后翻转（health 比对不会在 boot 文件半新半旧时提前放行）、
完成后与 status 的 ID 一致；全程无错盘写入/错盘解锁；[仅 secure] 盘上 FIT == 包内 boot.itb；
settle 后 ota_boot_next=normal。观察项：backup 创建（--no-backup 之外）raw-FIT 分支现含
分区回读校验（b85fec2e，journal 记录即可，无须主动破坏）。

**261124 批教训：B-2 全程必须拔 SD 克隆盘**（见 A-6b 源码级结论——SD 在位劫持引导，
武装事务被搁置 + rescue 路由失败 fallback 到 SD normal）；克隆干扰用 USB 盘即可
（rkimg 扫描不含 USB）。apply 后凭据存续行为以实测为准（261124 批未走到，待补）。

### B-3 负例复测（§3b recovery 同语义；B-2 自动回 normal，先重挂 /mnt/imgs）

```sh
/usr/sbin/armbian-ota start bad.tar.gz      # 同款拒绝（extract 后断言，分钟级）
/usr/sbin/armbian-ota start <AB 包>         # G6 反向：require_mode_backend 拒（have: recovery）
```

### B-3b recovery 专属 flags（A-5c 遗留项的执行位）

- `--force`：B-2 装过同 ID 包后直接再 `start` 同包 → INSTALLED_ID 重复闸拒绝；加 `--force` → 放行重装
  （素材允许同 VERSION 包，正好构造真实重复场景）
- `--no-backup`：`start --no-backup` → 跳过 lazy backup 创建（journal/串口无 backup 建段、state
  BACKUP_ID 空）——与 B-6 配对验证"无备份时中断的后果"

### B-4 双坏自愈（§2b）

```sh
dd if=/dev/zero of=/dev/nvme0n1 bs=512 seek=8128 count=64 conv=notrunc
dd if=/dev/zero of=/dev/nvme0n1 bs=512 seek=8064 count=64 conv=notrunc
sync && reboot
```

261124 批发现（历史背景）：recovery 侧零 env 时无路由 → 默认 bootcmd 直进 normal → 重建代码
永不触发，且武装因缺 bootdev token 硬失败 = 远程 OTA 死锁。**修复已合入 main**：
①0d8c9f34 init-bottom 探测+重建路由集（锚定经 find-userdata raw 分区解析，无需 cmdline token）；
②809c0676 排序修复——原 `99-ota-normal-marker` 按字典序跑在 overlayroot init-bottom 之前、
整条重建分支是死代码，改 `zz-` 前缀后生效；③a5c6d1c8 `fw_env_boot_disk` 接受 mapper 锚
（trixie udev 实节点），武装死锁随之解除。

**B-4 复测判定口径（261125+ 镜像）**：零化双副本 → 重启 →

- 串口/initramfs 日志见 `armbian-ota: env unreadable; rebuilding routing set on <disk>`，
  **同 boot 内**完成（fail-open：失败仅 `WARNING env rebuild failed; continuing boot` 不砖）
- 双坏时有武装事务（state ready_to_boot/boot_verifying）：改走
  `env unreadable; re-arming for Rescue (STATUS=…)`——armed 事务不丢
- 重启后 `fw_printenv` 路由集+基底完整、bootdev token 回归、**直接武装可跑**（原死锁解除）
- 保留注记：坏 env 首启解密仍走 scan order（窗口期勿插克隆盘）；loader 默认 env 仍不带路由
  （方向 ③ 未做，但 init-bottom 重建已消解其影响）

未做项（源码级）：rootfs 侧 AB 式重建服务；仅影响重建窗口内 rootfs 层观察。

### B-5 收口

拔克隆盘；[仅 secure] 串口/journal 确认 decryption-disk 的 LUKS 容器取自启动盘；终态记录（见报告节）。

### B-6 Rescue apply 中断恢复（F1 的 recovery 对应物；需一次人工断电窗口）

261124 批实测（--no-backup --force 载体，中断于 rootfs 解包中段）：

- **恢复语义 = 候选重放**（非备份恢复；备份在场未被使用）：上电 → rescue 路由 → campaign 重识别 →
  clean + 从头重解 → applied → 自动回 normal，一次上电 <2 min 收敛——幂等重放，arm 期
  "power loss is safe at any point" 证实
- 时序：上电 ~20s 进 apply、解包窗 ~28-66s（喊断电的窗口）、apply 全程 ~74s
- **载体武装前置**：须 bootdev token 在场（cmdline `armbian.bootdev=nvme`）——env 异常时先按
  B-4 恢复链修 env 再武装
- 串口全程存档（中断前 + 重放全程各一份）
- **261125 增验（c9200e89）pre-OTA armbianEnv 断电存续**：事务前在 armbianEnv 加自定义行
  （如 `myoverlay=1`）→ 中断选在 boot 清除后的 boot.tar 解包窗（分钟级宽窗）→ 重放收敛后
  自定义行**仍在**（捕获副本落 userdata/ota-recovery/staging，首捕获即定格；修复前 RAM 副本
  随断电死+盘上副本刚被清 → 重放折叠基线叠基线、用户 overlay 静默回出厂）。boot 分区捕获
  优先、rootfs /boot 仅兜底（e03211e9）——断电后 `ls` staging 副本在手前不 reboot

**B-4 双坏 env 手工恢复链（261124 处方，仅旧镜像兜底——261125+ 镜像 B-4 自愈后不再需要）**：`fw_setenv -f /etc/u-boot-initial-env <key> <val>`
只重建**最小集**（无 OTA 路由、无 bootdev token——仍不能武装）；完整恢复须**镜像 blob 级**：
office 从镜像 xz 提取 `xz -dc <img>.xz | dd bs=512 skip=8064 count=128`（0x3f0000..0x40000 双副本）
→ 推设备 → `dd of=/dev/nvme0n1 bs=512 seek=8064 conv=notrunc` → reboot（token 回归）→ 武装。

---

## 失败处置

- 合并事务任一观察点失败：先清残留（`umount -R` + [仅 secure] `cryptsetup luksClose armbian-ota-root-*`）再单项重跑；各项可独立复测，不依赖整链
- §7 未按预期（走到明文挂载/继续启动）→ 立即断电，保存串口全文
- 失败时附：`fw_printenv` 输出、`/etc/fw_env.config`、双偏移 od 头 16 字节、
  `journalctl -b -u armbian-ota-*`

## 本 rig 不覆盖（沿 261121 批残余缺口；A-6b 已部分收口）

- `p?` optional-p 的 sd/vd **启动盘侧**分支：锚定正则前缀由启动盘决定，devkit boot 盘为 NVMe 走不到
  （A-6b 的 mmcblk 克隆只覆盖干扰侧/BLKID 扫描分支，启动盘侧正则仍不触发）
- 真跨 release 全量切换：缺 resolute cli crypt 包
- itb-on-ext4 混装闸（4f70b1b1，campaign_fail `boot.itb payload but device exposes an ext4 boot
  partition` + start 侧拒配对）：需加密 ext4-boot 镜像 + itb 包素材，现有两形态包会先被加密闸拦
- find_raw_boot_dev 无锚定 fail-closed（4f70b1b1：cmdline 缺 bootdev token 拒写、campaign 失败）
  ——B-4 修复后正常路径 token 总在场，负例需手工剥离 cmdline
- rescue boot 分区挂载失败 fail-campaign（e101e111，warn-and-skip 已废）：需破坏挂载性（损 superblock）
- payload 哈希/ID 失败传播（cc22b869）：构建侧+备份侧故障注入，非真机路径
- raw 口令 SSKR 前缀拒写（acb4d747）：构建期守卫，非设备路径

## 报告

每类镜像独立一份报告（形态自检判定哪类就生成哪类，互不混写）：
`ab-plain` / `ab-secure` / `recovery-plain` / `recovery-secure` 四选二（本批 F 形态的两份），
文件名 `<form>-ota-test-report.md` 落 `.claude/test_plan/<批号>/`，**头部必含镜像身份块**：

```
- 设备/介质/loader：reComputer RK3576 DevKit / NVMe / SPI-NOR loader
- 镜像身份（/etc/armbian-image-release）：BOARD / RELEASE / IMAGE_VARIANT / VERSION /
  SEEED_EXTENSION_REPO + SEEED_EXTENSION_COMMIT / BUILD_REPOSITORY_COMMIT / IMAGE_UUID
- 烧录镜像 sha256 / OTA 包 sha256；烧录时间与 boot 计数基线
```

每项记录：板型 / 镜像（sha）/ boot 次数 / journal 关键行 / 通过-失败。
