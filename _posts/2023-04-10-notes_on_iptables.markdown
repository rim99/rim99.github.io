---
layout: post
title: "iptables笔记"
date: 2023-04-10 17:13:12 +0800
categories: 笔记
---

`iptables`可用于探测、修改、转发甚至丢弃IP协议包，是Linux系统里常用的软件防火墙设施，其底层依赖于Linux内核的netfilter模块。

`iptables`包含以下基础概念：
- 表（table）
- 链（chain）
- 规则（rule）
- 目标（target）

iptables包含5张表，每一张表用于不同的场景。每一张表都预定义有若干默认的规则链。
- `filter`：过滤IP协议包，包含`input`，`forward`，`output`三条链
- `nat`：转发协议包，包含`prerouting`，`output`，`postrouting`三条链
- `raw`，`mangle`，`security`三张表不常用

每条链的作用范围不同
- `input`作用于目标为本机套接字的IP协议包
- `output`作用于本机套接字产生的IP协议包
- `forward`作用于来源且目标均为外部的IP协议包
- `prerouting`作用于刚刚到达本机的IP协议包
- `postrouting`作用于即将离开本机的IP协议包

每条链都含有一组有序的规则。而规则指明对特定IP协议包执行特定动作，其中包含：
- IP协议包需要满足的要求，例如传输层协议，源/目标端口号，IP地址等等
- 目标，也就是期望执行的行为，可以是 `ACCEPT`，`DROP`，`REJECT`等等

当IP协议包进入Linux内核的时候，IP协议包会按照对应的表链中的规则依次尝试匹配。如果匹配成功，就会根据的规则里的目标来执行。如果找不到匹配规则，就会按照链的默认目标来执行。

## 参考资料

- [iptables - ArchWiki - Arch Linux](https://wiki.archlinux.org/title/Iptables)
