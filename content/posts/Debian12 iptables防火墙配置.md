---
title: Debian12 iptables防火墙配置
id: de6ddee8-5cef-4d4e-a8e3-b8c859bb0047
date: 2024-02-08 09:48:29
auther: limxwu
cover: https://images.unsplash.com/photo-1602604354236-01e2b7739b21?ixid=M3w1NjQ3Mzh8MHwxfHRvcGljfHxibzhqUUtUYUUwWXx8fHx8Mnx8MTcwNzM4Nzc1NHw&ixlib=rb-4.0.3
excerpt: 保存规则 通过 iptables-persistent 保存 netfilter-persistent save 通过iptables保存 要稍后保存规则设置，请使用命令 iptables-save > /etc/iptables/rules.v4ip6tables-save > /etc/ip
permalink: /archives/1707356909106
categories:
 - 文章
tags: 
 - 网络
 - linux
---

## 保存规则

### 通过 iptables-persistent 保存

```
netfilter-persistent save
```

### 通过iptables保存

要稍后保存规则设置，请使用命令

```
iptables-save > /etc/iptables/rules.v4
ip6tables-save > /etc/iptables/rules.v6
```

## 恢复规则

### 现行规则的恢复和替换

```
iptables-restore < /etc/iptables/rules.v4
```

### 恢复同时保持当前规则

```
iptables-restore -n < /etc/iptables/rules.v4
```

## iptables的使用

**语法：**

iptables (选项) (参数)

iptables -t 表名 规则链名 [规则号] -p 协议名 --sport 源端口 --dport 目标端口 -j 动作

| 选项 | 解释                                                                                                                                                                                                                     |
| ---- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| -t   | --table table 对指定的表 table 进行操作， table 必须是 raw， nat，filter，mangle 中的一个。如果不指定此选项，默认的是 filter 表                                                                                          |
| -p   | 指定要匹配的数据包协议类型                                                                                                                                                                                               |
| -s   | --source [!] address[/mask] ：把指定的一个／一组地址作为源地址，按此规则进行过滤。当后面没有 mask 时，address 是一个地址，比如：192.168.1.1；当 mask 指定时，可以表示一组范围内的地址，比如：192.168.1.0/255.255.255.0。 |
| -d   | --destination [!] address[/mask] ：地址格式同上，但这里是指定地址为目的地址，按此进行过滤                                                                                                                                |
| -i   | --in-interface [!] ：指定数据包的来自来自网络接口，比如最常见的 eth0 。注意：它只对 INPUT，FORWARD，PREROUTING 这三个链起作用。如果没有指定此选项， 说明可以来自任何一个网络接口。同前面类似，"!" 表示取反。             |
| -o   | --out-interface [!] ：指定数据包出去的网络接口。只对 OUTPUT，FORWARD，POSTROUTING 三个链起作用                                                                                                                           |
| -L   | --list [chain] 列出链 chain 上面的所有规则，如果没有指定链，列出表上所有链的所有规则。                                                                                                                                   |
| -A   | --append chain rule-specification 在指定链 chain 的末尾插入指定的规则，也就是说，这条规则会被放到最后，最后才会被执行。规则是由后面的匹配来指定。                                                                        |
| -I   | --insert chain [rulenum] rule-specification 在链 chain 中的指定位置插入一条或多条规则。如果指定的规则号是1，则在链的头部插入。这也是默认的情况，如果没有指定规则号。                                                     |
| -D   | --delete chain rule-specification -D, --delete chain rulenum 在指定的链 chain 中删除一个或多个指定规则。                                                                                                                 |
| -R   | --replace 替换/修改第几条规则                                                                                                                                                                                            |
| -F   | --flush [chain] 清空指定链 chain 上面的所有规则。如果没有指定链，清空该表上所有链的所有规则                                                                                                                              |
| -N   | --new-chain chain 用指定的名字创建一个新的链。                                                                                                                                                                           |
| -X   | --delete-chain [chain] ：删除指定的链，这个链必须没有被其它任何规则引用，而且这条上必须没有任何规则。如果没有指定链名，则会删除该表中所有非内置的链。                                                                    |
| -E   | --rename-chain old-chain new-chain ：用指定的新名字去重命名指定的链。这并不会对链内部造成任何影响                                                                                                                        |
| -Z   | --zero [chain] ：把指定链，或者表中的所有链上的所有计数器清零。                                                                                                                                                          |
| -j   | --jump target ：即满足某条件时该执行什么样的动作。target 可以是内置的目标，比如 ACCEPT，也可以是用户自定义的链。                                                                                                         |
| -h   | 显示帮助信息                                                                                                                                                                                                             |

| **参数** | **作用**                             |
| -------- | ------------------------------------ |
| -P       | 设置默认策略:iptables -P INPUT (DROP |
| -F       | 清空规则链                           |
| -L       | 查看规则链                           |
| -A       | 在规则链的末尾加入新规则             |

**四表：**

表的作用：容纳各种规则链

* raw ：确定是否对该数据包进行状态跟踪
* mangle ：修改数据包内容，用来做流量整形的，给数据包设置标记
* nat ：负责网络地址转换，用来修改数据源包中的源、目标IP地址或端口
* filter ：负责过滤数据包，确定是否放行该数据包（过滤）

**五链：**

规则链的作用：容纳各种防火墙规则

* INPUT链 ：处理输入数据包,匹配目标IP为本机的数据包
* OUTPUT链 ：处理输出数据包,一般不在此链上做配置
* FORWARD链 ：处理转发数据包。
* PREROUTING链 ：在进行路由选择前处理数据包，用来修改目的地址，用来做DNAT,相当于把内网服务器的IP和端口映射到路由器的外网IP和端口上
* POSTOUTING链 ：在进行路由选择后处理数据包，用来修改源地址，用来做SNAT,相当于内网通过路由器NAT转换功能实现内网主机通过一个公网IP地址上

### 设置步骤

1. iptables默认规则是全部放通，所以第一步将所有外部流量屏蔽。

   ```bash
   iptables -P INPUT DROP
   iptables -P FORWARD DROP
   ```
2. 放通22端口、ICPM、出流量

   ```bash
   iptables -I INPUT -p tcp --dport 22 -j ACCEPT
   #允许icmp包通过,也就是允许ping
   iptables -A INPUT -p icmp -m icmp --icmp-type 8 -j ACCEPT

   #本机对外请求相当于OUTPUT,对于返回数据包必须接收啊，这相当于INPUT了
   iptables -I INPUT -i lo -j ACCEPT
   iptables -I INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
   ```
3. 保存iptable

   ```bash
   netfilter-persistent save
   ```
