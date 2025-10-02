---
title: VPS DD Windows10精简版
published: 2025-08-16
description: 让VPS装上win
#image: https://bd076fc.webp.li/2025/04/27832734d6536dd1d761ace15fee2118.jpg
tags: [VPS, 教程]
category: VPS技术
draft: false
lang: zh_CN      # 仅当文章语言与 `config.ts` 中的网站语言不同时需要设置
---


:::important
请注意，以下命令适用于基于**Linux**的系统。对于其他操作系统，操作命令可能有所不同。
:::


> **操作环境**：Debian12 KVM VPS 2C2G10G/ipv4
> **目标**：dd win10精简版


## 第一步：更新VPS环境

Ubuntu/Debian
```
apt update -y && apt install -y curl socat wget sudo
```


## 第二步：开始dd win

### 1. 利用[bin456789](https://github.com/bin456789/reinstall)大佬脚本

国外服务器：

```bash
curl -O https://raw.githubusercontent.com/bin456789/reinstall/main/reinstall.sh || wget -O reinstall.sh $_
```

国内服务器：

```bash
curl -O https://cnb.cool/bin456789/reinstall/-/git/raw/main/reinstall.sh || wget -O reinstall.sh $_
```

### 2. 选择[dd包](https://dd.wx.mk/)

功能 1: 安装 Linux(不需要直接看功能2)

用户名 `root` 默认密码 `123@@@`
```bash
bash reinstall.sh anolis      7|8|23
                  rocky       8|9|10
                  oracle      8|9
                  almalinux   8|9|10
                  opencloudos 8|9|23
                  centos      9|10
                  fedora      41|42
                  nixos       25.05
                  debian      9|10|11|12|13
                  opensuse    15.6|tumbleweed
                  alpine      3.19|3.20|3.21|3.22
                  openeuler   20.03|22.03|24.03|25.03
                  ubuntu      16.04|18.04|20.04|22.04|24.04|25.04 [--minimal]
                  kali
                  arch
                  gentoo
                  aosc
                  fnos
                  redhat      --img="http://access.cdn.redhat.com/xxx.qcow2"
```

功能 2: DD

- 支持 `raw` `vhd` 格式的镜像（未压缩，或者压缩成 `.gz` `.xz` `.zst` `.tar` `.tar.gz` `.tar.xz` `.tar.zst`）
- DD Windows 镜像时，会自动扩展系统盘，静态 IP 的机器会配置好 IP，可能首次开机几分钟后才生效
- DD Linux 镜像时，**不会**修改镜像的任何内容

镜像网站：https://dd.wx.mk/

```bash
bash reinstall.sh dd --img "https://example.com/xxx.xz"
```
我用的镜像是https://dd.wx.mk/cxthhhhh/Disk_Windows_10_x64_Lite_by_CXT_v1.0.vhd.gz

最后输入`reboot`开始dd

:::tip
可通过多种方式（SSH、HTTP 80 端口、商家后台 VNC、串行控制台）查看安装进度。
<br />即使安装过程出错，也能通过 SSH 运行 `/trans.sh alpine` 安装到 Alpine。
:::

## 最后：dd 成功

利用win远程连接输入用户名Administrator/密码cxthhhhh.com
记得一定要改密码