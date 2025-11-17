---
title: OpenClash设置
author: orange
categories: [OpenWrt]
---
一、插件设置

1、模式设置

使用Meta内核

运行模式：Fake-IP(Tun混合)

2、外部控制

设置OpenClash面板登录的账号密码

3、GEO规则订阅

自动更新GEO MMDB数据库

自动更新GEO Site数据库

多了一个自动更新GEO ASN数据库（需要查一下用不用）

4、大陆白名单订阅

自动更新

二、覆写设置

1、DNS设置

自定义上游DNS服务器

FakeIP持久化

Fake-IP-Filter（访问过程中出现问题要配置成直连的IP填入下面文本框）

2、Meta设置

启动TCP并发

启用流量探测

三、配置订阅

1、自动更新（按需设置）

编辑页

1、在线订阅转换

订阅转换模板
