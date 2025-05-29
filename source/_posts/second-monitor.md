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

可以在moonlight的设定中，设置分辨率和帧率

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
# 设置虚拟显示器分辨率为2880x1800，刷新率为120Hz，在主屏右侧，逆时针旋转90度
monitor = DP-2,2880x1800@120.00,auto-right,auto,transform,1
```

想要修改分辨率，刷新率，位置和旋转见[Hyprland wiki](https://wiki.hyprland.org/Configuring/Monitors/)

最后，sunshine设置中，指定要串流的屏幕，在`Audio/Video`中的Display number

通过查看命令行启动sunshine的输出，可以看到有关屏幕的number

## 更多设置

### 笔记本电脑开机自动开始串流

#### 开机自动登录

在Windows11中，首先将启动密码设置为空，确保电脑自动登录，或者设置人脸识别或指纹登录。

#### 开启自动启动moonlight并串流

按下`Win`+`R`，输入`shell:startup`，打开启动文件夹，创建moonlight的快捷方式到这个文件夹即可。

在moonlight的快捷方式的属性，添加参数`stream <服务器名字> <应用名字>`

我用的是`"C:\Program Files\Moonlight Game Streaming\Moonlight.exe" stream myArch Desktop`

#### 设置按下电源键自动关机

另外还要记得设置sunshine自启动，以及静音副屏。

### 启动不带副屏的选项

由于笔者的Arch Linux装在移动硬盘上，有时会在没有副屏的电脑上启动，因此需要系统启动时不启用虚拟显示器。

通过内核参数添加的虚拟显示器，如果要不启用，除了重新更改内核参数并再生成GRUB设置文件外，目前笔者没有找到其他好的方法（sysctl对于这个参数无效），Hyprland 的禁用显示器命令稍微有点用。

频繁修改内核参数并再生成GRUB设置文件，实在是太麻烦了。

根据主板型号动态加载内核参数是一种显而易见的方法，然而笔者对于如何在内核启动前判断主板型号并修改对应的内核参数并没有什么头绪。

在GRUB里增加一个不带副屏的启动条目，以后每次启动时手动选择。勉强算是一个办法。

#### Hyprland根据主板型号自动启用/禁用副屏

```bash
#!/bin/bash
[ "$(cat /sys/class/dmi/id/board_name)" = "Your board name" ] && sunshine || hyprctl keyword monitor <Your Screen Name>,disable
```

#### 添加GRUB不同内核参数条目

添加GRUB条目一般是修改`/etc/grub.d/40_custom`，然后再运行`sudo grub-mkconfig -o /boot/grub/grub.cfg`。

这里不搞那么麻烦，复制`/etc/grub.d/10_linux`为`/etc/grub.d/11_linux_custom`，然后修改这个文件的内核参数，这会另外生成一个除内核参数外完全一样的启动条目。

为了方便修改参数，先在`/etc/default/grub`删去增加虚拟显示器的内核参数，我们会在新文件中再添加启用虚拟显示器的内核参数。

在`11_linux_custom`中，找到这段代码：

```
if [ "x${GRUB_DISTRIBUTOR}" = "x" ] ; then
  OS=Linux
else
  OS="${GRUB_DISTRIBUTOR} Linux"
  CLASS="--class $(echo ${GRUB_DISTRIBUTOR} | tr 'A-Z' 'a-z' | cut -d' ' -f1|LC_ALL=C sed 's,[^[:alnum:]_],_,g') ${CLASS}"
fi
```

我们修改内核参数和系统名字——与原系统区别：

```
# Added by coinhere to add 2rd screen grub start entry.
GRUB_CMDLINE_LINUX_DEFAULT="${GRUB_CMDLINE_LINUX_DEFAULT} drm.edid_firmware=DP-2:edid/EDID-fix.bin video=DP-2:e"

if [ "x${GRUB_DISTRIBUTOR}" = "x" ] ; then
  OS=Linux
else
  OS="${GRUB_DISTRIBUTOR} Linux (2rd Screen)"
  CLASS="--class $(echo ${GRUB_DISTRIBUTOR} | tr 'A-Z' 'a-z' | cut -d' ' -f1|LC_ALL=C sed 's,[^[:alnum:]_],_,g') ${CLASS}"
fi
```

最后再运行`sudo grub-mkconfig -o /boot/grub/grub.cfg`。
