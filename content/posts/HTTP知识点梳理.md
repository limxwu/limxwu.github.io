---
title: HTTP知识点梳理
id: add9e587-c024-49d0-ba3b-7e72fa153b62
date: 2023-08-24 00:17:09
auther: limxwu
cover: 
excerpt: 什么是HTTP HTTP全称Hypertext Transfer Protocol,意为超文本传输协议，是一个应用层用于出书超媒体文档（如HTML)的协议。它是专为浏览器和Web服务器之间的通信而设计的，但它也可用于其他目的。HTTP遵循典型的BS模型，客户端打开连接发出请求，等待收到相应。HTTP
permalink: /archives/httpzhi-shi-dian-shu-li
categories:
 - 文章
tags: 
 - web
---

## 什么是HTTP

HTTP全称Hypertext Transfer Protocol,意为超文本传输协议，是一个应用层用于出书超媒体文档（如HTML)的协议。它是专为浏览器和Web服务器之间的通信而设计的，但它也可用于其他目的。HTTP遵循典型的BS模型，客户端打开连接发出请求，等待收到相应。HTTP是一个无状态协议，表示服务器不会在两个请求之间保留任何数据（状态）。

## HTTP消息的组成

### 请求

URL(统一资源定位器)

请求方法:GET、POST、PUT、PATCH、DELETE、OPTIONS、HEAD....

请求头：Accept、Accept-Encoding、Connection、Authorization、User-Agent、Cookie、Content-Type（application/json、multipart/form-data、application/x-www-from-urlencoded、application/octet-stream....）、Content-Length、Refer、Host....

请求体（GET通常没有）

### 响应

响应码：消息-1xx，成功-2xx,重定向-3xx,请求错误-4xx,服务器错误-5xx

响应头：Content-Type、Content-Length、Content-Encoding、cookie、Cache-Control...

响应体



## 版本变化

HTTP1.0:短链接

HTTP1.1:默认长连接，一个TCP连接可以发起多个HTTP请求；断点续传，请求头添加range

HTTP2.0：

 - 二进制分帧，采用二进制传输（之前都是传输文本），编排后用帧传输
 - 首部压缩，将Header压缩，对于出现过Header传输索引号
 - 多路复用
 - 优先级控制
 - 服务器推送，提前将资源推送到客户端

HTTP3.0:传输层由TCP改为UDP=>线头阻塞问题彻底解决；移动网络更灵活



## HTTPS

HTTP+SSL/TLS

在原有的HTTP基础上添加了加密层SSL/TLS

SSL层的建立：

1.验证证书有效性

2.利用SSL证书中的公钥，由非对称加密协商出对称加密的算法和主密钥

**怎么保证保证服务器给客户端下发的公钥是真正的公钥，而不是中间人伪造的公钥呢？**

**证书如何安全传输，被掉包了怎么办？**

SSL证书包含：颁发者（一般为CA)、使用者（服务器域名）、有效期、服务器公钥、签名算法、签名摘要

签名摘要通过对证书内容HASH后得到摘要，再由CA私钥签名，此签名摘要可由CA公钥结合证书内容验证签名是否有篡改。