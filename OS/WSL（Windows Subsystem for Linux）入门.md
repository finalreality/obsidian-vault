---
title: "WSL（Windows Subsystem for Linux）入门"
source: "https://blog.csdn.net/qq_40773212/article/details/147797518"
author:
  - "[[qq_40773212]]"
published: 2025-05-08
created: 2025-06-04
description: "文章浏览阅读1.1k次，点赞5次，收藏20次。WSL 是 Windows 系统内置的 Linux 兼容层，允许直接在 Windows 中运行 Linux 命令行工具和应用程序，无需虚拟机或双系统。WSL 1：早期版本，通过翻译层兼容 Linux 系统调用，文件系统性能较低，但启动快。WSL 2：基于轻量级虚拟机（Hyper-V），支持完整 Linux 内核，文件系统性能接近原生，推荐使用。_windows wsl"
tags:
  - "clippings"
---
## 1.简介

WSL 是 Windows 系统内置的 Linux 兼容层，允许直接在 Windows 中运行 Linux [命令行工具](https://so.csdn.net/so/search?q=%E5%91%BD%E4%BB%A4%E8%A1%8C%E5%B7%A5%E5%85%B7&spm=1001.2101.3001.7020) 和应用程序，无需虚拟机或双系统。

WSL 1：早期版本，通过翻译层兼容 Linux 系统调用，文件系统性能较低，但启动快。

[WSL 2](https://so.csdn.net/so/search?q=WSL%202&spm=1001.2101.3001.7020) ：基于轻量级虚拟机（Hyper-V），支持完整 Linux 内核，文件系统性能接近原生，推荐使用。

## 2.安装与配置

- 步骤 1：启用 WSL 功能  
	wsl --install  
	==此命令自动安装 WSL 2 内核和默认发行版（如 Ubuntu）。==
- 步骤 2：手动指定发行版  
	安装其他发行版（如 Ubuntu 22.04）：  
	wsl --install -d Ubuntu-22.04
- 步骤 3：设置默认 WSL 版本  
	wsl --set-default-version 2

## 3.常用命令

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/403e0c0888a1495d8554dcd0c59d993d.png)

## 4.进阶使用

### 4.1 文件系统交互

从 Windows 访问 Linux 文件：  
路径格式：\\wsl KaTeX parse error: Undefined control sequence: \\< at position 1: \\̲<̲发行版名称>\\（如 \\\\wsl \\Ubuntu-22.04\\home\\user）。

从 Linux 访问 Windows 文件：  
挂载在 /mnt/ 目录下（如 /mnt/c/Users）。

### 4.2 网络互通

Windows 访问 WSL 服务：  
WSL 2 的 IP 地址可通过 ip addr show eth0 查看。

WSL 访问 Windows 服务：  
使用 hostname -I 获取 Windows 主机 IP（通常为 172.x.x.x）。

### 4.3 配置代理

在 WSL 中继承 Windows 代理设置：

```bash
# 获取 Windows 主机 IP
export HOST_IP=$(cat /etc/resolv.conf | grep nameserver | awk '{print $2}')
# 设置代理（假设 Windows 代理端口为 7890）
export http_proxy="http://$HOST_IP:7890"
export https_proxy="http://$HOST_IP:7890"
```

### 4.4 运行 GUI 程序

WSL 2 支持 Linux GUI 应用（需 Windows 11 或手动配置）：

```bash
# 安装图形工具（如 Firefox）
sudo apt install firefox
# 启动 GUI 应用（需先安装 X Server，如 VcXsrv）
export DISPLAY=$(cat /etc/resolv.conf | grep nameserver | awk '{print $2}'):0.0
firefox
```

### 4.5 Docker 集成

在 WSL 2 中直接使用 Docker：

安装 Docker Desktop for Windows，并启用 WSL 2 集成：  
Docker Desktop → Settings → Resources → WSL Integration → 勾选对应发行版。

在 WSL 中验证：

```bash
docker run hello-world
```