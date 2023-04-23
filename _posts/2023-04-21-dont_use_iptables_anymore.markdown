---
layout: post
title: "别再用iptables了"
date: 2023-04-21 18:27:44 +0800
categories: 笔记
---

[Netfilter团队](https://www.netfilter.org)的精力都放在了[nftables](https://www.netfilter.org/projects/nftables/index.html)项目上。这个才是Linux的现役IP协议层管理工具。

iptables已经是一个处于legacy状态的项目，不会有新的feature增加。如果看看iptables项目[最近的提交](https://git.netfilter.org/iptables/log/)，我们可以发现，其大多数提交都与nftables的兼容性相关。

在当今的主流Linux发行版中：
- Debian，Fedora/RHEL系提供的iptables二进制执行文件，其本质上是iptables-nft，是对nftables的封装
- openSUSE，Alpine，Arch则提供的是iptables的原生legacy版本

可以看出来，目前Linux生态对于iptables的态度仅仅是维持原有的功能。发展已经停滞。

## nftables vs iptables

那么新的nftables究竟有哪些优势呢？

举个例子，一台服务器需要处理5个客户的流量，各自有不同的来源/目标CIDR。如果有一天，老板告诉你，客户C不合作了，我们需要与其有关的规则全部干掉。

假如我们使用iptables来管理这些规则，那这五个客户的规则都混杂在一起，全部设置在nat/filter表的某些链上。那我们需要仔细的查看所有的规则，挑出与C相关的规则，然后一一删除。

```
➜  sudo iptables -L -t nat
Chain PREROUTING (policy ACCEPT)
target  prot opt source     destination
DNAT    tcp  --  anywhere   <client-A-host-1>    tcp dpt:<client-a-port> to:<ip1>:<port1>
DNAT    tcp  --  anywhere   <client-A-host-2>    tcp dpt:<client-a-port> to:<ip2>:<port2>
DNAT    tcp  --  anywhere   <client-A-host-3>    tcp dpt:<client-a-port> to:<ip3>:<port3>
DNAT    tcp  --  anywhere   <client-A-host-4>    tcp dpt:<client-a-port> to:<ip4>:<port4>
DNAT    tcp  --  anywhere   <client-B-host-5>    tcp dpt:<client-b-port> to:<ip5>:<port5>
DNAT    tcp  --  anywhere   <client-B-host-1>    tcp dpt:<client-b-port> to:<ip6>:<port6>
DNAT    tcp  --  anywhere   <client-B-host-2>    tcp dpt:<client-b-port> to:<ip7>:<port7>
DNAT    tcp  --  anywhere   <client-C-host-1>    tcp dpt:<client-c-port> to:<ip8>:<port8>
DNAT    tcp  --  anywhere   <client-C-host-2>    tcp dpt:<client-c-port> to:<ip9>:<port9>
DNAT    tcp  --  anywhere   <client-C-host-3>    tcp dpt:<client-c-port> to:<ip10>:<port10>
DNAT    tcp  --  anywhere   <client-D-host-1>    tcp dpt:<client-d-port> to:<ip11>:<port11>
DNAT    tcp  --  anywhere   <client-D-host-2>    tcp dpt:<client-d-port> to:<ip12>:<port12>
DNAT    tcp  --  anywhere   <client-D-host-3>    tcp dpt:<client-d-port> to:<ip13>:<port13>
DNAT    tcp  --  anywhere   <client-E-host-1>    tcp dpt:<client-e-port> to:<ip14>:<port14>
DNAT    tcp  --  anywhere   <client-E-host-2>    tcp dpt:<client-e-port> to:<ip15>:<port15>
DNAT    tcp  --  anywhere   <client-E-host-3>    tcp dpt:<client-e-port> to:<ip16>:<port16>
DNAT    tcp  --  anywhere   <client-E-host-4>    tcp dpt:<client-e-port> to:<ip17>:<port17>
DNAT    tcp  --  anywhere   <client-E-host-5>    tcp dpt:<client-e-port> to:<ip18>:<port18>
...
```

如果我们用的nftables，那么我们可以给每个用户建立自己的专属表。各自的规则都在自己对应的表内。如果不需要了，直接删除对应的表就好了。

```
➜  sudo nft list ruleset
table inet client_a {
    chain source_aaa {
        type nat hook prerouting priority raw + 10; policy accept;
        tcp daddr <client-A-host-1> dport <client-a-port> counter packets 0 bytes 0 dnat to <ip1>:<port1>
        ...
    }
    chain source_bbb {
        type nat hook prerouting priority raw + 10; policy accept;
        tcp daddr <client-A-host-3> dport <client-a-port> counter packets 0 bytes 0 dnat to <ip3>:<port3>
        ...
    }
}
table inet client_b {
    chain source_bbb {
        type nat hook prerouting priority raw + 10; policy accept;
        tcp daddr <client-B-host-1> dport <client-b-port> counter packets 0 bytes 0 dnat to <ip5>:<port5>
        ...
    }
}

...
```

不同于iptables把系统层面的表filter/nat直接暴露给用户，nftables将系统表抽象为类型。用户可以任意创建逻辑层面的表，并在其中的链中另行定义对应的系统表类型。这就允许用户按照业务场景，抽象出更多逻辑表，将各种规则组织成为结构体系，从而便于管理。

显而易见，nftables在管理大量规则的场景中更有优势。

## 从iptables迁移至nftables

对于单独的iptables语句，我们可以使用`iptables-translate`直接翻译出对应的nft命令。

```
# iptables-translate -t nat -A PREROUTING -p tcp --dport 1000 -j DNAT --to-destination 1.1.1.1:1234
nft add rule ip nat PREROUTING tcp dport 1000 counter dnat to 1.1.1.1:1234
```

对于已经建立好的iptables规则体系，我们可以导出并统一翻译。

```
# iptables-save > backup.iptables
# iptables-restore-translate -f backup.iptables > backup.nft
# nft -f backup.nft
```

## 总结

尽管无论是iptables还是nftables，都只是一个IP协议包的控制工具。但是nftables可以将控制规则组织出一个良好的逻辑结构，在管理复杂规则场景中有更大的优势。

## 延伸阅读

- [nftables - wiki](https://wiki.nftables.org/wiki-nftables/index.php/Main_Page)
- [RHEL Manual - Chapter 2. Getting started with nftables](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/configuring_firewalls_and_packet_filters/getting-started-with-nftables_firewall-packet-filters)
- [nftables - Gentoo Wiki](https://wiki.gentoo.org/wiki/Nftables)
- [nftables - ArchWiki](https://wiki.archlinux.org/title/Nftables)
