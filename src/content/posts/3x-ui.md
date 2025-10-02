---
title: Linux系统手动安装3x-ui
published: 2025-10-02
description: 让linux无视系统手动安装3x-u
#image: https://bd076fc.webp.li/2025/04/27832734d6536dd1d761ace15fee2118.jpg
tags: [VPS, X-ui, 教程]
category: VPS技术
draft: false
lang: zh_CN      # 仅当文章语言与 `config.ts` 中的网站语言不同时需要设置
---


:::important
请注意，以下命令适用于基于**Linux**的系统。对于其他操作系统，操作命令可能有所不同。
<br />本教程适用Alpine、Debian、Ubuntu......
:::


> **操作环境**：Alpine3.20 LXC 1C128M2G/nat-ipv4
> **目标**：手动安装3x-ui


## 第一步：更新LXC环境

```Alpine
apk update
apk add curl wget unzip systemd
```


## 第二步：安装3x-ui

### 1. 下载3x-ui包，上传到LXC

下载地址：https://pan.yizong.de/d/Share/BlogAPP/xui.tar.gz?sign=QQj8xZ8KS5GbwJLl4mb3ZxUlju81OpFx8VxUiQQFy_g=:0
<br/>该3x-ui包为手动编译，不依赖 glibc
<br/>下载后通过ssh工具上传到/opt目录

### 2. 开始安装

#### 解压，并赋予可执行权限
```bash
mkdir -p /opt/3x-ui
mkdir -p /opt/3x-ui/bin
tar -xvzf /opt/xui.tar.gz -C /opt/3x-ui
cd /opt/3x-ui
chmod +x x-ui
```

#### 初始化
```bash
cat <<EOF > /opt/3x-ui/bin/config.json
{
  "log": { "loglevel": "info" },
  "inbounds": [
    {
      "port": 1080,
      "protocol": "socks",
      "settings": {
        "auth": "noauth",
        "udp": true,
        "ip": "127.0.0.1"
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom",
      "settings": {}
    }
  ]
}
EOF
```

#### 更改面板端口
```bash
cd /opt/3x-ui
./x-ui setting -port 30000
```

#### 开始运行3x-ui
```bash
cd /opt/3x-ui
./x-ui
```

#### 后台运行
```bash
nohup ./x-ui run > /var/log/x-ui.log 2>&1 &
```

:::tip
如果你的系统没有systemctl就不用执行下面这两步
:::

#### 开机自启
```bash
cat <<EOF > /etc/systemd/system/x-ui.service
[Unit]
Description=3-XUI Panel
After=network.target

[Service]
Type=simple
ExecStart=/opt/3x-ui/x-ui run
WorkingDirectory=/opt/3x-ui
Restart=always

[Install]
WantedBy=multi-user.target
EOF
```

#### 启用并启动服务
systemctl daemon-reexec
systemctl enable --now x-ui


## 第三步：访问并设置面板

浏览器访问：http://你的IP:30000
<br/>用户admin/密码admin
**请立即修改密码**

### 1.安装xray
点击面板系统状态-xray-后面的版本号，选择一个点击

### 2.更新geoip.dat
面板系统状态有个扳手，点进去只更新**geoip.dat**，**其余不要更新**
<br/>重启xray和面板即可正常使用