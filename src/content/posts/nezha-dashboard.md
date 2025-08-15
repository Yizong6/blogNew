---
title: 搭建/卸载哪吒V1(CF-CDN)，适合纯V4/V6的VPS
published: 2025-04-13
description: 搭建/卸载哪吒面板监控VPS状态
#image: https://bd076fc.webp.li/2025/04/409904fa8c6625b03ddd1264bc51c25e.png
tags: [教程, VPS, Cloudflare]
category: VPS技术
draft: false
lang: zh_CN      # 仅当文章语言与 `config.ts` 中的网站语言不同时需要设置
---



本文内容整理至[Nodeseek论坛](https://www.nodeseek.com/)与**个人实践**

:::important
请注意，以下命令适用于基于**Linux**的系统。对于其他操作系统，操作命令可能有所不同。
:::



## 一. [Cloudflare](https://dash.cloudflare.com/)配置

1. 进入域名里的DNS-记录，添加记录，设置子域名前缀，ipv4添加A记录，ipv6添加AAAA记录，开启小黄云
2. 进入SSL/TLS-概述，将加密模式改成完全(严格)
3. 进入SSL/TLS-源服务器，以域名example.com举例，创建一个\*.example.com的证书，将源证书的代码和密钥的代码分别保存好
4. 进入网络，开启**Websockets**和**gRPC**

## 二. 安装哪吒面板

**纯v6小鸡必需要warp**创建ipv4出站才能下载代码

```bash
wget -N https://gitlab.com/fscarmen/warp/-/raw/main/menu.sh && bash menu.sh
```

执行安装代码，建议端口默认

```bash
curl -L https://raw.githubusercontent.com/nezhahq/scripts/refs/heads/main/install.sh -o nezha.sh && chmod +x nezha.sh && sudo ./nezha.sh
```

## 三. 安装Caddy

1. 执行安装脚本

   ```bash
   sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
   curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
   curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
   sudo apt update
   sudo apt install caddy
   ```

2. 创建并保存证书文件，此处的**example可以不更改**，以下两行代码分别执行，然后将保存的代码粘贴进去，并输入:wq保存退出
    
    ```bash
    vim /etc/caddy/example.pem    # 公钥
    vim /etc/caddy/example.key    # 私钥
    ```
    
3. 配置Caddyfile，执行以下代码打开配置文件
    
    ```bash
    vim /etc/caddy/Caddyfile
    ```
    
4. 输入%d清空，并粘贴如下代码，其中第一行的**nezha.example.com**需要替换成实际解析的子域名，证书路径如果第2步保存的文件名没改，这里也不用改，最后输入:wq保存退出
    
    ```bash
    nezha.example.com {
        reverse_proxy /proto.NezhaService/* h2c://127.0.0.1:8008
        tls /etc/caddy/example.pem /etc/caddy/example.key
        reverse_proxy /* 127.0.0.1:8008
    }
    ```
    
5. 启用并启动Caddy:
    
    ```bash
     systemctl enable caddy
     systemctl start caddy
    ```
    
6. 如果先安装的Caddy后安装的面板，这里需要重启下Caddy:
    
    ```bash
    sudo systemctl reload caddy
    systemctl restart caddy
    sudo systemctl status caddy
    ```
    

## 四. 面板设置与添加探针

1.  通过**nezha.example.com/dashboard**进入后台，默认账号密码都是admin，登录后添加新用户和密码，退出登录并用新账号密码登录，再进入设置删掉admin账号
    
2.  将**nezha.example.com**添加在仪表板服务器域名/IP(无 CDN)中并保存
    
3.  在其他小鸡新增探针时，粘贴的代码里需要修改如下位置，也可以在面板后台设置
    
    ```bash
    修改 NZ_SERVER=nezha.example.com:443
    开启 NZ_TLS=true
    ```


## 五. 面板卸载（仅适用**Systemd安装**）

### 1. 使用安装代码卸载

```bash
curl -L https://raw.githubusercontent.com/nezhahq/scripts/refs/heads/main/install.sh -o nezha.sh && chmod +x nezha.sh && sudo ./nezha.sh
```


### 2. 手动卸载

1. 停止服务：

    ```bash
    systemctl stop nezha-dashboard
    ```

2. 禁用开机自启：

    ```bash
    systemctl disable nezha-dashboard
    ```

3. 删除服务文件：

    ```bash
    rm /etc/systemd/system/nezha-dashboard.service
    ```

4. 重新加载 Systemd 配置：

    ```bash
    systemctl daemon-reload
    ```

5. 删除程序文件：
如果你的面板安装在 /opt/nezha/dashboard：

    ```bash
    rm -rf /opt/nezha/dashboard
    ```

6. 删除日志文件（如果有）：
如果日志保存在 /var/log/nezha，可以删除：

    ```bash
    rm -rf /var/log/nezha
    ```



## 六. 常用指令

停止panel指令
```bash
sudo systemctl stop nezha-dashboard
```

重启panel指令
```bash
sudo systemctl restart nezha-dashboard
```

查看panel状态
```bash
sudo systemctl status nezha-dashboard
```