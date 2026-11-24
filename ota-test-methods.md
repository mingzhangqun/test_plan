# OTA 真机回归清单（fix/ota @ 94806596 后、合并前）

针对 PR #42 追加提交 `94806596`（统一冗余 env + fw_setenv -f 重建）的三项真机验证。
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
```

通过标准：arm 两键写入即成功（无 WARNING "rebuilding env"——坏 CRC 重试不应触发）、
读回一致、事务全程 env 写入零失败、mark-success 后状态与 env 收敛。

## 4. 记录模板

每项记录：板型 / 镜像（sha）/ boot 次数 / journal 关键行 / 通过-失败。
失败时附：`fw_printenv` 输出、`/etc/fw_env.config`、双偏移 od 头 16 字节、
`journalctl -b -u armbian-ota-*`。
