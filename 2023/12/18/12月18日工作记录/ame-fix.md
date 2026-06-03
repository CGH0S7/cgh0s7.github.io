---
title: Archlinux问题记录
date: 2025-03-10 22:47:08
tags: [生活, Archlinux]
---

前几周在使用 Arch Linux 时遇到了两个有趣的问题，顺手记录在此，或许能帮到有类似困扰的朋友～

---

### 1. 显卡功耗上限解锁
**问题现象**  
使用 `nvidia-smi` 查看显卡功耗时，发现最大功耗被限制在 55W，性能无法完全释放：

```bash
nvidia-smi  # 输出显示 Power Limit: 55.00 W
```

**解决方案**  
启用 NVIDIA 动态功耗管理服务即可：
```bash
sudo systemctl enable --now nvidia-powerd
```

---

### 2. Steam Proton 输入失效
**问题描述**  
使用 proton-cachyos 和 proton-ge-custom 版本时，Steam 游戏无法接收任何输入（键盘/手柄）。  
⚠️ 经排查发现是 Wayland 协议兼容性问题导致。

**解决方案**  
在 Steam 游戏启动选项中加入环境变量禁用 Wayland：
```bash
PROTON_ENABLE_WAYLAND=0 %command%
```

---

