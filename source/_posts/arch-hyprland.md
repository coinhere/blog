---
title: 移动硬盘安装配置Archlinux+Hyprland
date: 2024-10-13 14:03:45
tags: 
 - Archlinux
 - Hyprland
 - 移动硬盘
---

之前一直使用Archlinux+KDE，也花费了一定时间来美化，自觉效果还不错，也用了较长一段时间，但KDE在使用中时常出现些奇奇怪怪的问题，十分折磨。

前几天看了几个Hyprland的视频，美观程度和流畅感简直甩了KDE两条街，当时就想投奔Hyprland，只是考虑到自己从头配置加美化要花不少时间，加上不能保证Hyprland就不会出各种各样的毛病，所以一直没有转Hyprland。

这几天拆机空出了一块硬盘，又淘了个硬盘和，正好组成一个移动硬盘，加上GitHub上有许多美观的Hyprland的配置，直接免去了配置加美化的一环，正好试一试平铺式窗口管理的感觉如何。

配置：

- Lenovo Xiaoxin pro 14
- CPU AMD Ryzen 7 7840HS with Radeon 780M Graps
- 1TB RC20固态硬盘+硬盘盒

## Archlinux安装

我参考的是[archlinux 简明指南](https://arch.icekylin.online/)和官方的[Archlinux Installation guide](https://wiki.archlinux.org/title/Installation_guide)，[Install Arch Linux on a removable medium](https://wiki.archlinux.org/title/Install_Arch_Linux_on_a_removable_medium)，[Install Arch Linux from existing Linux](https://wiki.archlinux.org/title/Install_Arch_Linux_from_existing_Linux)，以及[GRUB](https://wiki.archlinux.org/title/GRUB)。

### 安装前准备

因为是在已有的Archlinux上安装Archlinux，因此事先不需要配置网络、时钟、镜像源那些东西。

首先安装[arch-install-scripts](https://archlinux.org/packages/?name=arch-install-scripts)这个包，里面是一些安装Archlinux要用到的命令，以及[dosfstools](https://archlinux.org/packages/core/x86_64/dosfstools/)，包含格式化分区的命令。

连上硬盘，切换到Root。

### 分区格式化

之后用`cfdisk`分区，这里这里按官方的提示，增加了一个128G大小的windows的NTFS分区，专门用于跨平台存储数据，并作为第一个分区，其它按安装指南即可。

这里按简明指南使用Btrfs文件系统，因此`/home/`和`/`都在一个分区上。

{% asset_img cfdisk-example.png 分区示例 %}

格式化分区，注意将各个分区名替换成你对应的分区：

```bash
mkfs.ntfs -Q /dev/sda1 # 格式化NTFS分区
mkfs.fat -F 32 /dev/sda2 # 格式化EFI分区，双系统注意不要运行这步
mkswap /dev/sda3 # 格式化Swap分区
mkfs.btrfs -L myArch /dev/sda4 # 格式化Btrfs分区
mount -t btrfs -o compress=zstd /dev/sda4 /mnt # 挂载Btrfs分区
btrfs subvolume create /mnt/@ # 创建 / 目录子卷
btrfs subvolume create /mnt/@home # 创建 /home 目录子卷
umount /mnt
```

### 安装系统

首先挂载各个分区：

```bash
mount -t btrfs -o subvol=/@,compress=zstd /dev/sda4 /mnt # 挂载 / 目录
mount --mkdir -t btrfs -o subvol=/@home,compress=zstd /dev/sda4 /mnt/home # 挂载 /home 目录
mount --mkdir /dev/sda2 /mnt/boot # 挂载 /boot 目录
swapon /dev/sda3 # 挂载交换分区
```

因为我用于安装Archlinux的系统本身启用了swap分区，因此还要把这个系统的swap分区取消挂载，否则生成fstab文件时会把这个分区也加进去。

```bash
swapoff /dev/nvme0n1p6 # 取消原系统swap分区
mount --mkdir -t ntfs3 /dev/sda1 /mnt/media/windows # 挂载windows数据分区，用于生成fstab后自动挂载
df -h # 查看分区结果
free -h # 查看swap
```

安装系统，因为是从已有的Archlinux上安装Archlinux，因此加上`-c`参数，直接使用本地的缓存：

```bash
pacstrap -c /mnt base base-devel linux linux-firmware btrfs-progs
# 如果使用btrfs文件系统，额外安装一个btrfs-progs包
pacstrap -c /mnt amd-ucode intel-ucode # 移动硬盘跨平台，因此都AMD和intel的微码都装上
pacstrap -c /mnt sof-firmware networkmanager ntfs-3g dosfstools # 板载声卡驱动，网络，挂载NTFS分区
pacstrap -c /mnt neovim sudo fish man-db man-pages texinfo
```

生成并修改fstab文件，移除btrfs分区中两个字卷的subvolid参数，避免Timeshift恢复 Btrfs 快照时，可能出现由于子卷 ID 变更导致无法挂载目录而无法进入系统。

```bash
genfstab -U /mnt >> /mnt/etc/fstab # 生成fstab文件
nvim /mnt/etc/fstab # 修改fstab文件
```

### 系统设置

此处均参照[archlinux 简明指南](https://arch.icekylin.online/)，仅记录执行过的命令。
这里建议按[archlinux 简明指南](https://arch.icekylin.online/)，一步一步来：

```bash
arch-chroot /mnt
nvim /etc/hostname # 添加主机名，这里是myArch
```

设置hosts：

```bash
nvim /etc/hosts
```

加入，注意要和主机名一致：

```
127.0.0.1   localhost
::1         localhost
127.0.1.1   myArch.localdomain myArch
```

```bash
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime # 设置时区
hwclock --systohc # 同步硬件时间
nvim /etc/locale.gen # 去掉注释 en_US.UTF-8 UTF-8 以及 zh_CN.UTF-8 UTF-8
locale-gen
echo 'LANG=en_US.UTF-8' >> /etc/locale.conf # 设置系统语言
passwd root # 设置root密码
```

按[Install Arch Linux on a removable medium](https://wiki.archlinux.org/title/Install_Arch_Linux_on_a_removable_medium)的要求，需要修改`/etc/mkinitcpio.conf`文件：

```bash
nvim /etc/mkinitcpio.conf
```

在`HOOKS`中将`block`和`keyboard`移到`autodetect`之前：

```
HOOKS=(base udev keyboard block autodetect microcode modconf kms keymap consolefont filesystems fsck)
```

然后运行：

```bash
mkinitcpio -P
```

### 启动引导

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

### 重启后一般设置

此处均参照[archlinux 简明指南](https://arch.icekylin.online/)，仅记录执行过的命令。

#### 连接网络

```bash
systemctl enable --now NetworkManager
# 如果使用WiFi，运行以下命令
nmcli dev wifi list
nmcli dev wifi connect "wifi SSID" password "password"
```

#### 准备普通用户

```bash
useradd -m -G wheel -s /bin/bash myusername # 创建用户及用户家目录
passwd myusername # 设置密码
EDITOR=nvim visudo # 启用用户组权限，注释掉#%wheel ALL=(ALL:ALL) ALL
```

#### 用户设置

更改默认编辑器，在`/root/.bash_profile`及`/home/myusername/.bash_profile`中加入：

```
export EDITOR='nvim'
```

更改默认shell:

```bash
su myusername # 切换用户
chsh -l # 查看安装了哪些 Shell
chsh -s /usr/bin/zsh # 修改当前账户的默认 Shell
```

#### 开启 32 位支持库与 Arch Linux 中文社区仓库

修改`/etc/pacman.conf`：

取消下面两行的注释：

```
#[multilib]
#Include = /etc/pacman.d/mirrorlist
```

并添加：

```
[archlinuxcn]
Server = https://mirrors.ustc.edu.cn/archlinuxcn/$arch # 中国科学技术大学开源镜像站
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinuxcn/$arch # 清华大学开源软件镜像站
Server = https://mirrors.hit.edu.cn/archlinuxcn/$arch # 哈尔滨工业大学开源镜像站
Server = https://repo.huaweicloud.com/archlinuxcn/$arch # 华为开源镜像站
```

#### 更新`pacman`数据库，安装一些基础功能包

```
pacman -Syyu
sudo pacman -S sof-firmware alsa-firmware alsa-ucm-conf # 声音固件
sudo pacman -S adobe-source-han-serif-cn-fonts wqy-zenhei # 安装几个开源中文字体。一般装上文泉驿就能解决大多 wine 应用中文方块的问题
sudo pacman -S noto-fonts noto-fonts-cjk noto-fonts-emoji noto-fonts-extra # 安装谷歌开源字体及表情
sudo pacman -S archlinuxcn-keyring # cn 源中的签名（archlinuxcn-keyring 在 archlinuxcn）
sudo pacman -S yay # yay 命令可以让用户安装 AUR 中的软件（yay 在 archlinuxcn）
yay -S ttf-ms-win11-auto-zh_cn # 微软字体
```

### 设置`timeshift`备份

建议在安装驱动和Hyprland前备份，我习惯在安装hyprland前将显卡驱动装好。

```bash
sudo pacman -S timeshift
```

修改`/etc/timeshift/timeshift.json`，修改的行如下，运行`lsblk`查看UUID：

```
  "backup_device_uuid" : "21b2199f-38e4-4bf1-ae1e-0c9d4d0431f8", # 修改为自己备份分区的UUID
  "do_first_run" : "false",
  "btrfs_mode" : "true",
  "include_btrfs_home" : "true", # 是否备份/home
  "include_btrfs_home_for_restore" : "true", # 是否恢复/home
```

安装Hyprland之后，如果遇到timeshift GUI 无法启动的情况，需要安装`xorg-xhost`，原因见[arch wiki](https://wiki.archlinux.org/title/Timeshift#Timeshift_GUI_not_launching_on_Wayland)

```bash
sudo pacman -S xorg-xhost
```

这里先用命令行启用备份，可见[简明教程里的介绍](https://arch.icekylin.online/guide/advanced/system-ctl.html#%E7%B3%BB%E7%BB%9F%E5%BF%AB%E7%85%A7-%E5%A4%87%E4%BB%BD-%E4%B8%8E%E6%96%87%E4%BB%B6%E4%BC%A0%E8%BE%93)：

```bash
timeshift --list # 查看快照
timeshift --create --comments "after update" --tags D # 创建快照，标签为每日 
# timeshift  --restore  --snapshot '2014-10-12_16-29-08' --skip-grub # 恢复指定快照，并跳过grub
```

### 安装显卡驱动

**这一步可以跳过，因为安装HyDE的时候，会自动检测显卡并安装驱动及设定相关参数，包括NVIDIA**

没有什么说的比[简明教程](https://arch.icekylin.online/guide/rookie/graphic-driver)，和Arch wiki说的更清楚的了。

因为是移动硬盘，且主力机是40系显卡配Intel核显以及AMD核显笔记本，我安装的是：

```
sudo pacman -S mesa lib32-mesa vulkan-intel lib32-vulkan-intel # Intel 核芯显卡
sudo pacman -S mesa lib32-mesa xf86-video-amdgpu vulkan-radeon lib32-vulkan-radeon #AMD 集成显卡
sudo pacman -S nvidia nvidia-settings lib32-nvidia-utils # NVIDIA 独立显卡
```

Arch Wiki 提示：
> NVIDIA还需要将 `kms` 从 `0/etc/mkinitcpio.conf` 里的 `HOOKS` 数组中移除，并重新生成 `initramfs`。 这能防止 `initramfs` 包含 `nouveau` 模块，以确保内核在早启动阶段不会加载它。

在`HOOKS`中，删除`kms`

```
HOOKS=(base udev keyboard block autodetect microcode modconf keymap consolefont filesystems fsck)
```

然后运行：

```bash
sudo mkinitcpio -P
```

此外Arch Wiki提到，Wayland + Nvidia 还需要设置[DRM 内核级显示模式设置](https://wiki.archlinuxcn.org/wiki/Wayland#%E7%B3%BB%E7%BB%9F%E9%9C%80%E6%B1%82)，不然可能会导致黑屏：

设置[环境变量](https://wiki.archlinuxcn.org/wiki/%E7%8E%AF%E5%A2%83%E5%8F%98%E9%87%8F)，在`/etc/environment`中添加:

```
GBM_BACKEND=nvidia-drm
__GLX_VENDOR_LIBRARY_NAME=nvidia
```

## Hyprland安装

推荐两个Hyprland的配置，一个是[prasanthrangan's HYDE](https://github.com/prasanthrangan/hyprdots)，简洁、干净、美观，包含多个不同风格的主题和基本的一些应用，适合于想要在一个美观的主题上搭建自己的Hyprland的用户，这也是GitHub上目前Star数最多的Hyprland配置。

### HYDE

要安装HYDE，执行以下命令，安装过程中会安装许多包，为此你可能需要魔法：

```bash
pacman -Sy git
git clone --depth 1 https://github.com/prasanthrangan/hyprdots ~/HyDE
cd ~/HyDE/Scripts
./install.sh
```

可以添加其他想要安装的软件到`Scripts/custom_apps.lst`：

```bash
./install.sh custom_apps.lst
```

更新则运行：

```bash
cd ~/HyDE/Scripts
git pull
./install.sh -r
```

更新前修改`Scripts/restore_cfg.lst`来避免之前的配置被覆盖，我修改的是：

```
N|Y|${HOME}/.config/kitty|kitty.conf|kitty
N|Y|${HOME}/.config/waybar|config.ctl|waybar
```

### JaKooLit

另一个是[JaKooLit's Hyprland Dotfiles](https://github.com/JaKooLit/Hyprland-Dots)，不同之处是有更丰富的功能，例如下拉式终端、工作区概览，该配置为各种系统都提供率安装脚本，这里使用的是[archlinux 的安装脚本](https://github.com/JaKooLit/Arch-Hyprland)。

安装请执行以下命令，安装过程中会安装许多包，为此你可能需要魔法：

```bash
git clone --depth=1 https://github.com/JaKooLit/Arch-Hyprland.git ~/Arch-Hyprland
cd ~/Arch-Hyprland
chmod +x install.sh
./install.sh
```

此后便可按照自己的喜好安装其它程序以及更改配置了。

## Archlinux配置

### 服务启用

Hyprland没有启用蓝牙服务和Timeshift自动备份服务，因此手动启动：

```bash
sudo pacman -S bluez bluez-utils # HYDE 会自动安装这两个包
sudo systemctl enable --now bluetooth # 启用蓝牙服务
sudo systemctl enable --now cronie # 启用Timeshift自动备份
```

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

# Maps capslock to escape and a self-defined key -- capslock_layer
capslock = overload(capslock_layer, esc)

# Remaps the escape key to capslock
esc = capslock

# Swap rightalt and rightcontrol
rightalt = rightcontrol
rightctrol = rightalt

# When capslock_layer is pressed
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

直接运行timeshift-gtk，或点击图标运行，会发现无法打开图形界面。详细说明见<https://wiki.archlinux.org/title/Running_GUI_applications_as_root#Wayland>

解决方法包括：

- `xorg-xhost`，原因见[arch wiki](https://wiki.archlinux.org/title/Timeshift#Timeshift_GUI_not_launching_on_Wayland)

- 运行以下命令打开timeshift:

```bash
sudo -E timeshift-gtk # -E 表示保留当前用户环境
```

详细解释见<https://github.com/linuxmint/timeshift/issues/147>

### 输入法安装配置

#### 安装输入法相关包

```bash
sudo pacman -S fcitx5-im # 输入法基础包组
sudo pacman -S fcitx5-chinese-addons # 官方中文输入引擎
sudo pacman -S fcitx5-configtool # 输入法设置工具
sudo pacman -S fcitx5-rime # 安装rime输入法
```

#### 添加环境变量

创建`~/.config/environment.d/im.conf`，并添加(详细信息见[fcitx5 in wayland](https://fcitx-im.org/wiki/Using_Fcitx_5_on_Wayland#Chromium_.2F_Electron))：

```
XMODIFIERS=@im=fcitx
QT_IM_MODULE=fcitx
```

#### 启动fcitx5，安装rime输入法

```bash
fcitx5 -d # 启动fcitx5
```

在终端中运行`fcitx5-configtool`(或者右键任务栏的输入法图标，点击configure)，打开设置界面，取消勾选`Only Show Current Language`，在上方搜索Rime并将其移动到左侧，点击`Apply`并退出即可。

此时，使用`Ctrl`+`Space`即可切换至`Rime`并输入中文。

第一次使用时，会自动生成rime配置文件，在`~/.local/share/fcitx5/rime`中。

#### 使用雾凇拼音词库

```bash
yay -S rime-ice # 雾凇拼音输入方案

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

之后重新启动fcitx5(右键图标点击`restart`)，`Ctrl+Space`即可输入中文。

记得设置自动启动，在`~/.config/hypr/hyprland.conf`:

```
exec-once = fcitx5 --replace -d
```

#### 输入法美化

这里使用fcitx5的[FluentDark](https://github.com/Reverier-Xu/Fluent-fcitx5)皮肤，透明黑色。

安装：

```bash
# Dark theme
yay -S fcitx5-skin-fluentdark-git
# Light theme
yay -S fcitx5-skin-fluentlight-git
```

之后进入fcitx5-configtool，在`Addons`-`UI`-`Classic User Interface`中，在`Theme`和`Dark Theme`中下拉选中自己想要的主题即可。

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

或者：

> Open the browser >> navigate chrome://flags/ >> search for Preferred Ozone platform >> Select wayland

### neovim

#### neovim配置

直接使用[lazyvim](http://www.lazyvim.org/installation)的配置。

```bash
git clone https://github.com/LazyVim/starter ~/.config/nvim
rm -rf ~/.config/nvim/.git
nvim
```

安装成功后在nvim中运行`:LazyHealth`、`:Mason`、`:MasonLog`检查是否存在问题。

`:LazyExtra`查看额外配置。

#### neovide

安装[neovide](https://neovide.dev/)(一个Neovim的图形用户界面)以获得更好的视觉及输入体验：

```bash
sudo pacman -S neovide
```

记得在`~/.config/hypr/keybindings.conf`中添加启动快捷键：

```
bind = $mainMod, N, exec, neovide # launch terminal emulator
```

neovide自动读取neovim的设置，因此neovide的设置也放在neovim中，如`~/.config/nvim/lua/config/options.lua`

```lua
-- neovide configurations
if vim.g.neovide then
  vim.o.guifont = "Maple Mono NF CN:h9" -- 设置neovide字体
  vim.g.neovide_scale_factor = 1.0 -- neovide缩放与系统缩放叠加了，因此取消neovide的缩放
end
```

#### catppuccin主题美化

创建并添加`~/.config/nvim/lua/plugins/colorscheme.lua`:

```
return {
  {
    "folke/tokyonight.nvim",
    opts = { style = "storm" },
  },
  { "rose-pine/neovim", name = "rose-pine" },
  { "EdenEast/nightfox.nvim" },
  { "catppuccin/nvim", opts = { term_colors = true, dim_inactive = { enabled = true } } }, # term_colors保证neovide内置terminal色彩正确
  {
    "LazyVim/LazyVim",
    opts = {
      -- colorscheme = "nordfox",
      -- colorscheme = "rose-pine",
      colorscheme = "catppuccin",
    },
  },
}

```

### vscode

#### vscode无法输入中文

在`~/.config/code-flags.conf`中加入：

```
--enable-wayland-ime
```

#### catppuccin主题美化

- [vscode](https://github.com/catppuccin/vscode)
- [vscode-icons](https://github.com/catppuccin/vscode-icons)

### Firefox

#### firefox字体

Hyprlan默认的字体有些奇怪，这里修改字体设置。需要安装Windows字体。

在`Settings`-`Fonts`-`Advanced`-`Fonts for Simplified Chinese`，设置为：

{% asset_img font-settings.png 字体设置 %}

#### 插件

onetab
欧路词典

#### 取消视频自动静音

在`Settings`中搜索`Autoplay`，将`Default for all websites:`从`Block Audio`修改为`Allow Audio and Video`

### wayland-electron设置

将需要修改的`desktop`文件从`/usr/share/applications/`复制到`~/.local/share/applications/`，再加上：

```

--enable-features=WebRTCPipeWireCapturer --ozone-platform-hint=auto --enable-wayland-ime

```

参数意义见<https://wiki.archlinux.org/title/Wayland#Electron>

#### 设置Electron配置文件

详细说明见<https://wiki.archlinux.org/title/Wayland#Configuration_file>

如`~/.config/electron-flags.conf`：

```

--ozone-platform-hint=auto
--enable-wayland-ime

```

以及`~/.config/electron13-flags.conf`：

```

--enable-features=UseOzonePlatform
--ozone-platform=wayland
--enable-wayland-ime

```

### btop 类似任务管理器

```bash
sudo pacman -S btop
```

catppuccin主题安装<https://github.com/catppuccin/btop>

### openRGB 光污染必备

```bash
sudo pacman -S openrgb
```

设置自动启动：

```
exec-once = openrgb --startminimized --profile "your-profile-name"
```

### npm换源

```bash
npm config set registry=https://registry.npmmirror.com # 最新淘宝源
```

## Hyprland配置

详细见[Hyprland Wiki](https://wiki.hyprland.org/Getting-Started/Preconfigured-setups/#prasanthrangan)

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

> *注意*：如果你使用的电脑和笔者一样是`联想小新14Pro 2023`，且搭载的CPU是`AMD7840HS`，那么你会发现运行`hyprctl monitor`的结果中没有刷新率为120Hz的显示器设置，然而在Windows中可以正常应用120HZ，并且在Hyprland中强制使用120Hz会发现屏幕闪烁、变色、模糊。
>
> 这是因为小新主板提供的EDID信息（主板提供给操作系统显示器的信息，包括可使用的分辨率和刷新率）的校验和错误，需要将错误的EDID反编译、更正再编译后加载进内核，Bug探讨和详细的解决方法见<https://bbs.archlinux.org/viewtopic.php?id=289701>。
>
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

### HyDE更新覆盖设置

在`~/HyDE/Scripts/restore_cfg.lst`中，我修改的是：

```
N|Y|${HOME}/.config/kitty|kitty.conf|kitty
N|Y|${HOME}/.config/waybar|config.ctl|waybar
```

### SDDM theme

#### SDDM主题字体过小

在`/usr/share/sddm/themes/<theme-name>/theme.conf`修改：

```
GeneralFontSize="18"
```

#### 修改SDDM背景图片

在`/usr/share/sddm/themes/Candy/backgrounds/`里添加自己想要的背景，再在`/usr/share/sddm/themes/Candy/theme.conf`里修改。

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
