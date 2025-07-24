---
title: Cloudflare CDN 全端口回源设置
published: 2025-03-19
description: IPv6only使用VMess/Vless
#image: https://bd076fc.webp.li/2025/03/499d7dc607485ddc89ce239904fa80bd.png
tags: [教程, VPS, X-ui]
category: VPS技术
draft: false
lang: zh_CN      # 仅当文章语言与 `config.ts` 中的网站语言不同时需要设置
---


# 准备工作

1.  一个已经转入CloudFlare的，且是全功能的域名，千万不要使用双向解析的域名
2.  一台VPS(IPv6和IPv4都可以)
3.  [WebSSH](https://ssh.090227.xyz/) (让没有IPv6环境的小白朋友也可以连上VPS的SSH)

***

# 部署开始

## 1\. 通过WebSSH安装X-ui

打开 [WebSSH](https://ssh.090227.xyz/) ，`Hostname / 主机`填入你VPS的IP即可，IPv6无需修改IP格式直接填入即可，  
然后填入你VPS的`Username / 用户名`和`Password / 密码`后点击`Connect`连上你的VPS

连上机器后安装X-ui，这里就使用[甬哥的X-ui脚本](https://github.com/yonggekkk/x-ui-yg)

```
curl -Ls https://raw.githubusercontent.com/yonggekkk/x-ui-yg/main/install.sh
```

注意: 如果你的机器未安装BBR，请**优先安装BBR**后再展开后续的工作。

**X-ui面板端口**设为`2052`，CF支持HTTP回源端口如下：`80`、`8080`、`8880`、`2052`、`2082`、`2086`、`2095`，推荐使用除了`80`之外的端口

X-ui面板**用户名**和**密码**`随意即可`

**注意: **如果机器出口没有IPv4，可通过**安装WARP**来给机器**添加IPv4出口**

***

## 2\. 设置域名

打开 CloudFlare， 选中你要使用的域名，以下例子使用**google.com**域名举例；

### 2.1. 进入`SSL/TLS` > `概述` > `SSL/TLS 加密模式`为 `完全`

### 2.2. 进入`SSL/TLS` > `边缘证书` > **关闭** `始终使用 HTTPS` 和 `自动 HTTPS 重写`

### 2.3. 进入`SSL/TLS` > `源服务器` > **创建证书** 将`证书`和`密钥` **复制出来！保存好！！！**

### 2.4. 进入`DNS` > `记录` 添加2条 **AAAA记录**， `xui.google.com`和`vmess.google.com`，地址就填入你的小鸡IPv6即可 ，其中`xui`必须**开启小黄云**

**注意: **`vmess.google.com`将会作为你VMess节点的**伪装域名**使用

***

## 3\. 登录X-ui面板，设置VMess节点

打开 `http://xui.google.com:2052`， 填入你设置的X-ui面板**用户名**和**密码**后点击登录即可

### 3.1. 添加节点VMess-ws noTLS节点

**协议**: `vmess`  
**端口**: `随意` **复制下来备用**  
**uuid**: **复制下来备用**  
**传输协议: `ws`**  
**TLS**: `关闭`

**点击`添加`完成**

### 3.2. 添加节点VMess-ws-TLS节点

**协议**: `vmess`  
**端口**: `随意` **复制下来备用**  
**uuid**: 将**第3.1步**复制下来的`uuid`粘贴在此处  
**传输协议: `ws`**  
**TLS**: `开启`  
**证书方式**: `certificate file content`  
**公钥内容**: 填入**第2.3步**复制下来的`证书`内容  
**密钥内容**: 填入**第2.3步**复制下来的`密钥`内容

**点击`添加`完成**

***

## 4\. 设置CF的**Origin Rules**回源规则

打开 CloudFlare， 选中你要使用的域名

### 4.1. 进入`规则` > `Origin Rules` > `创建规则`

**规则名称**: `noTLS`

**字段**: `主机名` > `等于` > `vmess.google.com` > `And`  
**字段**: `SSL/HTTPS` > `关闭`

**目标端口**: `重写到` > 填入**第3.1步**复制下来的`noTLS端口`内容

**点击`部署`完成**

### 4.2. 进入`规则` > `Origin Rules` > `创建规则`

**规则名称**: `TLS`

**字段**: `主机名` > `等于` > `vmess.google.com` > `And`  
**字段**: `SSL/HTTPS` > `开启`

**目标端口**: `重写到` > 填入**第3.2步**复制下来的`TLS端口`内容

**点击`部署`完成**

***

## 5\. 填入你VMess节点信息，完成优选订阅

```
https://VMess.cmliussss.net/sub?cc=[VMess服务名字]&amp;host=[你的VMess域名]&amp;uuid=[你的UUID]&amp;path=[你的ws路径]&amp;alterid=[额外ID]&amp;security=[加密方式]
```

**host 必填项**，你VMess节点的`伪装域名`  
**uuid 必填项**，你VMess节点的`uuid`  
**path 非必填项**，你VMess节点的`路径`，默认`\`

**cc 非必填项**，可能会影响使用在线订阅转换,推荐使用地区代号，例如HK、SG、US  
**alterid 非必填项**，默认`0`  
**security 非必填项**，默认`auto`

**快速订阅方式**

```
https://VMess.cmliussss.net/sub?host=[你的VMess域名]&amp;uuid=[你的UUID]&amp;path=[你的ws路径]
```

例如: [https://VMess.cmliussss.net/sub?host=obdii.cfd&uuid=05641cf5-58d2-4ba4-a9f1-b3cda0b1fb1d&path=/linkws](https://vmess.cmliussss.net/sub?host=obdii.cfd&uuid=05641cf5-58d2-4ba4-a9f1-b3cda0b1fb1d&path=/linkws)

***

## 参考

[https://blog.cmliussss.com/p/CM19/](https://blog.cmliussss.com/p/CM19/)

***