# OTA 真机测试计划（批次 261124-040053）

- 依据：`ota-test-methods.md`（main @ 849bdd25 版）§0–§7 **全项覆盖**（见矩阵）
- 设备：reComputer RK3576 DevKit / NVMe 系统盘 / loader 在 SPI-NOR（沿 261121 批 rig），串口在手
- 盘符替换：手册 `mmcblkX` → `/dev/nvme0n1`，偏移/扇区数不变

## 成本预算（总计）

| 项 | 次数 | 构成 |
|---|---|---|
| 全盘烧录 | 2 | A/B-secure、recovery-secure 各一 |
| 仅 loader 烧录 | 1 | §7 换 plain loader（SPI-NOR） |
| ssh 重启 | 7 | §1×2、§2a×1、§3×1、§6×1、§2b×2 |
| 人工断电 | 3 窗口 | W1 烧 A、W2 §7、W3 烧 B（见 🔌） |

## 覆盖矩阵

| 手册项 | 执行位 |
|---|---|
| §1 A/B 首启 init-uboot | A-1–A-3 |
| §2a A/B 双坏自愈 + PR45 增验 | A-4 |
| §2b Recovery 双坏自愈 | B-4 |
| §3 正常事务 | A-7（并入 §4 事务） |
| §3b 并发锁 / 负例 | A-5 负例、A-7 锁；B-3 recovery 复测 |
| §4 陈旧挂载自愈 | A-7（kill 真事务制造残留） |
| §5 克隆盘锚定（A/B secure / recovery / 加密双盘） | A-6–A-8、B-2/B-4 |
| §6 payload-id 翻转顺序 | B-2 |
| §7 loader/OS 失配硬停 | A-9（窗口 W2） |

## 排序理由

- §1 只能占烧录后首启，放最前；负例/锁等零重启项穿插消化
- §2a 先于 §3：自愈刚重建的健康 env，才能可信判定 §3"无 rebuilding 警告"
- §4 用真事务 kill 制造残留（比合成 bind mount 更贴原始事故），重跑的同一事务就是 §3 本体，并与 §3b 锁、§5 克隆盘合并 → 省 1 次重启
- §7 收尾 Session A：halt 后必断电，断电窗口直接滚进 Session B 烧录，零额外恢复

## 预备（host 侧，不占设备）

- 构建两镜像：A/B-secure（crypt）、recovery-secure（crypt），同 commit；各配一 OTA 包且 VERSION 互异（payload-id 才会翻转）
- 坏包 ×1：解包删 boot 成员（boot.itb 或 boot.tar.<fmt>）→ 重打包
- USB 克隆盘：dd 同镜像（两形态可复用一块盘，Session 间在 host 重 dd）
- plain / secure loader 文件（loaders/ 相对路径）；scp 通道、串口会话就绪
- 记录烧录镜像与 OTA 包的 sha256（host 侧，入报告）

## 前置确认（每 Session 首启后第一步；不通过则中止该 Session）

```sh
cat /etc/armbian-image-release
```

逐项核对并记入报告：

- `BOARD` / `RELEASE` / `IMAGE_VARIANT` / `VERSION`：与烧录镜像、OTA 包三方一致
- `SEEED_EXTENSION_REPO` + `SEEED_EXTENSION_COMMIT`：**必须等于被测 commit**（本批 = main 849bdd25 系列）
- `BUILD_REPOSITORY_COMMIT`（带 `-dirty` 需注明）、`IMAGE_UUID`

示例设备当前为 `SEEED_EXTENSION_COMMIT=4245749a...`——非本批被测版本，测试前必须先烧新构建镜像。

---

## Session A：A/B-secure（NVMe 全 LUKS）

### A-0 🔌 人工断电窗口 W1
关机 → NVMe 全盘烧 A/B-secure 镜像 + SPI-NOR 写 secure loader → 上电。

### A-1 首启（前置确认 + §1）

```sh
cat /etc/armbian-image-release     # § 前置确认：逐项核对，含 SEEED_EXTENSION_COMMIT
journalctl -b | grep -iE "SSKR|tee-supplicant"   # 首启 SSKR 迁移 86da→SSKR（仅首启可观察）
systemctl status armbian-ota-init-uboot.service
journalctl -b -u armbian-ota-init-uboot | grep -E "Installed|unreadable"
cat /etc/fw_env.config        # 双条目 3 字段，设备为 /dev/nvme0n1
fw_printenv | head
```

版本核对辅以 `uname -v`：事务后 image-release 可能被 upper 陈旧文件遮蔽显示旧值
（261121 批 E1 实证，属预期非失败），前置确认只在首启做。

### A-2/A-3 重启×2（§1）

```sh
reboot                        # 两次；每次 boot 后看
journalctl -b -u armbian-ota-init-uboot --no-pager | tail -3   # 安静通过、无耗时增加
```

### A-4 双坏自愈（§2a + PR45 增验）

```sh
sync
dd if=/dev/zero of=/dev/nvme0n1 bs=512 seek=8128 count=64 conv=notrunc
dd if=/dev/zero of=/dev/nvme0n1 bs=512 seek=8064 count=64 conv=notrunc
sync && reboot
# 重启后：§0 判据 OK；config 仍双条目（未被误判单副本）；双偏移 od 头合法 flags=01
```

### A-5 负例（§3b 前半，零重启）

```sh
/usr/sbin/armbian-ota start bad.tar.gz
# 拒绝在 extract 之后（断言跑在解压目录上，分钟级非秒级，261121 批事务 ~73s 量级）
# 报错含 "dual-payload assertion failed"（no boot payload）
fw_printenv -n ota_in_progress   # 未被触碰（仍 0/unset）
```

### A-6 插克隆盘（§5）

USB 克隆盘插入；`blkid | grep -c security` 应见双份 PARTLABEL。

### A-7 合并事务（§4 + §3b 锁 + §3 + §5）

```sh
# 1. 牺牲事务：后台起，等 target mapper 挂载出现后杀掉（制造残留）
/usr/sbin/armbian-ota start pkg.tar.gz &     # 记 start 的精确 PID
until findmnt | grep -q armbian-ota-root-; do sleep 5; done
sleep 30 && kill -9 <精确PID>
# 勿 pkill/pgrep -f（261121 批实证会匹配自身命令行、误杀 ssh 会话）
# kill 只杀顶层 bash；孤儿写者留下为预期——正是 Detaching 有界等待要验的场景
findmnt | grep armbi_                         # 记录残留挂载
cryptsetup status armbian-ota-root-<slot>     # 记录残留 mapper

# 2. 真事务：前台重跑
/usr/sbin/armbian-ota start pkg.tar.gz
# 期望：Stale LUKS mapper ... closing → Detaching stale target-root mount（§4）
# 事务期间另一终端并发：/usr/sbin/armbian-ota start pkg2 → Another OTA process is running（§3b 锁）
fw_printenv -n ota_in_progress               # =1，无 WARNING "rebuilding env"（§3）

reboot                                        # 候选 slot 启动（双盘在场，LUKS 仍从启动盘解锁）
/usr/sbin/armbian-ota status && /usr/sbin/armbian-ota mark-success
fw_printenv -n ota_in_progress               # =0
cat /var/lib/armbian-ota/ota-state.env       # 键齐全（单次原子写）
```

### A-8 §5 A/B 收口

journal 确认 security 分区选中 `/dev/nvme0n1p*`（非克隆盘）；拔克隆盘。

### A-9 🔌 人工断电窗口 W2（§7）
关机 → SPI-NOR 写 **plain** loader（盘不动）→ 上电。
串口预期：`[Decryption-disk] Error: loader/OS mismatch: no OP-TEE/BL32 under this loader`
→ `panic: ... this image requires a secure loader (BL32). Boot halted; rootfs stays locked.`
确认：启动停止、无明文 fallback。观察完 → **人工断电**（设备 halt，ssh 不可达）。

---

## Session B：recovery-secure

### B-0 🔌 人工断电窗口 W3（紧接 W2，设备已在断电态）
NVMe 全盘烧 recovery-secure 镜像 + SPI-NOR 写回 secure loader → 上电。

### B-1 首启

前置确认（同 A-1：`cat /etc/armbian-image-release` 逐项核对）。串口观察首启
**明文口令窗口**（security marker 86da → tee-supplicant → SSKR 迁移，仅首启可见）；
健康进入 normal（decryption-disk 正常解锁）；记录 `cat /boot/payload-id` 旧值（§6 基线）；插克隆盘。

### B-2 Recovery OTA（§6 + §5 recovery/加密面）

```sh
/usr/sbin/armbian-ota start pkg.tar.gz
fw_printenv ota_in_progress ota_boot_next    # =1 / =recovery
reboot                                        # 进 Rescue apply（期间 ssh 不可达，观察走串口）
# apply 完成后设备自动 reboot 回 normal（串口 "OTA done, rebooting"，手动重启预算不变）
# 硬断言（前后）：apply 前 payload-id=旧值；回 normal 后 == status 的 CANDIDATE_ID → mark-success
# 串口日志若含 boot 文件交换序，记录 payload-id 行的相对位置（应最晚）
# 克隆盘在场：armbi_boot/rootfs/LUKS 解析均落启动盘（§5）
```

### B-3 负例复测（§3b recovery 同语义）

```sh
/usr/sbin/armbian-ota start bad.tar.gz      # 同款拒绝（extract 后断言，分钟级）
```

### B-4 双坏自愈（§2b）

```sh
dd if=/dev/zero of=/dev/nvme0n1 bs=512 seek=8128 count=64 conv=notrunc
dd if=/dev/zero of=/dev/nvme0n1 bs=512 seek=8064 count=64 conv=notrunc
sync && reboot
# 预期：默认 env 路由进 Rescue → reconciliation 重建 → 回 normal（克隆盘在场仍锚定启动盘）
fw_printenv | grep -E "ota_in_progress|boot_slot"    # 健康、状态一致
reboot                                       # 确认安静 boot 一次
```

### B-5 收口

拔克隆盘；串口/journal 确认 decryption-disk 的 LUKS 容器取自启动盘；终态记录（手册 §8 模板）。

---

## 失败处置

- 合并事务任一观察点失败：先清残留（`umount -R` + `cryptsetup luksClose armbian-ota-root-*`）再单项重跑；各手册项均可独立复测，不依赖整链
- §7 未按预期（走到明文挂载/继续启动）→ 立即断电，保存串口全文

## 本 rig 不覆盖（沿 261121 批残余缺口）

- `p?` optional-p 的 sd/vd 分支：锚定正则前缀由启动盘决定，devkit boot 盘为 NVMe 走不到该分支（克隆 USB 盘是 /dev/sdX 但只充当"重复 LABEL 锚定"干扰源）
- 真跨 release 全量切换：缺 resolute cli crypt 包

- 报告：沿用 261121 批格式，按形态各一份（ab-secure / recovery-secure），**头部必含镜像身份块**：

```
- 设备/介质/loader：reComputer RK3576 DevKit / NVMe / SPI-NOR loader
- 镜像身份（/etc/armbian-image-release）：BOARD / RELEASE / IMAGE_VARIANT / VERSION /
  SEEED_EXTENSION_REPO + SEEED_EXTENSION_COMMIT / BUILD_REPOSITORY_COMMIT / IMAGE_UUID
- 烧录镜像 sha256 / OTA 包 sha256；烧录时间与 boot 计数基线
```
