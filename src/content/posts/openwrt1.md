---
title: Openwrt防检测固件使用教程(详细)
published: 2024-11-08
description: 用于校园网防检测等功能。
#image: https://bd076fc.webp.li/2024/11/5eca459b9832c2897f435eddb44556b7.png
tags: [教程, 路由]
category: 路由技术
draft: false
lang: zh_CN      # 仅当文章语言与 `config.ts` 中的网站语言不同时需要设置
---


# Openwrt实现校园网防检测

![https://bd076fc.webp.li/2024/11/88e47729ebe964e265c5347cf732f6ef.png](https://bd076fc.webp.li/2024/11/88e47729ebe964e265c5347cf732f6ef.png)

自编译*Xiaomi Redmi Router AC2100*路由器固件，可用于**STBU校园网**防检测等功能。

本文将对新拿到已刷好固件的朋友们出一期详细的**PC端**配置教程，方便大家使用到更多功能。

:::important
请低调使用本路由器，最好放在不显眼的位置，否则后果自负
:::

## 一.  固件构成



###     1.1 检测类型

​            1.基于 IPv4 数据包包头内的 *TTL 字段*的检测（固定TTL）

​            2.基于 HTTP 数据包请求头内的 *User-Agent 字段*的检测(使用UA2F)

​            3.基于 IPv4 数据包包头内的 *Identification 字段*的检测（rkp-ipid 设置 IPID）

​            4.基于网络协议栈时钟偏移的检测技术（统一时间）

​            本人测试第2、3条检测STBU好像没有，但是以防万一我还是加在里面了。*



###     1.2 本固件集成的软件包(部分)

```
luci-app-ttyd luci-app-timedreboot luci-app-argon-config kmod-rkp-ipid iptables-mod-filter iptables-mod-ipopt iptables-mod-u32 iptables-nft ua2f luci-app-ua2f kmod-ipt-u32 kmod-ipt-ipopt ipset iptables-mod-conntrack-extra luci-compat luci-theme-argon
```

## 二. 固件基础使用教程



###     2.1 路由器启动和接入网线

​        先连接电源线，**不要连接网线**，等待路由器灯**从蓝变黄闪再变蓝**

{% note warning flat %}Warning 现在连网线可能会导致校园网被检测哦{% endnote %}

​        通过后面*2.2—2.6*的设置完成后，再接入已连接校园网的网线到**wan口** *（靠近电源线的口）*



###     2.2 进入固件后台管理地址192.168.1.1

​        推荐使用edge浏览器

![https://bd076fc.webp.li/2024/11/5bc3b473bf7cf6e5a99928eeced8f62a.png](https://bd076fc.webp.li/2024/11/5bc3b473bf7cf6e5a99928eeced8f62a.png)



###     2.3 授权进入

​        默认用户名为root

​        密码为password

![https://bd076fc.webp.li/2024/11/e96d075286198f95306f212a508a6a40.png](https://bd076fc.webp.li/2024/11/e96d075286198f95306f212a508a6a40.png)



###     2.4 进入Openwrt主界面简介

​        概览含一些路由的基本信息

![https://bd076fc.webp.li/2024/11/599c282dd5cb5e45e99283346ca4a2de.png](https://bd076fc.webp.li/2024/11/599c282dd5cb5e45e99283346ca4a2de.png)



​        往下可以看见当前**在线的用户**数量和连接WiFi的主机名及IP地址

![](https://bd076fc.webp.li/2024/11/2774c65c3f77d1601b3afcdf47b21741.png)



###     2.5 检查基本设置

​        进入系统，查看本地时间是否和浏览器一致，不一致请**同步浏览器时间**

![](https://bd076fc.webp.li/2024/11/f2a7906c54d8585096a71f4fd293d5c4.png)



​        管理权可以更改后台登录密码，**不建议修改**，忘记密码后果自负

![](https://bd076fc.webp.li/2024/11/6e3fe828ed53138f84450be903f6cccf.png)



​        可以选择一个特定时间**定时重启**，让路由器每天可以休息几十秒

![https://bd076fc.webp.li/2024/11/6817bc9a2c5c2ff2b86f489f41e0e0fd.png](https://bd076fc.webp.li/2024/11/6817bc9a2c5c2ff2b86f489f41e0e0fd.png)



​        后台有**重启按钮**，可以不用去拔电源了

![](https://bd076fc.webp.li/2024/11/de41333db5aa52b72b1d829f780ecf6c.png)



​        服务里带有**U2F检测开关**，实测关了也没有被检测，以防万一还是打开吧

​        *只应用前三个就行*

![](https://bd076fc.webp.li/2024/11/c501485d173ffbf882ac8414e3e51ba6.png)



​        网络-无线，设置WiFi名称和密码，上面是2.4G，下面是5G，点基本设置进入

![](https://bd076fc.webp.li/2024/11/ba584e4e54f5b68ef396b813c4dcf58f.png)



​        优先**使用5G信号**，本人200M宽带能跑到260M左右

​        SSID设置WiFi名，密码栏设置WiFi密码，图片是5G信号设置，2.4G不用也要进去设置

![](https://bd076fc.webp.li/2024/11/7eac2411986fbe1da41dd07f7d2539c1.png)



​        检查防火墙-自定义规则代码是否一致，修改后重启防火墙

![](https://bd076fc.webp.li/2024/11/4da7adcead1e1c8b60934f9a7ee596ee.png)

```
# This file is interpreted as shell script.
# Put your custom iptables rules here, they will
# be executed with each firewall (re-)start.

# Internal uci firewall chains are flushed and recreated on reload, so
# put custom rules into the root chains e.g. INPUT or FORWARD or into the
# special user chains, e.g. input_wan_rule or postrouting_lan_rule.
#
# 通过 rkp-ipid 设置 IPID
iptables -t mangle -N IPID_MOD
iptables -t mangle -A FORWARD -j IPID_MOD
iptables -t mangle -A OUTPUT -j IPID_MOD
iptables -t mangle -A IPID_MOD -d 0.0.0.0/8 -j RETURN
iptables -t mangle -A IPID_MOD -d 127.0.0.0/8 -j RETURN
iptables -t mangle -A IPID_MOD -d 172.16.0.0/12 -j RETURN
iptables -t mangle -A IPID_MOD -d 192.168.0.0/16 -j RETURN
iptables -t mangle -A IPID_MOD -d 255.0.0.0/8 -j RETURN
iptables -t mangle -A IPID_MOD -j MARK --set-xmark 0x10/0x10

# 防时钟偏移检测
iptables -t nat -N ntp_force_local
iptables -t nat -I PREROUTING -p udp --dport 123 -j ntp_force_local
iptables -t nat -A ntp_force_local -d 0.0.0.0/8 -j RETURN
iptables -t nat -A ntp_force_local -d 127.0.0.0/8 -j RETURN
iptables -t nat -A ntp_force_local -d 192.168.0.0/16 -j RETURN
iptables -t nat -A ntp_force_local -s 192.168.0.0/16 -j DNAT --to-destination 192.168.1.1

# 通过 iptables 修改 TTL 值
iptables -t mangle -A POSTROUTING -j TTL --ttl-set 128

```



###     2.6 高级功能扩展

​        ***附加高级功能可能导致运行不稳定，谨慎使用，不使用请关闭***

​        MentoHUST暂时用不上，**保持关闭**即可

​        解锁网易云灰色歌曲打开后连接WiFi可自动解锁，但可能让网易云加载变慢，自行测试使用

​        UU游戏加速器可登陆自己的账号，自行测试使用

​        KMS可自动解锁局域网内的Office 365全家桶，自行测试使用

![](https://bd076fc.webp.li/2024/11/0b68a04d6deb9a3a5d00c76dd1ee3916.png)



###     2.7 接入校园网

​        **保证*2.2—2.6*的设置无误**后，再接入已连接校园网的网线到**wan口** *（靠近电源线的口）*

​        自动弹出或自己1.1.1.1进入认证页面，登陆自己的宽带

​        宽带有问题请加认证页面左下角群，**自行找管理人员解决**

![](https://bd076fc.webp.li/2024/11/fd5c2611c0b1ddb8264b11168005ec35.png)



## 三.注意事项

​    被检测到五分钟后会自动恢复正常，或者等待五分钟去校园网系统重新登陆

​    本路由实测最高连接14台设备没被检测，但是100M宽带建议****不超过4台设备**，200M宽带建议**不超过6台设备**

​    多设备同时接入与断开路由有高机率被检测，比如室友集体下课回寝，或集体出去上课时

​    不建议连接此破解WiFi使用翻墙代理软件，小概率被检测共享

​    后期会整理大家使用当中存在的问题尝试出一期解决方案，有什么问题评论区留言哦！