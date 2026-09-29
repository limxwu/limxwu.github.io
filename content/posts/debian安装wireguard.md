---
title: debian安装wireguard
date: 2025-08-28 12:00:00
---
# 安装wireguard

    apt install wireguard

# 设置配置文件

在`/etc/wireguard` 路径下创建配置文件wg.conf

    [Interface]
    PrivateKey = ****
    Address = 192.168.xxx.xx/32
    # ListenPort not defined
    DNS = 192.168.xxx.xxx
    
    [Peer]
    PublicKey = ****
    PresharedKey = ****
    AllowedIPs = 0.0.0.0/0, ::/0
    Endpoint = *****:****
    PersistentKeepAlive = 25

启动服务

    wg-quick up wg0

启动服务过程可能由于缺少解析配置文件的依赖导致报错：

    root@debian:/etc/wireguard# wg-quick up wg0
    [#] ip link add wg0 type wireguard
    [#] wg setconf wg0 /dev/fd/63
    [#] ip -4 address add 192.168.***.***/32 dev wg0
    [#] ip link set mtu 1420 up dev wg0
    [#] resolvconf -a wg0 -m 0 -x
    /usr/bin/wg-quick: line 32: resolvconf: command not found
    [#] ip link delete dev wg0

可以安装openresolv解决

    apt install openresolv

创建开机启动

    systemctl enable wg-quick@wg0.service

# 配置转发

    echo net.ipv4.ip_forward=1>>/etc/sysctl.conf && sysctl -p
    
    apt update && apt install iptables
    
    iptables -t nat -A POSTROUTING -j MASQUERADE
    
    apt install iptables-persistent

PS：debian13配置ipv4转发

    echo net.ipv4.ip_forward=1>>/etc/sysctl.conf.d/99-custome.conf && sysctl --system