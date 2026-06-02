---
title: Ubuntu 24.04 系统下黑屏的急救与彻底修复
date: 2026-06-01 22:50:00
categories:
  - Linux
  - Ubuntu
tags:
  - Ubuntu
  - NVIDIA
  - AMD
  - HWE
  - GRUB
  - GDM
---
# Linux 相关: Ubuntu 24.04 系统下黑屏的急救与彻底修复

> **本篇博客的背景**：Ubuntu 24.04 LTS 系统，使用 AMD Ryzen + NVIDIA Optimus 双显卡笔记本（联想拯救者 R7000P 2021H），升级了英伟达驱动后重启黑屏，左上角有闪烁光标。

## 环境清单

| 项目 | 值 |
|------|-----|
| OS | Ubuntu 24.04.3 LTS (Noble) |
| 内核 | 6.17.0-35-generic（HWE） |
| GPU | NVIDIA GeForce RTX 3060 Laptop GPU + AMD Renoir 集显 |
| 显示管理器 | gdm3 |
| 显示服务器 | Xorg |
| NVIDIA 驱动 | 580.159.03（server LTS 分支） |
| Secure Boot | 开启 |

---

## 1. 确认本篇博客是否有可能解决问题

在尝试任何修复之前，请读者先确认自己电脑的症状是否匹配以下情况：

- 重启后黑屏，左上角有闪烁光标（也有可能是完全的黑屏，连光标都没有）
- 系统最近自动更新过内核（比如升级英伟达驱动导致内核被捎带着一起升级了），在 grub 界面中，输入 `ls /boot/vmlinuz-*` 能看到多个版本
- 能通过 CTRL+ALT+F2（或者 F3~F6）进入 TTY 界面并且登录账号

如果 TTY 都进不去，那这篇文章有可能无法解决你的电脑故障。

---

## 2. 急救方法：先从 GRUB 命令行启动到旧内核

这是首要任务，首先要能成功启动图形化桌面，否则对于不太熟悉命令行界面的读者来说，大部分操作都不方便进行。

### 2.1 进入 GRUB 命令行

重启电脑，在启动时反复按 `Esc` 或按住 `Shift` 进入 GRUB 菜单。如果只看到命令行界面（`grub>`），直接跳到下一步。

如果看到菜单但找不到 "Advanced options for Ubuntu"：
- 有些 GRUB 配置隐藏了菜单，按 `Esc` 或 `e` 进入编辑模式
- 如果完全卡死，强制关机再开机，反复按 `Esc`

### 2.2 探测你的分区位置

> **提醒：不要照抄以下命令，要看自己电脑的输出内容**：

在 `grub>` 提示符下，逐条输入以下命令。

```
ls
```

你会看到类似 `(hd0) (hd0,gpt1) (hd0,gpt2)` 的输出。从第一个分区开始排查：

```
ls (hd0,gpt2)/
```

**找根分区**：如果看到 `boot/ etc/ home/ usr/ var/` 这些目录，这就是你的根分区。如果只看到 `EFI/`，那是 ESP 分区，继续排查其他分区，直到找出来哪个是根分区。

```
ls (hd0,gpt2)/boot/
```

确认里面有 `vmlinuz-<版本号>-generic` 和 `initrd.img-<版本号>-generic` 文件，比如我的是 `vmlinuz-6.17.0-29-generic` 等文件。

从**版本号第二大**的内核开始尝试，记住版本号。这么做是因为，通常版本号最大的那个内核就是当前会导致黑屏故障的那个。

```
cat (hd0,gpt2)/etc/fstab
```

看根挂载点用的标识符，通常是 `UUID=xxxx-xxxx` 或 `/dev/nvme0n1p2` 这种设备名，记住它。

### 2.3 启动旧内核

假设根分区是 `(hd0,gpt2)`，fstab 里根设备是 `/dev/nvme0n1p2`（一般联想拯救者就是这个设备名了），要启动的内核是 `6.17.0-29-generic`：

```
set root=(hd0,gpt2)
linux /boot/vmlinuz-6.17.0-29-generic root=/dev/nvme0n1p2 ro quiet splash
initrd /boot/initrd.img-6.17.0-29-generic
boot
```

> **注意**：`/dev/nvme0n1p2` 可能在 GRUB 阶段不存在（那是 Linux 内核启动后才创建的设备节点），但内核启动后能识别。如果你在 GRUB 下 `ls /dev/` 只找到了一个 `/dev/null`，这是正常情况。

如果 `root=/dev/nvme0n1p2` 不工作，换成 fstab 里看到的 UUID 写法：

```
linux /boot/vmlinuz-6.17.0-29-generic root=UUID=<你的UUID> ro quiet splash
```

### 2.4 进入桌面后第一件事

如果能成功进入图形化桌面了，那请立即打开终端，输入以下内容：

```bash
# 让 GRUB 菜单以后都显示出来（避免下次又要手敲）
sudo sed -i 's/GRUB_TIMEOUT_STYLE=hidden/GRUB_TIMEOUT_STYLE=menu/' /etc/default/grub
sudo sed -i 's/GRUB_TIMEOUT=0/GRUB_TIMEOUT=5/' /etc/default/grub
sudo update-grub
```

---

## 3. 永久修复（推荐方案，约 5 分钟）

### 3.1 黑屏的根因

以下结论来自我自己的电脑，请读者按自己的电脑详细情况调整命令：本机最新的内核版本为 `6.17.0-35-generic`。但是这个 6.17.0-35 HWE 内核安装时，**`linux-modules-extra-6.17.0-35-generic` 包没有自动安装。** 这个包里面有 `amdgpu` 等非核心内核模块。

对于 R7000P 这个 AMD Ryzen + NVIDIA 双显卡笔记本，显示屏是接在 AMD 集显上的，没有 amdgpu 模块等于屏幕没驱动，屏幕没驱动等于 Xorg 启动失败，然后就是 GDM 反复重启，电脑成功黑屏。

**所以其实也不是 NVIDIA 驱动的问题，是 AMD 集显模块没装上。**

### 3.2 修复步骤

1. 安装 35 内核缺失的 extras 包和 headers
```bash
sudo apt install linux-modules-extra-6.17.0-35-generic linux-headers-6.17.0-35-generic
```

2. 保证固件最新
```bash
sudo apt install --reinstall linux-firmware
```

3. 验证 amdgpu 模块已安装
```bash
ls /lib/modules/6.17.0-35-generic/kernel/drivers/gpu/drm/amd/amdgpu/amdgpu.ko*
```
应该能看到 amdgpu.ko.zst 或者 amdgpu.ko

4. 重建 `6.17.0-35-generic` 内核的 initramfs
```bash
sudo update-initramfs -u -k 6.17.0-35-generic
```
5. 重启
```bash
sudo reboot
```

### 3.3 验证修复

在 3.2 步重启后，如果桌面能正常显示，那么可以开始确认以下内容：

```bash
uname -r                    # 对于我自己的电脑，应输出 6.17.0-35-generic
ls /dev/dri/                # 对于我自己的电脑，输出为: card1 card2 renderD128 renderD129
lsmod | grep amdgpu         # 应有 amdgpu 模块
nvidia-smi                  # 应正常输出 GPU 信息
```

如果输出符合预期，修复就算完成了。

### 3.4 防止黑屏可做的进一步操作

> **提醒**：以下操作不一定真的管用，如果你的电脑能跑，那就没必要再费心了

每次 linux 内核升级都有可能缺 extras 包。为了防止旧病复发，可以尝试以下操作：

```bash
# 方案 A：锁定 HWE meta 包（阻止自动拉新内核）
sudo apt-mark hold linux-generic-hwe-24.04 linux-headers-generic-hwe-24.04 linux-image-generic-hwe-24.04

# 方案 B：让 GRUB 默认启动用名字锁定（菜单顺序变了也不会指错）
sudo sed -i 's|^GRUB_DEFAULT=.*|GRUB_DEFAULT="Advanced options for Ubuntu>Ubuntu, with Linux 6.17.0-35-generic"|' /etc/default/grub
sudo update-grub

# 方案 C：关闭无人值守自动升级，可选，不推荐长期关闭，除非你会手动更新
sudo sed -i 's|^APT::Periodic::Unattended-Upgrade.*|APT::Periodic::Unattended-Upgrade "0";|' /etc/apt/apt.conf.d/20auto-upgrades
```

> **注意**：你可能之前在网上搜索教程，并运行过 `apt-mark hold linux-image-generic linux-headers-generic`，但是这条命令 **对HWE 内核无效**。6.17 是 HWE 内核，由 `linux-generic-hwe-24.04` 这个 meta 包管理，不是 `linux-image-generic`。先 `sudo apt-mark unhold linux-image-generic linux-headers-generic` 解掉，再 hold 上面正确的包。

### 3.5 以后手动升级内核的正确方法

当你做完了 3.4 的方案 A，那么以后升级到新内核的时候，就可以按下面的操作来：

```bash
# 先解 hold
sudo apt-mark unhold linux-generic-hwe-24.04 linux-headers-generic-hwe-24.04 linux-image-generic-hwe-24.04

# 升级
sudo apt update && sudo apt upgrade

# 立刻装 extras（关键！）
sudo apt install linux-modules-extra-$(uname -r) linux-headers-$(uname -r)

# 验证 amdgpu 存在
ls /lib/modules/$(uname -r)/kernel/drivers/gpu/drm/amd/amdgpu/amdgpu.ko*

# 重新锁住
sudo apt-mark hold linux-generic-hwe-24.04 linux-headers-generic-hwe-24.04 linux-image-generic-hwe-24.04
```

---

## 4. 完整排查复盘（可选阅读）

以下是我自己记录的完整排查过程。如果读者对"为什么"感兴趣，或者电脑状况跟上面的不完全匹配，可以对照排查一下。

### 4.1 第一层：GDM 服务状态

```bash
systemctl status gdm3
# 显示 active(running)，但有报错：
# Gdm: on_display_removed: assertion 'GDM_IS_REMOTE_DISPLAY (display)' failed

loginctl
# 只列出 tty session，没有图形化 session
# 这导致 GDM 虽然是 running 状态，但是没能成功拉起图形界面
```

同时发现 `/etc/gdm3/custom.conf` 中 `AutomaticLogin = user1` 指向不存在的用户，这会导致 GDM 启动时卡住。修复：

```bash
sudo sed -i 's/AutomaticLogin = user1/AutomaticLogin = <我的用户名>/' /etc/gdm3/custom.conf
```

### 4.2 第二层：DKMS 与 NVIDIA 驱动

```bash
dkms status
# nvidia-srv/580.159.03, 6.17.0-35-generic: installed (WARNING! Diff between built and installed module!)
# DKMS 编译的模块与已安装模块不一致
```

`dkms autoinstall` 报错：35 内核缺少 headers。从这里就能看出 35 内核的安装是不完整的。

> 以下操作是我排查过程中走的弯路，**仅作记录，请勿执行**。
>
> - 进图形化桌面后就执行 `sudo ubuntu-drivers autoinstall`——这导致了安装 `nvidia-driver-595-open` 驱动，并且我们又和没安装完整的 35 内核见面了。595-open 也与当前系统不兼容，导致黑屏更严重，状况直接从左上角闪光标变成了完全的黑屏
> - `sudo apt-mark hold linux-image-generic linux-headers-generic`——对 HWE 内核无效，锁错了 meta 包
> - `GRUB_DEFAULT="5>4"`——grub 启动的数字索引写错了一次，然后启动的时候找不到对应内核，就又退回了 35 内核了（我们又见面啦）

### 4.3 第三层：NVIDIA 用户态库版本污染

以下文件名是我自己电脑上的，随着时间推移导致内容过期，读者可能要自行搜索适配的驱动版本：
```bash
dpkg -S $(which nvidia-persistenced)
# nvidia-compute-utils-580-server

nvidia-persistenced --version
# version 590.48.01
# 包管理器认为是 580，实际磁盘上是 590 残留
```

我之前执行过一次 `apt purge`，但是东西没清干净。磁盘上残留了 44 个 590 版本的 `.so` 文件和 6 个 590 版本的二进制文件，导致内核模块（580）与用户态工具（590）API 不匹配。

修复：

```bash
# 删除所有 590 残留
find /usr/lib /usr/lib32 -name '*nvidia*590*' -delete
sudo rm /usr/bin/nvidia-persistenced /usr/bin/nvidia-powerd \
        /usr/bin/nvidia-pcc /usr/bin/nvidia-ngx-updater \
        /usr/bin/nvidia-cuda-mps-control /usr/bin/nvidia-cuda-mps-server

# 强制重装 580 包
sudo apt install --reinstall nvidia-compute-utils-580-server nvidia-utils-580-server \
                             libnvidia-compute-580-server libnvidia-decode-580-server \
                             libnvidia-encode-580-server libnvidia-fbc1-580-server \
                             libnvidia-gl-580-server libnvidia-cfg1-580-server
```

### 4.4 第四层：modprobe 配置错误

这里是最稀奇的地方，不知道有没有读者和我有相似的问题：`/lib/modprobe.d/nvidia-kms.conf` 有两行被错误拼接：

```
options nvidia-drm modeset=1options nvidia "NVreg_DynamicPowerManagement=0x02"
```

`modeset=1options` 不是有效参数。修复：

```bash
sudo tee /lib/modprobe.d/nvidia-kms.conf <<'EOF'
options nvidia-drm modeset=1
EOF
sudo rm /lib/modprobe.d/nvidia-runtimepm.conf  # 激进 runtime PM 导致 GPU 冷启动问题
```

### 4.5 第五层：Secure Boot 误判

```bash
mokutil --sb-state
# SecureBoot enabled

modinfo nvidia.ko | grep signer
# signer: Canonical Ltd. Kernel Module Signing
# 签名链完整有效，不是 Secure Boot 问题
```

判断标准：如果日志中出现 `module verification failed: signature and/or required key missing` 才是 Secure Boot 签名问题。看到 `loading out-of-tree module taints kernel` 说明模块加载成功，签名 OK。

### 4.6 第六层：/etc/X11/xorg.conf 遗留

可能是之前某次 `nvidia-xconfig` 生成了 `/etc/X11/xorg.conf` 文件，里面写死了 nvidia 作为唯一显示设备。在双显卡笔记本上，这个东西阻止了 Xorg 使用 PRIME 路径（amdgpu 显示 + nvidia 渲染）。

```bash
sudo mv /etc/X11/xorg.conf /etc/X11/xorg.conf.bak
sudo apt install nvidia-prime
sudo prime-select on-demand
```

### 4.7 第七层（根因）：amdgpu 模块缺失

35 内核的启动日志里面完全没有 amdgpu 加载记录：

```
# 35 内核（失败）
grep 'amdgpu' ——只有 CPU 名称 "AMD Ryzen 7 5800H with Radeon Graphics"
ls /dev/dri/ ——card1 card2（只有 nvidia，没有 amdgpu 的 card0）

# 29 内核（成功）
kernel: [drm] amdgpu kernel modesetting enabled.
kernel: [drm] Detected VRAM RAM=4096M
ls /dev/dri/ → card1 card2 renderD128 renderD129
```

排查到 Xorg 日志，里面的内容也能和上面的记录进行交叉验证：

```
(EE) systemd-logind: failed to take device /dev/dri/card0: No such file or directory
(EE) open /dev/dri/card0: No such file or directory
(EE) No devices detected.
Fatal server error:
```

`/dev/dri/card0` 是 amdgpu 的 DRM 节点，它不存在，那 Xorg 就找不到任何可用显示设备，GDM 也就跑不起来了。

最终修复就是本文第 3 节的内容：安装 `linux-modules-extra-6.17.0-35-generic`。

---

## 5. 防黑屏脚本

以下脚本读者可以复制粘贴到终端运行，检查以下自己的系统是否有类似隐患：

```bash
#!/bin/bash
echo "=== 内核版本 ==="
uname -r

echo ""
echo "=== 当前内核 extras 包状态 ==="
dpkg -l "linux-modules-extra-$(uname -r)" 2>/dev/null | tail -1

echo ""
echo "=== amdgpu 模块是否存在 ==="
ls /lib/modules/$(uname -r)/kernel/drivers/gpu/drm/amd/amdgpu/amdgpu.ko* 2>/dev/null || echo "缺失！需要安装 linux-modules-extra-$(uname -r)"

echo ""
echo "=== NVIDIA 用户态/内核模块版本一致性 ==="
KERNEL_VER=$(modinfo nvidia 2>/dev/null | grep '^version' | awk '{print $2}')
USER_VER=$(nvidia-smi --version 2>/dev/null | grep 'NVIDIA-SMI version' | awk '{print $4}')
echo "内核模块: $KERNEL_VER"
echo "用户态:   $USER_VER"
if [ "$KERNEL_VER" != "$USER_VER" ] && [ -n "$KERNEL_VER" ]; then
    echo "⚠️ 版本不匹配！"
fi

echo ""
echo "=== xorg.conf 遗留检查 ==="
if [ -f /etc/X11/xorg.conf ]; then
    echo "⚠️ /etc/X11/xorg.conf 存在，可能阻止 PRIME 正常工作"
else
    echo "无遗留 xorg.conf"
fi

echo ""
echo "=== PRIME 模式 ==="
prime-select query 2>/dev/null || echo "nvidia-prime 未安装"

echo ""
echo "=== NVIDIA 孤儿库残留检查 ==="
ORPHANS=$(find /usr/lib -name '*nvidia*' ! -path '*/modules/*' -exec sh -c 'dpkg -S "$1" >/dev/null 2>&1 || echo "$1"' _ {} \; 2>/dev/null | head -5)
if [ -n "$ORPHANS" ]; then
    echo "⚠️ 发现孤儿文件（不属于任何已安装包）："
    echo "$ORPHANS"
else
    echo "无孤儿文件"
fi

echo ""
echo "=== GRUB 默认启动项 ==="
grep GRUB_DEFAULT /etc/default/grub

echo ""
echo "=== 可用内核列表 ==="
ls /boot/vmlinuz-* | sed 's/.*vmlinuz-/  /'
```

---

## 6. FAQ

### Q: 需要关 Secure Boot 吗？
**不需要。** 本文场景中 Secure Boot 不是根因。Canonical 签名的 NVIDIA 模块在 Secure Boot 开启时正常工作。只有当日志中出现 `module verification failed` 时才是 Secure Boot 问题。

### Q: 能不能直接退回旧版本内核，长期不升级？

可以，但不推荐把这个当成最终的解决方法。

如果只是想先把桌面抢救回来，回退旧内核完全合理。本文第 2 节的 GRUB 手动启动目的就是这个。不过换用旧内核会带来几个长期隐患：

1. 安全更新可能跟不上
2. 新硬件、驱动、固件适配可能出问题
3. 以后安装 NVIDIA、VirtualBox、Docker、v4l2loopback 等依赖内核模块的东西时，仍然可能遇到 DKMS、headers、modules-extra 不匹配的问题
4. Ubuntu 的 HWE 内核本来就是给较新硬件准备的，长期停在旧内核可能只是把问题往后推，然后在未来炸飞自己

如果读者明确自己要这样做，可以按下面的方式把“能启动的旧内核”固定为默认启动项：

```bash
# 例：固定启动 6.17.0-29-generic，按读者电脑上的实际版本修改
sudo sed -i 's|^GRUB_DEFAULT=.*|GRUB_DEFAULT="Advanced options for Ubuntu>Ubuntu, with Linux 6.17.0-29-generic"|' /etc/default/grub
sudo update-grub
```

然后把它标记为手动安装，避免被 `autoremove` 删除：

```bash
sudo apt-mark manual \
  linux-image-6.17.0-29-generic \
  linux-headers-6.17.0-29-generic \
  linux-modules-6.17.0-29-generic \
  linux-modules-extra-6.17.0-29-generic
```

如果还想阻止 HWE 自动拉新内核：

```bash
sudo apt-mark hold linux-generic-hwe-24.04 linux-headers-generic-hwe-24.04 linux-image-generic-hwe-24.04
```

以后想恢复自动升级：

```bash
sudo apt-mark unhold linux-generic-hwe-24.04 linux-headers-generic-hwe-24.04 linux-image-generic-hwe-24.04
```

旧内核仅适合急救和备用，不适合作为长期方案。我认为最好还是保留一个能启动的旧内核，然后把新内核缺失的 `linux-modules-extra`、headers 和 NVIDIA 模块补齐。

### Q: Wayland 还是 Xorg？
本文场景使用 Xorg。如果你用 Wayland，`/etc/gdm3/custom.conf` 中 `WaylandEnable=false` 那行改成 `true` 即可。但 NVIDIA + Wayland 在 24.04 上仍有一些已知问题，建议先用 Xorg 稳定后再试。

### Q: 双系统 Windows 有影响吗？
没有。GRUB 和内核选择与 Windows 无关。

### Q: `grub-reboot` 是什么？
`sudo grub-reboot "菜单项名称"` 让 GRUB **仅下一次启动**使用指定内核。如果启动失败，第三次启动会自动回到默认内核。这是测试新内核的一种低风险方法。不过对于本文的情况，grub-reboot 不好用，因为电脑没启动失败，只是 gdm 没拉起图形化桌面。这样的话不论重启多少次都会从指定内核开机，只能通过 grub 再退回默认内核。

---

## 7. 排查过程中使用的一些命令

| 场景 | 命令 |
|------|------|
| GRUB 探测分区 | `ls` / `ls (hd0,gpt2)/` |
| GRUB 手动启动 | `linux /boot/vmlinuz-... root=...` + `initrd ...` + `boot` |
| 看内核列表 | `ls /boot/vmlinuz-*` |
| 看当前内核 | `uname -r` |
| 看 NVIDIA 状态 | `nvidia-smi` |
| 看 DKMS 状态 | `dkms status` |
| 看 DRM 设备 | `ls /dev/dri/` |
| 看已加载模块 | `lsmod \| grep -E 'nvidia\|amdgpu'` |
| 临时指定下次启动内核（本文场景慎用） | `sudo grub-reboot "Advanced options for Ubuntu>Ubuntu, with Linux X.X.X-XX-generic"` |
| 锁定 HWE 内核 | `sudo apt-mark hold linux-generic-hwe-24.04` |
| 查看锁定列表 | `apt-mark showhold` |
| 安装缺失 extras | `sudo apt install linux-modules-extra-$(uname -r)` |
| TTY 联网 (WiFi) | `nmcli device wifi connect "SSID" password "密码"` |
| TTY 联网 (有线) | `sudo dhclient <网卡名>` |
| 看启动日志 | `sudo journalctl -b -1 -p err --no-pager` |
| 看内核日志 | `sudo journalctl -b -1 -k --no-pager` |
