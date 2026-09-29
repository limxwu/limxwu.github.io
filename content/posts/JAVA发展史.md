---
title: JAVA发展史
id: 97ca8f6c-a544-4501-a60b-f72baaefd25e
date: 2023-08-24 00:02:20
auther: limxwu
cover: 
excerpt:   Java已经走过了二十多年的发展史了，作为一门长期霸占排行榜前三甲的编程语言，它被创造出来之时似乎并没有想到会有今天的辉煌，但回顾历史，Java的出现与兴起并非偶然，只是比别人早走了一步。(.net---------->(╯°Д°）╯︵ ┻━━━┻）   1991年4月，James Goslin
permalink: /archives/javafa-zhan-shi
categories:
 - 文章
tags: 
 - java
---



&emsp;&emsp;Java已经走过了二十多年的发展史了，作为一门长期霸占排行榜前三甲的编程语言，它被创造出来之时似乎并没有想到会有今天的辉煌，但回顾历史，Java的出现与兴起并非偶然，只是比别人早走了一步。(.net---------->(╯°Д°）╯︵ ┻━━━┻）


![11101967-cbb2666740b47397.png](/upload/11101967-cbb2666740b47397.png)

&emsp;&emsp;1991年4月，James Gosling（JAVA之父，就是上面那个小老头）博士带着一帮小弟开始了绿色计划，这个计划最初目标是为了开发一种能在各种电子产品上跑的程序架构。嗯…这个就是Java前身:Oak。不过似乎并不怎么吃香。

&emsp;&emsp;95年互联网兴起（互联网:Java老小弟，快上车啊🚐），Oak更名Java，在SunWorld大会上正式发布Java1.0（鼓掌，啪啪啪~🌹），并提出了“Write Once，Run Anywhere”。(Java：~~想偷懒么，啊呸，~~加入我吧，带给你足够的便利）

&emsp;&emsp;后面就开始数年份了😁。

&emsp;&emsp;96年1月，发布了第一个Java虚拟机Sun Classic VM。（**敲钟点**，面试官问你VM有哪些时，别只回答Hot Spot了，还可以带个IBM J9，再说个Classic VM，虚拟机之祖，汉语意思经典VM，岂不有逼格）

&emsp;&emsp;97年JDK1.1，代表技术：Jar文件格式、JDBC、JavaBeans、RMI。（你大爷还是厉害啊，看这远见）

&emsp;&emsp;98年JDK1.2，发展出了三个方向：Java2SE（面向桌面）、Java2EE（面向企业级）、Java2ME（面向移动端，小时候玩过手机上的Java游戏么，不过不是现在的Android），哦，对了JVM里还有了编译器，以前都是解释执行，加了编译器后Java运行速度就上去了。

&emsp;&emsp;99年HotSpot诞生，小公司开发的。（Sun:不错啊，小伙子，开个价吧，我买了）

&emsp;&emsp;00年JDK1.3，扩充了下类库。

&emsp;&emsp;02年JDK1.4，成熟，各大巨头加盟Compaq、Fukitsu、SAS、Symbian、IBM等（哭晕，就认识个IBM），这个版本有了正则、异常链、NIO、日志类、XML(果然熟了），嗯~今年微软发布了.net。(老大哥，一步慢、步步慢啊）

&emsp;&emsp;XX年JDK1.5(老铁们，猜猜前面哪一年），封箱、泛型、动态注解、枚举、可变长参数、便利循环、改进JVM内存模型。（微软牛逼）

&emsp;&emsp;06年JDK6（忘了说了，上一次发布的时候改叫法了），产品线J2EE、J2SE、J2ME改叫法了=>Java EE6、Java SE6、Java ME6，提供初步动态语言支持、微型动态语言支持，还有继续改虚拟机：锁与同步、垃圾回收、类加载等，还有一件大事Sun把Java开源了，（微软老大哥啊~）开源的版本叫OpenJDK。

&emsp;&emsp;JDK6之后，代码复杂度增加、Java开源、世界经济危机（08年应该没忘了吧），Sun公司在Java以外的事情上耗费了很多精力（就像你想静静的写代码，公司喊你去学企业文化，hehe），JDK两年一个版本速度没了。

&emsp;&emsp;09年，Oracle宣布以74亿美元收购市值曾超过2000亿美元的Sun公司。

&emsp;&emsp;JDK7是一个很重要的版本，很多的项目都使用JDK7不是没有原因的。JDK7原本的规划中有很多业界翘首期盼的项目：

*   Lambda项目：支持Lambda表达式，函数式编程

*   Jigsaw项目：虚拟机层面的模块化支持

*   动态语言支持

*   Garbage-First收集器

*   Coin项目：Java语法细节优化

&emsp;&emsp;可是由于Sun公司泥潭，无力推动JDK7的研发继续进行，公司股票跌至巅峰的3%。

* * *

&emsp;&emsp;Oracle收购Sun公司，实行B计划，也就是该砍的赶紧砍掉，推动JDK7的如期发布。在B计划中，砍掉了Lambda项目、Jigsaw项目和Coin项目的部分。最终JDK7包含的改进有：新的G1收集器，加强对非Java语言的支持，可并行的类加载架构

&emsp;&emsp;Oracle展现出极具商业化的风格。推出了Java SE Support产品，换句话说Oracle不提供免费的更新维护，要不你就用Java SE Support，要不使用JDK后续版本比如JDK8。

&emsp;&emsp;14年JDK8，提供了很多原计划于JDK7中的功能：

*   Lambda支持

*   Nashorn JavaScript引擎的支持

*   新的时间、日期API

*   彻底移除HotSpot永久代

*   ......

&emsp;&emsp;17年JDK9，Jigsaw、增强若干工具（JS Shell、JLink、JHSDB)、整顿HotSpot日志系统等。并且Oracle宣布，每年的3月和9月推出一个大版本。从此每六个大版本JDK才会划出一个支持版（LTS版），也就是JDK8、JDK11以及明年的JDK17是LTS版

&emsp;&emsp;18年3月JDK10，内部重构=>统一源仓库、统一垃圾回收器接口、统一即时编译器接口。  在18年Android侵权Java案落幕，Google赔偿Oracle合计88亿美金（骚操作啊，不到10年就靠一场官司把收购Sun的钱挣回来了还略有小赚，当然Oracle还有其他的一些骚操作不列举了）。

&emsp;&emsp;18年10月JDK11，加入ZGC革命性的垃圾回收器。Oracle将JDK分为Open JDK11和Oracle JDK11两个版本，功能基本一样，差别在于：前者随便用，只有半年更新支持；后者商业收费。

&emsp;&emsp;19年3月JDK12，switch表达式、Java微测试套件、加入Redhat领导开发的shenandoah垃圾收集器。

&emsp;&emsp;19年9月JDK13，动态CDS存档、ZGC取消提交未使用的内存、重写SocketAPI。

&emsp;&emsp;20年3月JDK14，实例的阵列匹配（预览）、包装工具（孵化器）、G1 的 NUMA 感知内存分配  、JFR 事件流式处理、非易失性映射字节缓冲区。

> 本文参考:
>
> [1]  周志明.深入理解JAVA虚拟机.机械工业出版社[M]
>
> [2]  openJdk社区 [http://openjdk.java.net/](http://openjdk.java.net/)