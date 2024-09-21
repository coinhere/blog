---
title: Archlinux+Hyprland安装配置
date: 2024-09-21 14:03:45
tags:
---

之前一直使用Archlinux+KDE，也花费了一定时间来美化，自觉效果还不错，也用了较长一段时间，但KDE在使用中时常出现些奇奇怪怪的问题，十分折磨。
前几天看了几个Hyprland的视频，美观程度和流畅感简直甩了KDE两条街，当时就想投奔Hyprland，只是考虑到自己从头配置加美化要花不少时间，加上不能保证Hyprland就不会出各种各样的毛病，所以一直没有转Hyprland。

刚好网上有许多大牛把自己Hyprland的配置分享到GitHub上，直接免去了配置加美化的一环，正好试一试平铺式窗口管理的感觉如何。
旧的Arch+KDE仍然保留，Hyprland就直接装在另一台笔记本电脑上，使用一段时间后再考虑是否把主力机也改成Hyprland。

笔记本配置：

- CPU AMD Ryzen 7 7840HS with Radeon 780M Graps
- GPU AMD Radeon 780M

## Archlinux+Hyprland安装

### Archlinux安装

Archlinux安装就不赘述了，我参考的是[archlinux 简明指南](https://arch.icekylin.online/)和官方的[Archlinux Installation guide](https://wiki.archlinux.org/title/Installation_guide)。

### 广式烧鹅

这里使用archlinux简明指南推荐的dae。

安装：

```bash
sudo pacman -S dae dae
```

参见[吃鹅直通手册](https://github.com/daeuniverse/dae/blob/main/docs/zh/README.md)，添加配置文件`/etc/dae/config.dae`。之后启动dae：

```bash
sudo systemctl enable --now dae
```

### Hyprland安装

此时你应该以及走完官方的安装指南没有遗漏，archlinux简明教程也到了`进阶安装`中的`桌面环境与常用应用安装`，已经有了一个无桌面窗口环境纯命令行的Archlinux系统了。

我选择的Hyprland的配置是[HYDE](https://github.com/prasanthrangan/hyprdots)，这是GitHub上目前Star数最多的Hyprland配置。

要安装HYDE，执行以下命令，安装过程中会安装许多包，为此你可能需要烧鹅：

```bash
pacman -Sy git
git clone --depth 1 https://github.com/prasanthrangan/hyprdots ~/HyDE
cd ~/HyDE/Scripts
./install.sh
```

此后便可按照自己的喜好安装其他程序、更改配置了。

## Archlinux配置

除桌面环境为HYDE而非KDE外，其余配置基本参照[archlinux 简明指南](https://arch.icekylin.online/)，这里只记录不同的配置。

### 改键

需求：

- `CapsLock`单击为`Escape`
- `CapsLock` + `f,b,p,n,a,e,u,d` = `right`, `left`, `up`, `down`, `home`, `end`, `pageup`, `pagedown`
- `CapsLock` + `h,j,k,l` = `left`, `down`, `up`, `right`
- `Escape`为`CapsLock`
- 右`Ctrl`键与右`Alt`键互换

这里选用[keyd](https://github.com/rvaiya/keyd)来改键，安装并启用：

```bash
sudo pacman -S keyd
sudo systemctl enable --now keyd
```

添加配置文件，如下所示：

```/etc/keyd/default.conf
[ids]

*

[main]

# Maps capslock to escape when pressed and control when held.
capslock = overload(capslock_layer, esc)

# Remaps the escape key to capslock
esc = capslock

rightalt = rightctrl
rightctrl = rightalt

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

更改用户默认shell:

```bash
chsh -l # 查看安装了哪些 Shell
chsh -s /usr/bin/fish # 修改当前账户的默认 Shell
```

### 输入法安装配置

```bash
sudo pacman -S fcitx5-im # 输入法基础包组
sudo pacman -S fcitx5-chinese-addons # 官方中文输入引擎
sudo pacman -S fcitx5-configtool # 输入法设置工具
sudo pacman -S fcitx5-rime # 安装rime输入法
```

环境变量：

```~/.config/environment.d/im.conf
XMODIFIERS=@im=fcitx
```

创建`~/.local/share/fcitx5/rime/default.custom.yaml`并添加：

```~/.local/share/fcitx5/rime/default.custom.yaml

patch:
  menu:
    page_size: 7                   # 候选词数量

  schema_list:
    - schema: double_pinyin_flypy  # 使用小鹤双拼
```

在终端中运行`fcitx5-configtool`，打开设置界面，取消勾选`Only Show Current Language`，搜索Rime并移动到左侧，应用并退出即可。

之后启动fcitx5，`Ctrl+Space`即可输入中文。

```bash
fcitx5 -d # 启动fcitx5
```

记得设置自动启动:

```~/.config/hypr/hyprland.conf
exec-once = fcitx5 -d
```

小鹤双拼详细配置见<https://whatacold.io/zh-cn/blog/2022-10-04-rime-double-pinyin-flypy/>

### 输入法美化

这里使用fcitx5的[烛光皮肤](https://github.com/thep0y/fcitx5-themes-candlelight)中的macOS-dark，安装见<https://github.com/thep0y/fcitx5-themes-candlelight>

皮肤调整，使用`catppuccin`配色：

在`~/.local/share/fcitx5/themes/macOS-dark/highlight.svg`文件中，修改 fill="#015ad7" 为 fill="#89b4fa"，

在`~/.local/share/fcitx5/themes/macOS-dark/panel.svg`文件中，修改 fill="#383838" 为 fill="#313244"。

### 终端字体

我使用[Maple Font](https://github.com/subframe7536/maple-font)

Release中下载MapleMono-NF-CN.zip，解压并放在`~/.local/share/fonts/`中。

在kitty配置文件`~/.config/kitty/kitty.conf`将字体修改为`Maple Mono NF CN`

### Kitty点击链接时浏览器为`Brave`，无缩放

解决方法：设置默认浏览器为Firefox：

```bash
xdg-settings set default-web-browser firefox.desktop
```

### timeshift GTK报错，图形界面无法启动

## HYDE配置

### 触摸板方向不符合直觉，滑动速度过快

Hyprland配置文件为`~/.config/hypr/hyprland.conf`，其中添加：

```~/.config/hypr/hyprland.conf
input {
  touchpad {
    natural_scroll = true
    scroll_factor = 0.5
  }
}
```

### waybar任务栏设置

waybar配置文件为`~/.config/waybar/config.ctl`。
第一个数字代表正在启用的配置。
HYDE中`win+alt+UP`、`win+alt+DOWN`切换waybar配置。
