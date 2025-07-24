---
title: VPS常用命令/工具
published: 2099-12-30
description: VPS常用命令/工具
#image: https://bd076fc.webp.li/2025/04/27832734d6536dd1d761ace15fee2118.jpg
tags: [脚本, VPS]
category: VPS技术
draft: false
lang: zh_CN      # 仅当文章语言与 `config.ts` 中的网站语言不同时需要设置
---


:::important
请注意，以下命令适用于基于**Linux**的系统。对于其他操作系统，操作命令可能有所不同。
:::


# 命令类

## 更新VPS环境
Ubuntu/Debian
```Ubuntu/Debian
apt update -y && apt install -y curl socat wget sudo
```

Alpine
```Alpine
apk update && apk add curl
```

## [Tree命令](https://www.runoob.com/linux/linux-comm-tree.html)
```Ubuntu/Debian
sudo apt install tree
```

## vim安装
```Ubuntu/Debian
sudo apt-get install vim
```

## unzip安装
```Ubuntu/Debian
sudo apt install zip
```

## [Nginx命令](https://www.runoob.com/linux/linux-comm-tree.html)
```Ubuntu/Debian
sudo nginx -t #检查配置文件语法

sudo systemctl start nginx #启动 Nginx

sudo systemctl stop nginx #停止 Nginx

sudo systemctl restart nginx #重启 Nginx

sudo systemctl reload nginx #重新加载配置（不中断服务）

sudo systemctl status nginx #查看状态
```



# 工具类

## [ssh综合工具箱](https://github.com/eooce/Sing-box)
```dash
curl -fsSL https://raw.githubusercontent.com/eooce/ssh_tool/main/ssh_tool.sh -o ssh_tool.sh && chmod +x ssh_tool.sh && ./ssh_tool.sh

bash <(curl -sL kejilion.sh)
```

## [IP质量体检脚本](https://github.com/xykt/IPQuality)
```dash
bash <(curl -Ls IP.Check.Place)
```

## [老王四协议](https://github.com/eooce/Sing-box)
```dash
bash <(curl -Ls https://raw.githubusercontent.com/eooce/sing-box/main/sing-box.sh)

PORT=11111 CFIP=www.visa.com.tw CFPORT=443 bash <(curl -Ls https://raw.githubusercontent.com/eooce/sing-box/main/sing-box.sh)
```

## [甬哥X-ui](https://github.com/yonggekkk/x-ui-yg)
```dash
bash <(curl -Ls https://raw.githubusercontent.com/yonggekkk/x-ui-yg/main/install.sh)
```



# IPv6专区

## [warp](https://github.com/fscarmen/warp)
```dash
wget -N https://gitlab.com/fscarmen/warp/-/raw/main/menu.sh && bash menu.sh
```

## 纯IPv6修改DNS64(NAT64)，可访问IPV4地址
```dash
vim /etc/resolv.conf
nameserver 2001:67c:2b0::4 
nameserver 2001:67c:27e4::64
```