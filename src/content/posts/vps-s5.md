---
title: 在 Debian 12 VPS 上搭建带认证的类家宽 SOCKS5 代理
published: 2026-05-30
description: "使用 AimiliVPN + VPNGate 为 VPS 添加类家宽出口，并通过 SOCKS5 用户名密码认证对外提供代理服务。"
tags: ["VPS", "SOCKS5", "VPNGate", "Debian", "Proxy"]
category: Guides
draft: false
---

> 本文仅用于个人网络学习、跨区域测试、出口连通性验证等合法用途。请勿用于垃圾注册、撞库、爬虫滥用、绕过风控、攻击或其他违反服务条款与法律法规的行为。

## 前言

普通 VPS 的公网 IP 通常来自云厂商或机房 ASN，在某些网站上可能会被识别为数据中心 IP。如果希望让 VPS 的部分流量通过 VPNGate 节点出口，可以使用 [AimiliVPN](https://github.com/baoweise-bot/aimili-vpngate) 自动采集 VPNGate 节点，并在本地提供 HTTP/SOCKS5 代理。

最终效果如下：

```text
客户端
  ↓ SOCKS5 用户名密码认证
VPS:52000
  ↓ 本地认证转发器
127.0.0.1:7928
  ↓ AimiliVPN
VPNGate 节点
  ↓
目标网站看到 VPNGate 出口 IP
```

本文配置：

```text
系统：Debian 12
AimiliVPN 本地代理：127.0.0.1:7928
对外 SOCKS5 端口：52000
SOCKS5 用户名：yizong
SOCKS5 密码：yizong
```

> 注意：VPNGate 是公共志愿节点池，无法保证 IP 一定“纯净”、一定是家宽、一定独享或长期稳定。本文中的“类家宽出口”指的是通过筛选 ISP、ASN、PTR 等信息，尽量选择看起来像家庭宽带的出口节点。

## 一、安装 AimiliVPN

Debian 12 上先安装基础依赖：

```bash
sudo apt update
sudo apt install -y curl ca-certificates git openvpn iproute2 iptables python3
```

AimiliVPN 官方安装脚本默认限制 Ubuntu，Debian 需要临时绕过系统判断：

```bash
curl -Ls https://raw.githubusercontent.com/baoweise-bot/aimili-vpngate/main/install.sh -o install.sh
sed -i 's/"${ID:-}"/"ubuntu"/g' install.sh
sudo bash install.sh
```

安装完成后，可以使用 `ml` 命令管理服务：

```bash
ml status
ml start
ml stop
ml restart
ml logs
```

查看当前运行状态：

```bash
ml status
```

正常情况下，你应该能看到类似信息：

```text
本地 HTTP/SOCKS5 代理：127.0.0.1:7928
当前 VPNGate 节点：xxx.xxx.xxx.xxx
代理检测出口 IP：xxx.xxx.xxx.xxx
```

## 二、测试 AimiliVPN 本地代理

先确认 `127.0.0.1:7928` 可以正常使用：

```bash
curl --proxy socks5h://127.0.0.1:7928 https://api.ipify.org
echo
```

如果返回的是 VPNGate 出口 IP，而不是 VPS 原始 IP，说明 AimiliVPN 已经正常工作。

也可以测试 HTTP 代理：

```bash
curl -x http://127.0.0.1:7928 https://api.ipify.org
echo
```

> AimiliVPN 的 `7928` 端口默认只适合作为本机代理出口，不建议直接暴露到公网。

## 三、创建带认证的 SOCKS5 转发器

Debian 12 默认软件源里不一定有 `3proxy`，所以这里使用 Python 写一个轻量 SOCKS5 认证转发器。

它的作用是：

```text
监听公网 0.0.0.0:52000
要求用户名密码认证
认证成功后转发到 127.0.0.1:7928
```

创建脚本：

```bash
sudo tee /opt/auth-socks-aimili.py >/dev/null <<'PYEOF'
#!/usr/bin/env python3
import select
import socket
import threading

LISTEN_HOST = "0.0.0.0"
LISTEN_PORT = 52000

USERNAME = b"yizong"
PASSWORD = b"yizong"

UPSTREAM_HOST = "127.0.0.1"
UPSTREAM_PORT = 7928


def recv_exact(sock, n):
    data = b""
    while len(data) < n:
        chunk = sock.recv(n - len(data))
        if not chunk:
            raise ConnectionError("connection closed")
        data += chunk
    return data


def read_socks_addr(sock, atyp):
    if atyp == 1:
        addr = recv_exact(sock, 4)
    elif atyp == 3:
        length = recv_exact(sock, 1)
        addr = length + recv_exact(sock, length[0])
    elif atyp == 4:
        addr = recv_exact(sock, 16)
    else:
        raise ValueError("unsupported address type")
    port = recv_exact(sock, 2)
    return addr, port


def relay(a, b):
    sockets = [a, b]
    while True:
        readable, _, errored = select.select(sockets, [], sockets, 300)
        if errored:
            return
        if not readable:
            return
        for src in readable:
            dst = b if src is a else a
            data = src.recv(65536)
            if not data:
                return
            dst.sendall(data)


def handle_client(client, addr):
    upstream = None
    try:
        client.settimeout(30)

        ver = recv_exact(client, 1)
        if ver != b"\x05":
            return

        nmethods = recv_exact(client, 1)[0]
        methods = recv_exact(client, nmethods)

        if 2 not in methods:
            client.sendall(b"\x05\xff")
            return

        client.sendall(b"\x05\x02")

        auth_ver = recv_exact(client, 1)
        if auth_ver != b"\x01":
            client.sendall(b"\x01\x01")
            return

        ulen = recv_exact(client, 1)[0]
        user = recv_exact(client, ulen)
        plen = recv_exact(client, 1)[0]
        passwd = recv_exact(client, plen)

        if user != USERNAME or passwd != PASSWORD:
            client.sendall(b"\x01\x01")
            return

        client.sendall(b"\x01\x00")

        head = recv_exact(client, 4)
        ver, cmd, rsv, atyp = head

        if ver != 5 or cmd != 1:
            client.sendall(b"\x05\x07\x00\x01\x00\x00\x00\x00\x00\x00")
            return

        addr_bytes, port_bytes = read_socks_addr(client, atyp)
        request = head + addr_bytes + port_bytes

        upstream = socket.create_connection((UPSTREAM_HOST, UPSTREAM_PORT), timeout=20)
        upstream.settimeout(30)

        upstream.sendall(b"\x05\x01\x00")
        resp = recv_exact(upstream, 2)

        if resp != b"\x05\x00":
            client.sendall(b"\x05\x01\x00\x01\x00\x00\x00\x00\x00\x00")
            return

        upstream.sendall(request)

        rhead = recv_exact(upstream, 4)
        ratyp = rhead[3]
        raddr, rport = read_socks_addr(upstream, ratyp)

        client.sendall(rhead + raddr + rport)

        if rhead[1] != 0:
            return

        client.settimeout(None)
        upstream.settimeout(None)
        relay(client, upstream)

    except Exception:
        pass
    finally:
        try:
            client.close()
        except Exception:
            pass
        if upstream:
            try:
                upstream.close()
            except Exception:
                pass


def main():
    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server.bind((LISTEN_HOST, LISTEN_PORT))
    server.listen(256)

    print(
        f"Authenticated SOCKS5 listening on {LISTEN_HOST}:{LISTEN_PORT}, "
        f"upstream {UPSTREAM_HOST}:{UPSTREAM_PORT}",
        flush=True,
    )

    while True:
        client, addr = server.accept()
        threading.Thread(target=handle_client, args=(client, addr), daemon=True).start()


if __name__ == "__main__":
    main()
PYEOF

sudo chmod +x /opt/auth-socks-aimili.py
```

## 四、创建 systemd 服务

创建服务文件：

```bash
sudo tee /etc/systemd/system/auth-socks-aimili.service >/dev/null <<'EOF'
[Unit]
Description=Authenticated SOCKS5 wrapper for AimiliVPN
After=network.target aimilivpn.service
Wants=aimilivpn.service

[Service]
Type=simple
ExecStart=/usr/bin/python3 /opt/auth-socks-aimili.py
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
EOF
```

启动服务：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now auth-socks-aimili
```

查看服务状态：

```bash
sudo systemctl status auth-socks-aimili --no-pager
```

检查端口是否监听：

```bash
ss -lntp | grep 52000
```

正常情况下会看到类似：

```text
LISTEN 0 256 0.0.0.0:52000 0.0.0.0:*
```

## 五、测试 SOCKS5 用户名密码认证

在 VPS 本机测试：

```bash
curl --proxy socks5h://yizong:yizong@127.0.0.1:52000 https://api.ipify.org
echo
```

如果返回的是 AimiliVPN 面板里的 VPNGate 出口 IP，说明 SOCKS5 认证代理已经正常工作。

测试错误密码：

```bash
curl --proxy socks5h://wrong:wrong@127.0.0.1:52000 https://api.ipify.org
echo
```

正常情况下应该连接失败。

## 六、客户端使用方式

在浏览器、Proxifier、Shadowrocket、Clash、v2rayN 或其他支持 SOCKS5 的客户端中填写：

```text
代理类型：SOCKS5
服务器地址：你的 VPS IP
端口：52000
用户名：yizong
密码：yizong
```

连接后访问：

```text
https://api.ipify.org
```

看到的 IP 应该是 VPNGate 出口 IP，而不是 VPS 原始 IP。

## 七、防火墙安全设置

SOCKS5 的用户名密码只是认证，不是加密。公网暴露时，强烈建议限制来源 IP。

如果只允许自己的公网 IP 访问，例如你的本地公网 IP 是 `1.2.3.4`：

```bash
sudo iptables -I INPUT -p tcp -s 1.2.3.4 --dport 52000 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 52000 -j DROP
```

查看规则：

```bash
sudo iptables -L INPUT -n --line-numbers
```

如果你使用的是 `ufw`，可以这样写：

```bash
sudo ufw allow from 1.2.3.4 to any port 52000 proto tcp
sudo ufw deny 52000/tcp
```

> 不要长期使用弱密码。示例里的 `yizong/yizong` 方便测试，但公网使用建议改成高强度随机密码。

## 八、修改用户名、密码或端口

编辑脚本：

```bash
sudo nano /opt/auth-socks-aimili.py
```

修改下面几行：

```python
LISTEN_PORT = 52000

USERNAME = b"yizong"
PASSWORD = b"yizong"
```

修改后重启服务：

```bash
sudo systemctl restart auth-socks-aimili
```

查看日志：

```bash
journalctl -u auth-socks-aimili -f
```

## 九、常用排错命令

查看 AimiliVPN 状态：

```bash
ml status
```

查看 AimiliVPN 日志：

```bash
ml logs
```

查看认证 SOCKS5 服务状态：

```bash
sudo systemctl status auth-socks-aimili --no-pager
```

查看端口监听：

```bash
ss -lntp | grep 52000
```

测试 AimiliVPN 原始本地代理：

```bash
curl --proxy socks5h://127.0.0.1:7928 https://api.ipify.org
echo
```

测试带认证的 SOCKS5 代理：

```bash
curl --proxy socks5h://yizong:yizong@127.0.0.1:52000 https://api.ipify.org
echo
```

重启所有相关服务：

```bash
sudo systemctl restart aimilivpn
sudo systemctl restart auth-socks-aimili
```

## 十、让 Xray 或 3x-ui 使用这个出口

如果你希望 VMess、VLESS 或 Trojan 入站流量也走 AimiliVPN，可以让 Xray 的出站指向 `127.0.0.1:7928`。

示例出站配置：

```json
{
  "tag": "vpngate-out",
  "protocol": "socks",
  "settings": {
    "servers": [
      {
        "address": "127.0.0.1",
        "port": 7928
      }
    ]
  }
}
```

然后在路由规则里把指定入站转到这个出口：

```json
{
  "type": "field",
  "inboundTag": [
    "你的入站 tag"
  ],
  "outboundTag": "vpngate-out"
}
```

链路如下：

```text
用户客户端
  ↓ VMess / VLESS / Trojan
Xray / 3x-ui
  ↓ 127.0.0.1:7928
AimiliVPN
  ↓ VPNGate
目标网站
```

相比直接暴露 SOCKS5，这种方式更适合长期公网使用，因为 VMess、VLESS、Trojan 这类协议可以提供更完整的传输加密与访问控制。

## 总结

本文实现了一个简单的 VPS 类家宽 SOCKS5 代理方案：

```text
AimiliVPN 负责连接 VPNGate 节点
127.0.0.1:7928 作为本地代理出口
Python SOCKS5 转发器负责用户名密码认证
52000 作为公网 SOCKS5 入口
```

最终客户端只需要填写：

```text
SOCKS5 地址：你的 VPS IP
端口：52000
用户名：yizong
密码：yizong
```

即可通过 VPS 使用 VPNGate 出口访问网络。

再次提醒：VPNGate 公共节点无法保证纯净、独享、稳定或真实住宅属性。如果你需要长期稳定的真实家宽出口，最可靠的方式仍然是在自己的家庭宽带环境中部署 WireGuard、OpenVPN 或 SoftEther，然后让 VPS 回连家宽网络作为出口。
