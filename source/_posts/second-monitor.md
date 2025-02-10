---
title: linux利用sunshine串流将笔记本屏幕作为副屏
date: 2025-02-10 09:35:00
tags: 
 - Archlinux
 - sunshine
 - hyprland
 - 副屏
---
自从有了台式机，除了回家和出差，笔记本基本上就是闲置的了。但是，笔记本的屏幕还是很好的，如果能够利用起来，那就再好不过了。这里介绍一种利用sunshine串流将笔记本屏幕作为副屏的方法。

本文参考了：

- <https://www.reddit.com/r/linux_gaming/comments/199ylqz/streaming_with_sunshine_from_virtual_screens/>
- <https://www.w568w.eu.org/spacedesk-on-linux.html>

## 原理

原理是首先创建一个虚拟显示器——这个虚拟显示器可以是硬件的（在某宝上买），或者模拟出来的。

虚拟显示器除了不能显示画面外，和其他的屏幕没有区别。

然后在你的桌面环境或窗口管理器中，便可以将这个虚拟显示器设置为副屏，此时仍没有画面。

最后再将这个副屏串流投影到实际的笔记本屏幕上。

最后享受你的副屏吧！

## 实现台式机到笔记本的串流

服务端为Archlinux+hyprland，客户端(也就是笔记本电脑)为Windows11.

### 服务端安装sunshine

```bash
sudo pacman -S sunshine
```

启动sunshine，右键单击系统托盘图标，选择`Open sunshine`，便可打开Web设定页面。

一开始默认设置即可。

### 客户端安装moonlight

在[官网](https://github.com/moonlight-stream/moonlight-qt/releases)下载安装moonlight即可

输入服务端的IP地址，点击连接，输入PIN码，即可连接。

至此，笔记本屏幕已经可以同步显示台式机桌面画面了，但还不是我们想要的副屏。

## 设置虚拟显示器

背景知识：

[EDID](https://en.wikipedia.org/wiki/Extended_display_identification_data)是一种用于显示器的元数据，包括显示器的名称、分辨率、刷新率、制造商等信息。

对Linux系统，一块显示器，就是一个接口（DP或者HDMI）加上这个接口所连显示器的信息（EDID）——向哪里传输图像以及需要怎样的图像。

因此我们首先创建EDID文件，并将其指定给一个接口，然后强制启用这个接口，于是就有了一个虚拟显示器。

### 获取并存放EDID文件

我们首先获取笔记本屏幕的EDID文件，并将这个文件放在`/usr/lib/firmware/edid/`目录下，各种屏幕的EDID文件你也可以直接在[这个网站](https://git.linuxtv.org/edid-decode.git/tree/data)下载。

Windows中。下载[MonitorInfoView](https://www.nirsoft.net/utils/monitor_info_view.html),运行后，输出EDID文件。

Linux中，直接复制`/sys/class/drm/card<接口名>/edid`这个文件到`/usr/lib/firmware/edid/`目录下，注意改名

### 绑定EDID文件到指定接口，并启用该接口

运行这个命令查看存在的接口：

```bash
for p in /sys/class/drm/*/status; do con=${p%/status}; echo -n "${con#*/card?-}: "; cat $p; done 
```

这里绑定到DP-2，
编辑内核参数，添加`drm.edid_firmware=DP-2:edid/EDID-fix.bin video=DP-2:e`

第一个参数是绑定EDID文件，第二个参数是启用这个接口，`:e`表示`enable`。

```bash
sudo nvim /etc/default/grub
```

这里我使用的EDID文件是之前修复小新EDID文件错误时用的，具体见<https://coinhere.fra1.zeabur.app/2024/10/13/arch-hyprland/>

```grub
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=5 nowatchdog drm.edid_firmware=eDP-1:edid/EDID-fix.bin,DP-2:edid/EDID-fix.bin video=DP-2:e nvidia_drm.modeset=1"
```

```bash
# 记得更新grub
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

## 设置副屏

在Hyprland中，修改`~/.config/hypr/monitors.conf`，加入:

```conf
monitor = DP-2,auto,auto,auto
```

想要修改分辨率，刷新率，和位置见[Hyprland wiki](https://wiki.hyprland.org/Configuring/Monitors/)

最后，sunshine设置中，指定要串流的屏幕，在`Audio/Video`中的Display number

通过查看命令行启动sunshine的输出，可以看到有关屏幕的number

```bash
