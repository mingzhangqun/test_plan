# recovery-secure OTA 真机测试报告（批 261126-063300）

图例：✅ 通过 / ❌ 失败 / ⚠ 偏差（有解释）/ ⏸ 阻塞

- 通道：ssh rk3576（127.0.0.1:50003 隧道；重刷后清 host key + root/root + enable ssh + 推 key）；串口 office screen `USB0`
- 设备/介质/loader：reComputer RK3576 DevKit / NVMe / SPI-NOR secure loader
- 镜像身份（/etc/armbian-image-release）：recomputer-rk3576-devkit / trixie / cli / 26.11.0-trunk /
  SEEED_EXTENSION_COMMIT=**b048f73f**（被测基线；烧录时 main tip）/ BUILD_REPOSITORY_COMMIT=0bf4c8e-dirty /
  IMAGE_UUID=8e338755-…；镜像 sha256 91035e5c…（261126-023102 secure/recovery trixie-crypt）
- OTA 包 sha256：pkg-old a96505c7…（261124 构建，同 VERSION 异内容→内容寻址翻转）；
  pkg-sameid 2d2f0956…（261126 构建，同 ID）；pkg-ab 9fabe74f…（G6 反向）；
  bad.tar.gz ba650639…（删 boot.itb+boot.sha256 重打包）
- gates/：g1/g2/g4 = pkg-old sed 改 package.env 重打包（BOARD/RELEASE+VARIANT/VARIANT）；
  g5 = plain REC 正包软链；g7 = raw 镜像软链（闸门在签名前生效，篡改即触发，sha 不记）
- 基线：2026-11-26 02:50 首启 boot 0；journal 存续 6 normal boot（rescue boot 不写 journal）；
  `uname -v` 与身份块一致
- 素材注记：异 VERSION 包不可得（同月构建均 26.11.0-trunk），本批以异构建包承载 §6 翻转

## 结果总览

| 步骤 | 结果 | 说明 |
|---|---|---|
| B-1 首启 | ✅ | 身份块/SSKR 迁移/resize/fw_env.config 全过；首启串口窗口错过（journal 补证） |
| B-2 OTA | ✅ | payload-id 翻转、itb 写回精确、health 自动 commit；发现 libyaml 缺陷（bd5211b8 修复） |
| B-2b 复装 | ✅ | b048f73f 侧 boot_next=normal 复位实证 |
| B-3 负例 | ✅ | 坏包断言拒、AB 包模式拒，RC=1，env 不触碰 |
| B-3b flags | ✅ | 重复闸 / --force / --no-backup（语义实测修正计划预期） |
| B-3c 闸门负例 | ✅ | G1–G5/G7 六例 metadata 阶段秒级拒（G6 反向在 B-3） |
| B-3d 命令面轻负例 | ✅ | 非 root start / usage / 未知选项 / mark-success 无事务 |
| B-3e 参数组合/重复执行 | ✅ | 并发锁、abandon、--no-backup 单独、三命令拒 |
| B-3f 复跑（新计划序） | ✅ | 3 循环全断言幂等复现；闸序发现（重复闸先于 dual-payload 断言） |
| B-4 双坏自愈 | ⏸ | libyaml 阻断（重刷 261126-090103 后转新批报告） |
| B-5 收口 | ✅ | 克隆盘面跳过（用户裁定，261124 批已验） |
| B-6 中断恢复 | ⏸ | 重刷 261126-090103 后转新批报告（261126 镜像上未执行断电） |

## 🔴 批次发现：initramfs fw_* 缺 libyaml（已修复 bd5211b8，未推）

- 现象：Rescue boot env 对账硬停——`fw_setenv: libyaml-0.so.2 not found` →
  `env CRC bad`（假阳性）→ 重建失败 → `OTA_RESCUE: env CRC rebuild failed (shell on ttyFIQ0)`
- 根因：261126 构建 libubootenv-tool 新增 libyaml 链接，99-copy-tools 只 stage libubootenv；
  fw_printenv 起不来 → fw_env_crc_bad 空输出判坏。261124 构建（0.3.5-0.1+b2，/usr/sbin 实文件）
  无此链接——261124 批 rescue 正常由此解释
- 影响面：rescue env 对账硬停（panic shell）；normal-boot marker 重建 fail-open → B-4/B-6 阻断
- 修复：copy_exec 独挑（binary+闭包+符号链接）+ 落盘断言；设备实测闭包含 libyaml、
  模拟静默丢断言响亮失败
- workaround（重建前再跑 rescue 用）：panic shell 注入
  `mount /dev/mapper/armbian-root /r && cp /r/usr/lib/aarch64-linux-gnu/libyaml-0.so.2* /usr/lib/aarch64-linux-gnu/`
  → 卸 /r、/mnt/userdata → `sh /scripts/init-premount/99-ota-apply`
- 串口证据：`uart/b2-rescue-fail-full.txt`（故障全程）、`uart/b2-rerun-start.txt` 及三轮循环存档

## B-1 首启

命令：身份块核对、resize 四查、journal 首启解密链、fw_env.config。

| 检查项 | 结果 | 说明 |
|---|---|---|
| 身份块 | ✅ | 逐项一致；status=Mode: unknown/idle（无事务时 state 空，预期路径） |
| SSKR 迁移 | ✅ | journal `marker: 86da → raw passphrase → tee-supplicant → retrieved`；分区头置 `SSKR` |
| userdata resize | ✅ | exit 0；journal partition→LUKS→ext4 三级单 boot；userdata 114.4G 盘尾；upper 108G |
| fw_env.config | ✅ | 双条目 3 字段（0x3f8000/0x3f0000）；fw_printenv RC=0 |
| 首启串口窗口 | ⚠ | 错过（设备 02:51 自启、06:33 接入）；journal 全链补证，非判定缺口 |

## B-2 Recovery OTA（§6 + itb 写回硬断言）

命令：`start pkg-old.tar.zst` → reboot → Rescue apply → 回 normal 核对。
注记：apply 由 b048f73f rescue 引擎执行，仅缺 lib 以 RAM 注入补齐；env 实为健康，走正常对账
路径（CRC-rebuild 分支未走到）。workaround 配方见「批次发现」节。

| 检查项 | 结果 | 说明 |
|---|---|---|
| 空间/容量双闸 | ✅ | avail 110G/need 6.3G；rootfs 2.56G→4.07G；boot 74MB→512M |
| 签名校验链 | ✅ | manifest sig → LUKS stash 解密 → rootfs/boot.itb checksum 全绿 |
| backup 建段 | ✅ | `Last-good backup created: 3dde17a1…`（含 p1 FIT 回读 2560B） |
| 武装 | ✅ | in_progress=1 / boot_next=recovery；state STATUS=prepared、CANDIDATE/BACKUP 齐 |
| Rescue apply | ✅ | campaign attempt 1/3 → clean → extract（20s，NVMe）→ handoff → 自动重启 |
| payload-id 翻转 | ✅ | 8eedf865 → **8ee2ed5b**（== CANDIDATE，§6 内容寻址成立） |
| itb 写回 | ✅ | p1 前 74,257,920B md5 == 包内 boot.itb（写前基线 0c5829aa） |
| health 自动 commit | ✅ | `STATUS=applied → Backup rolled → committed`，全程无人工 |
| /tmp/syspw | ✅ | 不存在（78e7ff29 keybox 清理） |

取证注记：apply 完成行（handing off）被内核输出滚出串口缓冲，journal+state 双源补证；payload-id
行相对位置未逐行核对（代码序+终态 marker 间接覆盖）。凭据存续（261124 遗留项）：四循环 apply
后 ssh 公钥认证均复活。

## B-2b 补充循环（b048f73f 复装）

命令：`start pkg-sameid`（ID 异于装机，正常安装路径）+ 注入循环。

| 检查项 | 结果 | 说明 |
|---|---|---|
| payload-id | ✅ | 回 8eedf865（== INSTALLED/CANDIDATE） |
| itb 写回 | ✅ | p1 前 74,286,592B md5 == 包内 boot.itb（2bfcf71f） |
| boot_next 复位 | ✅ | =normal 落位——b048f73f recovery_commit_success 复位实证（B-2 时 261124 旧 runtime 不复位，非回归） |
| 懒 backup 复用 | ✅ | `Existing backup payload verified structurally`（复用现货不重打包） |
| /tmp/syspw | ✅ | 不存在 |

## B-3 负例复测

命令：`start bad.tar.gz`；`start pkg-ab.tar.xz`。串口取证本步豁免（无 reboot，ssh 输出即主证据）。

| 检查项 | 结果 | 说明 |
|---|---|---|
| AB 包模式拒 | ✅ | `ab OTA backend is not installed (have: recovery)`，RC=1（G6 反向对称） |
| 坏包断言拒 | ✅ | extract 后 `no boot payload (dual-payload assertion failed)`，RC=1 |
| env 不触碰 | ✅ | 两例 ota_in_progress 前后均 0 |

## B-3b recovery 专属 flags

命令：`start pkg-sameid` → `start --force pkg-sameid` → `start --force --no-backup pkg-sameid`。

| 检查项 | 结果 | 说明 |
|---|---|---|
| 重复闸 | ✅ | `already installed (INSTALLED_ID); use --force`，RC=1，env 不触碰 |
| --force 重装 | ✅ | 武装 RC=0 → apply → payload-id 不变（同 ID 语义）、itb/boot_next/syspw 全绿 |
| --no-backup | ✅ | 无 backup 建段/复核段；BACKUP_ID 保留既有 last-good（实测语义，计划已修正） |

## B-3c 闸门负例（G1–G5/G7，两形态通用）

命令：office 制包（sed 改 package.env）→ `start` 逐例。串口取证豁免（无 reboot，秒级拒）。

| 检查项 | 结果 | 说明 |
|---|---|---|
| G1 BOARD | ✅ | `BOARD mismatch: package=recomputer-rk3588-devkit, device=…3576…`，RC=1 |
| G2 RELEASE | ✅ | `RELEASE mismatch: package=resolute …--allow-release-change`，RC=1 |
| G3 放行 release 后再拦 | ✅ | WARN `Cross-release OTA allowed` → `IMAGE_VARIANT mismatch`，RC=1（双改防误入真事务） |
| G4 VARIANT | ✅ | `IMAGE_VARIANT mismatch: package=desktop, device=cli`，RC=1 |
| G5 加密方向 | ✅ | `encryption mismatch: package=no, device=yes`，RC=1 |
| G7 原始镜像 | ✅ | `metadata missing package.env`，RC=1 |
| env 不触碰 | ✅ | 七例（含 G3）ota_in_progress 前后均 0 |

## B-3d 命令面轻负例（C 类）

| 检查项 | 结果 | 说明 |
|---|---|---|
| 非 root start | ✅ | `This command must be run as root`，RC=1（守卫先于包检查） |
| usage（无参/坏命令） | ✅ | usage 文本，RC=1 |
| start 未知选项 | ✅ | `Unknown option: --bogus (supported: …)`，RC=1 |
| mark-success 无事务 | ✅ | `requires an active applied transaction (STATUS=success)`，RC=1 |

注记：`status` 唯一无 root 守卫（CLI 只读设计，mutating 五命令全守卫；非 root 可见
Mode/Status/ID，信息面轻微，261124 批未验此项）。

## B-3e 参数组合与重复执行

命令：三命令拒；`start --no-backup pkg-old`（武装 A）→ 事务中并发 `start` → armed 重复
`start`（同 ID）→ `start --force` 穿闸（abandon A、武装 B）→ 注入循环 apply。

| 检查项 | 结果 | 说明 |
|---|---|---|
| switch-slot（recovery 上） | ✅ | `ab OTA backend is not installed (have: recovery)`，RC=1 |
| rollback（recovery 上） | ✅ | `Rollback is not a user command in recovery mode`，RC=1 |
| reconcile（recovery 上） | ✅ | `reconciles on every boot; nothing to do`，RC=0（信息性成功，设计语义） |
| 并发锁 | ✅ | 事务中第二个 start：`Another OTA process is running (PID: …)`，RC=1，env 未动 |
| --no-backup 单独 | ✅ | start 日志零 backup 行（grep=0）；BACKUP_ID 保留 last-good |
| armed 重复 start（同 ID） | ✅ | 撞 INSTALLED_ID 重复闸（先于 abandon），armed 态未扰动 |
| --force 穿闸 abandon | ✅ | `Removing orphan payload directory: …/8ee2ed5b…` → 新 campaign 重武装 |
| abandon 后 apply | ✅ | 第五循环四项硬断言全绿（payload-id 8eedf865/itb md5/boot_next/syspw） |

注记：mark-success 正向路径结构性走不到（health auto-confirm 与其共用
recovery_commit_success，总先到）；三 flag 并用组合未枚举（低价值）。

## B-3f 复跑（新计划序验证）

命令：按重组后计划序重跑 REC 全流程（B-2 → 同窗负例批 → --force → E1/E2/E3 → E4 收尾），
3 个注入循环（首跑 5 个）。设备起点 = 8eedf865/b048f73f idle，终点同。

| 检查项 | 结果 | 说明 |
|---|---|---|
| 循环 1（B-2 pkg-old） | ✅ | payload-id 8eedf865→8ee2ed5b；p1 md5==71860179；懒 backup 复用；syspw 净；boot_next 停 recovery（261124 旧 runtime 已知偏差） |
| 同窗批（B-3×2 + G 六例 + C 类 + 重复闸） | ✅ | 全 RC=1、G3 WARN+VARIANT 链、env 全程 0 |
| 循环 2（--force pkg-old） | ✅ | payload-id 不变（同 ID 重装语义）；p1 md5==71860179；syspw 净 |
| E1/E2/E3 | ✅ | --no-backup 零 backup 行（grep=0）；并发锁 PID 拒；重复闸且 armed CANDIDATE 不扰动 |
| 循环 3（E4 pkg-sameid --force） | ✅ | abandon 重武装 → apply → 8eedf865、p1 md5==2bfcf71f、**boot_next=normal**（b048f73f 复位再现）、syspw 净 |
| 幂等性 | ✅ | 全部断言与首跑逐项一致 |

注记：

- **E4+B-6 合并执行位**（新计划优化）受 libyaml 阻断——E4 改正常注入 apply；合并路径待重建镜像批验证
- **闸序发现**：坏包负例本轮先撞 INSTALLED_ID 重复闸（bad.tar.gz 由 pkg-old 制成，ID==装机）——重复闸
  先于 extract 后 dual-payload 断言；`start --force bad.tar.gz` 穿闸后断言照常命中。素材序约束：
  坏包须在"包 ID ≠ 装机 ID"窗口跑，或一律带 --force
- 串口证据：`uart/rerun-cycle{1,2,3}.txt`

## B-4 双坏自愈 ⏸

| 检查项 | 结果 | 说明 |
|---|---|---|
| 双坏 env 同 boot 重建 | ⏸ | marker fw_* 受 libyaml 缺陷影响（fail-open，路由集不重建），261126 镜像不可能通过 |

重刷 261126-090103 后在新批报告继续（见 `261126-090103/`；B-4 在新镜像上暴露 busybox ls
列布局缺陷，修复 21a80ff9）。

## B-5 收口

| 检查项 | 结果 | 说明 |
|---|---|---|
| 终态 | ✅ | payload-id 8eedf865、status success、in_progress=0/boot_next=normal、syspw 净 |
| LUKS 取启动盘 | ✅ | journal `security partition resolved: /dev/nvme0n1p3`，全链解锁落启动盘 |
| 克隆盘面 | ⚠ | 本批跳过（用户裁定；§5 干扰面 261124 批已验，记未覆盖） |

## B-6 中断恢复 + armbianEnv 存续 ⏸

重刷 261126-090103 后转新批报告（见 `261126-090103/`）。

| 检查项 | 结果 | 说明 |
|---|---|---|
| 断电后自动重放收敛 | ⏸ | 261126 镜像上未执行断电（libyaml 缺陷使 rescue 落 RESCUE SHELL） |
| --no-backup 载体语义 | ✅ | 已在 B-3b 第四循环覆盖 |

## 批次总结

- B-1 / B-2（含补充循环）/ B-3 全系（b/c/d/e/f，含新计划序复跑）/ B-5 全过；B-4/B-6 被
  libyaml 构建缺陷阻塞——重刷 261126-090103（08834155，含修复）后转新批报告
  `../261126-090103/` 继续（B-4 在新镜像又暴露 busybox ls 列布局缺陷，修复 21a80ff9）
- 附加实证：懒 backup 复用、boot_next=normal 复位（b048f73f 侧）、--no-backup 保留 last-good、
  itb 写回三轮 md5 精确、payload-id 内容寻址跨构建翻转、**新计划序可行性**（3 循环完成 REC 可跑面）
