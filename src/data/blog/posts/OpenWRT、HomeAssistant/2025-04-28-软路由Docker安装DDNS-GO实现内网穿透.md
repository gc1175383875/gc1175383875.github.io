---
author: orange
pubDatetime: 2025-04-28
title: 软路由Docker安装DDNS-GO实现内网穿透
tags: [DDNS, Docker, Cloudflare, 内网穿透]
description: 在软路由上通过Docker安装DDNS-GO，配合Cloudflare实现动态域名解析和内网穿透。
---

## 前言

家庭宽带通常没有固定公网IP，每次重启光猫或路由器后IP都会变化。DDNS（动态域名解析）可以将域名实时解析到当前的公网IP，配合端口转发或Cloudflare Tunnel，即可实现外网访问家中服务。

## Cloudflare 配置

### 1. 添加域名

1. 注册 [Cloudflare](https://dash.cloudflare.com/) 账号
2. 添加你的域名（需要将域名的 DNS 服务器改为 Cloudflare 提供的地址）
3. 等待 DNS 生效（通常几分钟到几小时）

### 2. 创建 API Token

1. 进入 [API Tokens](https://dash.cloudflare.com/profile/api-tokens) 页面
2. 点击「Create Token」
3. 选择「Edit zone DNS」模板，或自定义权限：
   - Zone - DNS - Edit
   - Zone - Zone - Read
4. Zone Resources 选择「Include - All zones」或指定域名
5. 创建后复制 Token，**只显示一次，请妥善保存**

## Docker 安装 DDNS-GO

### 方法一：Docker Compose（推荐）

创建 `compose.yaml` 文件：

```yaml
services:
  ddns-go:
    image: jeessy/ddns-go:latest
    container_name: ddns-go
    restart: unless-stopped
    network_mode: host
    volumes:
      - /opt/docker/ddns-go:/root
    environment:
      - TZ=Asia/Shanghai
```

启动容器：

```bash
docker compose up -d
```

### 方法二：Docker Run

```bash
docker run -d \
  --name ddns-go \
  --restart=unless-stopped \
  --net=host \
  -v /opt/docker/ddns-go:/root \
  -e TZ=Asia/Shanghai \
  jeessy/ddns-go:latest
```

## 配置 DDNS-GO

### 1. 访问管理界面

浏览器打开 `http://软路由IP:9876`

### 2. DNS 服务商配置

选择「Cloudflare」，填入：

- **API Token**：之前创建的 Token
- **域名**：如 `home.example.com` 或 `*.example.com`（泛域名）

### 3. IPv4/IPv6 配置

根据你的网络情况选择：

- **IPv4**：
  - 启用
  - 获取方式：通过接口获取 / 通过命令获取 / 通过URL获取
  - 推荐使用「通过URL获取」：`https://api.ipify.org` 或 `https://ifconfig.me/ip`

- **IPv6**：
  - 如果有公网 IPv6 地址可以启用
  - 获取方式选择对应接口

### 4. 其他设置

- **TTL**：建议 600 秒（10分钟）
- **Webhook**：可选，IP 变化时通知
- **保存配置**

## 端口转发（可选）

如果需要从外网访问内网服务，还需要在路由器上配置端口转发：

1. 进入路由器管理页面
2. 找到「端口转发」或「虚拟服务器」
3. 添加规则：
   - 外部端口：如 443、8080
   - 内部 IP：内网服务地址
   - 内部端口：服务实际端口

## Cloudflare Tunnel（推荐）

相比端口转发，Cloudflare Tunnel 更安全，无需暴露公网端口。

### 安装 Cloudflared

```yaml
services:
  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared
    restart: unless-stopped
    command: tunnel --no-autoupdate run --token <你的Tunnel Token>
    environment:
      - TZ=Asia/Shanghai
```

### 创建 Tunnel

1. 进入 Cloudflare Zero Trust 控制台
2. Networks → Tunnels → Create a tunnel
3. 选择「Cloudflared」
4. 命名并创建，复制生成的 Token
5. 在本地服务配置中添加 Public Hostname

## 验证

```bash
# 查看当前解析的 IP
nslookup home.example.com

# 或
dig home.example.com
```

## 常见问题

### 1. IP 没有更新

- 检查 API Token 权限是否正确
- 查看 DDNS-GO 日志：`docker logs ddns-go`
- 确认公网 IP 获取方式正确

### 2. 无法访问内网服务

- 确认端口转发规则正确
- 检查防火墙是否放行端口
- 尝试使用 Cloudflare Tunnel 替代端口转发

### 3. IPv6 解析问题

- 确认运营商分配了公网 IPv6
- 检查路由器 IPv6 设置
- 部分地区 IPv6 可能不稳定

## 参考链接

- [DDNS-GO GitHub](https://github.com/jeessy2/ddns-go)
- [Cloudflare API Tokens](https://dash.cloudflare.com/profile/api-tokens)
- [Cloudflare Tunnel 文档](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
