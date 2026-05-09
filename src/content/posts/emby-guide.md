---
title: 从零部署 Emby 媒体服与用户管理
published: 2026-05-09
description: "基于 Debian 12 的 Emby 媒体服务器完整搭建笔记：Docker、rclone、Google Drive/OpenList、Caddy、Cloudflare、Telegram 上传机器人、EmbyPulse 与 EmbyTGBot。"
# image: "./cover.jpeg"
tags: ["Emby", "Docker", "rclone", "Telegram", "Cloudflare", "Media Server"]
category: HomeLab
draft: false
---

> 本文整理自一次完整的 Emby 搭建过程，覆盖 NAT VPS 与独立服务器两种场景。涉及到的 Token、API Key、域名、Telegram ID 请全部替换成你自己的值，不要直接复制示例里的占位符。

## 0. 最终目标

我们要搭建的是一套完整的个人媒体系统：

```text
Google Drive / OpenList / OneDrive
        ↓ rclone 挂载
/mnt/gdrive 或 /mnt/openlist
        ↓
Emby 读取媒体库
        ↓
Caddy / Cloudflare 提供访问线路
        ↓
客户端播放、网页管理、用户自助注册、统计面板
```

同时还会额外搭建一个 Telegram 上传机器人：

```text
你把资源发给 Telegram 机器人
        ↓
服务器自动接收文件
        ↓
rclone 上传到网盘指定目录
        ↓
你整理命名后 Emby 扫库
```

可选管理系统有两个方向：

| 方案 | 用途 | 适合人群 |
|---|---|---|
| EmbyPulse | Web 面板、统计、用户中心、播放排行、缺集等 | 想要图形化管理面板 |
| EmbyTGBot | Telegram 管理用户、注册码、续期、查询在线人数 | 想通过 TG 管理用户 |

:::warning
不要把机器人 Token、Emby API Key、Google OAuth Secret、rclone.conf 公开到博客或 GitHub。
:::

---

## 1. 服务器场景与端口规划

本文覆盖两种服务器。

### 1.1 NAT VPS 场景

假设只有少量端口，比如：

```text
服务器 IP：176.123.6.17
可用端口：48111-48119
```

推荐规划：

```text
48111  Emby 反代线路
48112  EmbyPulse 管理后台
48113  EmbyPulse 用户中心
48114  Emby 直连播放线路
48119  SSH，若服务商已经分配给 SSH 就不要占用
```

NAT VPS 的重点是：**服务商面板必须把外部端口映射到 VPS 内部端口**。如果外部访问超时，第一优先排查端口映射。

### 1.2 独立服务器场景

假设服务器有独立公网 IP：

```text
服务器 IP：147.224.40.215
SSH 端口：22700
```

已有占用端口示例：

```text
80/tcp      caddy
443/tcp     caddy
22700/tcp   sshd
8001/tcp    sing-box
50501/tcp   sing-box
50503/udp   sing-box
50504/udp   sing-box
40823/udp   sing-box
5353/udp    avahi-daemon / openclaw-gateway
```

新增端口建议：

```text
8096/tcp    Emby 直连播放线路
18080/tcp   可选 Web API，仅监听 127.0.0.1
18081/tcp   可选 Web 前端，仅监听 127.0.0.1
18082/tcp   本地 Telegram Bot API，仅监听 127.0.0.1
```

独立服务器推荐线路：

```text
https://emby.example.com     Cloudflare 橙云线路，用于网页管理
https://play.example.com     DNS only 灰云线路，用于直连播放
http://147.224.40.215:8096   IP 直连线路，用于调试或客户端播放
```

:::important
视频播放不建议长期走 Cloudflare 橙云线路。Cloudflare 线路适合网页访问、管理、登录；播放大文件建议走灰云直连域名或 IP:8096。
:::

---

## 2. 系统初始化

以下命令以 Debian 12 为例。

```bash
ssh root@你的服务器IP -p 你的SSH端口
```

更新系统并安装基础组件：

```bash
apt update && apt upgrade -y
apt install -y ca-certificates curl gnupg lsb-release nano vim unzip fuse3 jq socat openssl git
hostnamectl set-timezone Asia/Shanghai
```

检查端口占用：

```bash
ss -lntup
```

---

## 3. 安装 Docker 和 Docker Compose

```bash
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do
  apt remove -y "$pkg" || true
done

install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc

cat > /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

systemctl enable --now docker

docker version
docker compose version
docker run --rm hello-world
```

创建 Docker 网络：

```bash
docker network create media-net 2>/dev/null || true
```

---

## 4. 配置 rclone 与 Google Drive

### 4.1 安装 rclone

```bash
curl https://rclone.org/install.sh | bash
rclone version
```

允许 FUSE 挂载被 Docker 容器读取：

```bash
grep -qxF 'user_allow_other' /etc/fuse.conf || echo 'user_allow_other' >> /etc/fuse.conf
```

创建目录：

```bash
mkdir -p /mnt/gdrive
mkdir -p /var/cache/rclone-gdrive
mkdir -p /root/.config/rclone
chmod 700 /root/.config/rclone
```

### 4.2 给 rclone 配自己的 Google Drive Client ID

不建议长期使用 rclone 默认共享 client_id，容易遇到 Google Drive API 限流：

```text
googleapi: Error 403: Quota exceeded
RATE_LIMIT_EXCEEDED
```

去 Google Cloud Console 创建项目：

```text
1. 创建项目，例如 rclone-emby-upload
2. APIs & Services -> Library
3. 启用 Google Drive API
4. Google Auth Platform / OAuth consent screen
5. Data Access 添加 scope:
   https://www.googleapis.com/auth/drive
6. Audience 添加自己的 Google 账号为测试用户，或发布到 In production
7. Clients -> Create OAuth client
8. Application type 选择 Desktop app
9. 复制 Client ID 和 Client Secret
```

然后在服务器配置：

```bash
rclone config
```

按提示填写：

```text
n) New remote
name> gdrive
Storage> drive
client_id> 你的 Client ID
client_secret> 你的 Client Secret
scope> drive
root_folder_id> 直接回车
service_account_file> 直接回车
Edit advanced config? n
Use web browser to automatically authenticate rclone with remote? n
```

如果服务器没有浏览器，rclone 会要求你在本地电脑执行：

```powershell
rclone authorize "drive" "一大串配置"
```

如果本地网络访问 Google 有问题，PowerShell 先设置代理：

```powershell
$env:HTTP_PROXY="http://127.0.0.1:7890"
$env:HTTPS_PROXY="http://127.0.0.1:7890"
```

授权成功后，把 JSON token 粘贴回服务器。

测试：

```bash
rclone about gdrive:
rclone lsd gdrive:
```

创建媒体目录：

```bash
rclone mkdir "gdrive:媒体/电影"
rclone mkdir "gdrive:媒体/电视剧"
rclone mkdir "gdrive:媒体/动漫"
rclone mkdir "gdrive:TelegramUploads/待整理"
```

### 4.3 挂载 Google Drive

创建 systemd 服务：

```bash
cat > /etc/systemd/system/rclone-gdrive.service <<'EOF'
[Unit]
Description=Rclone mount Google Drive for Emby
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
ExecStartPre=/bin/mkdir -p /mnt/gdrive /var/cache/rclone-gdrive
ExecStart=/usr/bin/rclone mount gdrive: /mnt/gdrive \
  --config=/root/.config/rclone/rclone.conf \
  --allow-other \
  --read-only \
  --uid=1000 \
  --gid=1000 \
  --umask=002 \
  --dir-cache-time=72h \
  --poll-interval=15s \
  --vfs-cache-mode=full \
  --vfs-cache-max-size=50G \
  --vfs-cache-max-age=6h \
  --buffer-size=64M \
  --cache-dir=/var/cache/rclone-gdrive
ExecStop=/usr/bin/fusermount3 -uz /mnt/gdrive
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF
```

启动：

```bash
systemctl daemon-reload
systemctl enable --now rclone-gdrive
systemctl status rclone-gdrive --no-pager
ls -la /mnt/gdrive
```

---

## 5. 可选：通过 OpenList WebDAV 挂载网盘

如果你已经搭好了 OpenList，可以通过 WebDAV 挂到本地。

OpenList WebDAV 地址格式：

```text
https://openlist.example.com/dav/
```

配置 rclone：

```bash
rclone config
```

示例：

```text
n) New remote
name> openlist
Storage> webdav
url> https://openlist.example.com/dav/
vendor> other
user> OpenList 用户名
password> OpenList 密码
bearer_token> 直接回车
```

测试：

```bash
rclone lsd openlist:
```

创建挂载：

```bash
mkdir -p /mnt/openlist
mkdir -p /var/cache/rclone-openlist
```

创建 systemd：

```bash
cat > /etc/systemd/system/rclone-openlist.service <<'EOF'
[Unit]
Description=Rclone mount OpenList WebDAV for Emby
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
ExecStartPre=/bin/mkdir -p /mnt/openlist /var/cache/rclone-openlist
ExecStart=/usr/bin/rclone mount openlist: /mnt/openlist \
  --config=/root/.config/rclone/rclone.conf \
  --allow-other \
  --read-only \
  --uid=1000 \
  --gid=1000 \
  --umask=002 \
  --dir-cache-time=72h \
  --poll-interval=30s \
  --vfs-cache-mode=full \
  --vfs-cache-max-size=20G \
  --vfs-cache-max-age=6h \
  --buffer-size=32M \
  --cache-dir=/var/cache/rclone-openlist
ExecStop=/usr/bin/fusermount3 -uz /mnt/openlist
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF
```

启动：

```bash
systemctl daemon-reload
systemctl enable --now rclone-openlist
ls -la /mnt/openlist
```

---

## 6. 部署 Emby

### 6.1 判断服务器架构

```bash
uname -m
docker info | grep -i architecture
```

如果是 `x86_64` / `amd64`，用：

```text
emby/embyserver:latest
```

如果是 `aarch64` / `arm64`，用：

```text
emby/embyserver_arm64v8:latest
```

:::warning
ARM64 服务器如果误用 amd64 镜像，会出现 `exec /init: exec format error`。
:::

### 6.2 创建 Compose

```bash
mkdir -p /opt/media-stack/emby
mkdir -p /opt/media/emby/config
mkdir -p /opt/media/emby/transcode
chown -R 1000:1000 /opt/media/emby
```

amd64 示例：

```bash
cat > /opt/media-stack/emby/docker-compose.yml <<'EOF'
services:
  emby:
    image: emby/embyserver:latest
    container_name: embyserver
    restart: unless-stopped
    ports:
      - "8096:8096"
    environment:
      UID: "1000"
      GID: "1000"
      GIDLIST: "1000"
      TZ: Asia/Shanghai
    volumes:
      - /opt/media/emby/config:/config
      - /opt/media/emby/transcode:/transcode
      - type: bind
        source: /mnt/gdrive
        target: /mnt/gdrive
        read_only: true
        bind:
          propagation: rslave
    networks:
      - media-net

networks:
  media-net:
    external: true
EOF
```

ARM64 改成：

```yaml
image: emby/embyserver_arm64v8:latest
platform: linux/arm64/v8
```

启动：

```bash
cd /opt/media-stack/emby
docker compose pull
docker compose up -d
docker logs --tail=100 embyserver
curl -I http://127.0.0.1:8096
```

让 Docker 尽量等 rclone 挂载后启动：

```bash
mkdir -p /etc/systemd/system/docker.service.d

cat > /etc/systemd/system/docker.service.d/override.conf <<'EOF'
[Unit]
Wants=rclone-gdrive.service
After=rclone-gdrive.service
EOF

systemctl daemon-reload
```

---

## 7. Caddy、Cloudflare 与播放线路

### 7.1 独立服务器推荐配置

Cloudflare DNS：

```text
A  emby  服务器IP  Proxied / 橙云
A  play  服务器IP  DNS only / 灰云
```

Caddy 配置：

```bash
mkdir -p /etc/caddy/conf.d
cp /etc/caddy/Caddyfile /etc/caddy/Caddyfile.bak.$(date +%F-%H%M%S)

grep -q 'import /etc/caddy/conf.d/\*.caddy' /etc/caddy/Caddyfile || \
  printf '\nimport /etc/caddy/conf.d/*.caddy\n' >> /etc/caddy/Caddyfile

cat > /etc/caddy/conf.d/emby.caddy <<'EOF'
emby.example.com {
  encode zstd gzip
  reverse_proxy 127.0.0.1:8096
}

play.example.com {
  encode zstd gzip
  reverse_proxy 127.0.0.1:8096
}
EOF

caddy fmt --overwrite /etc/caddy/Caddyfile /etc/caddy/conf.d/emby.caddy
caddy validate --config /etc/caddy/Caddyfile
systemctl reload caddy
```

最终入口：

```text
Cloudflare 代理线路：
https://emby.example.com

直连 HTTPS 线路：
https://play.example.com

IP 直连线路：
http://服务器IP:8096
```

### 7.2 NAT VPS 推荐配置

如果只有高端口：

```text
48111 -> Emby 反代线路
48114 -> Emby 直连播放线路
```

Caddyfile 示例：

```caddyfile
{
  auto_https off
}

:48111 {
  encode gzip
  reverse_proxy 127.0.0.1:8096
}
```

直连播放线路可以用服务商面板直接配置：

```text
外部 48114 -> 内部 8096
```

如果服务商只能同端口映射，则用 socat：

```bash
apt install -y socat

cat > /etc/systemd/system/emby-direct-48114.service <<'EOF'
[Unit]
Description=Direct TCP forward 48114 to Emby 8096
After=network-online.target docker.service
Wants=network-online.target

[Service]
ExecStart=/usr/bin/socat TCP-LISTEN:48114,fork,reuseaddr TCP:127.0.0.1:8096
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now emby-direct-48114
```

访问：

```text
http://服务器IP:48111  Emby 反代线路
http://服务器IP:48114  Emby 直连播放线路
```

---

## 8. 初始化 Emby 与媒体库

浏览器打开：

```text
http://服务器IP:8096
```

或：

```text
https://play.example.com
```

首次向导：

```text
语言：中文
创建管理员账号
允许远程访问：开启
```

添加媒体库：

```text
电影库：
内容类型：电影
路径：/mnt/gdrive/媒体/电影

电视剧库：
内容类型：电视节目
路径：/mnt/gdrive/媒体/电视剧

动漫库：
内容类型：电视节目
路径：/mnt/gdrive/媒体/动漫
```

---

## 9. 媒体库整理规范

Emby 主要根据媒体库类型、文件夹名、文件名和季集编号识别，不会真正理解视频内容。

### 9.1 电影

```text
媒体/
  电影/
    流浪地球 (2019)/
      流浪地球 (2019).mkv

    Avatar (2009)/
      Avatar (2009).mkv
```

### 9.2 电视剧

```text
媒体/
  电视剧/
    庆余年 (2019)/
      Season 01/
        庆余年 S01E01.mkv
        庆余年 S01E02.mkv
      Season 02/
        庆余年 S02E01.mkv
```

### 9.3 动漫

```text
媒体/
  动漫/
    葬送的芙莉莲 (2023)/
      Season 01/
        葬送的芙莉莲 S01E01.mkv
        葬送的芙莉莲 S01E02.mkv
      Specials/
        葬送的芙莉莲 S00E01.mkv
```

### 9.4 不推荐

```text
电视剧/
  第01集.mkv
  第02集.mkv
  01.mp4
```

或把所有电影、剧集、动漫混在一个媒体库里。

:::tip
Telegram 上传机器人建议只上传到 `gdrive:TelegramUploads/待整理`，不要直接传到正式媒体库。确认片名、年份、季集后，再移动到正式目录。
:::

---

## 10. Telegram 上传网盘机器人

这个机器人负责：

```text
接收 Telegram 文件
自动排队
上传到当前设置的网盘目录
支持 /setdir 切换上传目录
支持 /cancel 中止当前任务
支持 /queue 查看等待队列
支持每 10 秒刷新上传进度
```

### 10.1 准备信息

需要：

```text
BOT_TOKEN
TELEGRAM_API_ID
TELEGRAM_API_HASH
你的 Telegram 数字 ID
```

`BOT_TOKEN` 从 `@BotFather` 创建。  
`TELEGRAM_API_ID` 和 `TELEGRAM_API_HASH` 从 `https://my.telegram.org` 创建 App 获取。

### 10.2 创建目录与配置

```bash
mkdir -p /opt/tg-bot-uploader/botapi-data
mkdir -p /opt/tg-bot-uploader/downloads
cd /opt/tg-bot-uploader
```

创建 `.env`：

```bash
cat > /opt/tg-bot-uploader/.env <<'EOF'
BOT_TOKEN=这里填你的BOT_TOKEN
TELEGRAM_API_ID=这里填你的API_ID
TELEGRAM_API_HASH=这里填你的API_HASH

ALLOWED_USER_IDS=

DEST_REMOTE=gdrive:TelegramUploads/待整理
ALLOWED_DEST_PREFIXES=gdrive:

BOT_API_URL=http://127.0.0.1:18082
FILE_API_URL=http://127.0.0.1:18082/file

LOCAL_BOT_API_PATH_PREFIX=/var/lib/telegram-bot-api
HOST_BOT_API_PATH_PREFIX=/opt/tg-bot-uploader/botapi-data

DOWNLOAD_DIR=/opt/tg-bot-uploader/downloads
STATE_FILE=/opt/tg-bot-uploader/state.json
PROGRESS_INTERVAL=10
SKIP_OLD_UPDATES_ON_START=true
EOF

chmod 600 /opt/tg-bot-uploader/.env
vim /opt/tg-bot-uploader/.env
```

### 10.3 部署本地 Telegram Bot API Server

```bash
cat > /opt/tg-bot-uploader/docker-compose.yml <<'EOF'
services:
  telegram-bot-api:
    image: aiogram/telegram-bot-api:latest
    container_name: telegram-bot-api
    restart: unless-stopped
    ports:
      - "127.0.0.1:18082:8081"
    env_file:
      - /opt/tg-bot-uploader/.env
    environment:
      TELEGRAM_LOCAL: "1"
    volumes:
      - /opt/tg-bot-uploader/botapi-data:/var/lib/telegram-bot-api
EOF
```

启动：

```bash
cd /opt/tg-bot-uploader
docker compose pull
docker compose up -d
docker logs --tail=100 telegram-bot-api
```

测试：

```bash
source /opt/tg-bot-uploader/.env
curl "http://127.0.0.1:18082/bot${BOT_TOKEN}/getMe"
```

返回 `"ok":true` 即成功。

给机器人发 `/start`，然后获取你的 Telegram ID：

```bash
curl -s "http://127.0.0.1:18082/bot${BOT_TOKEN}/getUpdates" | jq
```

把 `from.id` 写进 `.env`：

```env
ALLOWED_USER_IDS=123456789
```

### 10.4 上传机器人脚本

```bash
cat > /opt/tg-bot-uploader/tg_uploader.py <<'PY'
#!/usr/bin/env python3
import os
import re
import time
import json
import shutil
import select
import threading
import subprocess
from pathlib import Path
from collections import deque
from urllib.parse import quote
from urllib.request import urlopen, Request

BOT_TOKEN = os.environ["BOT_TOKEN"]
BOT_API_URL = os.environ.get("BOT_API_URL", "http://127.0.0.1:18082").rstrip("/")
FILE_API_URL = os.environ.get("FILE_API_URL", "http://127.0.0.1:18082/file").rstrip("/")
DEFAULT_DEST_REMOTE = os.environ.get("DEST_REMOTE", "gdrive:TelegramUploads/待整理").rstrip("/")
STATE_FILE = Path(os.environ.get("STATE_FILE", "/opt/tg-bot-uploader/state.json"))
DOWNLOAD_DIR = Path(os.environ.get("DOWNLOAD_DIR", "/opt/tg-bot-uploader/downloads"))
PROGRESS_INTERVAL = int(os.environ.get("PROGRESS_INTERVAL", "10"))
SKIP_OLD = os.environ.get("SKIP_OLD_UPDATES_ON_START", "true").lower() in ["1", "true", "yes", "y"]

ALLOWED_DEST_PREFIXES = [x.strip() for x in os.environ.get("ALLOWED_DEST_PREFIXES", "gdrive:").split(",") if x.strip()]
ALLOWED_USER_IDS = {int(x.strip()) for x in os.environ.get("ALLOWED_USER_IDS", "").split(",") if x.strip()}

LOCAL_PREFIX = os.environ.get("LOCAL_BOT_API_PATH_PREFIX", "")
HOST_PREFIX = os.environ.get("HOST_BOT_API_PATH_PREFIX", "")

DOWNLOAD_DIR.mkdir(parents=True, exist_ok=True)
STATE_FILE.parent.mkdir(parents=True, exist_ok=True)

state_lock = threading.Lock()
queue_cond = threading.Condition()
job_queue = deque()
current_job = None
job_seq = 0

class Cancelled(Exception):
    pass

def load_state_unlocked():
    if not STATE_FILE.exists():
        return {"users": {}, "last_update_id": None}
    try:
        data = json.loads(STATE_FILE.read_text("utf-8"))
    except Exception:
        return {"users": {}, "last_update_id": None}
    data.setdefault("users", {})
    data.setdefault("last_update_id", None)
    return data

def save_state_unlocked(data):
    tmp = STATE_FILE.with_suffix(".tmp")
    tmp.write_text(json.dumps(data, ensure_ascii=False, indent=2), "utf-8")
    tmp.replace(STATE_FILE)

def get_user_dest(user_id):
    with state_lock:
        state = load_state_unlocked()
        return state["users"].get(str(user_id), {}).get("dest", DEFAULT_DEST_REMOTE)

def set_user_dest(user_id, dest):
    dest = dest.strip().rstrip("/")
    if not dest:
        raise ValueError("目录不能为空")
    if not any(dest.startswith(prefix) for prefix in ALLOWED_DEST_PREFIXES):
        raise ValueError(f"不允许的目录。允许前缀：{', '.join(ALLOWED_DEST_PREFIXES)}")
    with state_lock:
        state = load_state_unlocked()
        state["users"].setdefault(str(user_id), {})
        state["users"][str(user_id)]["dest"] = dest
        save_state_unlocked(state)

def get_last_update_id():
    with state_lock:
        return load_state_unlocked().get("last_update_id")

def set_last_update_id(update_id):
    with state_lock:
        state = load_state_unlocked()
        state["last_update_id"] = update_id
        save_state_unlocked(state)

def api(method, data=None, timeout=300):
    url = f"{BOT_API_URL}/bot{BOT_TOKEN}/{method}"
    if data is None:
        req = Request(url)
    else:
        req = Request(url, data=json.dumps(data).encode("utf-8"), headers={"Content-Type": "application/json"})
    with urlopen(req, timeout=timeout) as r:
        return json.loads(r.read().decode("utf-8"))

def send(chat_id, text):
    try:
        res = api("sendMessage", {"chat_id": chat_id, "text": text})
        if res.get("ok"):
            return res["result"]["message_id"]
    except Exception as e:
        print("sendMessage error:", repr(e), flush=True)
    return None

def edit(chat_id, message_id, text):
    if not message_id:
        return
    try:
        api("editMessageText", {"chat_id": chat_id, "message_id": message_id, "text": text})
    except Exception as e:
        print("editMessageText error:", repr(e), flush=True)

def human_size(num):
    try:
        num = float(num)
    except Exception:
        return "未知"
    units = ["B", "KB", "MB", "GB", "TB"]
    for unit in units:
        if abs(num) < 1024:
            return f"{num:.2f} {unit}"
        num /= 1024
    return f"{num:.2f} PB"

def safe_name(name):
    return name.replace("/", "_").replace("\\", "_").strip() or f"telegram_file_{int(time.time())}"

def rclone_join(remote_dir, file_name):
    remote_dir = remote_dir.strip().rstrip("/")
    return remote_dir + file_name if remote_dir.endswith(":") else remote_dir + "/" + file_name

def ensure_remote_dir(remote_dir):
    subprocess.run(["rclone", "mkdir", remote_dir], check=True, stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)

def get_media(message):
    for key in ["document", "video", "audio"]:
        if key in message:
            obj = message[key]
            file_id = obj["file_id"]
            file_name = obj.get("file_name") or f"{key}_{message['message_id']}" + (".mp4" if key == "video" else "")
            size = obj.get("file_size", 0)
            return file_id, safe_name(file_name), size
    if "photo" in message:
        obj = message["photo"][-1]
        return obj["file_id"], f"photo_{message['message_id']}.jpg", obj.get("file_size", 0)
    return None, None, None

def get_file_path(file_id):
    info = api("getFile", {"file_id": file_id})
    if not info.get("ok"):
        raise RuntimeError(info)
    return info["result"]["file_path"]

def resolve_local_path(file_path):
    paths = []
    if file_path.startswith("/"):
        paths.append(file_path)
        if LOCAL_PREFIX and HOST_PREFIX and file_path.startswith(LOCAL_PREFIX):
            paths.append(file_path.replace(LOCAL_PREFIX, HOST_PREFIX, 1))
    for p in paths:
        if os.path.exists(p):
            return Path(p)
    return None

def download_via_http(file_path, file_name, job):
    url = f"{FILE_API_URL}/bot{BOT_TOKEN}/{quote(file_path)}"
    local_path = DOWNLOAD_DIR / file_name
    with urlopen(url, timeout=3600) as r, open(local_path, "wb") as f:
        total = r.headers.get("Content-Length")
        total = int(total) if total and total.isdigit() else 0
        done = 0
        last_update = 0
        while True:
            if job["cancel_event"].is_set():
                raise Cancelled("已中止当前任务")
            chunk = r.read(8 * 1024 * 1024)
            if not chunk:
                break
            f.write(chunk)
            done += len(chunk)
            now = time.time()
            if now - last_update >= PROGRESS_INTERVAL:
                if total:
                    percent = done / total * 100
                    text = f"正在从 Telegram 下载到服务器：\n{file_name}\n\n进度：{percent:.2f}%\n已下载：{human_size(done)} / {human_size(total)}\n\n可发送 /cancel 中止当前任务"
                else:
                    text = f"正在从 Telegram 下载到服务器：\n{file_name}\n\n已下载：{human_size(done)}\n\n可发送 /cancel 中止当前任务"
                edit(job["chat_id"], job["progress_message_id"], text)
                last_update = now
    return local_path

def strip_ansi(s):
    return re.sub(r"\x1b\[[0-9;]*[A-Za-z]", "", s).replace("\r", "").replace("\n", "").strip()

def get_queue_size():
    with queue_cond:
        return len(job_queue)

def format_progress_message(job, line):
    line = strip_ansi(line) or "等待 rclone 输出进度..."
    if len(line) > 900:
        line = line[-900:]
    return (
        f"正在上传：\n{job['file_name']}\n\n"
        f"目标目录：\n{job['dest_dir']}\n\n"
        f"rclone 进度：\n{line}\n\n"
        f"队列中等待：{get_queue_size()} 个\n"
        f"可发送 /cancel 中止当前任务"
    )

def terminate_process(proc):
    if not proc:
        return
    try:
        proc.terminate()
        proc.wait(timeout=8)
    except Exception:
        try:
            proc.kill()
        except Exception:
            pass

def upload_with_rclone_progress(src, job):
    src = Path(src)
    dest_path = rclone_join(job["dest_dir"], job["file_name"])
    ensure_remote_dir(job["dest_dir"])

    cmd = [
        "rclone", "copyto", str(src), dest_path,
        "-P",
        "--stats=1s",
        "--stats-one-line",
        "--stats-unit=bytes",
        "--stats-log-level=NOTICE",
        "--transfers=1",
        "--checkers=1",
        "--tpslimit=3",
        "--tpslimit-burst=3",
        "--drive-pacer-min-sleep=200ms",
        "--drive-pacer-burst=10",
        "--retries=20",
        "--retries-sleep=30s",
        "--low-level-retries=20",
        "--log-level=NOTICE"
    ]

    proc = subprocess.Popen(cmd, stdin=subprocess.DEVNULL, stdout=subprocess.PIPE, stderr=subprocess.STDOUT, text=True, bufsize=0)
    job["process"] = proc
    job["stage"] = "uploading"

    last_edit = 0
    line_buf = ""
    last_line = "等待 rclone 输出进度..."

    try:
        while True:
            if job["cancel_event"].is_set():
                terminate_process(proc)
                raise Cancelled("已中止当前任务")

            if proc.poll() is not None:
                break

            ready, _, _ = select.select([proc.stdout], [], [], 1.0)
            if not ready:
                continue

            ch = proc.stdout.read(1)
            if not ch:
                continue

            if ch in ["\r", "\n"]:
                clean = strip_ansi(line_buf)
                line_buf = ""
                if clean:
                    last_line = clean
                now = time.time()
                if now - last_edit >= PROGRESS_INTERVAL:
                    edit(job["chat_id"], job["progress_message_id"], format_progress_message(job, last_line))
                    last_edit = now
            else:
                line_buf += ch
                if len(line_buf) > 3000:
                    line_buf = line_buf[-3000:]

        rc = proc.wait()
        if job["cancel_event"].is_set():
            raise Cancelled("已中止当前任务")
        if rc != 0:
            raise RuntimeError(f"rclone 上传失败，退出码：{rc}\n最后输出：{last_line}")

        edit(
            job["chat_id"],
            job["progress_message_id"],
            f"上传完成：\n{job['file_name']}\n\n目标目录：\n{job['dest_dir']}\n\n网盘路径：\n{dest_path}\n\n队列中等待：{get_queue_size()} 个"
        )
        return dest_path
    finally:
        job["process"] = None

def queue_snapshot(limit=10):
    with queue_cond:
        items = list(job_queue)[:limit]
        total = len(job_queue)
    if not items:
        return "当前没有等待中的任务。"
    lines = [f"等待队列：共 {total} 个任务"]
    for i, job in enumerate(items, 1):
        lines.append(f"{i}. #{job['id']} {job['file_name']} -> {job['dest_dir']}")
    if total > limit:
        lines.append(f"... 还有 {total - limit} 个未显示")
    return "\n".join(lines)

def status_text():
    with queue_cond:
        cur = current_job
        total = len(job_queue)
    if cur:
        cur_text = f"当前任务：\n#{cur['id']} {cur['file_name']}\n阶段：{cur.get('stage', '未知')}\n目标目录：{cur['dest_dir']}\n"
    else:
        cur_text = "当前没有正在上传的任务。\n"
    return f"{cur_text}\n等待队列：{total} 个\n\n命令：\n/queue 查看队列\n/cancel 中止当前任务\n/clear 清空等待队列"

def help_text(user_id):
    current = get_user_dest(user_id)
    return (
        "上传机器人使用说明\n\n"
        "直接发送文件、视频或音频给我，我会自动上传到当前网盘目录。\n"
        "多个文件会自动排队，按顺序依次上传。\n\n"
        "目录命令：\n"
        "/dir 查看当前上传目录\n"
        "/setdir gdrive:TelegramUploads/待整理 设置上传目录\n"
        "/setdir gdrive:媒体/电视剧/剧名 (年份)/Season 01 切换到指定目录\n\n"
        "队列命令：\n"
        "/status 查看当前任务和队列数量\n"
        "/queue 查看等待队列\n"
        "/cancel 中止当前正在上传的任务\n"
        "/clear 清空等待队列，不影响当前正在上传的任务\n\n"
        f"当前上传目录：\n{current}\n\n"
        "建议发送时选择“作为文件发送 / Send as File”，不要选择压缩视频。"
    )

def normalize_command(text):
    if not text:
        return "", ""
    parts = text.strip().split(maxsplit=1)
    cmd = parts[0].split("@", 1)[0]
    arg = parts[1].strip() if len(parts) > 1 else ""
    return cmd, arg

def add_job(message, chat_id, user_id):
    global job_seq
    file_id, file_name, size = get_media(message)
    if not file_id:
        send(chat_id, help_text(user_id))
        return

    dest_dir = get_user_dest(user_id)

    with queue_cond:
        job_seq += 1
        job_id = job_seq
        position = len(job_queue) + (1 if current_job else 0) + 1

    progress_message_id = send(
        chat_id,
        f"已加入上传队列：\n#{job_id} {file_name}\n\n"
        f"大小：{human_size(size) if size else '未知'}\n"
        f"目标目录：\n{dest_dir}\n\n"
        f"当前排队位置：{position}\n"
        f"发送 /queue 查看队列，/cancel 中止当前上传。"
    )

    job = {
        "id": job_id,
        "chat_id": chat_id,
        "user_id": user_id,
        "file_id": file_id,
        "file_name": file_name,
        "size": size,
        "dest_dir": dest_dir,
        "progress_message_id": progress_message_id,
        "stage": "queued",
        "cancel_event": threading.Event(),
        "process": None,
    }

    with queue_cond:
        job_queue.append(job)
        queue_cond.notify()

def handle_message(message):
    chat_id = message.get("chat", {}).get("id")
    user_id = message.get("from", {}).get("id")
    if not chat_id or not user_id:
        return

    if ALLOWED_USER_IDS and user_id not in ALLOWED_USER_IDS:
        send(chat_id, "你没有权限使用这个上传机器人。")
        return

    text = (message.get("text") or "").strip()
    cmd, arg = normalize_command(text)

    if cmd in ["/start", "/help"]:
        send(chat_id, help_text(user_id))
        return
    if cmd == "/dir":
        send(chat_id, f"当前上传目录：\n{get_user_dest(user_id)}")
        return
    if cmd == "/setdir":
        if not arg:
            send(chat_id, "用法：\n/setdir gdrive:TelegramUploads/待整理\n/setdir gdrive:媒体/电视剧/剧名 (年份)/Season 01")
            return
        try:
            set_user_dest(user_id, arg)
            ensure_remote_dir(arg)
            send(chat_id, f"已切换上传目录：\n{arg}")
        except Exception as e:
            send(chat_id, f"设置失败：{e}")
        return
    if cmd == "/queue":
        send(chat_id, queue_snapshot())
        return
    if cmd == "/status":
        send(chat_id, status_text())
        return
    if cmd == "/clear":
        with queue_cond:
            count = len(job_queue)
            job_queue.clear()
        send(chat_id, f"已清空等待队列：{count} 个任务。当前正在上传的任务不受影响。")
        return
    if cmd == "/cancel":
        with queue_cond:
            cur = current_job
        if not cur:
            send(chat_id, "当前没有正在上传的任务。")
            return
        cur["cancel_event"].set()
        if cur.get("process"):
            terminate_process(cur["process"])
        send(chat_id, f"已请求中止当前任务：\n#{cur['id']} {cur['file_name']}")
        return

    add_job(message, chat_id, user_id)

def worker_loop():
    global current_job
    while True:
        with queue_cond:
            while not job_queue:
                queue_cond.wait()
            job = job_queue.popleft()
            current_job = job

        src = None
        should_delete = False

        try:
            job["stage"] = "preparing"
            edit(job["chat_id"], job["progress_message_id"], f"开始处理：\n#{job['id']} {job['file_name']}\n\n目标目录：\n{job['dest_dir']}\n\n正在获取 Telegram 文件信息...")

            if job["cancel_event"].is_set():
                raise Cancelled("已中止当前任务")

            file_path = get_file_path(job["file_id"])
            local_path = resolve_local_path(file_path)

            if local_path:
                src = local_path
                should_delete = False
                job["stage"] = "ready"
            else:
                job["stage"] = "downloading"
                src = download_via_http(file_path, job["file_name"], job)
                should_delete = True

            if job["cancel_event"].is_set():
                raise Cancelled("已中止当前任务")

            upload_with_rclone_progress(src, job)

        except Cancelled as e:
            edit(job["chat_id"], job["progress_message_id"], f"任务已中止：\n#{job['id']} {job['file_name']}\n\n{e}\n\n队列中等待：{get_queue_size()} 个")
        except Exception as e:
            edit(job["chat_id"], job["progress_message_id"], f"任务失败：\n#{job['id']} {job['file_name']}\n\n错误：{e}\n\n队列中等待：{get_queue_size()} 个")
            print("worker error:", repr(e), flush=True)
        finally:
            if should_delete and src and Path(src).exists():
                try:
                    Path(src).unlink()
                except Exception:
                    pass
            with queue_cond:
                current_job = None
                queue_cond.notify_all()

def initialize_offset():
    last = get_last_update_id()
    if last is not None:
        return int(last) + 1
    if not SKIP_OLD:
        return 0
    try:
        res = api("getUpdates", {"timeout": 0, "limit": 100}, timeout=30)
        updates = res.get("result", [])
        if updates:
            max_id = max(u["update_id"] for u in updates)
            set_last_update_id(max_id)
            return max_id + 1
    except Exception as e:
        print("initialize_offset error:", repr(e), flush=True)
    return 0

def polling_loop():
    offset = initialize_offset()
    print(f"polling started, offset={offset}", flush=True)
    while True:
        try:
            res = api("getUpdates", {"offset": offset, "timeout": 50, "allowed_updates": ["message"]}, timeout=70)
            for upd in res.get("result", []):
                update_id = upd["update_id"]
                offset = update_id + 1
                set_last_update_id(update_id)
                msg = upd.get("message") or {}
                if msg:
                    handle_message(msg)
        except Exception as e:
            print("polling error:", repr(e), flush=True)
            time.sleep(5)

def main():
    print("tg-uploader started", flush=True)
    worker = threading.Thread(target=worker_loop, daemon=True)
    worker.start()
    polling_loop()

if __name__ == "__main__":
    main()
PY
```

检查语法：

```bash
chmod +x /opt/tg-bot-uploader/tg_uploader.py
python3 -m py_compile /opt/tg-bot-uploader/tg_uploader.py
```

创建 systemd：

```bash
cat > /etc/systemd/system/tg-uploader.service <<'EOF'
[Unit]
Description=Telegram file uploader to rclone remote
After=network-online.target docker.service
Wants=network-online.target

[Service]
Type=simple
EnvironmentFile=/opt/tg-bot-uploader/.env
WorkingDirectory=/opt/tg-bot-uploader
ExecStart=/usr/bin/python3 /opt/tg-bot-uploader/tg_uploader.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now tg-uploader
journalctl -u tg-uploader -f
```

### 10.5 机器人命令

```text
/help
查看帮助

/dir
查看当前上传目录

/setdir gdrive:TelegramUploads/待整理
设置上传目录

/setdir gdrive:媒体/电视剧/剧名 (年份)/Season 01
切换到指定目录

/status
查看当前任务

/queue
查看等待队列

/cancel
中止当前正在上传的任务

/clear
清空等待队列
```

---

## 11. EmbyPulse 管理方案

EmbyPulse 适合需要 Web 管理面板、用户中心、统计、播放排行等功能的场景。

### 11.1 Emby 里安装 Playback Reporting

Emby 后台：

```text
管理服务器
插件
目录 / Catalog
搜索 Playback Reporting
安装
重启 Emby
```

### 11.2 创建 Emby API Key

```text
管理服务器
服务器
API
新增 API Key
名称：EmbyPulse
```

### 11.3 部署 EmbyPulse

```bash
mkdir -p /opt/embypulse/config
mkdir -p /opt/embypulse/data
mkdir -p /opt/media-stack/embypulse
cd /opt/media-stack/embypulse
```

`.env`：

```bash
cat > .env <<'EOF'
LOCAL_ADMIN_USERNAME=admin
LOCAL_ADMIN_PASSWORD=这里改成强密码

EMBY_HOST=http://host.docker.internal:8096
EMBY_API_KEY=这里填Emby_API_Key
EOF

chmod 600 .env
```

`docker-compose.yml`：

```yaml
services:
  emby-pulse:
    image: zeyu8023/embypulse-pro:latest
    container_name: emby-pulse
    restart: unless-stopped
    ports:
      - "127.0.0.1:10307:10307"
      - "127.0.0.1:10308:10308"
    extra_hosts:
      - "host.docker.internal:host-gateway"
    volumes:
      - /opt/embypulse/config:/workspace/config
      - /opt/embypulse/data:/workspace/data
    environment:
      TZ: Asia/Shanghai
      PORT: "10307"
      REQUEST_PORT: "10308"
      LOCAL_AUTH_ENABLED: "true"
      LOCAL_ADMIN_USERNAME: "${LOCAL_ADMIN_USERNAME}"
      LOCAL_ADMIN_PASSWORD: "${LOCAL_ADMIN_PASSWORD}"
      EMBY_HOST: "${EMBY_HOST}"
      EMBY_API_KEY: "${EMBY_API_KEY}"
```

启动：

```bash
docker compose pull
docker compose up -d
docker logs --tail=100 emby-pulse
```

Caddy 反代：

```caddyfile
admin.example.com {
  reverse_proxy 127.0.0.1:10307
}

user.example.com {
  reverse_proxy 127.0.0.1:10308
}
```

---

## 12. EmbyTGBot 管理方案

EmbyTGBot 适合通过 Telegram 管理 Emby 用户。

### 12.1 准备

需要：

```text
Emby API Key
Emby 模板用户 testone
管理员 Bot Token
客户端 Bot Token
管理员 Telegram ID
```

Emby 创建模板用户：

```text
用户名：testone
用途：新用户复制它的权限和配置
```

### 12.2 部署

```bash
mkdir -p /opt/emby_tg_admin
cd /opt/emby_tg_admin

git clone https://github.com/sd87671067/EmbyTGBot.git EmbyTGBot
cd /opt/emby_tg_admin/EmbyTGBot

cp .env.example .env
chmod 600 .env
```

生成密钥：

```bash
openssl rand -hex 32
```

编辑 `.env`：

```env
APP_NAME=Emby TG 管理中心
APP_ENV=production
APP_PORT=18080
APP_BASE_URL=http://127.0.0.1:18080
APP_TIMEZONE=Asia/Shanghai
APP_MASTER_KEY=这里填openssl生成的字符串

APP_WEB_ADMIN_USERNAME=admin
APP_WEB_ADMIN_PASSWORD=一个强密码

EMBY_BASE_URL=http://host.docker.internal:8096
EMBY_API_KEY=Emby_API_Key

EMBY_SERVER_PUBLIC_URL=http://play1.example.com:8096,https://play2.example.com

EMBY_TEMPLATE_USER=testone
EMBY_IMPORT_IGNORE_USERNAMES=admin,testone
EMBY_SYNC_LOCAL_DEFAULT_PASSWORD=1234

ADMIN_BOT_TOKEN=管理员BotToken
ADMIN_CHAT_IDS=你的Telegram数字ID

CLIENT_BOT_TOKEN=客户端BotToken

ADMIN_CONTACT_TG_USERNAME=@你的TG用户名
ADMIN_CONTACT_TG_USER_ID=你的Telegram数字ID

DEFAULT_USER_EXPIRE_DAYS=90
REGISTER_CODE_LENGTH=16
CODE_BATCH_LIMIT=500
WEB_EXPIRING_SOON_DAYS=3
EXPIRY_CHECK_SECONDS=3600
ONLINE_CHECK_SECONDS=60
```

修改 `docker-compose.yml`，给容器访问宿主机 Emby：

```yaml
extra_hosts:
  - "host.docker.internal:host-gateway"
```

启动：

```bash
docker compose up -d --build
docker logs -f --tail=120 emby_tg_admin
```

### 12.3 常见问题

#### Bot Token 无效

日志：

```text
TokenValidationError: Token is invalid!
```

检查：

```bash
source .env
curl "https://api.telegram.org/bot${ADMIN_BOT_TOKEN}/getMe"
curl "https://api.telegram.org/bot${CLIENT_BOT_TOKEN}/getMe"
```

#### ARM64 镜像问题

Emby 本体需要 ARM64 镜像：

```yaml
image: emby/embyserver_arm64v8:latest
platform: linux/arm64/v8
```

#### 查询有效用户只显示一个

可以确认 SQLite 数据：

```bash
python3 - <<'PY'
import sqlite3
from pathlib import Path

for db in Path("data").rglob("*.db"):
    print(f"\n===== {db} =====")
    conn = sqlite3.connect(db)
    cur = conn.cursor()
    for t in [r[0] for r in cur.execute("SELECT name FROM sqlite_master WHERE type='table' ORDER BY name")]:
        cols = [r[1] for r in cur.execute(f'PRAGMA table_info("{t}")')]
        if "username" in cols:
            print(f"\n--- {t} ---")
            cur.execute(f'SELECT * FROM "{t}"')
            for row in cur.fetchall():
                print(row)
    conn.close()
PY
```

如果数据库里有多个用户，但机器人只显示一个，需要修正 `send_users_page` 函数或等待项目更新。

#### 服务地址一行显示两个 URL

可以把客户端显示改成：

```text
服务地址：
直连线路：http://play1.example.com:8096
CF线路：https://play2.example.com
```

核心逻辑是把逗号分隔的 `EMBY_SERVER_PUBLIC_URL` 格式化成多行。

---

## 13. Foam 卸载

如果之前部署过 Foam，可以这样卸载：

```bash
cd /opt/media-stack/foam 2>/dev/null || true

if [ -f docker-compose.yml ]; then
  docker compose down --remove-orphans
fi

docker rm -f foam foam-api foam-mysql foam-redis foam-selenium 2>/dev/null || true

mkdir -p /root/backup
if [ -d /opt/media-stack/foam ]; then
  tar -czf /root/backup/foam-backup-$(date +%F-%H%M%S).tar.gz /opt/media-stack/foam
fi

rm -rf /opt/media-stack/foam
docker image prune -f
```

清理 Caddy 中 Foam 域名反代：

```bash
grep -R "foam" -n /etc/caddy 2>/dev/null || true
vim /etc/caddy/conf.d/media-stack.caddy
caddy validate --config /etc/caddy/Caddyfile
systemctl reload caddy
```

---

## 14. 启停与维护

### 14.1 停止全部服务

```bash
docker stop embyserver 2>/dev/null || true
docker stop emby_tg_admin 2>/dev/null || true
docker stop telegram-bot-api 2>/dev/null || true

systemctl stop tg-uploader 2>/dev/null || true
```

### 14.2 重启全部服务

```bash
systemctl restart rclone-gdrive

cd /opt/media-stack/emby
docker compose up -d --force-recreate

cd /opt/emby_tg_admin/EmbyTGBot
docker compose up -d --force-recreate

cd /opt/tg-bot-uploader
docker compose up -d --force-recreate

systemctl restart tg-uploader
systemctl reload caddy
```

### 14.3 查看日志

```bash
docker logs -f --tail=100 embyserver
docker logs -f --tail=100 emby_tg_admin
docker logs -f --tail=100 telegram-bot-api
journalctl -u tg-uploader -f
journalctl -u rclone-gdrive -n 100 --no-pager
```

### 14.4 更新 Emby

```bash
cd /opt/media-stack/emby
docker compose pull
docker compose up -d
```

### 14.5 更新 EmbyTGBot

```bash
cd /opt/emby_tg_admin/EmbyTGBot
git pull --ff-only
docker compose up -d --build
```

### 14.6 更新上传机器人

```bash
cd /opt/tg-bot-uploader
docker compose pull
docker compose up -d
systemctl restart tg-uploader
```

---

## 15. 备份

```bash
mkdir -p /root/backup

tar -czf /root/backup/media-server-backup-$(date +%F).tar.gz \
  /opt/media-stack \
  /opt/media/emby/config \
  /opt/tg-bot-uploader \
  /opt/emby_tg_admin/EmbyTGBot/.env \
  /opt/emby_tg_admin/EmbyTGBot/data \
  /root/.config/rclone \
  /etc/systemd/system/rclone-gdrive.service \
  /etc/systemd/system/tg-uploader.service \
  /etc/caddy
```

重点备份：

```text
/opt/media/emby/config
/root/.config/rclone/rclone.conf
/opt/tg-bot-uploader/.env
/opt/tg-bot-uploader/state.json
/opt/emby_tg_admin/EmbyTGBot/.env
/opt/emby_tg_admin/EmbyTGBot/data
/etc/caddy
```

---

## 16. 最终推荐使用流程

日常新增资源：

```text
1. Telegram 上传机器人：
   /setdir gdrive:TelegramUploads/待整理

2. 把资源作为文件发送给机器人

3. 上传完成后，用 rclone moveto 整理到正式目录：
   gdrive:媒体/电影
   gdrive:媒体/电视剧
   gdrive:媒体/动漫

4. Emby 扫描媒体库

5. 如果识别错误，在 Emby 中手动 Identify
```

用户管理：

```text
方式 A：EmbyPulse Web 面板
方式 B：EmbyTGBot Telegram 管理
```

播放线路：

```text
首选：直连线路 / 灰云域名
备用：IP:8096
管理：Cloudflare 橙云域名
```

---

## 17. 总结

这套系统的核心原则是：

```text
Emby 只负责播放和刮削
rclone 负责把网盘挂成本地目录
Telegram 上传机器人负责收资源和上传网盘
EmbyPulse 或 EmbyTGBot 负责用户管理
Caddy 和 Cloudflare 负责访问线路
```

如果想稳定，最重要的是三点：

```text
1. 媒体文件命名规范
2. 播放线路不要长期走 Cloudflare 橙云
3. rclone 使用自己的 Google Drive client_id
```

把这三点做好，整套 Emby + 网盘媒体库的体验会稳定很多。
