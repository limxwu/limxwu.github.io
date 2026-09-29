---
title: Debian12 SambaCIFS安装与配置
date: 2023-12-22
---
# 1.背景

主机pudding上部署了多个服务，现需要pudding向局域网内提供文件共享服务，服务对象包括不限于windows、android、ios、linux、k8s node以及电视盒子等。

# 2.规划

文件共享服务需要满足以下条件：

- 支持多设备、多平台、易操作。需要向多种OS提供文件共享服务且易于浏览
- 支持多用户的文件权限控制，各个用户之间既能相互共享文件又能对私有文件保持隔离性
- 目前主机pudding已经投入使用，文件共享服务不能对现有服务造成影响

现常见的文件共享协议有：[SMB](https://en.wikipedia.org/wiki/Server_Message_Block)、[NFS](https://en.wikipedia.org/wiki/Network_File_System)、[FTP](https://en.wikipedia.org/wiki/File_Transfer_Protocol)、[对象存储](https://en.wikipedia.org/wiki/Amazon_S3)、[iSCSI](https://en.wikipedia.org/wiki/ISCSI)、FC SAN，关于这几种协议的对比可以参考这篇[《**NFS、FTP、SMB、WebDav、DLNA协议，傻傻分不清？》**](https://zhuanlan.zhihu.com/p/411161467)，其中iSCSI需要对磁盘重新分区，FC SAN需要光纤网络的支持，现实条件不满足，对比下来SMB在设备兼容性以及权限控制方面有着其他协议不可替代的优势，在技术选型方面使用SMB作为局域网内文件共享协议。

# 3.概览

**服务器消息块( Service Message Block , SMB )** 是一种通信协议，主要由配备Microsoft Windows的计算机使用，通常用于在网络上的节点之间共享文件、打印机、串行端口和各种通信。它使用NTLM或Kerberos协议进行用户身份验证。它还提供了经过身份验证的进程间通信（IPC）机制。

服务器消息块 (SMB) 支持通过计算机网络进行文件共享、打印机共享、网络浏览和进程间通信（通过命名管道）。SMB 是 Microsoft分布式文件系统实施的基础。

# 4.安装

## 4.1.更新系统

```
sudo apt update && sudo apt upgrade -y
```

![2023-11-11-myvkogrn.png](http://s3.limxwu.fun/public/blog/2023-11-11-myvkogrn.png)

## 4.2.安装SMB

```
sudo apt install samba -y
```

![img](http://s3.limxwu.fun/public/blog/2023-11-11-wqqjedta.png)

# 5.配置

打开配置文件：

```
sudo vim /etc/samba/smb.conf
```

![img](http://s3.limxwu.fun/public/blog/2023-11-11-jhzxhzrm.png)

Samba的配置文件为/etc/samba/smb.conf，通过修改这个配置文件来完成我们的各种需求。打开这个配置文件，你会发现很多内容都用 `#` 或者 `;` 注视掉了。先看一下未被注释掉的部分:

```
[global]
        workgroup = MYGROUP
        server string = Samba Server Version %v
        security = user
        passdb backend = tdbsam
        load printers = yes
        cups options = raw
[homes]
        comment = Home Directories
        browseable = no
        writable = yes
[printers]
        comment = All Printers
        path = /var/spool/samba
        browseable = no
        guest ok = no
        writable = no
        printable = yes
```

主要有以上三个部分：[global], [homes], [printers]

[global] 定义全局的配置，workgroup用来定义工作组，相信如果你安装过windows的系统，你会对这个workgroup不陌生。一般情况下，需要我们把这里的MYGROUP改成WORKGROUP（windows默认的工作组名字）。

security = user #这里指定samba的安全等级。关于安全等级有四种：

share：用户不需要账户及密码即可登录samba服务器

user：由提供服务的samba服务器负责检查账户及密码（默认）

server：检查账户及密码的工作由另一台windows或samba服务器负责

domain：指定windows域控制服务器来验证用户的账户及密码。

passdb backend = tdbsam # passdb backend（用户后台），samba有三种用户后台：smbpasswd, tdbsam和ldapsam.

smbpasswd：该方式是使用smb工具smbpasswd给系统用户（真实用户或者虚拟用户）设置一个Samba密码，客户端就用此密码访问Samba资源。smbpasswd在/etc/samba中，有时需要手工创建该文件。

tdbsam：使用数据库文件创建用户数据库。数据库文件叫passdb.tdb，在/etc/samba中。passdb.tdb用户数据库可使用 `smbpasswd -a` 创建Samba用户，要创建的Samba用户必须先是系统用户。也可使用pdbedit创建Samba账户。pdbedit参数很多，列出几个主要的：

pdbedit -a username：新建Samba账户。

pdbedit -x username：删除Samba账户。

pdbedit -L：列出Samba用户列表，读取passdb.tdb数据库文件。

pdbedit -Lv：列出Samba用户列表详细信息。

pdbedit -c “[D]” -u username：暂停该Samba用户账号。

pdbedit -c “[]” -u username：恢复该Samba用户账号。

ldapsam：基于LDAP账户管理方式验证用户。首先要建立LDAP服务，设置 “passdb backend = ldapsam:ldap://LDAP Server”

load printers 和 cups options 两个参数用来设置打印机相关。

除了这些参数外，还有几个参数需要你了解：

netbios name = MYSERVER # 设置出现在网上邻居中的主机名

hosts allow = 127. 192.168.12. 192.168.13. # 用来设置允许的主机，如果在前面加 ”;” 则表示允许所有主机

log file = /var/log/samba/%m.log #定义samba的日志，这里的%m是上面的netbios name

max log size = 50 # 指定日志的最大容量，单位是K

[homes] 该部分内容共享用户自己的家目录，也就是说，当用户登录到samba服务器上时实际上是进入到了该用户的家目录，用户登陆后，共享名不是homes而是用户自己的标识符，对于单纯的文件共享的环境来说，这部分可以注视掉。

[printers] 该部分内容设置打印机共享。

此处将[home]下的**read only**设置为no，并通过pdbedit -a username新建用户。

# 6.连接

windows文件管理器中添加一个网络位置

![img](http://s3.limxwu.fun/public/blog/2023-11-12-bsorkput.png)

输入格式为：\\ip\username

![img](http://s3.limxwu.fun/public/blog/2023-11-12-hqlloamm.png)

输入账号及密码后下一步直到创建完成，成功挂载目录。

![img](http://s3.limxwu.fun/public/blog/2023-11-12-aeoeubyy.png)