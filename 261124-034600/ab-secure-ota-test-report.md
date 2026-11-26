# A/B Secure OTA 真机测试报告（261124-034600 批次，devkit/NVMe）

- 日期：2026-11-24；设备：reComputer RK3576 DevKit（NVMe，secure loader，**A/B 味，全 LUKS**）
- 被测 commit：main tip 849bdd25（PR43/45/46/47/48 全并入后顶点，首测该树）
- 镜像：26.11.0-trunk trixie cli crypt
- 烧录镜像 sha256：0b0ab3d2e00c1475c3920a9e9b894c5a2103a50533686bcf6ccbaedecc2247e2（crypt_AB_PART.img）
- OTA 包 sha256：AB 7d8989087562fafb5b2f8160c749db189c4b4a7a8942b7325cb28b55c6f3d4e7 / REC a96505c7…（备用）
- loader sha256：e7f244c622a87b624d016d5d6dd4b3bb879a24efcd9a568ea742cae032f185a0

## 形态自检

6 分区（boot_a/b raw-FIT 512M×2、security 4M、rootfs_a/b LUKS 3.8G×2、userdata LUKS）；
`armbian.slot=a`、`ab_boot_mode=raw-fit`、boot_fit 路由；`bl32-v1.08` 在固件链（OP-TEE 在）；
身份块 SEEED_EXTENSION_COMMIT=849bdd25505b… 与被测 commit 一致 ✓

## A-1 首启

| 检查点 | 证据 | 判定 |
|---|---|---|
| init-uboot | `Installed fw_env.config: /dev/nvme0n1 0x3f8000 + 0x3f0000 (redundant) 0x8000` | ✓ |
| fw_env.config | 双条目 3 字段（0x3f8000/0x3f0000 × 0x8000），父盘 nvme0n1 | ✓ |
| SSKR 首启迁移 | journal `[Decryption-disk] No SSKR marker, reading raw passphrase` → `tee-supplicant started`（仅首启可观察，已捕获） | ✓ |
| **userdata resize 四查** | service exit 0/SUCCESS；单 boot 三级链：`resizing userdata partition: p6`（parted -f 顺带修复 GPT 备份头，script(1) "Yes" 应答入 journal）→ `resizing LUKS mapping armbian-userdata to 231874560 sectors` → `resizing ext4`；p6 510M→盘尾（end 19189759s→250052607s）；`/media/root-rw` 104G | ✓✓ |
| 6bfacffe 单 boot LUKS 扩容 | LUKS 行与 ext4 行同 boot 0（04:56:02 / 04:56:11），无二次 reboot | ✓ |
| 双副本 blob 头 | 0x3f8000: `0c0f55f7 01` + `boot_`；0x3f0000: `350792a9 02` + `ab_`——CRC 合法、flags 01/02 不等（递增序号，大者新=冗余副本胜出） | ✓ |

boot 计数基线：boot 0 = 04:55:57 UTC（journal 唯一 boot，烧录后未重启过）。

## A-2/A-3 重启×2

| 检查点 | 证据 | 判定 |
|---|---|---|
| init-uboot 每 boot 运行 | `active (exited) 0/SUCCESS, CPU 77ms`（journal 无条目=健康路径零输出早退，"journal rotated" 提示源于 volatile 小配额 journal，非跳过） | ✓ |
| init-uboot 无 unreadable | 两次重启后 env 读回正常、boot_slot=a 不变 | ✓ |
| resize 安静复跑 ×2 | `resize2fs: The filesystem is already 115937280 (1k) blocks long. Nothing to do!` → Finished exit 0（复跑成本 <1s） | ✓ |
| boot 耗时 | reboot→ssh 可达 ~25-30s（隧道自动复活） | ✓ |

boot 计数：3（烧录首启 + 2 次重启）。

## A-5 负例

| 检查点 | 证据 | 判定 |
|---|---|---|| 断言拒绝（extract 后） | 05:11:46 读元数据 → 容量闸×2 OK → staging/space gate OK → 05:13:23 `[ERROR] KIND=full package has no boot payload (dual-payload assertion failed)`，全程 ~97s（分钟级） | ✓ |
| 退出码 | RC=1 | ✓ |
| env 不被触碰 | ota_in_progress: PRE=0 → POST=0 | ✓ |
| 容量闸锚定 | 目标算的是 /dev/nvme0n1p5/p2（启动盘 slot b），未受 sshfs 路径影响 | ✓ |

bad.tar.gz = 正包删 boot.itb+boot.sha256 重打包（office 侧制，sha256 未记——断言在成员级，包完整性无关判定）。

## A-5b 闸门负例套件（G1–G7，849bdd25 树实测）

制法：office 解包正包 → sed 改 package.env → tar -cf（gate 在签名校验前生效，篡改即触发）。

| # | 输入 | 拒绝点（实测输出） | RC | env | 判定 |
|---|---|---|---|---|---|
| G1 | BOARD 改 rk3588 | `BOARD mismatch: package=recomputer-rk3588-devkit, device=…3576…` | 1 | 0 | ✓ |
| G2 | RELEASE 改 bookworm | `RELEASE mismatch … (cross-release OTA needs --allow-release-change)` | 1 | 0 | ✓ |
| G3 | 同 G2 + `--allow-release-change` | WARN `Cross-release OTA allowed` 放行 → `IMAGE_VARIANT mismatch … CLI and Desktop cannot cross-upgrade`（双改防误入真事务，二次拦截生效） | 1 | 0 | ✓ |
| G4 | 仅 VARIANT 改 desktop | `IMAGE_VARIANT mismatch`（单因） | 1 | 0 | ✓ |
| G5 | plain AB 正包 | `encryption mismatch: package=no, device=yes` | 1 | 0 | ✓ |
| G6 | REC zst 正包 | `recovery OTA backend is not installed on this firmware (have: ab)`（措辞较 261121 演进，语义同） | 1 | 0 | ✓ |
| G7 | 原始 .img.xz | `tar: This does not look like a tar archive` → `metadata missing package.env`（秒级快败） | 1 | 0 | ✓ |

全套 `ota_in_progress` 零触碰，终态 env 干净。未压缩 tampered tar 元数据读 <1s（gate 在流式读
package.env 后即触发，不扫全档）；xz/zst 正包 ~17s。本 rig 仍不覆盖：容量/空间闸**拒绝**分支、
payload magic mismatch、ambiguous 双 boot 成员、zst 协商拒绝、加密反向闸（plain session）。

## A-5c 命令覆盖套件（C 系列，849bdd25 树实测）

| # | 证据 | 判定 |
|---|---|---|
| C1a | `switch-slot b`：boot_slot=b / **boot_success=a 锚保留** / ota_in_progress=1 / retry=3 / state ready_to_boot→reboot→slot b（p5）→健康自动确认 boot_success=b、STATUS=success | ✓✓ |
| C1b | 反向 `switch-slot a`：同链回 a（p4），env/state 收敛 | ✓ |
| C2 守卫 | 事务中 `Cannot switch slots while A/B OTA is in progress`；`Already booting slot a`；坏参 usage；非 root `must be run as root`；usage 文本 | ✓ |
| C3 | `No A/B OTA in progress, nothing to rollback` RC=0 | ✓ |
| C4 真回滚 | armed 后 `rollback` → `Rollback configured, rebooting back to slot a` **自动 reboot** → 回 p4、env 全复位（boot_slot=boot_success=a、retry=3）、state=rollback | ✓✓ |
| C5 reconcile | armed 后清 env 模拟 → `target slot b did not pass boot verification; running confirmed slot a` → state=rollback、TARGET 清空（U-Boot-rollback 记录分支） | ✓ |

UART 双源（`usb0` 存档 `uart/C4-rollbackboot-092735.txt`，1050 行）：rollback 后启动串口侧完整链
`security partition resolved: nvme0n1p3 → SSKR → p4 解锁 → mapper ready → overlayroot configured`，
与 journal/cmdline 三源一致。A-1~A-7 为 journal 单源（规则后补），C 系列起双源。

## A-5d 启动失败自动回落（F1，849bdd25 树实测）

| 检查点 | 证据 | 判定 |
|---|---|---|
| 坏槽硬停 | 串口：`Saving Environment … Writing to redundant nvme(0)... done`（扣减落盘）→ `FIT: No fit blob` → `### ERROR ### Please RESET`（无 CLI、不自动 reset） | ✓ |
| retry 耗尽自动回落 | 终态 env：`boot_slot=a`、`boot_success=a`、`ota_in_progress=0`、`slot_retry_left=3`（复位 max）——只有耗尽分支会翻转 boot_slot 并清 ota_in_progress | ✓✓ |
| 锚槽正常启动 | cmdline `armbian.slot=a` + root p4 + 串口解密全链（`uart/F1-final-*.txt`） | ✓ |
| reconcile 真实记录 | journal：`A/B OTA target slot b did not pass boot verification; running confirmed slot a` → state=rollback（C5 模拟分支的真机闭环） | ✓✓ |
| 修复路径 | 重跑 OTA start：`Writing FIT … boot.itb (73061888 bytes) … to slot b: p2` → reboot → slot b
（p5）健康确认 `boot_success=b`/STATUS=success；串口 `F1-repair-slotb.txt` p5 解锁链 + `AB OTA
marked successful on slot b` | ✓✓ |

串口存档：`uart/F1-attempt1-errorhalt.txt`（含 U-Boot 全段——未进内核故缓冲未滚）、`F1-cycle1.txt`、
`F1-final-*.txt`。人工断电 **4 次**（用户实测口述；扣减 3 次耗尽后回落，含首发 reboot 的尝试序
共 5 次 U-Boot 进入）。

## A-6 克隆盘（§5 干扰源就位）

`/dev/sda` 29.7G，完整 A/B secure 布局，PARTLABEL **与 PARTUUID 均与启动盘逐字节一致**（同镜像 dd，
最大干扰场景）。security×2、rootfs_a/b×4。**克隆盘 userdata 保持 510M 出厂尺寸**——resize 服务锚定
启动盘未被同名 PARTLABEL 干扰（锚定的顺带正向证据）。

## A-7 合并事务（§4 陈旧自愈 + §3b 锁 + §3 正常 + §5 锚定）

**第 1 步 牺牲事务**（PID 7618，目标挂载出现 70s 后 kill -9）：留下 stale mapper
`armbian-ota-root-b`（active and in use）+ stale mount `ab-root-mnt.fX6HSw`（ext4 rw）双残留；
`ota_in_progress=0`（arm 是最后一步，未武装）✓

**第 2 步 真事务**（05:18:08→05:19:54，全程 106s）：

| 阶段 | 证据 | 判定 |
|---|---|---|
| 陈旧锁清理 | `Removing stale lock file` | ✓ |
| manifest 签名链 | `signature verified` → stashed 口令解密 → `SHA256 matches signed manifest` → rootfs/boot.itb checksum 各自过 | ✓✓ |
| **Stale mapper 自愈** | `Stale LUKS mapper armbian-ota-root-b found (interrupted previous run?), closing it before unlocking` → `Unmounted target-slot mount ...fX6HSw` | ✓✓（PR47/46） |
| **§5 锚定（克隆盘在场）** | 容量闸 p5/p2、解锁/写入目标全部 `/dev/nvme0n1p*`；`Security partition marker: SSKR`（SSKR 仅存在于启动盘 p3，克隆 sda3 为出厂 86da——读到 SSKR 即证明锚定 nvme0n1p3） | ✓✓ |
| mkfs/写盘 | mkfs on mapper（LUKS 头零接触）→ rootfs extract 100% → boot.itb 73MB 裸写 p2 @192MB/s | ✓ |
| env 武装 | ota_in_progress=1、boot_slot=b、slot_retry_left=3（ota_boot_next 为空=AB 语义用 boot_slot 翻转，非 recovery 键） | ✓ |
| 并发锁（§3b） | extract 过半时并发 `start bad.tar.gz`：`Another OTA process is running (PID: 8618)` + `Cannot acquire OTA lock`，RC=1 | ✓ |
| 残留清零 | 事务后 findmnt 无 stale、mappers 仅 armbian-root/userdata | ✓ |

**第 3 步 候选 slot 启动确认**：

| 检查点 | 证据 | 判定 |
|---|---|---|
| slot b 引导 | 重启后 cmdline `armbian.slot=b`，隧道 25s 复活 | ✓ |
| 健康确认 | firstboot 健康检查 05:20:42 自动确认（ExecCondition OTA_MODE=ab）；`status`: Mode=ab Status=success slot=b，p2/p5 [BOOTING] | ✓ |
| mark-success | `AB OTA marked successful on slot b`（自动确认后幂等） | ✓ |
| env 收敛 | ota_in_progress=0（清除）、boot_slot=b、boot_success=b、slot_retry_left=3 | ✓ |
| state 文件 | OTA_MODE/STATUS/CURRENT_SLOT/COMPLETE_TIME 键齐全（单次原子写） | ✓ |

## A-4 双坏自愈（§2a + PR45 增验，补跑）

零当时 env 已带 OTA 后状态（boot_slot=b/boot_success=b）——比首启态更严：重建须重新探测槽。

| 检查点 | 证据 | 判定 |
|---|---|---|
| 双坏检测+重建 | `ERROR: persistent env unreadable; re-running full init (fw_setenv -f rebuild)` → 全量 init（layout 探测 nvme0n1p1-p5、boot media nvme 0、raw-fit 模式） | ✓ |
| 一次重启自愈 | 同一 boot 内完成，坏 env 判据 → OK，无手工干预 | ✓ |
| config 未误判单副本 | 仍双条目 3 字段 | ✓ |
| blob 头 | 主 `e12bb1be 02` / 冗余 `e186a3c8 01`——CRC 合法、flags 为递增序号 02/01（计划判据"flags=01"实义为合法非零序号，非固定值） | ✓ |
| slot 一致性 | 重建 `boot_slot=a`/`boot_success=a`；实际 `armbian-root` → **p4 = slot a** ✓ 一致 | ✓ |

语义注记：cmdline 本次无 `armbian.slot=`（U-Boot 回落内置默认 env 引导，非 ab 路由）；
decryption-disk 解 rootfs_a。零化 env 丢失槽记忆（b 的 OTA 成功态被遗忘、回落探测 a）——env 为
槽路由唯一真源的固有行为，p5 上 OTA 内容仍在盘但不再被引用。重建 env 的引导路由链另起一 boot 验证（见下）。

**重建 env 引导验证**（追加一 boot）：cmdline 带 `armbian.slot=a`（重建后的 ab_preboot/ab_select_fit_slot/boot_fit
路由链完整可用）、init-uboot 安静零输出、env/运行槽一致 → ✓

boot 计数：8（烧录首启 + A-2/A-3 ×2 + A-7 slot b + A-4 自愈 + A-4 路由验证 ×2 轮内含）。

## A-8 AB 收口

分区解析锚定证据（A-7 事务日志，/tmp 已随重启清空、全文存本报告）：容量闸 rootfs→p5 / boot→p2、
`Security partition marker: SSKR`（仅启动盘 p3 可读到）、boot.itb 裸写 p2、mkfs on p5 mapper——
全程零 `/dev/sda*` 引用，克隆盘（PARTUUID 级同款）在场未造成任何错盘解析 ✓

拔盘后终态：sda 消失、仅 nvme 单盘、mapper 仅 armbian-root/userdata、env 健康
（boot_slot=a/boot_success=a/ota_in_progress=0）、运行 slot a（cmdline+root p4 双证）✓

## A-9 loader/OS 失配硬停（软件化：dd mtdblock0）

| 检查点 | 证据 | 判定 |
|---|---|---|
| mtd 换写 | plain loader sha256 三级一致（9017a61a…），dd 16M 后读回 md5 全等（7e1007f1…） | ✓ |
| NVMe SSKR + plain loader | `marker: SSKR → SSKR marker found → Error: SSKR marker requires working OP-TEE and keybox_app` → 落 cryptroot 手动口令提示（**非 panic**，panic 仅 86da 分支）；内核 `optee: api uid mismatch`；口令为二进制 keyfile 控制台不可输入 → 事实死锁、rootfs 锁定、无明文 fallback | ✓（语义达标/形态偏差） |
| A-6b 引导设备选择 | SD 在场：`Bootdev(atags): mmc 1`、`armbian.bootdev=mmc`；拔 SD/USB 后 bootdev=nvme。**SD 优先为 RK boot ROM 既定行为（用户确认）**，非缺陷；A-9 类测试须拔 SD/USB 才能落到 NVMe 场景 | ✓（既定行为） |
| A-6b 三盘 PARTLABEL | 18 份（nvme+sd+mmcblk 各 6，LUKS UUID 亦全同）；mmcblk 分支首次真机在场 | ✓ |
| **发现：86da 闸缺口** | 86da 新鲜盘经 raw-FIT 从 SD 引导：FIT 自带 bl32 → /dev/tee0 存在 → keybox_ready 过 → raw passphrase 放行、系统完整启动。**源码层根因确认**：plain 拒 raw FIT 是 env-only 方案（C 闸 0c185134 已被否），盘上 secure env（dd 克隆）覆盖 plain 桩即绕过——属 11-20 已知边界的真机实证；FIT 自带 TEE 穿透 86da 属已知"未签名 OP-TEE 过 TEE 闸"缺口的新场景 | ⚠（已知缺口实证） |
| mmc vs nvme TEE 差异 | 同 plain loader 同源 FIT：SD 引导 /dev/tee0 在（86da 放行），NVMe 引导 api uid mismatch（SSKR 拒绝）——**SSKR 守住依赖 nvme 路径恰好不启 FIT 内 TEE（非显式防御）**；内因在 rkbin 闭源层，U-Boot 树内无 FIT optee 启动调用者 | ⚠ 待排查 |

串口存档：`A9-final.txt`（SD 引导 86da 放行全程）、`A9-nvme-plainloader.txt`（NVMe SSKR 拒绝+手动提示）、
`A9-poll*.txt`。设备现停于 LUKS 手动提示（不可输入），须断电进 W3。

## Session AB(secure) 结论

**除两项发现（86da 闸缺口、mmc/nvme TEE 差异）外全部通过**：A-1~A-8 + G1–G7 + C1–C7 + F1 全绿。
两项发现移交代码层排查：decryption-disk 86da 分支 keybox_ready 的判定面、U-Boot mmc/nvme 路径对
FIT 内 bl32 的处理差异。SD 最高引导优先为既定行为（非发现）。

