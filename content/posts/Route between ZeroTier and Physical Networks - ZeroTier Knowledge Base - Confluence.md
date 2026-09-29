---
title: Route between ZeroTier and Physical Networks - ZeroTier Knowledge Base - Confluence
id: 9552621d-771a-4769-be52-4d3b95b33f2e
date: 2023-08-24 00:22:55
auther: limxwu
cover: 
excerpt: 本文由 简悦 SimpRead 转码， 原文地址 zerotier.atlassian.net This seems to be the simplest pattern for getting remote access to your LAN. It doesn't require acces.
permalink: /archives/route-between-zerotier-and-physical-networks---zerotier-knowledge-base---confluence
categories:
 - 文章
tags: 
 - 网络
 - openwrt
---

> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [zerotier.atlassian.net](https://zerotier.atlassian.net/wiki/spaces/SD/pages/224395274/Route+between+ZeroTier+and+Physical+Networks)

This seems to be the simplest pattern for getting remote access to your LAN. It doesn't require acces......

This seems to be the simplest pattern for getting remote access to your LAN. It doesn't require access to the LAN's router or have some of the pitfalls of bridging. This requires a Linux PC or VM, something that runs iptables, on your LAN. A raspberrypi works. This is a [NAT/Masquerade](https://en.wikipedia.org/wiki/Network_address_translation "https://en.wikipedia.org/wiki/Network_address_translation") setup.

Possible Disadvantages:

*   No broadcast/multicast across networks (but the mobile OS's don't allow this anyways).

*   Can't initiate connections _from_ the LAN _to_ an external ZeroTier client.

### Summary

*   Install ZeroTier

*   Add a managed route to the ZeroTier network (at [my.zerotier.com](http://my.zerotier.com/ "http://my.zerotier.com"))

*   Enable IP Forwarding

*   Configure iptables

### Required information

For Example:

<table data-number-column="false"><tbody><tr><th rowspan="1" colspan="1" color=""><p data-renderer-start-pos="671">Info</p></th><th rowspan="1" colspan="1" color=""><p data-renderer-start-pos="679">Example</p></th><th rowspan="1" colspan="1" color=""><p data-renderer-start-pos="690">Shorthand Name Below</p></th></tr></tbody></table><table data-number-column="false"><tbody><tr><th rowspan="1" colspan="1" color=""><p data-renderer-start-pos="671">Info</p></th><th rowspan="1" colspan="1" color=""><p data-renderer-start-pos="679">Example</p></th><th rowspan="1" colspan="1" color=""><p data-renderer-start-pos="690">Shorthand Name Below</p></th></tr><tr><td rowspan="1" colspan="1" color=""><p data-renderer-start-pos="716">ZeroTier Network ID</p></td><td rowspan="1" colspan="1" color=""><p data-renderer-start-pos="739">d5e04297a19bbd70</p></td><td rowspan="1" colspan="1" color=""><p data-renderer-start-pos="759">$NETWORK_ID</p></td></tr><tr><td rowspan="1" colspan="1" color=""><p data-renderer-start-pos="776">ZeroTier Interface Name</p></td><td rowspan="1" colspan="1" color=""><p data-renderer-start-pos="803">zt7nnig26</p></td><td rowspan="1" colspan="1" color=""><p data-renderer-start-pos="816">$ZT_IFACE</p></td></tr><tr><td rowspan="1" colspan="1" color=""><p data-renderer-start-pos="831">Physical Interface Name</p></td><td rowspan="1" colspan="1" color=""><p data-renderer-start-pos="858">eth0</p></td><td rowspan="1" colspan="1" color=""><p data-renderer-start-pos="866">$PHY_IFACE</p></td></tr><tr><td rowspan="1" colspan="1" color=""><p data-renderer-start-pos="882">ZeroTier subnet</p></td><td rowspan="1" colspan="1" color=""><p data-renderer-start-pos="901">172.27.0.0/16</p></td><td rowspan="1" colspan="1" color=""></td></tr><tr><td rowspan="1" colspan="1" color=""><p data-renderer-start-pos="925">Physical subnet</p></td><td rowspan="1" colspan="1" color=""><p data-renderer-start-pos="944">192.168.100.0/24</p></td><td rowspan="1" colspan="1" color=""><p data-renderer-start-pos="964">$PHY_SUB</p></td></tr><tr><td rowspan="1" colspan="1" color=""><p data-renderer-start-pos="978">ZeroTier IP Address of "Router"</p></td><td rowspan="1" colspan="1" color=""><p data-renderer-start-pos="1013">172.27.0.1</p></td><td rowspan="1" colspan="1" color=""><p data-renderer-start-pos="1027">$ZT_ADDR</p></td></tr></tbody></table>

### Install ZeroTier

[https://www.zerotier.com/download/](https://www.zerotier.com/download/ "https://www.zerotier.com/download/")

`1 2` `sudo zerotier-cli join $NETWORK_ID sudo zerotier-cli listnetworks`

Authorize it at [my.zerotier.com/network/$NETWORK_ID](http://my.zerotier.com/network/$NETWORK_ID "http://my.zerotier.com/network/$NETWORK_ID")

The _listnetworks_ output has the ZeroTier Interface name under <dev>

### Configure the ZeroTier managed route

At [my.zerotier.com/network/$NETWORK_ID](http://my.zerotier.com/network/$NETWORK_ID "http://my.zerotier.com/network/$NETWORK_ID") -> Settings -> Managed Routes

This adds another route to every device joined to the ZeroTier network.

<table data-number-column="false"><tbody><tr><th rowspan="1" colspan="1" color=""><p data-renderer-start-pos="1469">Destination</p></th><th rowspan="1" colspan="1" color=""><p data-renderer-start-pos="1484">(Via)</p></th></tr><tr><td rowspan="1" colspan="1" color=""><p data-renderer-start-pos="1495">$PHY_SUB</p></td><td rowspan="1" colspan="1" color=""><p data-renderer-start-pos="1507">$ZT_ADDR</p></td></tr></tbody></table>

For example:

<table data-number-column="false"><tbody><tr><th rowspan="1" colspan="1" color=""><p data-renderer-start-pos="1537">Destination</p></th><th rowspan="1" colspan="1" color=""><p data-renderer-start-pos="1552">(Via)</p></th></tr><tr><td rowspan="1" colspan="1" color=""><p data-renderer-start-pos="1563">192.168.100.0/23</p></td><td rowspan="1" colspan="1" color=""><p data-renderer-start-pos="1583">172.27.0.1</p></td></tr></tbody></table>

Configure the destination route as slightly **larger** than the actual physical subnet, here /23 instead of /24 (a smaller number is a bigger subnet in this notation) This makes devices that are on both the physical and the ZeroTier network prefer the physical connection.

### Enable IP forwarding

This can vary depending on linux distribution. Typically:

Edit **/etc/sysctl.conf** to uncomment **net.ipv4.ip_forward**. This enables forwarding at boot.

To enable it now

`1` `sudo sysctl -w net.ipv4.ip_forward=1`

### Configure iptables

Assign some shell variables (personalize these)

`1` `PHY_IFACE=eth0; ZT_IFACE=zt7nnig26`

Add rules to iptables

`1 2 3` `sudo iptables -t nat -A POSTROUTING -o $PHY_IFACE -j MASQUERADE sudo iptables -A FORWARD -i $PHY_IFACE -o $ZT_IFACE -m state --state RELATED,ESTABLISHED -j ACCEPT sudo iptables -A FORWARD -i $ZT_IFACE -o $PHY_IFACE -j ACCEPT`

Save iptables rules for next boot

`1 2` `sudo apt install iptables-persistent sudo bash -c iptables-save > /etc/iptables/rules.v4`

### Test!

*   Turn off wifi on your phone

*   Join it to the zerotier network, authorize it

*   Try to access something on the physical LAN