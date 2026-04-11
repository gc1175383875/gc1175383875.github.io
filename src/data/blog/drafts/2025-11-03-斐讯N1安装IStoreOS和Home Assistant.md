---
author: orange
pubDatetime: 2025-11-03
title: 斐讯N1安装IStoreOS和Home Assistant
draft: true
tags: [OpenWRT, HA, Home Assistant]
description: 斐讯N1盒子安装IStoreOS系统和Home Assistant智能家居平台的详细教程。
---
# 一、安装 IStoreOS

## 固件下载与安装

固件地址：https://fw.koolcenter.com/iStoreOS/alpha/n1/

下载后使用 balenaEtcher 软件，把下载的 zip 解压出的 img 镜像，烧写到 U 盘。

把 U 盘插到 N1 盒子上，然后通电，等待几分钟，盒子会自动从 U 盘启动系统。

通过路由器查找 IP，用 SSH 工具连接，账号 `root`，密码 `password`。

连上之后执行：

```bash
install-to-emmc.sh
```

或浏览器直接访问该 IP，通过 IStoreOS 页面配置。

## 网络配置

使用网络向导 - 手动配置，快速配置静态路由和 DHCP 等，各配置项按需填写，旁路由做网关时，要打开 DHCP。

### 接口设置

网络 -> 接口 -> lan 编辑 -> DHCP 服务器 -> 高级设置：

- 动态 DHCP 确保开启
- DHCP 选项填写主路由 IP：`"3,192.168.31.1", "6,192.168.31.1"`（经测试，IStoreOS 版，直接填 192.168.31.1 无效，找不到 DHCP）

DNS 设置为主路由 IP `192.168.31.1`

### 标签设置

网络 -> DHCP/DNS -> 标签，新增一项：

```json
"Proxy":
[
  "3,192.168.31.2", "6,192.168.31.2"
]
```

然后进入主路由的管理页面，关闭 DHCP。

### 防火墙设置

配置好之后，如果没有网络，需要检查 `网络 -> 防火墙`，需要确认 Lan 区域的"IP 动态伪装"勾上保存。

## 安装 OpenClash

访问 https://github.com/vernesong/OpenClash/releases ，下载 OpenClash 的 IPK 版本，用 SSH 工具移到 OpenWrt(IStoreOS) 系统里。

### 安装依赖

```bash
opkg update
opkg install bash iptables dnsmasq-full curl ca-bundle ipset ip-full iptables-mod-tproxy iptables-mod-extra ruby ruby-yaml kmod-tun kmod-inet-diag unzip luci-compat luci luci-base
opkg install /tmp/openclash.ipk
```

安装完成后，刷新 IStoreOS 的网页，打开 `服务 - OpenClash`，自动弹出核心下载。如果进度慢，访问插件设置 - 版本更新来手动下载安装，按照"内核路径"后面的标识 `/etc/openclash/core/clash_meta` 来修改文件名放入对应位置。

## 扩容 Overlay（可选）

输入 `parted`（如果下面打印有 Using /dev/mmcblkboot0，说明找错了磁盘分区，使用 `parted /dev/mmcblk1` 手动选择分区）

```bash
unit s
resizepart 3 5G
quit
```

然后再执行：

```bash
resize2fs /dev/mmcblk1p3
```

### 挂载剩余空间

在磁盘管理里，把空闲分区用上，格式化。然后在"挂载点"页面，创建一个新分区的挂载点，设置成根分区，保存。

挂载指令：

```bash
mkdir -p /tmp/introot
mkdir -p /tmp/extroot
mount --bind / /tmp/introot
mount /dev/mmcblk1p4 /tmp/extroot
tar -C /tmp/introot -cvf - . | tar -C /tmp/extroot -xf -
umount /tmp/introot
umount /tmp/extroot
```

完成后 `reboot` 重启。

---

# 二、安装 Home Assistant

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
