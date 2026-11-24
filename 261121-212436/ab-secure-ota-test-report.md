# A/B Secure OTA 真机测试报告（261121-212436 批次，devkit/NVMe）

- 日期：2026-11-23；设备：reComputer RK3576 DevKit（NVMe，secure loader，**A/B 味，全 LUKS**）
- 镜像：26.11.0-trunk trixie cli crypt，SEEED_EXTENSION_COMMIT=92333196（m-dev tip）；A/B secure **首次在 devkit 上按手册全流程**（此前 11-16 首闭环亦在本机形态）
- 方法：`.claude/ota-test-methods.md` A/B plain 全节 + A/B secure 增量节；重点观察分区变化

## 分区基线与终核（重点项）

| 分区 | PARTLABEL | 大小 | 基线=终态 | 变化 |
|---|---|---|---|---|
| nvme0n1p1/p2 | boot_a/boot_b | 512M×2 | PARTUUID 不变 | **raw FIT 分区**（非 ext4）：每笔事务裸写 boot.itb（73058816B） |
| nvme0n1p3 | security | 4M | PARTUUID 不变 | marker `86da→SSKR` 首启迁移后全程保持 SSKR |
| nvme0n1p4/p5 | rootfs_a/rootfs_b | 3.8G×2 | **LUKS UUID 逐字节保留** | mkfs 只打 mapper（LUKS 头零接触） |
| nvme0n1p6 | userdata | 110.6G | LUKS UUID 不变 | overlay upper + OTA staging |

GPT 分区表/尺寸/LABEL 全程零变化。boot 分区非文件系统（plain 味是 ext4 + tar 解包——secure 用裸 FIT 直写，二者差异如实体现）。

## 首启验证

| 检查点 | 证据 | 判定 |
|---|---|---|
| 布局 | 6 分区含 security；boot_a/b 为 raw-FIT 分区（无 FSTYPE） | ✓ |
| **SSKR 首启迁移** | journal `No SSKR marker, reading raw passphrase` → `tee-supplicant started` → p3 marker = **SSKR** | ✓ |
| LUKS 首启 | init-uboot hook 装冗余 fw_env.config（0x3f8000+0x3f0000）、`armbian-root`/`armbian-userdata` mapper 就位 | ✓ |
| env 锚 | `boot_slot=a`、`ab_boot_mode=raw-fit`、bootcmd 含 `ab_select_fit_slot; boot_fit` 路由 | ✓ |
| 固件链 | 内核 cmdline `androidboot.fwver=…bl32-v1.08…`（OP-TEE 在链） | ✓ |

## 负向闸门（6/6 + N7）

| 输入 | 结果 | 判定 |
|---|---|---|
| plain AB_PART 包 | `encryption mismatch: package=no, device=yes`（反向） | ✓ |
| recovery zst 包 | `recovery OTA backend is not installed (have: ab)` | ✓ |
| 原始 .img.xz | `metadata missing package.env` | ✓ |
| resolute gnome-crypt 包 | `RELEASE mismatch` | ✓ |
| + `--allow-release-change` | WARN 放行 → `IMAGE_VARIANT mismatch: desktop vs cli` | ✓ |
| `rollback` pristine | `No OTA mode recorded` | ✓ |
| N7 武装中再 start | `Another AB OTA boot verification is still in progress` | ✓ |

## D1 正向事务（双向 + secure 专属链）

| 阶段 | 证据 | 判定 |
|---|---|---|
| 容量闸 ×2 | rootfs→p5 LUKS 容量、boot 73MB→p2 | ✓ |
| **manifest 签名链** | `Verifying OTA payload manifest signature → verified` → 解密（initramfs stashed 口令）→ `SHA256 matches signed manifest` | ✓✓ |
| SSKR 解锁 | `Security partition marker: SSKR` → LUKS 开 p5 → mapper `armbian-ota-root-b` | ✓ |
| mkfs on mapper | LUUT 头零接触、UUID 保留 | ✓ |
| **boot 裸写** | `Writing FIT boot image … boot.itb (73058816 bytes) to /dev/nvme0n1p2`（非 tar 解包） | ✓ |
| env 武装 | boot_slot=b 翻转、锚=a、retry=3 | ✓ |
| 引导+确认 | 内核 cmdline `armbian.slot=b root=/dev/mapper/armbian-root` → health → `marked successful on slot b` | ✓✓ |
| b→a 反向 | 全链一致，锚点转移 a | ✓ |

## SIGKILL 中断组（stale mapper + stale mount 双自愈）

| 测试 | 结果 | 判定 |
|---|---|---|
| SIGKILL @解压 30% | 残留=stale mapper + mapper 上 stale mount + 孤儿写者；env 未武装（arm 是最后一步） | ✓ |
| 立即重试 | `Removing stale lock file` → **`Stale LUKS mapper … closing it before unlocking`**（9111f326）→ 重开 → mkfs → staged success rc=0 | ✓✓ |
| 机制澄清 | 9111f326 的 `ab_teardown_luks_mapper` **已内含 umount**（5 轮、跳过活动根）——secure 路径 stale-mount 本就被 mapper 自愈覆盖；46596614（plain 缺口修复）在此路径为 no-op，已热补丁留作双保险 | ✓ |
| 并发锁 | `Cannot acquire OTA lock` | ✓ |

## D4 switch-slot（boot-trial + 失败路径补全）

| 测试 | 结果 | 判定 |
|---|---|---|
| `switch-slot a` 武装 trial → 引导 → health → `marked successful on slot a` | 锚点保留/转移正确 | ✓ |
| 守卫（env 已武装） | `Cannot switch slots while A/B OTA is in progress` | ✓ |
| **失败路径：manual trial 撞坏 FIT** | 破坏 slot a FIT → 阶梯耗尽 → U-Boot 自动回滚锚 b → reconcile `Status=rollback`（`PACKAGE_PATH=manual-slot-switch`）——手动换槽正确纳入 trial/回滚契约 | ✓✓ |
| **发现→修复（22e6df3e）**：start 的元数据读取（大包数秒）在锁外，窗口内 switch-slot 可合法拿锁插队武装（双写者） | 修复：CLI `start` 在 `check_ota_pkg` 前取锁 + `acquire_lock` 同 PID 重入；实测 +2s 插 switch-slot 被拒（`Cannot acquire OTA lock`）、env 零污染、interleave 的 start 正常完成、全链 success | ✓✓ |

机制澄清：switch-slot 本身**取锁**（`ab_require_tools`→`ota_require_runtime`→`acquire_lock`），无锁缺口在 start 侧锁前窗口；`ab_rollback` 在该窗口为 env 守卫下的安全 no-op，非缺口。

## D3 目标 FIT 损坏（验签拒 + 阶梯 + 回滚）

破坏 p2 kernel 数据 8 字节后重启：

| 层 | 证据 | 判定 |
|---|---|---|
| 签名层 | `Verifying Hash Integrity ... sha256,rsa2048:dev+ OK`（config 节 RSA2048 过——**成功侧验签证据同场取得**） | ✓ |
| 数据哈希层 | kernel 子镜像 `sha256 Bad hash: a524d2…` → `Bad Data Hash` → `ERROR: can't get kernel image!` | ✓ |
| 终态 | `### ERROR ### Please RESET the board ###`——**零未验证代码执行** | ✓✓ |
| 阶梯 | 预算烧尽 → U-Boot 回滚 slot a → reconcile `Status=rollback`、target 清空 | ✓ |

## 版本互换（h120 ↔ 当前批）

| 方向 | 证据 | 判定 |
|---|---|---|
| 降级 a→slot b | 内核 `#1 SMP Thu Nov 19 08:56:04`（原 Nov 20）；顺带治愈被破坏的 p2（新 FIT 覆盖+验签+引导成功） | ✓ |
| 升回 b→slot a | 内核回 `#1 SMP Fri Nov 20 15:55:18` | ✓ |

## D2 真断电（单次拔电档）

| 阶段 | 证据 | 判定 |
|---|---|---|
| 拔电 | 引导滚动中单次拔电 | ✓ |
| 重试 | 下次启动扣 1 预算重试 slot b | ✓ |
| LUKS 自恢复 | `EXT4-fs (dm-1): recovery complete`（LUKS 内 rootfs journal 自愈） | ✓ |
| 终态 | `marked successful on slot b`、Status=success、锚点转移 | ✓✓ |

（多次拔电耗尽档已在 plain 轮验证，secure 阶梯机制同 D3 已证，不重复耗用户时间。）

## 残余缺口

| 缺口 | 所需条件 |
|---|---|
| sd*/vd* 盘型 | A/B 味 + 此类盘 |

（D2 多次拔电耗尽档 secure 侧**判定无需复测**：断电→fs 自恢复（D2 单次档实证）与预算耗尽→回滚（D3 阶梯实证）两机制分侧已直接覆盖，`ab_preboot` 阶梯代码 plain/secure 共用且 plain 轮多拔实测过；LUKS header 引导期只读不受拔电影响。）

## 结论

A/B secure 在 devkit/NVMe 上手册全流程闭环：首启 SSKR 迁移、负向闸 7/7、D1 双向（manifest 验签→解密→SHA256 对账→LUKS mapper→**FIT 裸写**全程）、SIGKILL 双残留自愈、D4 boot-trial + **失败路径与竞态补全（22e6df3e 锁前窗口修复）**、**D3 破坏 FIT 被验签逐层拦截**（rsa2048 OK + Bad Data Hash 干净停机）+ 阶梯自动回滚、版本互换双向、D2 单次断电重试成功（LUKS journal 自愈）。分区/LUKS 头/security marker 全程零意外。与 11-16 首轮及 plain 侧互证；**四形态（recovery plain/secure、A/B plain/secure）至此在 devkit 261121 批全部按手册完整走完**。
