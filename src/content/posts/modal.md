---
title: Debian 12 VPS + Modal 部署 Docker 完整教程（免费额度内）
published: 2026-04-27
description: 从一台全新的 Debian 12 VPS 出发，使用 Modal 部署 Docker 镜像，并给出一个尽量保持在线且不超过免费额度的稳定方案。
#image: "/images/modal-debian12-cover.jpg"
tags: [Debian, Modal, Docker, VPS, Tutorial]
category: DevOps
draft: false
---

> 这篇文章基于 Modal 官方文档和官方定价页整理，目标是：**用一台全新的 Debian 12 VPS 当作控制端，在 Modal 上部署一个 Docker 镜像，并把费用控制在免费额度内。**

## 先说结论

如果你的目标是：

1. **部署后服务一直可访问**
2. **尽量像“常驻”一样保持在线**
3. **每月不超过 Modal Starter 的免费额度**

那么最稳的方案是：

- 使用 `modal deploy` 做**持久部署**
- 用 `modal.Image.from_registry()` 直接引用现成镜像
- 资源固定为 **`cpu=0.125` + `memory=128`**
- 配置 **`min_containers=1`**、**`max_containers=1`**、**`buffer_containers=0`**
- **不要指定区域**，除非你真的有延迟或合规需求

这套配置在 **31 天整月 24x7** 的最保守口径下，单个常驻热容器大约是：

- **不指定区域：约 `$5.13/月`**
- **1.25x 区域：约 `$6.41/月`**
- **2.5x 区域：约 `$12.82/月`**

而 Modal Starter 当前自带 **`$30/月` 免费 compute credits**，所以**一个最小规格的常驻热容器是明显在免费额度内的**。

## 你需要知道的 3 个前提

### 1）你的 Debian 12 VPS 只是“控制端”

Modal 的工作方式不是“在你的 VPS 上一直跑 Docker 守护进程”，而是：

- 你在 VPS 上安装 Modal CLI
- 你在 VPS 上执行 `modal deploy`
- 真正的容器由 Modal 在云端运行

所以这台 Debian 12 VPS 更像“部署控制台”，不是运行容器的宿主机。

### 2）“保持运行”不等于“同一个容器永不重启”

`@modal.web_server(...)` 本质上仍然属于 Modal 的 Web Endpoint，底层是**自动扩缩容的容器池**。把 `min_containers=1` 打开后，可以做到：

- 服务一直有 URL
- 至少保留 1 个 warm container
- 空闲时不缩到 0

但它**不保证永远是同一个容器**。你应该把它理解成“服务持续可用”，而不是“这一台容器永不被替换”。

### 3）为什么这套方案能控制在免费额度内

Modal 当前公开价格里：

- CPU：`$0.0000131 / core / sec`
- Memory：`$0.00000222 / GiB / sec`
- Starter：`$30 / month free credits`

而默认最小资源请求是：

- **0.125 CPU**
- **128 MiB 内存**

所以只保留 **1 个最小规格热容器**，即便整月不停，仍然离 `$30` 免费额度有比较大的安全边际。

---

## 一、在全新的 Debian 12 VPS 上安装环境

先登录你的 Debian 12 VPS，执行下面这组命令：

```bash
apt update
apt install -y python3 python3-pip python3-venv git curl
```

然后创建一个 Python 虚拟环境，避免污染系统环境：

```bash
python3 -m venv ~/modal-venv
source ~/modal-venv/bin/activate
```

安装 Modal CLI：

```bash
python -m pip install -U pip modal
```

初始化登录：

```bash
python -m modal setup
```

如果你不方便在浏览器里登录，也可以使用 Token 方式：

```bash
python -m modal token set
```

---

## 二、创建项目目录

```bash
mkdir -p ~/modal-docker-demo
cd ~/modal-docker-demo
```

---

## 三、准备你的镜像信息

这篇教程使用的是“**直接引用现成镜像**”的方式，也就是：

- 你已经有一个公开镜像，例如 Docker Hub 或 GHCR
- Modal 直接从镜像仓库拉取这个镜像

最典型的形式是：

```text
Docker Hub: your-user/your-image:latest
GHCR:      ghcr.io/your-user/your-image:latest
```

如果镜像本身没有合适的 Python 运行时，Modal 官方支持在 `from_registry()` 里通过 `add_python="3.11"` 补进去。

---

## 四、写一份“免费额度内、尽量常驻”的 `modal_app.py`

下面这份是本文的推荐配置。

你只需要改 4 个地方：

- `APP_NAME`
- `IMAGE_REF`
- `PORT`
- `START_CMD`

把下面内容保存成 `modal_app.py`：

```python
import os
import subprocess
import modal

APP_NAME = "docker-on-modal-min"
IMAGE_REF = "ghcr.io/your-user/your-image:latest"   # 改成你的镜像
PORT = 3000                                           # 改成你的容器实际监听端口
START_CMD = ["node", "index.js"]                    # 改成你的启动命令

app = modal.App(APP_NAME)

image = modal.Image.from_registry(
    IMAGE_REF,
    add_python="3.11",  # 让现成镜像更容易兼容 Modal Function
)

@app.function(
    image=image,
    cpu=0.125,
    memory=128,
    min_containers=1,
    max_containers=1,
    buffer_containers=0,
    timeout=24 * 60 * 60,
)
@modal.web_server(PORT)
def serve():
    print("provider =", os.environ.get("MODAL_CLOUD_PROVIDER"))
    print("region   =", os.environ.get("MODAL_REGION"))
    subprocess.Popen(START_CMD)
```

### 这份配置为什么这样写

#### `cpu=0.125` + `memory=128`

这是 Modal 当前默认最小资源请求。显式写出来有两个好处：

- 成本预估更清楚
- 后面自己回看代码时不会误会

#### `min_containers=1`

这会让函数**至少保留 1 个 warm container**，即使没有请求，也不会缩到 0。对于“希望尽量持续在线”的场景，这个参数最关键。

#### `max_containers=1`

这会把容器上限固定为 1，避免流量稍微高一点就自动扩成多个容器，从而出现意外账单。

#### `buffer_containers=0`

不预留额外容器。因为我们这篇教程的目标是**免费额度内稳定运行**，不是极致低延迟。

#### `timeout=24 * 60 * 60`

Modal Function 默认执行超时是 300 秒。手动拉到 24 小时，是为了尽量减少过早回收带来的不确定性。

#### 为什么这里没有写 `region`

因为只要你显式指定区域，就会触发区域价格乘数：

- `US / EU / UK / AP`：**1.25x**
- `CA / SA / ME / MX / AF`：**2.5x**

如果你的目标是“**优先稳在免费额度内**”，那么**不指定区域最省钱**。

---

## 五、部署到 Modal

确保你当前还在虚拟环境里：

```bash
source ~/modal-venv/bin/activate
```

执行部署：

```bash
python -m modal deploy modal_app.py
```

部署完成后，Modal 会为这个 Web Endpoint 分配可访问的 URL。即使你关闭 SSH 会话，**这个部署本身仍然存在**。

---

## 六、如何查看日志、状态和停止部署

### 查看 App 列表

```bash
python -m modal app list
```

### 实时查看日志

```bash
python -m modal app logs docker-on-modal-min -f
```

### 查看容器列表

```bash
python -m modal container list
```

### 查看某个容器日志

```bash
python -m modal container logs ta-xxxxxxxx -f
```

### 停止部署

```bash
python -m modal app stop docker-on-modal-min
```

注意：`modal app stop` 是永久停止当前部署，后面如果还要跑，需要重新 `modal deploy`。

---

## 七、怎么确认费用不会超出免费额度

### 方案 A：本文推荐的“常驻热容器版”

配置：

- `cpu=0.125`
- `memory=128`
- `min_containers=1`
- `max_containers=1`
- 不指定区域

按 **31 天整月不停机** 来算，约 **`$5.13/月`**。

这比 Starter 自带的 **`$30/月` 免费 credits`** 低很多，所以**长期放 1 个最小规格热容器是安全的**。

### 方案 B：如果你必须指定区域

如果你写：

```python
region="ap"
```

那么会触发 **1.25x** 的区域乘数。此时单个最小规格热容器大约 **`$6.41/月`**。

如果是 2.5x 的区域类别，则约 **`$12.82/月`**。

结论依然是：

- **1 个最小规格热容器**，即便指定区域，仍然通常在免费额度内
- **不要把 `max_containers` 调大**，否则费用会开始线性上涨

### 方案 C：如果你更在意费用，而不是“热容器常驻”

把 `min_containers=1` 去掉：

```python
@app.function(
    image=image,
    cpu=0.125,
    memory=128,
    max_containers=1,
    buffer_containers=0,
    timeout=24 * 60 * 60,
)
```

这时服务仍然是**持久部署**，URL 一直有效，但容器在空闲时会缩到 0。这样通常更便宜，不过首次请求会有冷启动。

如果你只是个人项目、低频访问站点，实际上**这是最省钱的方案**。

---

## 八、如何查看本月账单

```bash
python -m modal billing report --for "this month"
```

如果你想看今天的小时级用量：

```bash
python -m modal billing report --for today -r h
```

建议你在刚部署后的前几天每天看一次账单，确认没有因为镜像本身的高负载把 CPU 或内存实际使用拉高。

> Modal 官方文档写明：CPU 和内存是按“**请求值**”和“**实际使用值**”里更高的那个计费。

---

## 九、常见问题

### Q1：为什么我已经 `modal deploy` 了，但感觉容器还是会变？

因为 Modal 的核心模型不是传统 VPS，而是**服务持续存在、容器由平台调度维护**。`min_containers=1` 只能保证有至少 1 个 warm container，不代表某个容器实例永远不被替换。

### Q2：为什么我不推荐一开始就锁到 Tokyo、Seoul 这类更细区域？

因为区域越细，资源池越小；官方也建议优先使用更宽泛的区域，这通常会更有利于可用性和冷启动表现。并且一旦显式指定区域，就会引入价格乘数。

### Q3：为什么我不用自己的 Docker 直接在 VPS 上跑？

当然可以，但那是另一条路线。本文这条路的核心价值是：

- 用 VPS 只做“控制台”
- 把运行、扩缩容、日志、部署 URL 交给 Modal
- 省掉自己维护公网服务、进程守护和扩缩容的工作

### Q4：如果我想把现成 Dockerfile 直接丢给 Modal 呢？

也可以，Modal 还支持 `modal.Image.from_dockerfile("./Dockerfile")`。但如果你已经有 GHCR / Docker Hub 镜像，直接 `from_registry()` 通常更省事。

---

## 十、一份可直接复制的完整流程

### 1）安装环境

```bash
apt update
apt install -y python3 python3-pip python3-venv git curl
python3 -m venv ~/modal-venv
source ~/modal-venv/bin/activate
python -m pip install -U pip modal
python -m modal setup
```

### 2）创建项目

```bash
mkdir -p ~/modal-docker-demo
cd ~/modal-docker-demo
```

### 3）写 `modal_app.py`

```python
import os
import subprocess
import modal

APP_NAME = "docker-on-modal-min"
IMAGE_REF = "ghcr.io/your-user/your-image:latest"
PORT = 3000
START_CMD = ["node", "index.js"]

app = modal.App(APP_NAME)

image = modal.Image.from_registry(
    IMAGE_REF,
    add_python="3.11",
)

@app.function(
    image=image,
    cpu=0.125,
    memory=128,
    min_containers=1,
    max_containers=1,
    buffer_containers=0,
    timeout=24 * 60 * 60,
)
@modal.web_server(PORT)
def serve():
    print("provider =", os.environ.get("MODAL_CLOUD_PROVIDER"))
    print("region   =", os.environ.get("MODAL_REGION"))
    subprocess.Popen(START_CMD)
```

### 4）部署

```bash
source ~/modal-venv/bin/activate
python -m modal deploy modal_app.py
```

### 5）看日志

```bash
python -m modal app logs docker-on-modal-min -f
```

### 6）看账单

```bash
python -m modal billing report --for "this month"
```

---

## 最后的建议

如果你只有一个小站点、一个小 API，或者只是想把一个 Docker 镜像稳定挂在公网：

- **优先用最小资源**：`0.125 CPU + 128 MiB`
- **优先锁死容器数**：`max_containers=1`
- **想更像常驻就加**：`min_containers=1`
- **想更省钱就去掉**：`min_containers=1`
- **没有强需求就不要指定区域**

这样做，基本就能在 Modal 的免费额度内，把一个最小规格的 Docker 服务长期挂住。

---

## 参考资料

- Modal 官方文档总览：`https://modal.com/docs/guide`
- 使用现成镜像：`https://modal.com/docs/guide/existing-images`
- 图片与镜像 API：`https://modal.com/docs/reference/modal.Image`
- Web endpoints：`https://modal.com/docs/guide/webhooks`
- `modal.web_server`：`https://modal.com/docs/reference/modal.web_server`
- 部署管理：`https://modal.com/docs/guide/managing-deployments`
- `modal app` CLI：`https://modal.com/docs/reference/cli/app`
- `modal container` CLI：`https://modal.com/docs/reference/cli/container`
- 资源配置与计费：`https://modal.com/docs/guide/resources`
- 自动扩缩容：`https://modal.com/docs/guide/scale`
- 冷启动与 warm container：`https://modal.com/docs/guide/cold-start`
- 区域选择：`https://modal.com/docs/guide/region-selection`
- Function 超时：`https://modal.com/docs/guide/timeouts`
- Pricing：`https://modal.com/pricing`
