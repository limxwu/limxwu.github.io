---
title: Openwrt配置BGP
id: 0fe34515-18c5-4b37-ac3a-728c0594beb8
date: 2024-05-08 08:50:44
auther: limxwu
cover: 
excerpt: 安装FRR opkg updateopkg install frr frr-babeld frr-bfdd frr-bgpd frr-eigrpd frr-fabricd frr-isisd frr-ldpd frr-libfrr frr-nhrpd frr-ospf6d frr-ospfd fr
permalink: /archives/1715129443633
categories:
 - 文章
tags: 
 - 网络
 - openwrt
---

## 安装FRR

```shell
opkg update
opkg install frr frr-babeld frr-bfdd frr-bgpd frr-eigrpd frr-fabricd frr-isisd frr-ldpd frr-libfrr frr-nhrpd frr-ospf6d frr-ospfd frr-pbrd frr-pimd frr-ripd frr-ripngd frr-staticd frr-vrrpd frr-vtysh frr-watchfrr frr-zebra
```

或者

```
 opkg update && opkg install quagga quagga-zebra quagga-bgpd quagga-vtysh
```

## 配置

```shell
$ vtysh

OpenWrt# conf t

OpenWrt(config)# router bgp 65000

OpenWrt(config-router)# neighbor 192.168.1.5 remote-as 65001

OpenWrt(config-router)# neighbor 192.168.1.5 description ubuntu-dev1

OpenWrt(config-router)# neighbor 192.168.1.6 remote-as 65001

OpenWrt(config-router)# neighbor 192.168.1.6 description ubuntu-dev2

OpenWrt(config-router)# exit

OpenWrt(config)# exit
```

## 查看

```shell
OpenWrt# show ip bgp summary
BGP router identifier 192.168.1.2, local AS number 65000
RIB entries 0, using 0 bytes of memory
Peers 2, using 18 KiB of memory

Neighbor        V         AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
192.168.1.5     4 65001       0       0        0    0    0 never    Active
192.168.1.6     4 65001       0       0        0    0    0 never    Active

Total number of neighbors 2

Total num. Established sessions 0
Total num. of routes received     0
```
