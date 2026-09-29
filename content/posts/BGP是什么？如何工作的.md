---
title: BGP是什么？如何工作的
id: f5d2e19f-ff87-44c4-887d-83fa6cda125f
date: 2024-02-27 11:07:29
auther: limxwu
cover: 
excerpt: 本文由 简悦 SimpRead 转码， 原文地址 info.support.huawei.com BGP（Border Gateway Protocol）是一种用于自治系统 AS（Autonomous System）之间的动态路由协议，BGP4 + 是对 BGP 的扩展。 为什么需要 BGP？ IG
permalink: /archives/1709003249553
categories:
 - 文章
tags: 
 - 网络
---

> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [info.support.huawei.com](https://info.support.huawei.com/info-finder/encyclopedia/zh/BGP.html)

> BGP（Border Gateway Protocol）是一种用于自治系统 AS（Autonomous System）之间的动态路由协议，BGP4 + 是对 BGP 的扩展。

为什么需要 BGP？
----------------

IGP（Interior Gateway Protocol，内部网关协议）被设计用来在单一的路由选择域内提供可达性信息并不适合提供域间路由选择功能，BGP（Border Gateway Protocol，域间路由协议）作为优秀的域间路由协议得以产生并发展。

当今的网络通常使用以下类型的 IGP：

* 距离矢量协议，例如路由信息协议（Routing Information Protocol, RIP）。
* 链路状态协议，例如开放式最短路径优先（Open Shortest Path First, [OSPF](https://info.support.huawei.com/info-finder/encyclopedia/zh/OSPF.html "OSPF")）协议和中间系统到中间系统（Intermediate System to Intermediate System, [IS-IS](https://info.support.huawei.com/info-finder/encyclopedia/zh/IS-IS.html "IS-IS")）协议。

虽然这些协议是为不同目的设计的，并且具有不同的行为特征，但是它们的共同目标是解决在一个路由选择域内的路径最优化问题。IGP 并不适合提供域间路由选择功能。比如说，一种域间路由选择协议应该能够提供广泛的策略控制，因为不同的域通常需要不同的路由选择策略和管理策略。

从一开始，BGP 就被设计成一种域间路由选择协议，其设计目标就是策略控制能力和可扩展性。但是，BGP 也不适合替代 IGP，因为它们适用的场景不同。

BGP 有两种运行方式，当 BGP 运行于同一 AS 内部时，被称为 IBGP（Internel BGP，内部边界网关协议）；当 BGP 运行于不同 AS 之间时，称为 EBGP（Externel BGP，外部边界网关协议）。如图 1 BGP 的运行方式所示：

![](https://download.huawei.com/mdl/image/download?uuid=7aba7e6f8d6b4b1686e48442ba8e7526)
BGP 的运行方式

BGP 是怎么工作的？
------------------

### BGP 报文中的角色

Speaker：发送 BGP 报文的路由设备称为 BGP 发言者（Speaker），它接收或产生新的路由信息，并发布（Advertise）给其它 BGP Speaker。当 BGP Speaker 收到来自其它 AS 的新路由时，如果该路由比当前已知路由更优、或者当前还没有该路由，它就把这条路由发布给所有其他 BGP Speaker（发布该路由的 BGP Speaker 除外）。

Peer：相互交换报文的 BGP Speaker 之间互称对等体（Peer）。

### BGP 的报文

BGP 的运行是通过报文驱动的，共有 Open、Update、Notification、Keepalive 和 Route-refresh 五种报文类型。

* Open 报文：是 TCP 连接建立后发送的第一个报文，用于建立 BGP 对等体之间的连接关系。对等体在接收到 Open 报文并协商成功后，将发送 Keepalive 报文确认并保持连接的有效性。确认后，对等体间可以进行 Update、Notification、Keepalive 和 Route-refresh 报文的交换。
* Update 报文：用于在对等体之间交换路由信息。Update 报文可以发布多条属性相同的可达路由信息，也可以撤销多条不可达路由信息。
* Notification 报文：当 BGP 检测到错误状态时，就向对等体发出 Notification 报文，之后 BGP 连接会立即中断。
* Keepalive 报文：BGP 会周期性地向对等体发出 Keepalive 报文，用来保持连接的有效性。
* Route-refresh 报文：Route-refresh 报文用来请求对等体重新发送所有的可达路由信息。

### BGP 处理过程

因为 BGP 的传输层协议是 TCP 协议，所以在 BGP 对等体建立之前，对等体之间首先进行 TCP 连接。BGP 邻居间会通过 Open 报文协商相关参数，建立起 BGP 对等体关系。建立连接后，BGP 邻居之间交换整个 BGP 路由表。BGP 会发送 Keepalive 报文来维持邻居间的 BGP 连接，BGP 协议不会定期更新路由表，但当 BGP 路由发生变化时，会通过 Update 报文增量地更新路由表。当 BGP 检测到网络中的错误状态时（例如收到错误报文时），BGP 会发送 Notification 报文进行报错，BGP 连接会随即中断。示意图见图 1 邻居建立过程图。

![](https://download.huawei.com/mdl/image/download?uuid=62681d95de1c4d09abbcdfa0513c8714)
邻居建立过程图

### BGP 有限状态机

BGP 有限状态机共有六种状态，分别是 Idle、Connect、Active、Open-Sent、Open-Confirm 和 Established。

在 BGP 对等体建立的过程中，通常可见的三个状态是：Idle、Active、Established。

* Idle 状态下，BGP 拒绝任何进入的连接请求，是 BGP 初始状态。
* Connect 状态下，BGP 等待 TCP 连接的建立完成后再决定后续操作。
* Active 状态下，BGP 将尝试进行 TCP 连接的建立，是 BGP 的中间状态。
* Open-Sent 状态下，BGP 等待对等体的 Open 报文。
* Open-Confirm 状态下，BGP 等待一个 Notification 报文或 Keepalive 报文。
* Established 状态下，BGP 对等体间可以交换 Update 报文、Route-refresh 报文、Keepalive 报文和 Notification 报文。

BGP 对等体双方的状态必须都为 Established，BGP 邻居关系才能成立，双方通过 Update 报文交换路由信息。

![](https://download.huawei.com/mdl/image/download?uuid=19de49dfc35242d8b5365fd70ba51a0e)

### BGP 属性

BGP 路由属性是一套参数，它对特定的路由进一步的描述，使得 BGP 能够对路由进行过滤和选择。事实上，所有的 BGP 路由属性都可以分为以下 4 类：

* 公认必须遵循的（Well-known mandatory）：所有 BGP 设备都可以识别，且必须存在于 Update 报文中。如果缺少这种属性，路由信息就会出错。
* 公认任意（Well-known discretionary）：所有 BGP 设备都可以识别，但不要求必须存在于 Update 报文中，可以根据具体情况来选择。
* 可选过渡（Optional transitive）：在 AS 之间具有可传递性的属性。BGP 设备可以不支持此属性，但它仍然会接收这类属性，并通告给其他对等体。
* 可选非过渡（Optional non-transitive）：如果 BGP 设备不支持此属性，则相应的这类属性会被忽略，且不会通告给其他对等体。

下面介绍几种常用的 BGP 路由属性：

* Origin 属性，属于公认必须遵循属性，用来定义路径信息的来源，标记一条路由是怎么成为 BGP 路由的，包含 IGP、EGP 和 Incomplete 三种类型。
* AS_Path 属性，属于公认必须遵循属性，按矢量顺序记录了某条路由从本地到目的地址所要经过的所有 AS 编号。
* Next_Hop 属性，属于公认必须遵循属性。
* MED，属于可选非过渡属性，MED（Multi-Exit-Discriminator）属性仅在相邻两个 AS 之间传递，收到此属性的 AS 一方不会再将其通告给任何其他第三方 AS。
* Local_Pref 属性，属于公认任意属性，仅在 IBGP 对等体之间有效，不通告给其他 AS，用于表明路由设备的 BGP 优先级。

BGP 是如何处理路由的？
----------------------

BGP 对路由的处理如图 BGP 对路由的处理过程所示。BGP 路由来源包括从其他协议引入和从邻居学习两个部分，为了减少路由规模，可以对优选的 BGP 路由进行聚合。在引入路由、从邻居接收或发送路由的过程中，可以通过路由策略实现对路由的过滤，也可以修改路由的属性。

![](https://download.huawei.com/mdl/image/download?uuid=0d59fda86e9d42f2a0e69ab5f7615780)
BGP 对路由的处理过程

### 路由引入

BGP 协议自身不能发现路由，所以需要引入其他协议的路由（如 IGP 或者静态路由等）注入到 BGP 路由表中，从而将这些路由在 AS 之内和 AS 之间传播。

BGP 引入路由时支持 Import 和 Network 两种方式：Import 方式是按协议类型，将 RIP 路由、[OSPF](https://info.support.huawei.com/info-finder/encyclopedia/zh/OSPF.html "OSPF") 路由、[IS-IS](https://info.support.huawei.com/info-finder/encyclopedia/zh/IS-IS.html "IS-IS") 路由、静态路由和直连路由等某一协议的路由注入到 BGP 路由表中；Network 方式比 Import 方式更精确，将指定前缀和掩码的一条路由注入到 BGP 路由表中。

### 路由选择

当到达同一目的地存在多条路由时，BGP 采取路由选择策略进行路由选择，例如优选没有迭代到 Graceful Down（该 [SRv6](https://info.support.huawei.com/info-finder/encyclopedia/zh/SRv6.html "SRv6") TE-Policy 处于延迟删除状态）的 SRv6 TE-Policy 的路由、在与 RPKI（Resource Public Key Infrastructure）服务器进行连接的情景中，应用起源 AS 验证结果后的 BGP 路由优先级顺序为 Valid > NotFound > Invalid、优选没有误码的路由等。

### 路由聚合

在大规模的网络中，BGP 路由表十分庞大，使用路由聚合（Routes Aggregation）可以大大减小路由表的规模。

路由聚合实际上是将多条路由合并的过程。这样 BGP 在向对等体通告路由时，可以只通告聚合后的路由，而不是通告所有的具体路由。

BGP 路由聚合支持两种方式：

* 自动聚合：对 BGP 引入的路由进行聚合。配置自动聚合后，对参加聚合的具体路由进行抑制。配置自动聚合后，BGP 将按照自然网段聚合路由（如 10.1.1.1/32 和 10.2.1.1/32 将聚合为 A 类地址 10.0.0.0/8），并且 BGP 向对等体只发送聚合后的路由。
* 手动聚合：对 BGP 本地路由进行聚合。手动聚合可以控制聚合路由的属性，以及决定是否发布具体路由。

IPv4 支持自动聚合和手动聚合两种方式，而 [IPv6](https://info.support.huawei.com/info-finder/encyclopedia/zh/IPv6%2B.html "IPv6+") 仅支持手动聚合。

### BGP 发布路由

BGP 发布路由时采用如下策略：

* 存在多条有效路由时，BGP Speaker 只将最优路由发布给对等体。
* BGP Speaker 从 EBGP 获得的路由会向它所有 BGP 对等体发布（包括 EBGP 对等体和 IBGP 对等体）。
* BGP Speaker 从 IBGP 获得的路由不向它的 IBGP 对等体发布。
* BGP Speaker 从 IBGP 获得的路由是否通告给它的 EBGP 对等体要依据 IGP 和 BGP 同步的情况。
* 连接一旦建立，BGP Speaker 将把自己可发布的 BGP 最优路由发布给新对等体。

BGP 支持哪些扩展？
------------------

传统的 BGP-4 只能管理 IPv4 单播路由信息，对于使用其它网络层协议（如 [IPv6](https://info.support.huawei.com/info-finder/encyclopedia/zh/IPv6%2B.html "IPv6+")、[组播](https://info.support.huawei.com/info-finder/encyclopedia/zh/%E7%BB%84%E6%92%AD.html "组播")等）的应用就受到一定限制。

为了提供对多种网络层协议的支持，IETF（Internet Engineering Task Force）对 BGP-4 进行了扩展，形成 MP-BGP（Multi-protocol Extensions for Border Gateway Protocol）。MP-BGP 向前兼容，即支持 BGP 扩展的路由器与不支持 BGP 扩展的路由器可以互通。

MP-BGP 在现有 BGP-4 协议的基础上增强功能，使 BGP 能够为多种路由协议提供路由信息，包括 IPv6（即 BGP4+）和组播。

* MP-BGP 可以同时为单播和组播维护路由信息，将它们储存在不同的路由表中，保持单播和组播之间路由信息相互隔离。
* MP-BGP 可以同时支持单播和组播模式，为两种模式构建不同的网络拓扑结构。
* 原 BGP-4 支持的单播路由策略和配置方法大部分都可应用于组播模式，从而根据路由策略为单播和组播维护不同的路由。

BGP 采用地址族（Address Family）来区分不同的网络层协议，关于地址族的一些取值可以参考相关标准。支持多种 MP-BGP 扩展应用，包括对 VPN 的扩展、对 IPv6 的扩展等，不同的扩展应在各自的地址族视图下配置。

* BGP-IPv4 单播地址族有以下作用：维护公网 BGP 邻居，并且传递公网 IPv4 路由信息；传递公网 IPv4 标签路由，主要用在 Option C 方式的跨域 BGP/MPLS IP VPN 或 Option C 方式的跨域 BGP/MPLS IPv6 VPN 场景里。
* BGP-IPv6 单播地址族有以下作用：维护公网 IPv6 BGP 邻居，并且传递公网 IPv6 路由信息；传递 IPv6 标签路由，主要用于配置 [6PE](https://info.support.huawei.com/info-finder/encyclopedia/zh/6PE.html "6PE") 场景里。
* BGP-IPv4 组播地址族视图、BGP-MVPN 地址族视图、BGP-IPv6 MVPN 地址族视图、BGP-MDT 地址族视图等组播相关地址族可以传输跨 AS 的路由信息，主要应用于 MBGP、BIER、NG MVPN、[BIERv6](https://info.support.huawei.com/info-finder/encyclopedia/zh/BIERv6.html "BIERv6") 和 Rosen MVPN。
* BGP-VPNv4 地址族、BGP-VPNv6 地址族、BGP-VPN 实例视图、BGP 多实例 VPN 实例视图、BGP-L2VPN-AD 地址族视图、BGP-L2VPN-AD 地址族视图等 VPN 相关地址族主要应用于 BGP/MPLS IP VPN、VPWS 以及 VPLS。
* BGP-EVPN 地址族视图、BGP 多实例 [EVPN](https://info.support.huawei.com/info-finder/encyclopedia/zh/EVPN.html "EVPN") 地址族视图等 EVPN 相关地址族主要用于配置 BGP EVPN 对等体，应用于 EVPN VPLS、EVPN VPWS 以及 EVPN L3VPN。EVPN（Ethernet Virtual Private Network）是一种用于二层网络互联的 VPN 技术。EVPN 技术采用类似于 BGP/MPLS IP VPN 的机制，在 BGP 协议的基础上定义了一种新的网络层可达信息 NLRI（Network Layer Reachability Information）即 EVPN NLRI，EVPN NLRI 定义了几种新的 BGP EVPN 路由类型，用于处在二层网络的不同站点之间的 MAC 地址学习和发布。
* BGP IPv4 SR-Policy 地址族视图和 BGP IPv6 SR-Policy 地址族视图主要应用于 [Segment Routing MPLS](https://info.support.huawei.com/info-finder/encyclopedia/zh/SR-MPLS.html "SR-MPLS") 和 [Segment Routing](https://info.support.huawei.com/info-finder/encyclopedia/zh/Segment+Routing.html "Segment Routing") IPv6。
* BGP-Flow 地址族视图、BGP-Flow VPNv4 地址族视图、BGP-Flow VPNv6 地址族视图、BGP-Flow VPN 实例 IPv4 地址族视图、BGP-Flow VPN 实例 IPv6 地址族视图等 Flow 相关地址族主要用于防止 DoS/DDoS 攻击，可以提高网络安全性和可用性。
* BGP-Labeled 地址族视图和 BGP-Labeled-VPN 实例 IPv4 地址族视图的应用主要在于 BGP 分标签方案的运营商配置。
* BGP-LS 地址族视图主要用于汇总 IGP 协议收集的拓扑信息上送给上层控制器。
