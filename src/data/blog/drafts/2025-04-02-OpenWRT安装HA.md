---
author: orange
pubDatetime: 2025-04-02
title: OpenWRT安装HA
draft: true
tags: [OpenWRT, HA, Home Assistant]
description: OpenWRT系统安装Home Assistant智能家居平台的详细教程。
---

## 固件下载

https://github.com/ophub/amlogic-s9xxx-openwrt/releases

## 网络配置

OpenWRT 安装好后，默认 IP 是 192.168.1.1，如果路由器的网段不是 192.168.1.1，可能会导致默认连不上，需要先把连接 OpenWRT 对应的网络适配器设置手动 IP，网关设为 192.168.1.1。

### 方法一：命令行修改

```bash
nano /etc/config/network
```

修改 lan 口的 ipaddr 配置，可以设置为自己想要的局域网网段，经过尝试，最好是和路由器前缀保持同步，比如我的路由器是 192.168.31.1，我就可以把 OpenWRT 里的 IP 设置为 192.168.31.2。

### 方法二：页面修改

也可以在 OpenWRT 管理页面里找到"网络 - 接口"，页面里选择 lan 口，ipv4 地址设置为 192.168.31.2。DNS 服务器设置为路由器 IP。

## 扩容

系统 - 软件包（空闲空间较少，需要扩容）

Docker - 概览（Docker 根目录 可用空间少，需要扩容）

打开 Shell 连接工具：FinaShell、MobaXterm、XShell 等

```bash
lsblk    # 列出分区信息
cfdisk   # 分区工具
```

找不到指令时，执行：

```bash
opkg update
opkg install cfdisk
```

### 更换软件源

安装源时有连接问题时，更换软件源，"系统 - 软件包 - 配置"，snapshots/ 往前的部分全部替换成可用源：

```text
src/gz openwrt_core https://mirrors.ustc.edu.cn/openwrt/releases/24.10.0/targets/armsr/armv8/packages
src/gz openwrt_base https://mirrors.ustc.edu.cn/openwrt/releases/24.10.0/packages/aarch64_generic/base
src/gz openwrt_luci https://mirrors.ustc.edu.cn/openwrt/releases/24.10.0/packages/aarch64_generic/luci
src/gz openwrt_packages https://mirrors.ustc.edu.cn/openwrt/releases/24.10.0/packages/aarch64_generic/packages
src/gz openwrt_routing https://mirrors.ustc.edu.cn/openwrt/releases/24.10.0/packages/aarch64_generic/routing
src/gz openwrt_telephony https://mirrors.ustc.edu.cn/openwrt/releases/24.10.0/packages/aarch64_generic/telephony
```

## 安装 OpenClash 依赖

```bash
opkg install bash iptables dnsmasq-full curl ca-bundle ipset ip-full iptables-mod-tproxy iptables-mod-extra ruby ruby-yaml kmod-tun kmod-inet-diag unzip luci-compat luci luci-base
opkg install cfdisk libblkid1 libfdisk1
```

用 cfdisk 改好分区表后重启系统即可（可能用到指令 `partprobe /dev/mmcblk2`、`parted /dev/mmcblk2`）

然后在 docker 设置里修改 root 路径。

### 分区操作

```bash
parted /dev/mmcblk2
(parted) rm 3                # 删除 p3
(parted) rm 4                # 删除 p4
(parted) mkpart primary ext4 原p3起点 50GB  # 创建 Docker 分区
(parted) mkpart primary ext4 50GB 100%     # 创建共享存储分区
(parted) quit
```

```bash
umount -l /mnt/mmcblk2p4
umount -l /mnt
```

### 创建软链接

```bash
ln -sf /var/run/docker.sock /run/docker.sock
ln -sf /var/run/dbus /run/dbus
```

## 安装 Home Assistant Supervised

```bash
docker run -d --name hassio_supervisor --privileged \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/run/dbus:/var/run/dbus \
  -v /opt/docker/hassio:/data \
  -e SUPERVISOR_SHARE="/opt/docker/hassio" \
  -e SUPERVISOR_NAME=hassio_supervisor \
  -e SUPERVISOR_MACHINE="qemuarm-64" \
  -e HOMEASSISTANT_REPOSITORY="homeassistant/qemuarm-64-homeassistant" \
  --restart unless-stopped \
  homeassistant/aarch64-hassio-supervisor
```

默认源的包比较旧，改为从 ghcr.io 取包，能取到最新：

参考：https://github.com/home-assistant/supervisor/pkgs/container/aarch64-hassio-supervisor

```bash
docker run -d --name hassio_supervisor --privileged \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/run/dbus:/var/run/dbus \
  -v /opt/docker/hassio:/data \
  -e SUPERVISOR_SHARE="/opt/docker/hassio" \
  -e SUPERVISOR_NAME=hassio_supervisor \
  -e SUPERVISOR_MACHINE="qemuarm-64" \
  -e HOMEASSISTANT_REPOSITORY="homeassistant/qemuarm-64-homeassistant" \
  --restart unless-stopped \
  ghcr.io/home-assistant/aarch64-hassio-supervisor
```

### 常见问题

创建 `/opt/docker/hassio/jobs.json`：

```json
{"ignore_conditions": ["healthy"]}
```

查看日志：

```bash
docker logs hassio_supervisor
```

挂载共享目录：

```bash
mount --make-shared /
```

安装 HACS：

```bash
wget -O - https://get.hacs.xyz | bash -
```

---

## Armbian 相关

参考链接：

- https://github.com/ophub/amlogic-s9xxx-armbian/blob/main/README.cn.md
- https://wkdaily.cpolar.top/15

初始密码：`1234`

### Docker 运行 OpenWRT

```bash
docker run --name immortalwrt --restart=always -d --network macnet --privileged immortalwrt-image:latest /sbin/init
```

改为自启动：

```bash
docker update --restart=always immortalwrt
```

### Docker 备份与恢复

```bash
# 备份
docker export mycontainer > mycontainer.tar

# 恢复
docker import mycontainer.tar mycontainer:backup
```

### 创建 Macvlan 网络

```bash
docker network create -d macvlan \
  --subnet=192.168.31.0/24 \
  --gateway=192.168.31.1 \
  -o parent=eth0 \
  macnet
```

### 配置 Macvlan Shim

```bash
ip link add macvlan-shim link eth0 type macvlan mode bridge
ip addr add 192.168.31.9/24 dev macvlan-shim
ip link set macvlan-shim up
```

```bash
ip route add 192.168.31.0/24 dev macvlan-shim
```

### Netplan 配置

```yaml
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    eth0:
      dhcp4: yes
      addresses:
        - 192.168.31.9/24
      routes:
        - to: default
          via: 192.168.31.1
        - to: 0.0.0.0/0
          via: 192.168.31.2
          metric: 100
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
  vlans:
    macvlan-shim:
      id: 100
      link: eth0
      addresses: [192.168.31.9/24]
      routes:
        - to: 192.168.31.0/24
          via: 0.0.0.0
          scope: link
```

保存后，执行 `netplan try` 检查是否有错误，然后 `netplan apply` 应用。

```bash
# 生成配置并验证语法
sudo netplan generate

# 应用配置
sudo netplan apply

# 重启 NetworkManager
sudo systemctl restart NetworkManager
```

### 静态 IP 配置

编辑 `/etc/network/interfaces`：

```bash
nano /etc/network/interfaces
source /etc/network/interfaces.d/*
```

创建 `/etc/network/interfaces.d/eth0`：

```bash
auto eth0
iface eth0 inet static
    address 192.168.31.9/24
    gateway 192.168.31.1
    dns-nameservers 192.168.31.1 114.114.114.114
    hwaddress ether 01:02:03:04:05:06
    up ip link set $IFACE promisc on
```

文件建好后，重启 N1：

```bash
systemctl reboot
```

检查网络状态：

```bash
systemctl status systemd-networkd
```
