# OTA 真机回归清单（OTA 系列合入 main @ 849bdd25 后）

基线 PR #42（统一冗余 env + fw_setenv -f 重建），叠加 PR45/46/47 合入后的新增行为。
前置：目标板 ssh 可达（或串口）；root；fw_printenv/fw_setenv 在 PATH。

## 0. 公共参数与工具

- env 几何：主副本 `0x3f8000`、冗余副本 `0x3f0000`，各 `0x8000`（64 扇区）
- 冗余格式头：CRC(4) + flags(1)，flags=1（mkenvimage -r 语义）
- 坏 env 判据（`ab_env_healthy` 语义，勿看退出码）：

```sh
fw_printenv 2>&1 >/dev/null | grep -qiE "bad crc|cannot read environment|failed to read environment" && echo BAD || echo OK
```

- 校验盘上 blob 头（应见 5 字节头，第 5 字节 = 01）：

```sh
dd if=/dev/mmcblkX bs=512 skip=8128 count=1 status=none | od -A d -t x1 -N 8   # 0x3f8000
dd if=/dev/mmcblkX bs=512 skip=8064 count=1 status=none | od -A d -t x1 -N 8   # 0x3f0000
```

## 1. A/B 首启 init-uboot 全链（新路径：3 字段 config + sysfs 父盘解析）

烧写 A/B 镜像，首启后：

```sh
systemctl status armbian-ota-init-uboot.service     # oneshot, exit 0
journalctl -b -u armbian-ota-init-uboot             # 有 "Installed fw_env.config" 行
cat /etc/fw_env.config
# 预期两行、3 字段（无 erase 列），设备为父盘（mmcblkX 非 mmcblkXpN）：
#   /dev/mmcblkX  0x3f8000  0x8000
#   /dev/mmcblkX  0x3f0000  0x8000
fw_printenv | head                                 # 健康列出
```

重启两次，确认：

```sh
journalctl -b -u armbian-ota-init-uboot --no-pager | tail -3
# 每 boot 都启动（无 Condition 跳过）、健康时快速退出、无 "unreadable" 报错
# 且每次 boot 耗时无明显增加（早退 = 一次 fw_printenv）
```

通过标准：config 双条目 3 字段、boot_slot/分区变量读回正确、连续两 boot 服务安静通过、系统正常启动进 slot。

## 2. 双坏自愈演练（A/B 与 recovery 各一次）

### 2a. A/B

```sh
# 制造双坏（在运行的系统上直接写裸偏移）
sync
dd if=/dev/zero of=/dev/mmcblkX bs=512 seek=8128 count=64 conv=notrunc   # 0x3f8000
dd if=/dev/zero of=/dev/mmcblkX bs=512 seek=8064 count=64 conv=notrunc   # 0x3f0000
sync
fw_printenv 2>&1 >/dev/null | grep -i "cannot read"    # 确认 BAD
reboot
```

预期链：U-Boot 读双坏 → 编译默认 env → distro_bootcmd 起旧 slot → init-uboot 服务检出
"persistent env unreadable" → 全量重跑 → `fw_setenv -f` 重建。

重启后验证：

```sh
journalctl -b -u armbian-ota-init-uboot | grep -E "unreadable|Installed"
fw_printenv | grep -E "boot_slot|boot_success"        # slot 正确
# §0 判据应输出 OK；盘上双偏移 blob 头均合法（另一副本可为零/旧）
```

通过标准：一次重启内自愈完成、boot_slot/boot_success 与实际运行 slot 一致、无手工干预。

PR45 增验：双坏盘不得被 env-format probe 误判为单副本格式 —— 自愈后 `/etc/fw_env.config`
仍为双条目（非单条 `0x3f8000`）；两副本 blob 头均合法且 flags=01（即"双副本 flags"项收口）。

### 2b. Recovery

同款双坏 dd → reboot → 设备应进 Rescue（默认 env 路由）→ initramfs `99-ota-apply`
reconciliation 重建。Rescue 内/重启后验证 env 健康、`ota_in_progress` 状态一致、
能正常回到 normal（若处于 OTA 事务中则事务继续收敛）。

## 3. 普通 OTA 事务（partial 写入正常路径）

```sh
# 事务 arm 阶段本质是 partial 写（ota_in_progress/ota_boot_next 两键）
/usr/sbin/armbian-ota start <package.tar.gz>
fw_printenv -n ota_in_progress      # =1（fw_env_set_multi 首试写入+读回）
fw_printenv -n ota_boot_next
# 事务推进：候选启动 → health 确认
/usr/sbin/armbian-ota status
/usr/sbin/armbian-ota mark-success  # 或等 firstboot health 自动确认
fw_printenv -n ota_in_progress      # =0，flags 清除
systemctl status armbian-ota-firstboot.service
cat /var/lib/armbian-ota/ota-state.env   # OTA_MODE/STATUS/TARGET_SLOT 等键齐全
```

通过标准：arm 两键写入即成功（无 WARNING "rebuilding env"——坏 CRC 重试不应触发）、
读回一致、事务全程 env 写入零失败、mark-success 后状态与 env 收敛。

### 3b. 并发锁与负例（PR46/45 合入后）

```sh
# 并发锁：start 在读包元数据前先 acquire_lock（CLI 与 backend 同 PID 重入为 no-op）
/usr/sbin/armbian-ota start <pkg> &   # 先起一个
/usr/sbin/armbian-ota start <pkg2>    # 第二个立即被拒：Another OTA process is running

# 负例：包内缺 boot payload —— 断言已移到解压后目录上跑（不再流式列全包成员）
# 预期秒级失败：KIND=full package has no boot payload (dual-payload assertion failed)
```

通过标准：第二个 start 退出码非 0 且不碰 env/state；缺 boot payload 的包在 extract 后
立即被拒（A/B 与 recovery 同语义，报错行含 "dual-payload assertion failed"）。

## 4. A/B 陈旧目标根挂载自愈（PR46）

```sh
target_root=/dev/mmcblkXpN            # 非活动 slot 的 root 分区
mkdir -p /tmp/stale-root
mount "${target_root}" /tmp/stale-root
mount --bind /tmp /tmp/stale-root/tmp # 模拟被杀 hook 留下的嵌套挂载
/usr/sbin/armbian-ota start <package.tar.gz>
```

预期：journal 见 `Detaching stale target-root mount`，`umount -R` 递归拆干净（含嵌套）
后事务继续，mkfs 正常执行。残留写者 120s 拆不掉才失败（负例可不做）。
事后 `umount -R /tmp/stale-root 2>/dev/null` 清理。

## 5. 克隆盘在场时的磁盘锚定（PR45/46 锚定系 + PR47 optional-p）

第二块盘烧同款镜像（LABEL/PARTLABEL/UUID 全重复）接入后跑一轮 OTA：

- A/B secure：security 分区解析结果属于当前启动盘（journal 中 ab_get_security_part 路径）；
- recovery：armbi_boot / rootfs / LUKS 容器解析均落在启动盘，不落到克隆盘；
- 加密镜像双盘：decryption-disk 的 LUKS 容器来自启动盘（`p?` 可选锚定对 sda1/vda1 同样命中）。

通过标准：全程无错盘写入/错盘解锁，OTA 落盘分区全在启动盘。

## 6. Recovery OTA payload-id 翻转顺序（PR46）

recovery 事务 apply 阶段观察 `/boot/payload-id`：

```sh
watch -n1 cat /boot/payload-id   # apply 期间；所有 boot 文件换完前应保持旧值
/usr/sbin/armbian-ota status     # 事务完成后
cat /boot/payload-id             # == CANDIDATE_ID / INSTALLED_ID
```

通过标准：payload-id 归入 critical 类最后翻转 —— health 比对不会在 boot 文件半新半旧时
提前放行；完成后与 status 的 ID 一致。

## 7. secure 镜像 + 无 BL32 loader 硬停（PR47）

secure 加密镜像在 plain loader（无 OP-TEE/BL32，无 `/dev/tee*`）下上电：

串口预期（依次）：
`[Decryption-disk] Error: loader/OS mismatch: no OP-TEE/BL32 under this loader`
→ `panic: Decryption-disk: this image requires a secure loader (BL32). Boot halted; rootfs stays locked.`
→ 启动停止、rootfs 保持锁定。

通过标准：不得回退挂明文 rootfs、不得走到 overlayroot RW fallback。

## 8. 记录模板

每项记录：板型 / 镜像（sha）/ boot 次数 / journal 关键行 / 通过-失败。
失败时附：`fw_printenv` 输出、`/etc/fw_env.config`、双偏移 od 头 16 字节、
`journalctl -b -u armbian-ota-*`。
