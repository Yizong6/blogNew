---
title: 可执行文件部署SunPanel到debian12
published: 2025-06-28
description: 在Debian12上部署Sunpanel v1.3.0并使用Caddy反向代理。
#image: https://doc.sun-panel.top/images/icon-small-new.png
tags: [教程, VPS]
category: VPS技术
draft: false
lang: zh_CN      # 仅当文章语言与 `config.ts` 中的网站语言不同时需要设置
---


# 在Debian12上部署Sunpanel v1.3.0并使用Caddy反向代理

:::important
Sunpanel v1.3.0 为最后一个免捐赠版本，如需使用最新版请访问作者GitHub查看
:::

**项目位置**：https://github.com/hslr-s/sun-panel


# 第一步：系统环境准备

更新系统软件包并安装必要工具

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl tar vim
```



# 第二步：部署 Sunpanel v1.3.0

1. 准备工作目录
```bash
sudo mkdir -p /opt/sunpanel
cd /opt/sunpanel
sudo rm -rf ./*
```

2. 下载 Sunpanel v1.3.0 二进制文件
```bash
sudo curl -L -o sunpanel.tar.gz https://github.com/hslr-s/sun-panel/releases/download/v1.3.0/sun-panel_v1.3.0_linux_amd64.tar.gz
```

3. 解压缩二进制包
```bash
sudo tar -zxvf sunpanel.tar.gz 
sudo rm sunpanel.tar.gz
```

4. 确认目录名称
```bash
ls -l
# 应该能看到一个目录：sun-panel_v1.3.0_linux_amd64
```



# 第三步：配置 Systemd 服务

```bash
sudo vim /etc/systemd/system/sunpanel.service
```
粘贴以下内容：

```ini
[Unit]
Description=Sunpanel Service
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/opt/sunpanel/sun-panel_v1.3.0_linux_amd64
ExecStart=/opt/sunpanel/sun-panel_v1.3.0_linux_amd64/sun-panel
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```


# 第四步：启动服务并排除端口冲突（大概率不会出现此情况）

启动服务并设置开机自启
```bash
sudo systemctl daemon-reload
sudo systemctl start sunpanel
sudo systemctl enable sunpanel
```

查看运行状态
```bash
sudo systemctl status sunpanel
```

如果运行失败（复制问ChatGPT）：
```bash
# 查找是否端口占用
sudo ss -tulnp | grep 3002

# 终止冲突进程
sudo kill -9 <PID>

# 再次启动
sudo systemctl start sunpanel
sudo systemctl status sunpanel
```


# 第五步：安装 Caddy

```bash
# 安装 Caddy
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/caddy-stable-archive-keyring.gpg] https://dl.cloudsmith.io/public/caddy/stable/debian any-version main" | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install -y caddy
```



# 第六步：配置 Caddy 反向代理

```bash
sudo vim /etc/caddy/Caddyfile
```

清空原内容并填写如下配置（请将 sun.yourdomain.com 替换为你的域名）：
```bash
sun.yourdomain.com {
    reverse_proxy localhost:3002
}
```

重载 Caddy 配置
```bash
sudo systemctl reload caddy
```

# 🎉 最终访问
利用cloudflare、edgeone等部署域名
打开浏览器，访问你的域名，你应该可以看到 Sunpanel 的登录界面

建议定期备份database文件