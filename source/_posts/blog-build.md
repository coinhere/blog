---
title: Hexo+GitHub搭建博客
date: 2024-9-7 14:03:45
tags: 
 - Hexo
 - Github
---

搭建博客一般需要这些：

- 文章。这里是 Markdown 文档。
- 博客框架。用于生成博客网页和提供 Web 服务。
- 版本管理。简言之就是备份和备份管理。
- 服务器。一台空闲的机器，用于运行框架，提供 Web 服务。

[Hexo](https://hexo.io/) 是一个快速、简洁且高效的博客框架，使用 Markdown 来编写帖子，生成静态网页。

[Git](https://git-scm.com/)、[Github](https://github.com/) 用于版本管理和云托管代码。

如果选择部署网站到服务器，可以选择购买云服务器或者自建（需要公网IP，否则内网穿透）。

也可以选择让将 Web 服务部署到别人的服务器上。
~~鉴于 Github 国内访问不稳定，因此没有选择部署在 [Github Pages](https://pages.github.com/) 上。
~~而是选择 [Zeabur](https://zeabur.com/) 一个国人团队开发的部署服务的平台。~~
Zeabur隔一段时间停止服务，因此重回Github Pages

## Quick Start

### 安装必要软件

请根据自己的平台安装 [nodejs](https://nodejs.org/en/) 和 [Git](https://git-scm.com/), 这里是 Archlinux。

```bash
sudo pacman nodejs git
```

### 安装 Hexo

首先换源以提升安装速度。

运行下面命令查看当前源：

```bash
npm config get registry
```

默认官方源是：<https://registry.npmjs.org/>
这里更换到淘宝源：<http://registry.npmmirror.com>，之前的淘宝源：<http://registry.npm.taobao.org> 已停止服务。

```bash
npm config set registry https://registry.npmmirror.com/
```

安装 Hexo:

```bash
npm install -g hexo-cli
```

官方安装文档请参考：[安装文档](https://hexo.io/zh-cn/docs/)

### 建立博客网站

运行以下命令，初始化博客网站：

```bash
hexo init <folder>
cd <folder>
npm install
```

随后生成网页静态文件，再部署 Web 服务就可访问博客网站了：

```bash
hexo generate
hexo server
```

在浏览器中访问：<http://localhost:4000/>，即可查看博客。

### 更换博客主题

如果对默认主题不满意，可以选择自己喜欢的主题更换。主题可在<https://hexo.io/themes/>中查看，或者在 Github 中搜索 Hexo Themes。

这里选择 [Fluid](https://github.com/fluid-dev/hexo-theme-fluid)

### 托管代码至 Github

注册好 Github 账号后，新建一个空仓库，命名就行，其他全默认。

然后在博客文件夹中运行：

```bash
git init
git add .
git commit -m "first commit"
git branch -M main
git remote add origin https://github.com/<your-github-username>/<your-repo-name>.git
git push -u origin main
```

以上 git 命令的含义请查阅：<https://www.runoob.com/git/git-basic-operations.html>

Push 时，你可能需要输入密码，如需要免密，可自行配置 GitHub SSH Key，具体请查阅 <https://docs.github.com/en/authentication/connecting-to-github-with-ssh>。

### ~~部署服务至 [Zeabur](https://zeabur.com/) 平台~~

~~注册账号后，新建项目，选择服务器地区，添加服务选择 Github，选择自己的博客仓库即可（如果没有，点击配置 Github。~~

~部署完成后，在`网络-公开`中生成域名后访问即可。~

### 部署至 Github Pages
