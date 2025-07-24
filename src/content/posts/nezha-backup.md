---
title: 哪吒面板自动打包并上传GitHub仓库备份
published: 2025-06-16
description: 哪吒面板备份到GitHub脚本
#image: https://bd076fc.webp.li/2025/06/e71c2e9a623fa8988658bd52749f07a5.png
tags: [教程, VPS, GitHub]
category: VPS技术
draft: false
lang: zh_CN      # 仅当文章语言与 `config.ts` 中的网站语言不同时需要设置
---



**此脚本为 Nezha面板 每日自动备份到 GitHub 并通过 Telegram 通知**

操作环境：Debian11 VPS **nezha非docker安装**
> **目标**：每天早上 6:00（北京时间）自动  
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

```bash
#!/bin/bash

# === 配置项 ===
BACKUP_DIR="/opt/nezha"
GIT_REPO_DIR="/root/nezha-backup"
GIT_REMOTE="origin"
GIT_BRANCH="main"
GITHUB_REPO="github.com/你的用户名/nezha-backup.git"       # ⚠️ 替换为你的仓库地址
GITHUB_TOKEN="ghp_xxx你的GitHubToken"                       # ⚠️ 替换为你的 GitHub Token
MAX_DAYS=7

# === Telegram 配置 ===
TG_BOT_TOKEN="123456789:ABC-你的BotToken"                   # ⚠️ 替换为你的 Bot Token
TG_CHAT_ID="你的ChatID"                                     # ⚠️ 替换为你的 Chat ID

# === 发送 Telegram 通知函数 ===
send_telegram() {
  curl -s -X POST "https://api.telegram.org/bot$TG_BOT_TOKEN/sendMessage" \
    -d chat_id="$TG_CHAT_ID" \
    -d text="$1" \
    -d parse_mode="Markdown"
}

# 时间戳与归档名
TODAY=$(date +"%Y-%m-%d")
ARCHIVE_NAME="nezha-backup-$TODAY.tar.gz"

# 切换到仓库目录
mkdir -p "$GIT_REPO_DIR"
cd "$GIT_REPO_DIR" || exit 1

# 第一次初始化 Git 仓库
if [ ! -d ".git" ]; then
  git clone "https://$GITHUB_TOKEN@$GITHUB_REPO" .
fi

# 清理本地超期归档
find . -name "nezha-backup-*.tar.gz" -type f -mtime +$MAX_DAYS -exec rm -f {} \;

# 同步远程
git pull "$GIT_REMOTE" "$GIT_BRANCH"

# 清理 Git 超期归档记录
git ls-files | grep 'nezha-backup-.*\.tar\.gz' | while read -r file; do
  FILE_DATE=$(echo "$file" | grep -oE '[0-9]{4}-[0-9]{2}-[0-9]{2}')
  if [[ $(date -d "$FILE_DATE" +%s) -lt $(date -d "$MAX_DAYS days ago" +%s) ]]; then
    git rm -f "$file"
  fi
done

# 创建新归档
tar -czf "$ARCHIVE_NAME" "$BACKUP_DIR"

# 提交并推送
git add "$ARCHIVE_NAME"
git commit -m "Backup on $TODAY"
if git push "$GIT_REMOTE" "$GIT_BRANCH"; then
  send_telegram "🎉 *Nezha 备份成功！"
else
  send_telegram "⚠️ *Nezha 备份失败！*"
fi

```

---

## 第三步：设置权限

```bash
chmod +x /root/nezha_backup.sh
```

---

## 第四步：Git 用户信息 & 首次推送

配置 Git 全局身份（仅需一次）：

```bash
git config --global user.name "YourName"
git config --global user.email "you@example.com"
```

初次在仓库目录执行：

```bash
cd /root/nezha-backup
git branch -M main
git push -u origin main
```

---

## 第五步：添加定时任务（每天北京时间早上 6 点）

```bash
crontab -e
```

添加以下内容（北京时间早上 6 点 = UTC 22 点）：

```cron
0 22 * * * /bin/bash /root/nezha_backup.sh >/dev/null 2>&1
```

保存并退出（vim：:wq；nano：Ctrl+O → Ctrl+X）

---

## 第六步：手动测试

```bash
bash /root/nezha_backup.sh
```

✔️ 检查点
GitHub 仓库出现 nezha-backup-YYYY-MM-DD.tar.gz
Telegram 收到「备份成功」通知
