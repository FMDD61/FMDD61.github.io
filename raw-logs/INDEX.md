# 原始日志索引

本目录保存了本次会话中所有关键诊断日志，按排查阶段组织。

## 目录结构

### 01-initial-diagnosis — 初步诊断
| 文件 | 内容 |
|------|------|
| `01-gdm-status.txt` | GDM 服务状态、custom.conf、loginctl 输出 |
| `02-dkms-and-kernels.txt` | DKMS 状态（含 WARNING）、autoinstall 报错、可用内核列表 |

### 02-grub-rescue — GRUB 救援
| 文件 | 内容 |
|------|------|
| `01-grub-detection-and-boot.txt` | GRUB ls 探测分区、fstab 内容、成功启动的 3 条命令 |
| `02-grub-default-wrong.txt` | GRUB_DEFAULT="5>4" 错误写法、apt-mark hold 锁错 meta 包 |

### 03-nvidia-cleanup — NVIDIA 驱动清理
| 文件 | 内容 |
|------|------|
| `01-ubuntu-drivers-autoinstall.txt` | ubuntu-drivers autoinstall 触发 595-open + 35 内核重装 |
| `02-network-and-tailscale.txt` | 35 内核下 nmcli 无网卡、tailscale IP |
| `03-orphan-590-binaries.txt` | nvidia-persistenced 版本 590 vs 580 不匹配、44 个孤儿库文件 |

### 04-kernel-35-investigation — 35 内核深度排查
| 文件 | 内容 |
|------|------|
| `01-boot-logs-comparison.txt` | 35 vs 29 启动日志对比：nvidia 加载但 amdgpu 缺失 |
| `02-secureboot-and-modprobe.txt` | Secure Boot 验证（非根因）、nvidia-kms.conf 格式错误 |
| `03-xorg-conf.txt` | /etc/X11/xorg.conf 内容（nvidia-xconfig 遗留） |

### 05-root-cause-fix — 根因修复
| 文件 | 内容 |
|------|------|
| `01-the-fix.txt` | 安装 linux-modules-extra + 验证 amdgpu.ko 存在 + 修复后成功日志 |

## 关键时间线

1. 重启黑屏 → TTY 诊断 → 发现 GDM 没拉起图形 session
2. GRUB 命令行手动 boot 6.17.0-29 内核 → 成功进桌面
3. 尝试 apt-mark hold linux-image-generic → 无效（HWE 内核）
4. ubuntu-drivers autoinstall → 触发 595-open + 35 内核重装 → 再次黑屏
5. 清理 590 孤儿二进制/库 → 修复 nvidia-smi 版本不匹配
6. 排查 35 内核：nvidia 模块加载正常但 amdgpu 缺失
7. 发现 /etc/X11/xorg.conf 遗留 → 删除
8. 安装 nvidia-prime + prime-select on-demand → 仍黑屏
9. 安装 linux-modules-extra-6.17.0-35-generic → 修复成功

## 最终状态

- 内核：6.17.0-35-generic（默认启动）
- NVIDIA 驱动：580.159.03（server LTS 分支）
- PRIME 模式：on-demand（amdgpu 显示 + nvidia 按需渲染）
- 备用内核：6.17.0-29、6.14.0-37
- GRUB 默认：名字写法锁定 6.17.0-35
