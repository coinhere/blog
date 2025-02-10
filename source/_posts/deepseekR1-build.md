---
title: 本地在线\离线部署deepseek-R1
date: 2025-01-28 02:35:00
tags: 
 - Archlinux
 - Deepseek
 - 离线
---
部署两个模型：

* 一个放在公司内网
* 一个放在自己电脑

## 安装ollama

官网下载安装包: <https://www.ollama.com/download/windows>

Archlinux可以：

```bash
 yay -S ollama
```

## 部署deepseek R1

可在ollama官网阅读deepseek R1各个模型的部署命令：<https://www.ollama.com/library/deepseek-r1>

这里选择部署DeepSeek-R1-Distill-Qwen-7B

```bash
allama serve # 另开一个终端启动allama
ollama run deepseek-r1:7b
```

## 下载开源客户端 Chatbox

用于获得更好的、现代的、美观的AI交互界面

官网：<https://chatboxai.app/zh>

Archlinux:
```
yay -S chatbox-bin
```
