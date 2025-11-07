---
title: OpenWRT安装HA
author: orange
categories: [OpenWRT,HA,Home Assistant]
tags: [http, https]
---
https://github.com/ophub/amlogic-s9xxx-openwrt/releases

OpenWRT安装好后，默认IP是192.168.1.1，如果路由器的网段不是192.168.1.1，可能会导致默认连不上，需要先把连接OpenWRT对应的网络适配器设置手动IP，网关设为192.168.1.1。

nano /etc/config/network

修改lan口的ipaddr配置，可以设置为自己想要的局域网网段，经过尝试，最好是和路由器前缀保持同步，比如我的路由器是192.168.31.1，我就可以把OpenWRT里的IP设置为192.168.31.2

也可以在OpenWRT管理页面里找到“网络-接口”，页面里选择lan口，ipv4地址设置为192.168.31.2。DNS服务器设置为路由器IP

扩容：

系统-软件包（空闲空间较少，需要扩容）

Docker-概览（Docker根目录 可用空间少，需要扩容）

打开Shell连接工具：FinaShell、MobaXterm、XShell等

lsblk（列出分区信息）

cfdisk（分区工具）

找不到指令时，执行opkg update和opkg install cfdisk

安装源时有连接问题时，更换软件源，“系统-软件包-配置”

snapshots/往前的部分全部替换成可用源

```
src/gz openwrt_core https://mirrors.ustc.edu.cn/openwrt/releases/24.10.0/targets/armsr/armv8/packages
src/gz openwrt_base https://mirrors.ustc.edu.cn/openwrt/releases/24.10.0/packages/aarch64_generic/base
src/gz openwrt_luci https://mirrors.ustc.edu.cn/openwrt/releases/24.10.0/packages/aarch64_generic/luci
src/gz openwrt_packages https://mirrors.ustc.edu.cn/openwrt/releases/24.10.0/packages/aarch64_generic/packages
src/gz openwrt_routing https://mirrors.ustc.edu.cn/openwrt/releases/24.10.0/packages/aarch64_generic/routing
src/gz openwrt_telephony https://mirrors.ustc.edu.cn/openwrt/releases/24.10.0/packages/aarch64_generic/telephony

```

OpenClash依赖

opkg install bash iptables dnsmasq-full curl ca-bundle ipset ip-full iptables-mod-tproxy iptables-mod-extra ruby ruby-yaml kmod-tun kmod-inet-diag unzip luci-compat luci luci-base

opkg install cfdisk libblkid1 libfdisk1

用cfdisk改好分区表后重启系统即可（可能用到指令partprobe /dev/mmcblk2 、parted /dev/mmcblk2）

然后在docker设置里修改root路径

parted /dev/mmcblk2
(parted) rm 3                # 删除 p3
(parted) rm 4                # 删除 p4
(parted) mkpart primary ext4 原p3起点 50GB  # 创建 Docker 分区
(parted) mkpart primary ext4 50GB 100%     # 创建共享存储分区
(parted) quit

umount -l /mnt/mmcblk2p4

umount -l /mnt

```
ln -sf /var/run/docker.sock /run/docker.sock
ln -sf /var/run/dbus /run/dbus
```

```

```

```
docker run -d --name hassio_supervisor --privileged \
-v /var/run/docker.sock:/var/run/docker.sock \
-v /var/run/dbus:/var/run/dbus \
-v /opt/docker/hassio:/data \
-e SUPERVISOR_SHARE="/opt/docker/hassio" \
-e SUPERVISOR_NAME=hassio_supervisor \
-e SUPERVISOR_MACHINE="qemuarm-64" \
-e HOMEASSISTANT_REPOSITORY="homeassistant/qemuarm-64-homeassistant" \
--restart unless-stopped homeassistant/aarch64-hassio-supervisor
```

默认源的包比较旧，改为从ghcr.io取包，能取到最新

https://github.com/home-assistant/supervisor/pkgs/container/aarch64-hassio-supervisor

```
docker run -d --name hassio_supervisor --privileged \
-v /var/run/docker.sock:/var/run/docker.sock \
-v /var/run/dbus:/var/run/dbus \
-v /opt/docker/hassio:/data \
-e SUPERVISOR_SHARE="/opt/docker/hassio" \
-e SUPERVISOR_NAME=hassio_supervisor \
-e SUPERVISOR_MACHINE="qemuarm-64" \
-e HOMEASSISTANT_REPOSITORY="homeassistant/qemuarm-64-homeassistant" \
--restart unless-stopped ghcr.io/home-assistant/aarch64-hassio-supervisor
```

aiohttp

/opt/docker/hassio/jobs.json

{"ignore_conditions": ["healthy"]}

docker logs hassio_supervisor

mount --make-shared /

/mnt/mmcblk2p4/docker/hassio/homeassistant

wget -O - https://get.hacs.xyz | bash -

https://github.com/wuwentao/midea_ac_lan.git

node-red启动失败

配置里关闭ssl，保存

Armbian

https://github.com/ophub/amlogic-s9xxx-armbian/blob/main/README.cn.md

https://wkdaily.cpolar.top/15

初始密码1234

docker run --name immortalwrt --restart=always -d --network macnet --privileged immortalwrt-image:latest /sbin/init

改为自启动

docker update --restart=always immortalwrt

备份

docker export mycontainer > mycontainer.tar

恢复

docker import mycontainer.tar mycontainer:backup

```
docker network create -d macvlan \
  --subnet=192.168.31.0/24 \
  --gateway=192.168.31.1 \
  -o parent=eth0 \
  macnet
```

```
ip link add macvlan-shim link eth0 type macvlan mode bridge
ip addr add 192.168.31.9/24 dev macvlan-shim
ip link set macvlan-shim up
```

```
ip route add 192.168.31.0/24 dev macvlan-shim
```

```
network:
  version: 2
  renderer: NetworkManager  # 使用 NetworkManager 管理
  ethernets:
    eth0:
      dhcp4: yes
      addresses:
        - 192.168.31.9/24     # 宿主机 IP
      routes:
        - to: default
          via: 192.168.31.1    # 主路由 IP（临时，后续切换为旁路由）
        - to: 0.0.0.0/0           # 额外添加默认路由（旁路由 IP）
          via: 192.168.31.2
          metric: 100             # 优先级低于 DHCP 下发的路由（可选）
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
  vlans:
    macvlan-shim:
      id: 100                  # VLAN ID（自定义，非真实 VLAN）
      link: eth0
      addresses: [192.168.31.9/24]  # 与宿主机同 IP（macvlan 特性）
      routes:
        - to: 192.168.31.0/24
          via: 0.0.0.0          # 强制路由到本地接口
          scope: link  
```

保存后，执行netplan try检查是否有错误

执行这个保存应用netplan apply

```
# 生成配置并验证语法
sudo netplan generate

# 应用配置
sudo netplan apply

# 重启 NetworkManager
sudo systemctl restart NetworkManager
```

```
nano /etc/network/interfaces
source /etc/network/interfaces.d/*

armbian网卡设置静态地址的方法:
1. 创建/etc/network/interfaces.d/eth0 文件，内容如下：
#auto eth0
## 设置静态IP地址
#iface eth0 inet static
#        # 自动开启网卡混杂模式
#        up ip link set $IFACE promisc on
#        # 给eth0设置固定的mac地址，自己编一个
#        hwaddress ether 01:02:03:04:05:06
#        # armbian的ip地址
#        address 192.168.31.9
#        broadcast 192.168.31.255
#        netmask 255.255.255.0
#        #  主路由的ip地址
#        gateway 192.168.31.1
#        dns-nameservers 192.168.31.1
#        dns-nameservers 114.114.114.114

# /etc/network/interfaces.d/eth0 内容
auto eth0
iface eth0 inet static
    address 192.168.31.9/24
    gateway 192.168.31.1
    dns-nameservers 192.168.31.1 114.114.114.114
    hwaddress ether 01:02:03:04:05:06
    up ip link set $IFACE promisc on
 
2. 文件建好后，重启N1即可, 重启命令：
systemctl reboot
```

systemctl status systemd-networkd
