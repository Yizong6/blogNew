---
title: 1秒卸载哪吒探针v1
published: 2025-04-13
description: 卸载哪吒探针v1
#image: https://bd076fc.webp.li/2025/04/409904fa8c6625b03ddd1264bc51c25e.png
tags: [教程, VPS]
category: VPS技术
draft: false
lang: zh_CN      # 仅当文章语言与 `config.ts` 中的网站语言不同时需要设置
---



要卸载 **哪吒探针V1** 的监控端(Agent)，请按照以下步骤操作：

:::important
请注意，以下命令适用于基于**Linux**的系统。对于其他操作系统，操作命令可能有所不同。
:::


## 一. 一键卸载(1s完成)

```bash
cd /opt/nezha/agent/ && ./nezha-agent service uninstall && cd /root && rm -rf /opt/nezha/agent/
```

如果安装了多个服务并想要全部卸载，可以使用 Agent 安装脚本的卸载功能：

```bash
./agent.sh uninstall
```



## 二. 手动卸载

### 1\. 停止 Agent 服务

首先，停止正在运行的 `nezha-agent` 服务。

```bash
sudo systemctl stop nezha-agent
```

### 2\. 禁用开机自启
防止服务在系统启动时自动运行。

```bash
sudo systemctl disable nezha-agent
```

### 3\. 删除服务文件

移除 `nezha-agent` 的服务配置文件。

```bash
sudo rm /etc/systemd/system/nezha-agent.service
```

### 4\. 删除 Agent 文件

删除安装目录中的 Agent 文件。

```bash
sudo rm -rf /opt/nezha/agent
```

### 5\. 重新加载 systemd 配置

确保系统服务管理器加载最新的配置。

```bash
sudo systemctl daemon-reload
```

### 6\. 检查残留进程

确认没有遗留的 `nezha-agent` 进程在运行。

```perl
ps -ef | grep nezha-agent
```

如果发现仍有相关进程运行，请使用 `kill` 命令终止。

### 7\. 删除日志文件（可选）

如果需要，您可以删除与 `nezha-agent` 相关的日志文件。

```bash
sudo rm -rf /var/log/nezha
```



## 三. 常用指令

停止agent指令
```bash
sudo systemctl stop nezha-agent
```

重启agent指令
```bash
sudo systemctl restart nezha-agent
```

查看agent状态
```bash
sudo systemctl status nezha-agent
```