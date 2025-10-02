---
title: 哪吒面板自动上传GitHub仓库备份
published: 2025-06-16
description: 哪吒面板每天自动打包备份到GitHub脚本
#image: https://bd076fc.webp.li/2025/06/e71c2e9a623fa8988658bd52749f07a5.png
tags: [教程, VPS, GitHub]
category: VPS技术
draft: false
lang: zh_CN      # 仅当文章语言与 `config.ts` 中的网站语言不同时需要设置
---



**此脚本为 Nezha面板 每日自动备份到 GitHub 并通过 Telegram 通知**

操作环境：Debian12 VPS **nezha非docker安装**
> **目标**：每天凌晨 3:00（北京时间）自动
> 1. 打包 `/opt/nezha` 为 `.tar.gz`  
> 2. 上传到 GitHub 仓库  
> 3. 自动清理 7 天前的旧备份  
> 4. 通过 Telegram Bot 推送成功/失败通知 

## 第一步：准备工作

### 1. 安装所需软件

```bash
sudo apt update
sudo apt install git zip curl -y
```

### 2. 获取 GitHub Token 并新建仓库

1. 打开 [https://github.com/settings/tokens](https://github.com/settings/tokens)
2. 创建一个  **Classic Token**
3. 勾选权限：✅ `repo`
4. 复制 Token（如：`ghp_xxxxxxxxxxxxxxxxxxxxxxxx`）
5. 新建一个仓库，命名为 `nezha-backup`（建议私有）
6. 仓库地址形如：https://github.com/<你的GitHub用户名>/nezha-backup.git

### 3. 获取 Telegram Bot Token 和 Chat ID

#### 创建 Telegram Bot：

1. 搜索 `@BotFather`，发送 `/newbot`
2. 设置名称和用户名，获取 Bot Token

#### 获取 Chat ID：

1. 给你的 Bot 发一条消息
2. 访问：

```
https://api.telegram.org/bot<你的BotToken>/getUpdates
```

3. 找到 `"chat":{"id":xxxxx,...}`，这个 `id` 就是 Chat ID

---

## 第二步：创建备份脚本

### 1. 创建脚本文件

```bash
vim /root/nezha_backup.sh
```

### 2. 粘贴以下内容（⚠️ 替换标注内容）

#!/bin/bash
set -Eeuo pipefail

# ===== cron 环境修正 =====
export SHELL=/bin/bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
export HOME=/root
export LANG=C.UTF-8

### ====== 用户配置 START ======
GITHUB_USER="xxxxxxx"
GITHUB_REPO="nezha-backup"
GITHUB_TOKEN="ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
BOT_TOKEN="xxxxxxx:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
CHAT_ID="xxxxxxx"
BACKUP_DIR="/opt/nezha"
KEEP_DAYS=7
### ====== 用户配置 END ======

WORKDIR="/root/nezha-backup"
DATE=$(date +%F)
TARFILE="nezha-backup-$DATE.tar.gz"

# 绝对路径命令
GIT=$(command -v git)
CURL=$(command -v curl)
FIND=$(command -v find)
TAR=$(command -v tar)

send_telegram() {
    local msg="$1"
    $CURL -s -X POST "https://api.telegram.org/bot${BOT_TOKEN}/sendMessage" \
        -d "chat_id=${CHAT_ID}" \
        -d "parse_mode=Markdown" \
        -d "text=${msg}" >/dev/null || true
}

echo "[INFO] 初始化仓库..."
if [ ! -d "$WORKDIR/.git" ]; then
    rm -rf "$WORKDIR"
    mkdir -p "$WORKDIR"
    $GIT clone "https://${GITHUB_USER}:${GITHUB_TOKEN}@github.com/${GITHUB_USER}/${GITHUB_REPO}.git" "$WORKDIR" || {
        send_telegram "⚠️ *Nezha 备份失败*：无法克隆仓库"
        exit 1
    }
    cd "$WORKDIR" || exit 1
    $GIT config user.name "$GITHUB_USER"
    $GIT config user.email "${GITHUB_USER}@users.noreply.github.com"
    $GIT config --global --add safe.directory "$WORKDIR" || true

    # 检查仓库是否为空（忽略 .git 目录）
    if [ -z "$($FIND "$WORKDIR" -mindepth 1 -maxdepth 1 -not -name '.git' -print -quit)" ]; then
        echo "# Nezha Backup Repo" > README.md
        $GIT add README.md
        $GIT commit -m "init repo"
        $GIT branch -M main
        $GIT push -u origin main
        echo "[INFO] 已完成 GitHub 仓库初始化"
    fi
else
    cd "$WORKDIR" || exit 1
    $GIT config --global --add safe.directory "$WORKDIR" || true
    $GIT pull origin main >/dev/null 2>&1 || true
fi

# 先确保备份目录存在
if [ ! -d "$BACKUP_DIR" ]; then
    send_telegram "⚠️ *Nezha 备份失败*：备份目录不存在：$BACKUP_DIR"
    exit 1
fi

echo "[INFO] 打包 $BACKUP_DIR..."
ERRFILE="/tmp/nezha-tar-$DATE.err"
# 直接输出到 WORKDIR，避免 /tmp->mv
set +e
$TAR --warning=no-file-changed -czf "$WORKDIR/$TARFILE" -C "$BACKUP_DIR" . 2>"$ERRFILE"
tar_ec=$?
set -e

if [ $tar_ec -ne 0 ]; then
    if [ $tar_ec -eq 1 ]; then
        # 1 多为非致命警告（例如文件在读时被改动），忽略并继续
        echo "[WARN] tar 返回 1（警告），继续。最后几行："
        tail -n 5 "$ERRFILE" || true
    else
        # >=2 视为失败
        tailmsg=$(tail -n 10 "$ERRFILE" 2>/dev/null || true)
        send_telegram "⚠️ *Nezha 备份失败*：打包错误（exit $tar_ec）
$tailmsg"
        exit 1
    fi
fi

$GIT add .

# 删除超过 KEEP_DAYS 的旧备份
echo "[INFO] 删除超过 $KEEP_DAYS 天的旧备份..."
$FIND "$WORKDIR" -name "nezha-backup-*.tar.gz" -type f -mtime +$KEEP_DAYS -exec $GIT rm -f {} \; >/dev/null 2>&1 || true

# 提交并推送（只有变更时才提交）
if $GIT diff --cached --quiet; then
    echo "[INFO] 没有新的备份文件需要提交"
else
    $GIT commit -m "Backup on $DATE"
    $GIT push origin main || {
        send_telegram "⚠️ *Nezha 备份失败*：推送错误"
        exit 1
    }
    send_telegram "🎉 *Nezha 备份成功！* 已保存：$DATE，已自动清理超过 ${KEEP_DAYS} 天的旧备份"
    echo "[INFO] 备份成功"
fi

```

---

## 第三步：设置权限

```bash
chmod +x /root/nezha_backup.sh
```

---

## 第四步：添加定时任务（每天北京时间凌晨 3 点）

```bash
crontab -e
```

添加以下内容（北京时间凌晨 3 点 ）：

```cron
0 3 * * * env -i HOME=/root /bin/bash /root/nezha_backup.sh >/dev/null 2>&1
```

保存并退出（vim：:wq；nano：Ctrl+O → Ctrl+X）

---

## 第五步：手动运行测试

手动运行
```bash
bash /root/nezha_backup.sh
```

假装corn运行
```bash
env -i /bin/bash -c 'HOME=/root /bin/bash /root/nezha_backup.sh'
```

---


✔️ 检查点

GitHub 仓库出现 nezha-backup-YYYY-MM-DD.tar.gz

Telegram 收到「备份成功」通知
