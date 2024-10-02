---
title: 移动硬盘安装配置Archlinux+Hyprland
date: 2024-09-21 14:03:45
tags:
---

之前一直使用Archlinux+KDE，也花费了一定时间来美化，自觉效果还不错，也用了较长一段时间，但KDE在使用中时常出现些奇奇怪怪的问题，十分折磨。

前几天看了几个Hyprland的视频，美观程度和流畅感简直甩了KDE两条街，当时就想投奔Hyprland，只是考虑到自己从头配置加美化要花不少时间，加上不能保证Hyprland就不会出各种各样的毛病，所以一直没有转Hyprland。

这几天拆机空出了一块硬盘，又淘了个硬盘和，正好组成一个移动硬盘，加上GitHub上有许多美观的Hyprland的配置，直接免去了配置加美化的一环，正好试一试平铺式窗口管理的感觉如何。

配置：

- Lenovo Xiaoxin pro 14
- CPU AMD Ryzen 7 7840HS with Radeon 780M Graps
- 1TB RC20固态硬盘+硬盘盒

## Archlinux+Hyprland安装

### Archlinux安装

我参考的是[archlinux 简明指南](https://arch.icekylin.online/)和官方的[Archlinux Installation guide](https://wiki.archlinux.org/title/Installation_guide)，[Install Arch Linux on a removable medium](https://wiki.archlinux.org/title/Install_Arch_Linux_on_a_removable_medium)，[Install Arch Linux from existing Linux](https://wiki.archlinux.org/title/Install_Arch_Linux_from_existing_Linux)，以及[GRUB](https://wiki.archlinux.org/title/GRUB)。

#### 安装前准备

因为是在已有的Archlinux上安装Archlinux，因此事先不需要配置网络、时钟、镜像源那些东西。

首先安装[arch-install-scripts](https://archlinux.org/packages/?name=arch-install-scripts)这个包，里面是一些安装Archlinux要用到的命令，以及[dosfstools](https://archlinux.org/packages/core/x86_64/dosfstools/)，包含格式化分区的命令。

连上硬盘，切换到Root。

#### 分区格式化

之后用`cfdisk`分区，这里这里按官方的提示，增加了一个128G大小的NTFS分区，专门用于跨平台存储数据，并作为第一个分区，其它按安装指南即可。

这里按简明指南使用Btrfs文件系统，因此`/home/`和`/`都在一个分区上。

![分区示例](/home/atmos//Pictures/Screenshots/241002_22h14m48s_screenshot.png "分区示例")

格式化分区，注意将各个分区名替换成你对应的分区：

```bash
mkfs.fat -F 32 /dev/sda2 # 格式化EFI分区，双系统注意不要运行这步
mkswap /dev/sda3 # 格式化Swap分区
mkfs.btrfs -L myArch /dev/sda4 # 格式化Btrfs分区
mount -t btrfs -o compress=zstd /dev/sda4 /mnt # 挂载Btrfs分区
btrfs subvolume create /mnt/@ # 创建 / 目录子卷
btrfs subvolume create /mnt/@home # 创建 /home 目录子卷
umount /mnt
```

#### 安装系统

首先挂载各个分区：

```bash
mount -t btrfs -o subvol=/@,compress=zstd /dev/sda4 /mnt # 挂载 / 目录
mkdir /mnt/home # 创建 /home 目录
mount -t btrfs -o subvol=/@home,compress=zstd /dev/sda4 /mnt/home # 挂载 /home 目录
mount --mkdir /dev/sda2 /mnt/boot # 挂载 /boot 目录
swapon /dev/sda3 # 挂载交换分区
```

因为我用于安装Archlinux的系统本身启用了swap分区，因此还要把这个系统的swap分区取消挂载，否则生成fstab文件时会把这个分区也加进去。

```bash
swapoff /dev/nvme0n1p6
```

安装系统，因为是从已有的Archlinux上安装Archlinux，因此加上`-c`参数，直接使用本地的缓存：

```bash
pacstrap -c /mnt base base-devel linux linux-firmware btrfs-progs
# 如果使用btrfs文件系统，额外安装一个btrfs-progs包
pacstrap -c /mnt amd-ucode intel-ucode # 移动硬盘跨平台，因此都AMD和intel的微码都装上
pacstrap -c /mnt sof-firmware networkmanager ntfs-3g dosfstools # 板载声卡驱动，网络，挂载NTFS分区
pacstrap -c /mnt neovim sudo zsh zsh-completions man-db man-pages texinfo
```

生成fstab文件：

```bash
genfstab -U /mnt >> /mnt/etc/fstab
cat /mnt/etc/fstab # 查看fstab文件
```

#### 系统设置

这里建议按[archlinux 简明指南](https://arch.icekylin.online/)，一步一步来：

```bash
arch-chroot /mnt
nvim /etc/hostname # 添加主机名
```

设置hosts：

```bash
nvim /etc/hosts
```

加入：

```
127.0.0.1   localhost
::1         localhost
127.0.1.1   myArch.localdomain myArch
```

```bash
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime # 设置时区
hwclock --systohc # 同步硬件时间
nvim /etc/locale.genfstab # 去掉注释 en_US.UTF-8 UTF-8 以及 zh_CN.UTF-8 UTF-8
locale-gen
echo 'LANG=en_US.UTF-8' >> /etc/locale.conf # 设置系统语言
passwd root # 设置root密码
```

按[Install Arch Linux on a removable medium](https://wiki.archlinux.org/title/Install_Arch_Linux_on_a_removable_medium)的要求，需要修改`/etc/mkinitcpio.conf`文件：

在`HOOKS`中将`block`和`keyboard`移到`autodetect`之前：

```
HOOKS=(base udev keyboard block autodetect microcode modconf kms keymap consolefont filesystems fsck)
```

然后运行：

```bash
mkinitcpio -P
```

#### 启动引导

首先安装grub:

```bash
pacman -S grub efibootmgr os-prober # 前两个是GRUB必需的，os-prober用于引导windows系统
```

按[Install Arch Linux on a removable medium](https://wiki.archlinux.org/title/Install_Arch_Linux_on_a_removable_medium)的要求，
运行以下命令来安装为BIOS和UEFI分别安装GRUB，并加上`--removable`以保证将硬盘移至另一台计算机时能够从硬盘启动：

```bash
grub-install --target=i386-pc /dev/sda2 --recheck # 未知原因，这个命令运行失败，可能是要另建一个BIOS分区
grub-install --target=x86_64-efi --efi-directory=/boot --removable --recheck
```

接下来修改`/etc/default/grub`文件，并生成配置：

```bash
nvim /etc/default/grub # loglevel=5 nowatchdog 取消注释最后一行启用os-prober
grub-mkconfig -o /boot/grub/grub.cfg
```

完成安装，退出系统：

```bash
exit # 退回安装环境
umount -R /mnt # 卸载新分区
reboot # 重启
```

### Hyprland安装

我选择的Hyprland的配置是[HYDE](https://github.com/prasanthrangan/hyprdots)，这是GitHub上目前Star数最多的Hyprland配置。

要安装HYDE，执行以下命令，安装过程中会安装许多包，为此你可能需要魔法：

```bash
pacman -Sy git
git clone --depth 1 https://github.com/prasanthrangan/hyprdots ~/HyDE
cd ~/HyDE/Scripts
./install.sh
```

此后便可按照自己的喜好安装其它程序以及更改配置了。

详细见[Hyprland Wiki](https://wiki.hyprland.org/Getting-Started/Preconfigured-setups/#prasanthrangan)

## Archlinux配置

除桌面环境为HYDE而非KDE外，其余配置基本参照[archlinux 简明指南](https://arch.icekylin.online/)，这里只记录不同的配置。

### 改键

需求：

- `CapsLock`单击为`Escape`
- `CapsLock` + `f,b,p,n,a,e,u,d` = `right`, `left`, `up`, `down`, `home`, `end`, `pageup`, `pagedown`
- `CapsLock` + `h,j,k,l` = `left`, `down`, `up`, `right`
- `Escape`为`CapsLock`
- 右`Ctrl`键与右`Alt`键互换

这里选用[keyd](https://github.com/rvaiya/keyd)来改键，如果只是单纯交换键位可以使用Hyprland自带的[改键配置]()，

安装并启用keyd：

```bash
sudo pacman -S keyd
sudo systemctl enable --now keyd
```

要查看各个键位的名称，运行并按下要查看的键位：

```bash
sudo keyd monitor
```

添加配置文件`/etc/keyd/default.conf`，如下所示：

```
[ids]

*

[main]

# Maps capslock to escape when pressed and control when held.
capslock = overload(capslock_layer, esc)

# Remaps the escape key to capslock
esc = capslock

rightalt = rightcontrol
rightctrol = rightalt

[capslock_layer]
f = right
b = left
p = up
n = down
a = home
e = end
u = pageup
d = pagedown
h = left
j = down
k = up
l = right
```

运行以下命令来重载配置：

```bash
sudo keyd reload
```

### fish shell

开箱即用，无需配置

安装并更改用户默认shell:

```bash
sudo pacman -S fish
chsh -l # 查看安装了哪些 Shell
chsh -s /usr/bin/fish # 修改当前账户的默认 Shell
```

### timeshift备份

运行以下命令打开timeshift:

```bash
sudo -E timeshift-gtk # -E 表示保留当前用户环境
```

直接运行timeshift-gtk，或点击图标运行，会发现无法打开图形界面。这是因为运行timeshift需要root权限，而切换到root后便失去了wayland的环境。

详细解释见<https://github.com/linuxmint/timeshift/issues/147>

### 输入法安装配置

安装输入法相关包：

```bash
sudo pacman -S fcitx5-im # 输入法基础包组
sudo pacman -S fcitx5-chinese-addons # 官方中文输入引擎
sudo pacman -S fcitx5-configtool # 输入法设置工具
sudo pacman -S fcitx5-rime # 安装rime输入法
yay -S rime-ice # 雾凇拼音输入方案
```

环境变量`~/.config/environment.d/im.conf`(详细信息见[fcitx5 in wayland](https://fcitx-im.org/wiki/Using_Fcitx_5_on_Wayland#Chromium_.2F_Electron))：

```
XMODIFIERS=@im=fcitx
```

创建`~/.local/share/fcitx5/rime/default.custom.yaml`并添加：

```
patch:
  # 仅使用「雾凇拼音」的默认配置，配置此行即可
  __include: rime_ice_suggestion:/
  # 以下根据自己所需自行定义
  __patch:
    menu/page_size: 7 #候选词个数
    schema_list: # 不使用小鹤双拼去掉下两行
      - schema: double_pinyin_flypy
```

在终端中运行`fcitx5-configtool`，打开设置界面，取消勾选`Only Show Current Language`，搜索Rime并移动到左侧，应用并退出即可。

之后启动fcitx5，`Ctrl+Space`即可输入中文。

```bash
fcitx5 -d # 启动fcitx5
```

记得设置自动启动，在`~/.config/hypr/hyprland.conf`:

```
exec-once = fcitx5 -d
```

### 输入法美化

这里使用fcitx5的[FluentDark](https://github.com/Reverier-Xu/Fluent-fcitx5)皮肤，透明黑色。

### kitty

#### 终端字体

我使用[Maple Font](https://github.com/subframe7536/maple-font)，支持连字符。

Release中下载MapleMono-NF-CN.zip，解压并放在`~/.local/share/fonts/`中。

在kitty配置文件`~/.config/kitty/kitty.conf`将字体修改为`Maple Mono NF CN`

#### Kitty点击链接时浏览器为`Brave`，无缩放

解决方法：设置默认浏览器为Firefox：

```bash
xdg-settings set default-web-browser firefox.desktop
```

### nvim

直接使用[lazyvim](http://www.lazyvim.org/installation)的配置。

```bash
git clone https://github.com/LazyVim/starter ~/.config/nvim
rm -rf ~/.config/nvim/.git
nvim
```

### catppuccin主题美化

- [vscode](https://github.com/catppuccin/vscode)
- [vscode-icons](https://github.com/catppuccin/vscode-icons)
- [btop](https://github.com/catppuccin/btop)

### firefox插件

onetab
欧路词典

### 调整屏幕刷新率

HYDE中设置屏幕参数在`~/.config/hypr/monitors.conf`，详细设置查看<https://wiki.hyprland.org/Configuring/Monitors/>

默认是：

```config
monitor = ,preferred,auto,auto
```

首先查看屏幕支持的分辨率和刷新率设置:

```bash
hyprctl monitors
```

，选择你需要的设置，添加到`~/.config/hypr/monitors.conf`中，例如:

```config
monitor = ,2880x1800@120.00,auto,auto
```

> _注意_：如果你使用的电脑和笔者一样是`联想小新14Pro 2023`，且搭载的CPU是`AMD7840HS`，那么你会发现运行`hyprctl monitor`的结果中没有刷新率为120Hz的显示器设置，然而在Windows中可以正常应用120HZ，并且在Hyprland中强制使用120Hz会发现屏幕闪烁、变色、模糊。
>
> 这是因为小新主板提供的EDID信息（主板提供给操作系统显示器的信息，包括可使用的分辨率和刷新率）的校验和错误，需要将错误的EDID反编译、更正再编译后加载进内核，Bug探讨和详细的解决方法见<https://bbs.archlinux.org/viewtopic.php?id=289701>。

### wayland-electron设置

将需要修改的`desktop`文件从`/usr/share/applications/`复制到`~/.local/share/applications/`，再加上：

```
--enable-features=WebRTCPipeWireCapturer --ozone-platform-hint=auto --enable-wayland-ime
```

参数意义见<https://wiki.archlinux.org/title/Wayland#Electron>

## HYDE配置

### 触摸板方向不符合直觉，调转下滑方向

Hyprland配置文件为`~/.config/hypr/hyprland.conf`，其中添加：

```~/.config/hypr/hyprland.conf
input {
  touchpad {
    natural_scroll = true
  }
}
```

### waybar任务栏设置

waybar配置文件为`~/.config/waybar/config.ctl`。
第一个数字代表正在启用的配置。
HYDE中`win+alt+UP`、`win+alt+DOWN`切换waybar配置。

### SSDM theme

SSDM主题字体过小，这里是Corners主题：
在`/usr/share/sddm/themes/Corners/theme.conf`修改：

```
GeneralFontSize="18"
```

### hyprland drop-down terminal

使用[pyprland](https://hyprland-community.github.io/pyprland/)的内置插件[scratchpads](https://hyprland-community.github.io/pyprland/scratchpads.html)
我的配置如下`~/.config/hypr/pyprland.toml`：

```toml
[pyprland]
plugins = ["scratchpads"]

[scratchpads.term]
animation = "fromTop"
command = "kitty --class kitty-dropterm"
class = "kitty-dropterm"
size = "75% 50%" # percent of full screen
max_size = "1920px 100%"
margin = "10%" # percent of half screen
offset = "214%" # percent of half size, offset = (2*size + margin)/size
```

还需要设置自动启动和快捷键：

```conf
exec-once = /usr/bin/pypr
bind = ,F12,exec,pypr toggle term
```
