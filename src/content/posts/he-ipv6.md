---
title: Debian12 VPS 添加 HE IPv6 隧道并保持运行
published: 2026-04-26
description: 给只有 IPv4 的 Debian 12 VPS 配置 Hurricane Electric Tunnelbroker IPv6，并用 ifupdown 与 systemd 定时器保持运行
#image: https://example.com/your-cover.webp
tags: [VPS, Debian, IPv6, HE Tunnelbroker]
category: VPS技术
draft: false
lang: zh_CN
---

:::important
请注意，本文适用于 **Debian 12 纯 IPv4 VPS**。如果你的 VPS 已经自带原生 IPv6，一般不需要使用 HE Tunnelbroker。

本文方案不会接管主网卡，也不会重启主网络服务，目的是尽量避免远程 SSH 断连。
:::

> **操作环境**：Debian 12 KVM VPS / 仅 IPv4 入口  
> **目标**：给纯 IPv4 VPS 添加 HE Tunnelbroker IPv6，并在重启后自动恢复  
> **核心方案**：`ifupdown v4tunnel` + `systemd timer watchdog`

## 一、准备 HE Tunnelbroker 参数

先到 [HE Tunnelbroker](https://tunnelbroker.net/) 创建一个 Regular Tunnel。

创建完成后，在隧道详情页记录下面 4 个核心参数：

```text
Server IPv4 Address
Client IPv4 Address
Server IPv6 Address
Client IPv6 Address
```

例如：

```text
Server IPv4 Address: 74.82.46.6
Client IPv4 Address: 162.43.91.133
Server IPv6 Address: 2001:470:23:543::1/64
Client IPv6 Address: 2001:470:23:543::2/64
```

脚本里填写 IPv6 时，不要带 `/64` 后缀，例如：

```text
2001:470:23:543::2
2001:470:23:543::1
```

:::important
HE 的 6in4 隧道使用的是 **IPv4 protocol 41**，不是 TCP 41 端口，也不是 UDP 41 端口。

如果 VPS 服务商或上游网络不放行 protocol 41，隧道接口可能能创建成功，但 IPv6 不会真正通。
:::

## 二、确认 VPS 当前公网 IPv4

在 VPS 上执行：

```bash
ip -4 addr
ip route
```

如果已经安装 `curl`，也可以查看公网出口 IP：

```bash
curl -4 https://ifconfig.co
```

确认结果要和 HE 后台的 `Client IPv4 Address` 一致。

如果不一致，需要先在 HE 后台修改 `Client IPv4 Address`，否则隧道不会正常回包。

## 三、一键安装脚本

下面脚本适合全新的 Debian 12 VPS 使用。

它会做这些事情：

- 安装最小依赖：`ifupdown`、`iproute2`、`iputils-ping`
- 写入 `/etc/network/interfaces.d/he-ipv6`
- 只拉起 `he-ipv6` 隧道，不重启主网卡
- 启用 `networking`，保证重启后自动拉起
- 创建 `systemd timer`，每 2 分钟检查一次 IPv6，不通则自动重建隧道

:::important
执行前请先修改脚本开头的 4 个参数：

```bash
CLIENT4="你的 Client IPv4"
SERVER4="HE 的 Server IPv4"
CLIENT6="你的 Client IPv6，不带 /64"
SERVER6="HE 的 Server IPv6，不带 /64"
```
:::

```bash
cat > /root/install-he-ipv6.sh <<'EOF'
#!/bin/sh
set -eu

TUN="he-ipv6"

# ====== 请修改下面 4 个参数 ======
CLIENT4="YOUR_CLIENT_IPV4"
SERVER4="YOUR_SERVER_IPV4"
CLIENT6="YOUR_CLIENT_IPV6"
SERVER6="YOUR_SERVER_IPV6"
# =================================

# 用于 watchdog 测试 IPv6 出网，不依赖 DNS
TEST6="2001:4860:4860::8888"

if [ "$(id -u)" != "0" ]; then
    echo "请使用 root 用户执行本脚本。"
    exit 1
fi

echo "[1/9] 检查并安装最小依赖..."
NEED_APT=0
command -v ifup >/dev/null 2>&1 || NEED_APT=1
command -v ifdown >/dev/null 2>&1 || NEED_APT=1
command -v ip >/dev/null 2>&1 || NEED_APT=1
command -v ping >/dev/null 2>&1 || NEED_APT=1

if [ "$NEED_APT" = "1" ]; then
    apt-get update
    apt-get install -y ifupdown iproute2 iputils-ping
fi

echo "[2/9] 备份现有网络配置..."
BACKUP_DIR="/root/he-ipv6-backup-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$BACKUP_DIR"
cp -a /etc/network/interfaces "$BACKUP_DIR/interfaces" 2>/dev/null || true
cp -a /etc/network/interfaces.d "$BACKUP_DIR/interfaces.d" 2>/dev/null || true

echo "[3/9] 开启 IPv6，并关闭 rp_filter 干扰..."
cat > /etc/sysctl.d/99-he-ipv6.conf <<EON
net.ipv6.conf.all.disable_ipv6=0
net.ipv6.conf.default.disable_ipv6=0
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
EON
sysctl -p /etc/sysctl.d/99-he-ipv6.conf >/dev/null 2>&1 || true
modprobe sit 2>/dev/null || true

echo "[4/9] 准备 /etc/network/interfaces.d ..."
mkdir -p /etc/network/interfaces.d

if [ ! -f /etc/network/interfaces ]; then
    cat > /etc/network/interfaces <<'EON'
auto lo
iface lo inet loopback

source /etc/network/interfaces.d/*
EON
else
    grep -q '^source /etc/network/interfaces.d/\*' /etc/network/interfaces 2>/dev/null || \
    echo 'source /etc/network/interfaces.d/*' >> /etc/network/interfaces
fi

echo "[5/9] 写入 HE IPv6 隧道配置..."
cat > /etc/network/interfaces.d/he-ipv6 <<EON
auto ${TUN}
iface ${TUN} inet6 v4tunnel
        address ${CLIENT6}
        netmask 64
        endpoint ${SERVER4}
        local ${CLIENT4}
        ttl 255
        gateway ${SERVER6}
        pre-up ip tunnel del ${TUN} 2>/dev/null || true
        post-down ip tunnel del ${TUN} 2>/dev/null || true
EON

echo "[6/9] 创建 watchdog 自恢复脚本..."
cat > /usr/local/sbin/he-ipv6-watchdog.sh <<EON
#!/bin/sh
set -eu

TUN="${TUN}"
TEST6="${TEST6}"

need_restart=0

if ! ip link show "\$TUN" >/dev/null 2>&1; then
    need_restart=1
fi

if ! ip -6 route show default 2>/dev/null | grep -q "dev \$TUN"; then
    need_restart=1
fi

if ! ping -6 -c 1 -W 3 "\$TEST6" >/dev/null 2>&1; then
    need_restart=1
fi

if [ "\$need_restart" = "1" ]; then
    echo "[he-ipv6-watchdog] IPv6 tunnel seems down, rebuilding..."
    ifdown --force "\$TUN" 2>/dev/null || true
    ip tunnel del "\$TUN" 2>/dev/null || true
    ifup "\$TUN" 2>/dev/null || true
else
    echo "[he-ipv6-watchdog] IPv6 tunnel is OK."
fi
EON
chmod +x /usr/local/sbin/he-ipv6-watchdog.sh

echo "[7/9] 创建 systemd timer..."
cat > /etc/systemd/system/he-ipv6-watchdog.service <<'EON'
[Unit]
Description=Check and recover HE IPv6 tunnel

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/he-ipv6-watchdog.sh
EON

cat > /etc/systemd/system/he-ipv6-watchdog.timer <<'EON'
[Unit]
Description=Run HE IPv6 tunnel watchdog periodically

[Timer]
OnBootSec=60
OnUnitActiveSec=120
AccuracySec=15
Persistent=true

[Install]
WantedBy=timers.target
EON

systemctl daemon-reload

echo "[8/9] 拉起 HE IPv6 隧道，只操作 he-ipv6，不重启主网络..."
ip tunnel del "${TUN}" 2>/dev/null || true
ifdown --force "${TUN}" 2>/dev/null || true
ifup "${TUN}"

echo "[9/9] 启用开机自启和 watchdog..."
systemctl enable networking >/dev/null 2>&1 || true
systemctl enable --now he-ipv6-watchdog.timer >/dev/null

echo
echo "[OK] HE IPv6 隧道配置完成。"
echo
echo "===== tunnel ====="
ip tunnel show "${TUN}" || true
echo
echo "===== address ====="
ip -6 addr show dev "${TUN}" || true
echo
echo "===== route ====="
ip -6 route || true
echo
echo "===== test ====="
ping -6 -c 3 "${TEST6}" || true
echo
echo "如果上面的 ping 有回包，可以继续测试："
echo "  getent hosts ipv6.google.com"
echo "  ping -6 -c 3 ipv6.google.com"
echo "  curl -6 https://ifconfig.co"
EOF

chmod +x /root/install-he-ipv6.sh
/root/install-he-ipv6.sh
```

## 四、使用示例

假设你的 HE 后台信息如下：

```text
Server IPv4 Address: 74.82.46.6
Client IPv4 Address: 162.43.91.133
Server IPv6 Address: 2001:470:23:543::1/64
Client IPv6 Address: 2001:470:23:543::2/64
```

那么脚本开头应改成：

```bash
CLIENT4="162.43.91.133"
SERVER4="74.82.46.6"
CLIENT6="2001:470:23:543::2"
SERVER6="2001:470:23:543::1"
```

然后执行：

```bash
chmod +x /root/install-he-ipv6.sh
/root/install-he-ipv6.sh
```

## 五、测试 IPv6 是否成功

先测试纯 IPv6 地址，不依赖 DNS：

```bash
ping -6 -c 3 2001:4860:4860::8888
```

再测试域名解析与 IPv6 出口：

```bash
getent hosts ipv6.google.com
ping -6 -c 3 ipv6.google.com
curl -6 https://ifconfig.co
```

如果 `curl -6 https://ifconfig.co` 返回你的 HE Client IPv6，例如：

```text
2001:470:23:543::2
```

说明 IPv6 已经配置成功。

:::tip
有些情况下，`ping -6 HE 的 Server IPv6` 可能不回包，但外网 IPv6 是通的。

最终建议以这两个结果为准：

```bash
ping -6 -c 3 2001:4860:4860::8888
curl -6 https://ifconfig.co
```
:::

## 六、DNS 解析失败时的处理

如果出现这种情况：

```text
ping -6 2001:4860:4860::8888 能通
ping -6 ipv6.google.com 不通
curl -6 https://ifconfig.co 提示 Could not resolve host
```

说明 IPv6 已经通了，问题只是 DNS。

可以直接写一个静态 `/etc/resolv.conf`：

```bash
rm -f /etc/resolv.conf

cat > /etc/resolv.conf <<'EOF'
nameserver 2001:470:20::2
nameserver 2606:4700:4700::1111
nameserver 2001:4860:4860::8888
nameserver 74.82.42.42
nameserver 1.1.1.1
nameserver 8.8.8.8
options timeout:2 attempts:2
EOF
```

然后重新测试：

```bash
getent hosts ipv6.google.com
ping -6 -c 3 ipv6.google.com
curl -6 https://ifconfig.co
```

## 七、查看运行状态

查看隧道：

```bash
ip tunnel show he-ipv6
ip -6 addr show dev he-ipv6
ip -6 route
```

查看 watchdog 定时器：

```bash
systemctl status he-ipv6-watchdog.timer --no-pager
systemctl list-timers | grep he-ipv6
```

查看 watchdog 日志：

```bash
journalctl -u he-ipv6-watchdog.service -n 50 --no-pager
```

手动重建隧道：

```bash
ifdown --force he-ipv6 2>/dev/null || true
ip tunnel del he-ipv6 2>/dev/null || true
ifup he-ipv6
```

## 八、故障排查

### 1. 提示 Network is unreachable

说明本机没有 IPv6 默认路由。

检查：

```bash
ip -6 route
```

正常应看到类似：

```text
default via 2001:470:23:543::1 dev he-ipv6
```

如果没有，重新拉起隧道：

```bash
ifdown --force he-ipv6 2>/dev/null || true
ip tunnel del he-ipv6 2>/dev/null || true
ifup he-ipv6
```

### 2. IPv6 路由有了，但 ping 外网 100% 丢包

先确认 IPv4 出口是否和 HE 后台一致：

```bash
ip route get HE_SERVER_IPV4
curl -4 https://ifconfig.co
```

例如：

```bash
ip route get 74.82.46.6
curl -4 https://ifconfig.co
```

如果 `curl -4` 返回的 IPv4 不是 HE 后台的 `Client IPv4 Address`，需要去 HE 后台更新 Client IPv4。

### 3. 抓包确认 protocol 41 是否有回包

安装 tcpdump 后可以抓包：

```bash
apt install -y tcpdump

tcpdump -ni any 'ip proto 41 or host HE_SERVER_IPV4'
```

另开一个窗口执行：

```bash
ping -6 -c 3 HE_SERVER_IPV6
```

如果只看到：

```text
Out IP CLIENT_IPV4 > SERVER_IPV4
```

没有看到：

```text
In IP SERVER_IPV4 > CLIENT_IPV4
```

通常说明 HE 后台 endpoint 没生效，或者上游网络没有把 protocol 41 回包送回来。

可以尝试在 HE 后台重新保存 `Client IPv4 Address`，或者删除重建 tunnel。

## 九、卸载脚本

如果以后不用 HE IPv6，可以执行：

```bash
systemctl disable --now he-ipv6-watchdog.timer 2>/dev/null || true
rm -f /etc/systemd/system/he-ipv6-watchdog.service
rm -f /etc/systemd/system/he-ipv6-watchdog.timer
rm -f /usr/local/sbin/he-ipv6-watchdog.sh
rm -f /etc/network/interfaces.d/he-ipv6
systemctl daemon-reload

ifdown --force he-ipv6 2>/dev/null || true
ip tunnel del he-ipv6 2>/dev/null || true
```

## 十、关于 Routed /64 和 LXC 容器

HE 一般会给两段 IPv6：

```text
Tunnel /64：用于 he-ipv6 隧道本身
Routed /64：用于分配给网站、Docker、LXC、虚拟机等业务
```

例如：

```text
Tunnel Client IPv6: 2001:470:23:543::2
Routed /64:         2001:470:24:543::/64
```

如果以后要给 LXC 容器分配公网 IPv6，建议使用 `Routed /64`，不要直接使用 tunnel 的 `/64`。

例如：

```text
宿主机网关：2001:470:24:543::1
容器 101： 2001:470:24:543::101
容器 102： 2001:470:24:543::102
```

## 总结

本文方案的核心是：

```text
Debian 12 纯 IPv4 VPS
        ↓
HE Tunnelbroker 6in4
        ↓
/etc/network/interfaces.d/he-ipv6
        ↓
systemd timer 定时检查并自动恢复
```

它的优点是：

- 不重启主网络
- 不接管主网卡
- SSH 断连风险低
- 重启后可自动恢复
- 隧道异常时会定时自修复

需要注意的是：DD 重装系统会清空系统文件，所以重装后需要重新执行本文的一键脚本。
