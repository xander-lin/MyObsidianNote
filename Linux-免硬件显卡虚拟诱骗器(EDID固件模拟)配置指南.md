# Linux 免硬件显卡虚拟诱骗器（EDID 固件模拟）配置指南

在无头服务器（Headless）、远程串流（如 Sunshine/Moonlight）、远程桌面或虚拟化直通场景中，通过内核 DRM 自定义 EDID 与强制输出，可免插实体“显卡欺骗插头/诱骗器”实现虚拟显示输出。

---

## 一、配置原理

1. **`drm.edid_firmware=<Port>:<Path>`**：指定端口（如 `DP-1`、`HDMI-A-1`）加载自定义的 EDID 二进制数据。
2. **`video=<Port>:e`**：强制将该显示接口状态置为 **启用（enabled）**，忽略物理连接检测。
3. **Initramfs 打包**：显卡驱动在系统启动极早期加载，需将 EDID 二进制固件打包进 Initramfs，否则内核在挂载根文件系统前无法加载固件。

---

## 二、复现与配置步骤

### 1. 确认显卡接口名称

查看显卡当前可用的输出接口：
```bash
ls -d /sys/class/drm/card*-*
# 常见名称：card0-DP-1, card0-HDMI-A-1 等，取端口名（如 DP-1）
```

---

### 2. 生成/放置 EDID 固件

创建固件目录并写入预设的 4K@60Hz (HDR / HDP-V104) 固件：

```bash
sudo mkdir -p /usr/lib/firmware/edid19

# 使用 Hex 数据直接写入 EDID 二进制文件
echo "00ffffffffffff000843040199999999011c0103804f00783eee91a3544c99260f5054bfef80d1c0d1e8d1fc950090408180814081c040d000a0f0703e803020350058c31000001a000000fc004844502d563130340a20202020000000ff0064656d6f7365742d310a203020000000fd0018900fde3c000a2020202020200158020356425e04051013141f2021222748494a4b4c5d5e5f606162636465666768696a6be200d5e305c00023097f0783010000e50f00000c006e030c00100038782000800102030467d85dc401788801e606050169694f023a801871382d40582c250058c31000001e011d8018711c1620582c250058c31000009e000000000016" | xxd -r -p | sudo tee /usr/lib/firmware/edid19/edid > /dev/null

sudo chmod 644 /usr/lib/firmware/edid19/edid
```

> **注**：也可使用现有显示器的 EDID（通过 `cat /sys/class/drm/card0-<Port>/edid > /usr/lib/firmware/edid19/edid` 提取）。

---

### 3. 将固件打包进 Initramfs

#### 针对 Arch Linux / 基于 mkinitcpio 的发行版：
编辑 `/etc/mkinitcpio.conf`：
```bash
FILES=("/usr/lib/firmware/edid19/edid")
```
重新生成 Initramfs：
```bash
sudo mkinitcpio -P
```

#### 针对 Ubuntu / Debian / 基于 initramfs-tools 的发行版：
编辑 `/etc/initramfs-tools/hooks/edid`，写入：
```bash
#!/bin/sh
PREREQ=""
prereqs() { echo "$PREREQ"; }
case $1 in prereqs) prereqs; exit 0;; esac
. /usr/share/initramfs-tools/hook-functions
copy_file firmware /usr/lib/firmware/edid19/edid /usr/lib/firmware/edid19/edid
```
赋予执行权限并更新：
```bash
sudo chmod +x /etc/initramfs-tools/hooks/edid
sudo update-initramfs -u
```

---

### 4. 配置内核启动参数

编辑 `/etc/default/grub`，在 `GRUB_CMDLINE_LINUX_DEFAULT` 中添加参数：

```text
drm.edid_firmware=DP-1:edid19/edid video=DP-1:e
```
*(如使用 HDMI-1，则改为 `HDMI-A-1:edid19/edid video=HDMI-A-1:e`)*

更新 GRUB 配置：
```bash
# Arch Linux / Fedora
sudo grub-mkconfig -o /boot/grub/grub.cfg

# Ubuntu / Debian
sudo update-grub
```

---

### 5. 重启并验证

重启后，检查是否生效：

```bash
# 1. 查看内核启动参数
cat /proc/cmdline

# 2. 检查端口连接状态（应显示 connected）
cat /sys/class/drm/card0-DP-1/status

# 3. 查看当前识别出的显示器模式/分辨率
cat /sys/class/drm/card0-DP-1/modes
```
