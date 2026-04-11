---
author: orange
pubDatetime: 2025-11-16
title: OpenClash设置
tags: [OpenWRT]
description: OpenClash插件的详细设置说明，包括模式设置、DNS配置、订阅配置等。
---

## 一、插件设置

### 1. 模式设置

- 使用 Meta 内核
- 运行模式：Fake-IP(Tun混合)

### 2. 外部控制

设置 OpenClash 面板登录的账号密码

### 3. GEO规则订阅

- 自动更新 GEO MMDB 数据库
- 自动更新 GEO Site 数据库
- 多了一个自动更新 GEO ASN 数据库（需要查一下用不用）

### 4. 大陆白名单订阅

自动更新

## 二、覆写设置

### 1. DNS设置

- 自定义上游 DNS 服务器
- FakeIP 持久化
- Fake-IP-Filter（访问过程中出现问题要配置成直连的 IP 填入下面文本框）

### 2. Meta设置

- 启动 TCP 并发
- 启用流量探测

## 三、配置订阅

### 1. 自动更新

按需设置

### 2. 编辑页

- 在线订阅转换
- 订阅转换模板
