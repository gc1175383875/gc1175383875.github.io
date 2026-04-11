---
author: orange
pubDatetime: 2025-12-04
title: 斐讯N1安装Home Assistant
tags: [HA, Home Assistant, Docker]
description: 在IStoreOS系统上通过Docker安装Home Assistant智能家居平台的详细教程。
---

## 方法一：Docker Compose（推荐）

```bash
opkg install docker-compose
```

找一个位置创建文件 `compose.yaml`，粘贴以下内容：

```yaml
services:
  homeassistant:
    container_name: homeassistant
    image: "ghcr.io/home-assistant/home-assistant:stable"
    volumes:
      - /opt/docker/hassio:/config
      - /run/dbus:/run/dbus:ro
    environment:
      - TZ=Asia/Shanghai
    restart: unless-stopped
    privileged: true
    network_mode: host
```

粘贴完成后执行：

```bash
docker compose up -d
```

### 安装 HACS

```bash
docker exec -it homeassistant bash
wget -O - https://get.hacs.xyz | bash -
```

## ~~方法二：HA Supervised（已弃用）~~

<details>
<summary>点击展开查看已弃用的安装方法</summary>

此方式官方已不再支持，新版本安装不进来，需要在后面加版本号下载 24 年末的旧版本，好处是 Supervised 版附带 Add-on。

创建 `/opt/docker/hassio/jobs.json`：

```json
{"ignore_conditions": ["healthy"]}
```

默认源的包比较旧，改为从 ghcr.io 取包，能取到最新。

参考：https://github.com/home-assistant/supervisor/pkgs/container/aarch64-hassio-supervisor

```bash
docker run -d --name hassio_supervisor --privileged \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/run/dbus:/var/run/dbus \
  -v /opt/docker/hassio:/data \
  -e SUPERVISOR_SHARE=/opt/docker/hassio \
  -e SUPERVISOR_NAME=hassio_supervisor \
  -e HOMEASSISTANT_REPOSITORY=homeassistant/qemuarm-64-homeassistant \
  --restart always \
  ghcr.io/home-assistant/aarch64-hassio-supervisor
```

查看日志：

```bash
docker logs -f hassio_supervisor
```

检查该目录是否存在，不存在要用 mkdir 创建：`/run/udev`

</details>

---

## Docker 镜像源

```text
https://docker.xuanyuan.me
https://docker.m.daocloud.io
https://docker.imgdb.de
https://docker-0.unsee.tech
https://docker.hlmirror.com
https://docker.1ms.run
```

## 参考链接

Home Assistant 最热门论坛：https://bbs.hassbian.com/
