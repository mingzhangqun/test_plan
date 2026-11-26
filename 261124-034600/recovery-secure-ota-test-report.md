# Recovery Secure OTA 真机测试报告（261124-034600 批次，devkit/NVMe）

- 日期：2026-11-24；设备：reComputer RK3576 DevKit（NVMe，secure loader 已写回，**recovery 味，全 LUKS**）
- 被测 commit：main tip 849bdd25（与 AB session 同树）
- 镜像：26.11.0-trunk trixie cli crypt（recovery 布局）
- 烧录镜像：`…-crypt.img.xz` sha256 a0e5f6d7ec872a57cac4e5e62043ebe763deb59ffb629309e9d53de91cdd9b38
  （未压缩 img 烧录后已删，xz 为源证物；设备身份以 SEEED_EXTENSION_COMMIT=849bdd25 三方核对为准）
- REC OTA 包：`…-crypt_OTA.tar.zst` a96505c721ea60193f0331cde23c780821d914488d41a4357e47351f2b71be8f
- 通道：ssh rk3576（重刷后清 host key + root/root + enable ssh + 推 key，全按 plan 配方）；串口 usb0

## 形态自检

五分区：boot(512M ext4 PARTLABEL=boot) + recovery(512M raw) + security(4M) + rootfs LUKS 3.8G +
userdata LUKS；cmdline 无 ota_role/slot（normal 从 rootfs 引导）；fwver 含 bl32-v1.08（secure loader
写回成功）；身份块 SEEED_EXTENSION_COMMIT=849bdd25505b… 一致 ✓

## B-1 首启

| 检查点 | 证据 | 判定 |
|---|---|---|
| userdata resize 四查 | service active；单 boot 三级链：`resizing userdata partition: p5`（510M→盘尾 end 11186175s→250052607s）→ `resizing LUKS mapping to 239878144 sectors` → `resizing ext4`；`/media/root-rw` 108G | ✓✓ |
| SSKR 首启迁移 | journal `[Decryption-disk] marker: 86da → No SSKR marker, reading raw passphrase`（首启明文窗口捕获）→ 迁移后 marker = **SSKR** | ✓ |
| payload-id 基线 | `installed-payload-id = 8ee2ed5b8593b99638f553e015ae53d6a7100203f1c92cff0d33d979cbf27cdb`（normal 态 boot 分区不挂载，rootfs 镜像为准——计划已修正） | ✓ |
| boot 计数基线 | boot 0 = 05:10:19 UTC（journal 唯一 boot，烧录后未重启） | ✓ |

## B-2 Recovery OTA

**第一段 武装**（05:35–05:36）：

| 检查点 | 证据 | 判定 |
|---|---|---|
| 同版未拒 | 无 --flag 直接武装成功（`Recovery OTA armed`）——新鲜 state 无 INSTALLED_ID，闸条件 `installed_id 非空` 不满足，**放行正确**；INSTALLED_ID 拒绝场景留 B-3b（apply 后 state 落盘再测） | ✓（语义确认） |
| lazy backup | B2-1 武装时创建、B2-2 复验：`Existing backup payload verified structurally: 15ba7caf…`，`Campaign prepared: candidate=8ee2ed5b… backup=15ba7caf…` | ✓ |
| 武装 env | ota_in_progress=1、ota_boot_next=recovery | ✓ |
| 克隆盘干扰 | sda 在场（REC 同镜像 dd，PARTUUID/LUKS UUID 全同），staging/武装全程锚定 nvme0n1 | ✓ |

**第二段 Rescue apply + 回 normal**（两轮劫持事故 + 机制发现）：

- **事故 1（SD 劫持）**：reboot 后 SD（AB 克隆镜像）抢引导 → SD 无 recovery 分区 → rescue 路由失败 →
  fallback 直接 normal 启动 SD rootfs_a（root/1234 默认态）——之前误判为"apply 后新系统锁出"
- **撤回"apply 后失联"产品发现**：apply 根本没执行（NVMe env 仍武装 `ota_in_progress=1`），
  锁出系统 = SD 克隆首启
- **机制发现（源码级，均已入计划）**：①引导设备选择在 C 级 `boot_devtype_init`，atag 恒报
  `mmc 1`、`bootdev_init` 只看卡在不在——**SD 在位必赢，可引导性无关**（清 idblock 实证无效）；
  ②**U-Boot env 跟随引导盘**——SD 引导读 SD env（无武装态），NVMe 武装事务被搁置
- **NVMe 介质错误虚惊**：YX-520 瞬时态（复测 1GB @352MB/s 零错误、坏点自愈），盘健康；
  p1 "bad superblock" 为误判（p1 本就是 raw FIT 分区，`d00dfeed` FDT magic ✓）
**第三段 正式 apply（拔 SD 后，串口全程证据 `recovery-secure-uart-log/B2-apply-capture.txt`）**：

| 阶段 | 证据 | 判定 |
|---|---|---|
| NVMe 引导 + 解锁 | `mmc1: card removed` → NVMe 引导 → SSKR → p4/p5 解锁全链 | ✓ |
| 99-ota-apply | `OTA_DEBG init-premount ota hook start (role=recovery)` → `campaign 8ee2ed5b… attempt 1/3` | ✓ |
| rootfs 重建 | `mkfs path skipped`（/boot on rootfs 路径）→ `cleaning /mnt/root (except boot)` → tar 解包 → `rootfs payload marker written: 8ee2ed5b…` | ✓ |
| upper 选择性清洗 | `overlay upper cleaned (lib/modules, overlayroot.conf)` + 移除 stale autologin override ×2——**authorized_keys/shadow/ssh enable 不在清洗列表，apply 后双认证全通**（KEY_OK+PW_OK；SD 劫持误判正式闭环） | ✓✓ |
| **boot 裸写+校验** | `writing raw FIT boot image … boot.itb (74257920 bytes) → /dev/nvme0n1p1` → `verifying written FIT against boot.sha256` → `boot.itb verified after write (sha256=b6b9181e…)` | ✓✓ |
| 收尾 | fstab/crypttab 修补（ROOT_UUID 954c91a9）→ `applied 8ee2ed5b…; handing off to normal` → login | ✓ |
| 终态 | status: Mode=recovery **Status=success**、Installed=8ee2ed5b（=candidate=rootfs marker 三方一致）、payload store 1.3G installed、ota_in_progress=0 | ✓✓ |

注记：`ota_boot_next=recovery` 残留（inert：路由只看 ota_in_progress=1）；apply 中 tar 时间戳
future 警告为设备时钟漂移，无害。§6 payload-id 翻转顺序：同版包无翻转可观察，boot.itb 裸写为
"boot OTA disabled"行之后的独立校验步骤（`writing→verifying→verified` 顺序完整在案）。

## B-3 负例 + G6 反向 + B-3b --force

| # | 输入 | 拒绝点 | 判定 |
|---|---|---|---|
| B-3a | bad.tar.gz（AB 味坏包） | `ab OTA backend is not installed (have: recovery)`——mode 闸先于断言（AB 味包做 recovery 坏包不可用，已另制 REC 味） | ✓ |
| B-3b | AB 正包 | 同上（G6 反向） | ✓ |
| B-3c | REC 正包（无 --force） | `Payload 8ee2ed5b… is already installed (INSTALLED_ID); use --force to reinstall`——apply 后 state 落盘，闸生效 | ✓ |
| B-3d | REC 味坏包 + --force | 穿过 INSTALLED_ID → space/capacity gate 过 → staging → `KIND=full package has no boot payload (dual-payload assertion failed)` RC=1、env 不触碰 | ✓✓ |

注：挂载瞬态 ENOENT 一次（sshfs 目录缓存，重试即愈）；`--force` 在 B-3c 留闸、B-3d 穿闸，
两侧语义均验证。`--no-backup` 的后果面并入 B-6 载体（--no-backup --force 武装后中断）。

## B-4 双坏自愈（§2b）

| 检查点 | 证据 | 判定 |
|---|---|---|
| 双坏后可启动 | 25s 回 normal；串口解密链完整（scan order 解 p4） | ✓（不砖） |
| **boot 时 env 重建** | **未发生**：blob 仍全零、`fw_printenv` "Cannot read environment, using default"——U-Boot 内置默认 env 直接 normal 引导，未走 Rescue/未跑 99-ota-apply | ✗（与 §2b 预期不符，**发现**） |
| **重测确认（纯 NVMe 单盘，用户质疑盘干扰后复测）** | 依然全零、无 reconciliation 活动、无 token——**盘干扰排除**。**根因钉死（源码级）**：99-ota-apply 的 env 重建代码入口条件是 cmdline `armbian.ota_role=recovery`（只有健康 env 的 ota_rescue_boot 路由才会设）——零 env → 默认 bootcmd（`boot_fit;` 无路由）→ role=normal → hook 秒退（实证：`/run/initramfs/ota.log` 未创建）；AB 则有 rootfs 侧 init-uboot 无条件重建。**鸡生蛋锁死** | ✗（发现+机理） |
| 默认 env 锚定 | 无 `bootdev` token → decryption-disk 走 scan order（克隆盘在场时有错盘风险——本次已拔盘无碍） | ⚠ 风险注记 |
| 武装链（B-6 首次武装失败实证） | 零 env → 无 token → `Cannot resolve boot disk for fw_env` → **OTA 武装硬失败 = 远程 OTA 死锁**，须 blob 级人工恢复（配方已固化计划 B-6 节） | ⚠ 产品级 |

**发现定性**：recovery 侧无 boot 时 env 重建（与 AB 的 init-uboot 每 boot 重建不同）；双坏 env 下设备
**不砖但远程 OTA 死锁**（B-6 首次武装硬失败实证——早前"lazy 修复"推断已证伪并撤回）。**修复分支
`fix/recovery-env-boot-rebuild`（ab6f681a，未合入）**：init-bottom 每 normal boot 探测+重建，真机
模拟验证全链通过；initramfs 端到端待构建复测。窗口期内盘序锚定退化为扫描序（克隆盘风险注记）。

## B-6 Rescue apply 中断恢复（W4，--no-backup --force 载体）

| 阶段 | 证据 | 判定 |
|---|---|---|
| 载体武装 | bootdev token 恢复后武装成功（`--no-backup`：仅不新建备份，B-2 旧备份同 ID 在库）；此前武装失败根因闭环：零 env → 无 token → `Cannot resolve boot disk for fw_env` | ✓ |
| 中断 | 串口冻结于 `[38.5s] rootfs 解包中段`（清理后重解 ~10s 处）——用户精确断电 | ✓ |
| **一次上电收敛** | 上电 → rescue 路由 → `campaign 8ee2ed5b attempt 1/3` → `cleaning` →**从头重放解包** → `[68.5s] applied …; handing off to normal` → 自动重启 → 解密链 → ssh 复活（全程 <2 min） | ✓✓ |
| **恢复语义** | 走**候选重放**而非备份恢复（备份在场未被使用）——幂等重放设计，arm 期承诺 "power loss is safe at any point" 真机证实 | ✓✓ |
| 终态 | ota_in_progress=0、Status=success、installed=marker=8ee2ed5b 三方一致 | ✓ |

串口存档：`B6-apply-capture.txt`（中断前）、`B6-recovery-capture.txt`（重放全程）。

## B-5 收口

克隆盘已拔（SD 劫持事故后全程无盘）；decryption-disk 的 LUKS 容器取自启动盘——串口各轮
`Found LUKS device: /dev/nvme0n1p4/p5` 恒定 ✓。终态 env/state/payload-id 全收敛 ✓

## Session REC(secure) 结论

**测试项全过**（B-4 的"自愈"预期除外：判定 = 不砖 ✓ / 自愈 ✗，已修分支待合入）。B-1（resize/SSKR/
payload-id 基线）+ B-2 三段（武装/SD 劫持事故反转/正式 apply 全链）+ B-3 四例 + B-6（中断重放）。
**三项发现**：①SD 劫持机制（bootdev_init 卡在位即胜 + env 跟随引导盘，recovery session 克隆
SD 必须拔——已入计划）；②recovery 双坏 env 死锁（**已修**：fix/recovery-env-boot-rebuild ab6f681a）；
③中断恢复走重放不依赖备份（设计确认项）。

## 261124-034600 批总结（两 Session）

- **Session AB(secure)**：全绿 + 86da 闸缺口 + mmc/nvme TEE 差异两项发现
- **Session REC(secure)**：全绿 + SD 劫持 + 双坏 env 死锁 + 重放语义三项发现
- 通用补充覆盖（计划新增章节实测）：G1–G7 闸门、C1–C7 命令面、F1 失败回落、A-6b 三盘、
  每步 UART 双源、loader 软件化换写
- 五项发现处置状态：**86da 闸（FIT 自带 TEE）= 设计级待方案**（keybox_ready 无法区分 SPI 链 TEE
  与 FIT 载入 TEE——同二进制同密钥，需换信任锚如 OTP/RPMB 绑定，非小修可解）；**mmc/nvme TEE
  差异 = rkbin 闭源层待查**；**SD 引导优先 = 既定行为入计划策略**；**recovery 双坏 env 死锁 = 已修
  （fix/recovery-env-boot-rebuild）**；**恢复重放不依赖备份 = 设计确认项**
