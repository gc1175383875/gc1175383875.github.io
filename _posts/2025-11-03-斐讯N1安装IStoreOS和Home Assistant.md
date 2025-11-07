---
title: 斐讯N1安装IStoreOS和Home Assistant
author: orange
categories: [OpenWRT,HA,Home Assistant]
tags: [http, https]
---
固件地址：https://fw.koolcenter.com/iStoreOS/alpha/n1/

下载后使用balenaEtcher软件，把下载的zip解压出的img镜像，烧写到U盘

把U盘插到N1盒子上，然后通电，等待几分钟，盒子会自动从U盘启动系统。

通过路由器查找IP，用SSH工具，连接查找到的IP，账号root，密码password

连上之后执行`install-to-emmc.sh`

或浏览器直接访问该IP，通过IStoreOS页面配置

网络配置

使用网络向导-手动配置，快速配置静态路由和dhcp等，各配置项按需填写，旁路由做网关时，要打开dhcp。

网络->接口，lan 编辑，DHCP服务器->高级设置，动态DHCP确保开启，DHCP选项填写主路由IP："3,192.168.31.1", "6,192.168.31.1"（经测试，IStoreOS版，直接填192.168.31.1无效，找不到DHCP）。

网络->DHCP/DNS->标签，新增一项

```json
"Proxy":
[
  "3,192.168.31.2", "6,192.168.31.2"
]
```

然后进入主路由的管理页面，关闭DHCP

安装OpenClash

访问`https://github.com/vernesong/OpenClash/releases`，下载OpenClash的IPK版本，用SSH工具移到OpenWrt(IStoreOS)系统里，注意最后一行的文件路径和文件名，按照下载过来的文件移到的路径来调整

执行

```opkg
opkg update
opkg install bash iptables dnsmasq-full curl ca-bundle ipset ip-full iptables-mod-tproxy iptables-mod-extra ruby ruby-yaml kmod-tun kmod-inet-diag unzip luci-compat luci luci-base
opkg install /tmp/openclash.ipk
```

安装完成后，刷新IStoreOS的网页，打开`服务-OpenClash`，自动弹出核心下载，如果进度慢，访问插件设置-版本更新来手动下载安装，按照“内核路径”后面的标识`/etc/openclash/core/clash\_meta`来修改文件名放入对应位置。

---

扩容Overlay（可选）

输入parted（如果下面打印有Using /dev/mmcblkboot0，说明找错了磁盘分区，使用parted /dev/mmcblk1手动选择分区）

```
unit s
resizepart 3 5G
quit
```

然后再执行`resize2fs /dev/mmcblk1p3`

挂载剩余空间

在磁盘管理里，把空闲分区用上，格式化

然后在“挂载点”页面，创建一个新分区的挂载点，设置成根分区，保存

挂载指令

```mkdir
mkdir -p /tmp/introot
mkdir -p /tmp/extroot
mount --bind / /tmp/introot
mount /dev/mmcblk1p4 /tmp/extroot
tar -C /tmp/introot -cvf - . | tar  -C /tmp/extroot -xf -
umount /tmp/introot
umount /tmp/extroot

```

完成后reboot重启

---

opkg install docker-compose

找一个位置创建文件`compose.yaml`，粘贴一下内容，粘贴完成后执行`docker compose up -d`

```yaml
services:
  homeassistant:
    container_name: homeassistant
    image: "ghcr.io/home-assistant/home-assistant:stable"
    volumes:
      - /opt/docker/hassio:/config
      # - /etc/localtime:/etc/localtime:ro
      - /run/dbus:/run/dbus:ro
    environment:
      - TZ=Asia/Shanghai  # 设置时区
    restart: unless-stopped
    privileged: true
    network_mode: host
```

安装HACS：

```
docker exec -it homeassistant bash

wget -O - https://get.hacs.xyz | bash -
```

方法二（安装HA Supervised）：

此方式官方已不再支持，新版本安装不进来，需要在后面加版本号下载24年末的旧版本，好处是Supervised版附带Add-on

/opt/docker/hassio/jobs.json

{"ignore_conditions": ["healthy"]}

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

```
docker run -d --name hassio_supervisor --privileged \
--restart always \
-v /var/run/docker.sock:/var/run/docker.sock \
-v /var/run/dbus:/var/run/dbus \
-v /opt/docker/hassio:/data \
-e SUPERVISOR_SHARE=/opt/docker/hassio \
-e SUPERVISOR_NAME=hassio_supervisor \
-e HOMEASSISTANT_REPOSITORY=homeassistant/qemuarm-64-homeassistant \
ghcr.io/home-assistant/aarch64-hassio-supervisor
```

docker logs -f hassio_supervisor

检查该目录是否存在，不存在要用mkdir创建

/run/udev

添加镜像源

```
https://docker.xuanyuan.me
https://docker.m.daocloud.io
https://docker.imgdb.de
https://docker-0.unsee.tech
https://docker.hlmirror.com
https://docker.1ms.run
```

结尾附上Home Assistant最热门论坛链接

https://bbs.hassbian.com/
