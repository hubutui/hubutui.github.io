---
title: "从 Arch 到 WSL2：在 Windows 下复刻纯正 Linux 开发体验"
date: 2026-04-25T00:00:00+08:00
toc: false
images:
tags:
  - WSL
  - ArchLinux
  - docker
  - podman
  - Linux
---

## 简介

本文档旨在为熟悉 Arch Linux 的开发者提供一份详尽的指南，帮助你在 Windows 11 环境下，通过 Windows Subsystem for Linux 2 (WSL2) 部署和配置一个与原生环境高度一致的 Arch Linux 系统。

本文将会不定期更新，仅供参考。

## 核心前提：启用 WSL2

WSL2 需要 Windows 专业版或企业版，**家庭版不支持 WSL2**。

具体安装步骤请参考 [微软官方文档](https://learn.microsoft.com/zh-cn/windows/wsl/install)。

## 选择 Linux 发行版

WSL 支持多种 Linux 发行版，建议使用 Arch Linux 以获得与原生环境一致的体验。具体安装方法请参考相应发行版的官方文档。

## WSL2 网络配置

为了获得更好的网络兼容性和更方便的 WSL 与宿主机之间的网络连接，建议将 WSL2 的网络模式配置为 **mirrored** 模式。

### 配置 mirrored 网络模式

现代 Windows 11 版本提供了图形化界面来配置 WSL2 设置：

1. 按 **Windows 键**，搜索 **"WSL Settings"**
2. 打开 **"WSL Settings"** 应用
3. 在设置界面中找到 **"网络模式"** (Networking Mode) 选项
4. 选择 **"镜像模式"** (Mirrored Mode)

**mirrored 网络模式的优势：**

- 使用 `localhost` 地址 `127.0.0.1` 即可从 Linux 内部连接到 Windows 服务器
- 改进了 VPN 的网络兼容性
- 支持 IPv6
- 支持多播
- 可以直接从局域网 (LAN) 连接到 WSL

**注意：**

- 此功能需要 Windows 11 22H2 或更高版本
- 配置完成后需要重启 WSL 服务：`wsl --shutdown`

现代 Windows 版本已自动处理防火墙配置，无需手动设置防火墙规则。

## 优化开发工作流

### Visual Studio Code 集成

对于 VS Code 用户，微软官方的 **"WSL" 扩展**是必装神器。它允许你的 VS Code 界面在 Windows 端运行，而所有后端操作（终端、调试、Git、语言服务）则无缝地在 WSL 的 Arch 环境中执行。

只需在 Arch 终端的项目目录中运行 `code .`，即可获得原生级的开发体验。

### 文件系统性能最佳实践

**核心原则**：始终将你的项目代码和开发文件存储在 **Arch Linux 的文件系统内部**（例如 `~/projects`），而不是挂载的 Windows 磁盘（`/mnt/c/...`）。前者提供了原生的 ext4 文件系统性能，而后者在进行大量 I/O 操作时（如 `npm install`）会非常缓慢。

### docker / podman

Windows 下的 docker-desktop 或者 podman-desktop，实际上也是一个 HyperV 虚拟机里装了个 Linux 然后再安装 docker 或者 podman，或者是用 WSL2 然后装了 docker 或者 podman，在此基础上再增加了一个方便操作管理的图形界面。

如果你已经在 ArchLinux 下习惯了命令行操作，那么完全可以照搬原有的操作，在 WSL2 里的 ArchLinux 安装 docker 或者 podman，继续使用即可。
