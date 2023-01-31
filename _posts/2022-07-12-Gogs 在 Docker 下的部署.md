---
layout: post
categories: [Git,Gogs]
description: none
keywords: Git
---
# Gogs简单的自助Git服务
Gogs 是一款极易搭建的自助 Git 服务。

## 简介
Gogs 的目标是打造一个最简单、最快速和最轻松的方式搭建自助 Git 服务。使用 Go 语言开发使得 Gogs 能够通过独立的二进制分发，并且支持 Go 语言支持的 所有平台，包括 Linux、Mac OS X、Windows 以及 ARM 平台。

## 特点

Gogs 是用 Go 语言开发的，最简单、最快速和最轻松的方式搭建自助 Git 服务。

- 易安装
除了可以根据操作系统平台通过 二进制运行，还可以通过 Docker 或 Vagrant，以及 包管理 安装。
- 跨平台
任何 Go 语言支持的平台都可以运行 Gogs，包括 Windows、Mac、Linux 以及 ARM。
- 轻量级
一个廉价的树莓派的配置足以满足 Gogs 的最低系统硬件要求。最大程度上节省服务器资源
- 开源化
所有的代码都开源在 GitHub 上并且支持多种数据库，例如 MySQL、MSSQL、SQLite3 等。

## Docker部署

使用Docker部署步骤：
- 先下载gogs的docker镜像
```text
docker pull gogs/gogs
```
- 创建gogs存储目录
```text
mkdir -p /var/gogs
```
- 创建容器,使用gogs镜像
```text
docker run -d --name=gogs -p 10022:22 -p 10080:3000 -v /var/gogs:/data gogs/gogs
```
- 直接使用使用 ip:10080,请求就可以执行图像界面安装





