---
title: 使用哪吒面板为 VPS 配置 Cloudflare DDNS
published: 2026-05-03
description: "用哪吒面板给 VPS 配置 Cloudflare DDNS，并说明 IP 检测间隔、IP 变更通知"
image: ""
tags: ["VPS", "DDNS", "Cloudflare"]
category: "VPS"
draft: false
---

:::note
本文记录的是：使用哪吒面板把 VPS 的公网 IP 自动同步到 Cloudflare DNS。适合家宽、动态 IP VPS、NAT 后端、经常重装或迁移 IP 的机器。
:::

## 一、先说结论

如果你想给 VPS 配置 Cloudflare DDNS，目前更推荐使用：

```text
哪吒面板 + Cloudflare API Token
```

哪吒面板可以在 Agent 上报新 IP 后，由 Dashboard 自动更新 Cloudflare 里的 DNS 记录。

而 Komari 目前更偏向监控面板，并没有像哪吒面板一样内置 Cloudflare DDNS 功能。如果你使用 Komari，可以搭配 `ddns-go` 或自写脚本来完成 DDNS。

## 二、准备工作

你需要提前准备好：

- 一个已经托管到 Cloudflare 的域名
- 一台已经接入哪吒 Agent 的 VPS
- 一个 Cloudflare API Token
- 哪吒 Dashboard 管理员权限

本文假设你的主域名是：

```text
example.com
```

准备用来指向 VPS 的 DDNS 域名是：

```text
vps.example.com
```

:::tip
实际操作时，请把 `example.com` 和 `vps.example.com` 换成你自己的域名。
:::

## 三、在 Cloudflare 里准备 DNS 记录

进入 Cloudflare 后，选择你的域名，然后进入：

```text
DNS → Records
```

如果你只需要 IPv4，添加一条 A 记录：

```text
类型：A
名称：vps
内容：先填当前 VPS IPv4，或者临时填一个占位 IP
代理状态：DNS only / 仅 DNS / 灰云
TTL：Auto 或 1 分钟
```

如果你还需要 IPv6，再添加一条 AAAA 记录：

```text
类型：AAAA
名称：vps
内容：先填当前 VPS IPv6，或者临时填一个占位 IPv6
代理状态：DNS only / 仅 DNS / 灰云
TTL：Auto 或 1 分钟
```

:::warning
如果这个域名是用来连接 SSH、VPS 面板、代理服务、游戏服或其他非 HTTP 服务，建议使用灰云，也就是 `DNS only`。不要开启 Cloudflare 橙云代理，否则可能导致真实端口无法连接。
:::

## 四、创建 Cloudflare API Token

进入 Cloudflare 账号设置：

```text
右上角头像
→ My Profile
→ API Tokens
→ Create Token
```

可以使用 `Edit zone DNS` 模板，也可以自定义 Token。

权限建议设置为：

```text
Zone → Zone → Read
Zone → DNS → Edit
```

资源范围建议限制到指定域名：

```text
Include → Specific zone → example.com
```

创建后，Cloudflare 会显示一次 API Token。

:::caution
Cloudflare API Token 只会显示一次，请立即复制保存。不要把 Token 发到公开群聊、博客正文、GitHub 仓库或截图里。
:::

## 五、在哪吒面板添加 DDNS 配置

登录哪吒 Dashboard，进入：

```text
动态域名解析
→ 新配置
```

按照下面这样填写：

```text title="哪吒 DDNS 配置示例"
名称：Cloudflare-DDNS
DDNS 供应商：cloudflare
域名：vps.example.com
最大重试次数：3
DDNS 凭据 1：留空
DDNS 凭据 2：填写 Cloudflare API Token
启用 DDNS IPv4：如果需要更新 A 记录就勾选
启用 DDNS IPv6：如果需要更新 AAAA 记录就勾选
```

:::important
使用 Cloudflare 作为 DDNS 供应商时，`DDNS 凭据 1` 一般留空，`DDNS 凭据 2` 填 Cloudflare API Token。
:::

如果你要同时更新多个域名，可以用英文逗号分隔：

```text
vps.example.com,node1.example.com,home.example.com
```

## 六、把 DDNS 配置绑定到指定 VPS

添加 DDNS 配置后，还需要把它绑定到具体的服务器。

进入：

```text
服务器
→ 找到目标 VPS
→ 编辑
```

然后开启：

```text
启用 DDNS
```

并选择刚才创建的配置：

```text
Cloudflare-DDNS
```

保存后，哪吒会在 Agent 上报 IP 变化时，自动更新 Cloudflare DNS 记录。

## 七、检查 DDNS 是否生效

可以先在哪吒面板里看日志：

```text
日志
```

正常情况下，应该能看到类似内容：

```text
正在尝试更新域名(vps.example.com)DDNS
尝试更新域名(vps.example.com)DDNS成功
```

也可以在本地终端检查解析结果。

检查 IPv4：

```bash
dig vps.example.com A +short
```

检查 IPv6：

```bash
dig vps.example.com AAAA +short
```

或者使用：

```bash
nslookup vps.example.com
```

如果返回的 IP 和 VPS 当前公网 IP 一致，就说明 DDNS 已经正常工作。

## 八、哪吒多久检测一次 IP？

哪吒不是秒级实时检测。默认情况下，Agent 的 IP 上报间隔通常是：

```text
1800 秒，也就是 30 分钟
```

也就是说，默认逻辑大概是：

```text
VPS 公网 IP 变化
→ Agent 下次上报新 IP
→ Dashboard 根据 DDNS 配置更新 Cloudflare DNS
```

所以默认情况下，IP 变化后可能需要最多约 30 分钟才会被发现。

如果你想让它更及时，可以修改 Agent 的 IP 上报间隔。

例如改成 60 秒：

```yaml title="agent 配置示例"
ip_report_period: 60
```

修改完成后重启哪吒 Agent：

```bash
sudo systemctl restart nezha-agent.service
```

推荐值：

```text
普通 VPS：300 秒，也就是 5 分钟
希望更快发现 IP 变化：60 秒
不建议长期使用 30 秒，除非确实需要非常快
```

:::note
哪吒更新 Cloudflare DNS 记录后，客户端实际解析到新 IP 的时间还会受 DNS TTL 和本地 DNS 缓存影响。Cloudflare 里建议把相关记录的 TTL 设置为 `Auto` 或 `1 分钟`。
:::

## 九、IP 变化后可以收到通知吗？

可以。哪吒面板可以配置 IP 变更通知。

大致入口是：

```text
右上角头像
→ 系统设置
→ 系统配置
→ IP 变更提醒
```

配置项一般包括：

```text
覆盖范围：监控全部服务器，或只监控指定服务器
特定服务器：选择需要包含或排除的服务器
提醒发送至通知分组：选择已经配置好的通知组
启用功能：开启
通知中显示完整 IP 地址：按需开启
```

在配置 IP 变更提醒之前，需要先配置通知方式。

进入：

```text
通知
```

然后添加你想使用的推送方式，例如：

```text
Telegram
Bark
Server 酱
企业微信
飞书
Slack
邮件
```

最终效果大概是：

```text
VPS 公网 IP 变化
→ 哪吒 Agent 检测并上报新 IP
→ 哪吒 Dashboard 更新 Cloudflare DNS
→ 哪吒发送 IP 变更通知
```

:::tip
如果你找不到 `IP 变更提醒`，不要只在 `通知` 页面找。通知页面通常只是创建通知方式和通知组，IP 变更提醒本身一般在 `系统设置 → 系统配置` 里。
:::

## 十、为什么我找不到 IP 变更提醒？

如果后台里看不到相关设置，可以按下面几个方向排查。

### 1. 是否使用管理员账号登录

IP 变更提醒属于系统配置，普通账号可能看不到。

请使用管理员账号登录后台后再查看：

```text
右上角头像
→ 系统设置
→ 系统配置
```

### 2. 是否在错误页面查找

很多人会去：

```text
通知
```

里面找 IP 变更提醒。

但这个页面一般只是用来配置通知方式，例如 Telegram、Bark、邮件等。真正的 IP 变更提醒通常在系统设置里。

### 3. 是否界面语言不同

如果你使用英文界面，可能会显示为类似：

```text
IP Change Notification
```

可以在系统设置页面用浏览器搜索：

```text
Ctrl + F
```

然后搜索：

```text
IP
```

### 4. 是否哪吒版本较旧或使用了第三方改版

不同版本、不同主题、不同第三方镜像的菜单位置可能不完全一样。

如果确实找不到，建议先查看当前面板版本，或者更新到官方较新的版本后再看。

## 十一、Komari 有类似功能吗？

目前 Komari 没有像哪吒面板这样内置 Cloudflare DDNS 的功能。

Komari 更偏向服务器监控，虽然可以配合 Cloudflare Tunnel 暴露面板，也有一些社区工具可以把节点 IP 同步到 Git，但这些都不等于 Cloudflare DDNS。

简单对比：

| 面板 | 是否内置 Cloudflare DDNS | 适合用途 |
| --- | --- | --- |
| 哪吒面板 | 支持 | 监控 + DDNS + 通知 |
| Komari | 暂不支持 | 轻量监控 |
| Komari + ddns-go | 支持，靠 ddns-go 实现 | 监控和 DDNS 分开管理 |

如果你使用 Komari，但也想做 Cloudflare DDNS，可以使用：

```text
Komari + ddns-go
```

或者：

```text
Komari + 自写 Cloudflare API 脚本 + cron
```

:::tip
如果只是为了 VPS 动态解析域名，`ddns-go` 会更直接；如果你已经在用哪吒面板，则直接用哪吒内置 DDNS 更省事。
:::

## 十二、常见问题排查

### Cloudflare DNS 没有更新

优先检查：

```text
Cloudflare API Token 是否正确
Token 是否有 Zone Read 和 DNS Edit 权限
Token 的资源范围是否包含当前域名
哪吒 DDNS 配置里供应商是否选择 cloudflare
DDNS 凭据 2 是否填写了 Token
目标服务器是否启用了 DDNS
```

### DDNS 日志显示失败

可以先确认 Token 权限：

```text
Zone → Zone → Read
Zone → DNS → Edit
```

再确认域名确实托管在 Cloudflare，并且 DNS 记录已经存在。

### IPv6 没有更新

检查 VPS 是否真的有公网 IPv6：

```bash
ip -6 addr
```

也可以测试公网 IPv6：

```bash
curl -6 ip.sb
```

如果 VPS 没有公网 IPv6，就不要勾选 `启用 DDNS IPv6`。

### 更新成功但访问还是旧 IP

这通常是 DNS 缓存问题。

可以尝试：

```bash
dig vps.example.com A +short @1.1.1.1
dig vps.example.com A +short @8.8.8.8
```

如果公共 DNS 已经返回新 IP，但你本地还是旧 IP，可以清理本地 DNS 缓存，或者等待 TTL 过期。

### 开了橙云后连不上

如果你是用这个域名连接 SSH、代理服务、游戏服或自定义端口，请把 Cloudflare 代理状态改成：

```text
DNS only / 灰云
```

不要使用橙云代理。

## 十三、推荐配置总结

我个人比较推荐这样配置：

```text
Cloudflare DNS：灰云
TTL：Auto 或 1 分钟
哪吒 DDNS：启用 IPv4，按需启用 IPv6
Agent IP 上报间隔：300 秒
需要快速通知：改为 60 秒
IP 变更通知：开启
```

如果你的 VPS IP 经常变动，建议使用：

```yaml
ip_report_period: 60
```

如果 IP 很少变化，只是防止偶尔变动，可以使用：

```yaml
ip_report_period: 300
```

## 十四、完整流程回顾

最后把完整流程压缩成一张清单：

```text
1. Cloudflare 中添加 A / AAAA 记录
2. 创建 Cloudflare API Token
3. Token 权限设置为 Zone Read + DNS Edit
4. 哪吒 Dashboard 添加 Cloudflare DDNS 配置
5. DDNS 凭据 2 填写 Cloudflare API Token
6. 到服务器编辑页启用 DDNS
7. 检查日志是否更新成功
8. 根据需要调整 ip_report_period
9. 配置通知方式
10. 开启 IP 变更提醒
```

这样配置完成后，VPS 公网 IP 变化时，哪吒就可以自动更新 Cloudflare DNS，并在检测到 IP 变化后给你发送通知。
