---
title: calico部署BGP模式的高可用k8s集群
date: 2024-03-14
categories:
 - 文章
tags: 
 - 容器
 - 网络
 - linux
 - kubernetes
---
 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [tinychen.com](https://tinychen.com/20230109-k8s-13-deploy-ha-k8s-with-calico-bgp/#5-3-%E9%85%8D%E7%BD%AEBGP)
>
> 本文主要在 centos7 系统上基于 containerd 和 v3.24.5 版本的 calico 组件部署 v1.26.0 版本的堆叠 ETCD 高可用 k8s 原生集群，在 LoadBalancer 上选择了 PureLB 和 calico 结合 bird 实现 BGP 路由可达的 K8S 集群部署。

本文主要在 centos7 系统上基于`containerd`和`v3.24.5`版本的 calico 组件部署`v1.26.0`版本的堆叠 ETCD 高可用 k8s 原生集群，在`LoadBalancer`上选择了`PureLB`和`calico`结合`bird`实现 **BGP 路由可达**的 K8S 集群部署。

此前写的一些关于 k8s 基础知识和集群搭建的一些[方案](https://tinychen.com/tags/k8s/)，有需要的同学可以看一下。

[](#1-1-集群信息 "1.1 集群信息")1.1 集群信息[](#1-1-集群信息)
---------------------------------------------

机器均为 16C16G 的虚拟机，硬盘为 100G。

|IP| Hostname                      |
|-|-------------------------------|
|10.31.90.0|k8s-calico-apiserver.tinychen.io|
|10.31.90.1|k8s-calico-master-10-31-90-1.tinychen.io|
|10.31.90.2|k8s-calico-master-10-31-90-2.tinychen.io|
|10.31.90.3|k8s-calico-master-10-31-90-3.tinychen.io|
|10.31.90.4|k8s-calico-worker-10-31-90-4.tinychen.io|
|10.31.90.5|k8s-calico-worker-10-31-90-5.tinychen.io|
|10.31.90.6|k8s-calico-worker-10-31-90-6.tinychen.io|
|10.33.0.0/17|podSubnet|
|10.33.128.0/18|serviceSubnet|
|10.33.192.0/18|LoadBalancerSubnet|

[](#1-2-检查mac和product-uuid "1.2 检查mac和product_uuid")1.2 检查 mac 和 product_uuid[](#1-2-检查mac和product-uuid)
--------------------------------------------------------------------------------------------------------

同一个 k8s 集群内的所有节点需要确保`mac`地址和`product_uuid`均唯一，开始集群初始化之前需要检查相关信息

```shell
ip link 
ifconfig -a


sudo cat /sys/class/dmi/id/product_uuid
```

[](#1-3-配置ssh免密登录（可选） "1.3 配置ssh免密登录（可选）")1.3 配置 ssh 免密登录（可选）[](#1-3-配置ssh免密登录（可选）)
-----------------------------------------------------------------------------------

如果 k8s 集群的节点有多个网卡，确保每个节点能通过正确的网卡互联访问

```shell
su root
ssh-keygen
cd /root/.ssh/
cat id_rsa.pub >> authorized_keys
chmod 600 authorized_keys

cat >> ~/.ssh/config <<EOF
Host k8s-calico-master-10-31-90-1
    HostName 10.31.90.1
    User root
    Port 22
    IdentityFile ~/.ssh/id_rsa

Host k8s-calico-master-10-31-90-2
    HostName 10.31.90.2
    User root
    Port 22
    IdentityFile ~/.ssh/id_rsa

Host k8s-calico-master-10-31-90-3
    HostName 10.31.90.3
    User root
    Port 22
    IdentityFile ~/.ssh/id_rsa

Host k8s-calico-worker-10-31-90-4
    HostName 10.31.90.4
    User root
    Port 22
    IdentityFile ~/.ssh/id_rsa

Host k8s-calico-worker-10-31-90-5
    HostName 10.31.90.5
    User root
    Port 22
    IdentityFile ~/.ssh/id_rsa

Host k8s-calico-worker-10-31-90-6
    HostName 10.31.90.6
    User root
    Port 22
    IdentityFile ~/.ssh/id_rsa
EOF


```

[](#1-4-修改hosts文件 "1.4 修改hosts文件")1.4 修改 hosts 文件[](#1-4-修改hosts文件)
-------------------------------------------------------------------

```shell
cat >> /etc/hosts <<EOF
10.31.90.0 k8s-calico-apiserver k8s-calico-apiserver.tinychen.io
10.31.90.1 k8s-calico-master-10-31-90-1 k8s-calico-master-10-31-90-1.tinychen.io
10.31.90.2 k8s-calico-master-10-31-90-2 k8s-calico-master-10-31-90-2.tinychen.io
10.31.90.3 k8s-calico-master-10-31-90-3 k8s-calico-master-10-31-90-3.tinychen.io
10.31.90.4 k8s-calico-worker-10-31-90-4 k8s-calico-worker-10-31-90-4.tinychen.io
10.31.90.5 k8s-calico-worker-10-31-90-5 k8s-calico-worker-10-31-90-5.tinychen.io
10.31.90.6 k8s-calico-worker-10-31-90-6 k8s-calico-worker-10-31-90-6.tinychen.io
EOF
```

[](#1-5-关闭swap内存 "1.5 关闭swap内存")1.5 关闭 swap 内存[](#1-5-关闭swap内存)
---------------------------------------------------------------

```shell
swapoff -a
sed -i '/swap / s/^(.*)$/#\1/g' /etc/fstab
```
[](#1-6-配置时间同步 "1.6 配置时间同步")1.6 配置时间同步[](#1-6-配置时间同步)
-----------------------------------------------------

这里可以根据自己的习惯选择 ntp 或者是 chrony 同步均可，同步的时间源服务器可以选择阿里云的`ntp1.aliyun.com`或者是国家时间中心的`ntp.ntsc.ac.cn`。

### [](#使用ntp同步 "使用ntp同步")使用 ntp 同步[](#使用ntp同步)

```shell
yum install ntpdate -y
ntpdate ntp.ntsc.ac.cn

hwclock
```

### [](#使用chrony同步 "使用chrony同步")使用 chrony 同步[](#使用chrony同步)

```shell
yum install chrony -y


systemctl enable chronyd.service
systemctl start chronyd.service
systemctl status chronyd.service


vim /etc/chrony.conf


$ grep server /etc/chrony.conf

server 0.centos.pool.ntp.org iburst
server 1.centos.pool.ntp.org iburst
server 2.centos.pool.ntp.org iburst
server 3.centos.pool.ntp.org iburst


$ grep server /etc/chrony.conf

server ntp.ntsc.ac.cn iburst


systemctl restart chronyd.service


chronyc sourcestats -v
chronyc sources -v


```

[](#1-7-关闭selinux "1.7 关闭selinux")1.7 关闭 selinux[](#1-7-关闭selinux)
------------------------------------------------------------------

```shell
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=disabled/' /etc/selinux/config
```

[](#1-8-配置防火墙 "1.8 配置防火墙")1.8 配置防火墙[](#1-8-配置防火墙)
-------------------------------------------------

k8s 集群之间通信和服务暴露需要使用较多端口，为了方便，直接禁用防火墙

```shell
systemctl disable firewalld.service
```

[](#1-9-配置netfilter参数 "1.9 配置netfilter参数")1.9 配置 netfilter 参数[](#1-9-配置netfilter参数)
-----------------------------------------------------------------------------------

这里主要是需要配置内核加载`br_netfilter`和`iptables`放行`ipv6`和`ipv4`的流量，确保集群内的容器能够正常通信。

```shell
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

[](#1-10-配置IPVS "1.10 配置IPVS")1.10 配置 IPVS[](#1-10-配置IPVS)
----------------------------------------------------------

IPVS 是专门设计用来应对负载均衡场景的组件，[kube-proxy 中的 IPVS 实现](https://github.com/kubernetes/kubernetes/blob/master/pkg/proxy/ipvs/README.md#run-kube-proxy-in-ipvs-mode)通过减少对 iptables 的使用来增加可扩展性。在 iptables 输入链中不使用 PREROUTING，而是创建一个假的接口，叫做 kube-ipvs0，当 k8s 集群中的负载均衡配置变多的时候，IPVS 能实现比 iptables 更高效的转发性能。

> 注意在 4.19 之后的内核版本中使用`nf_conntrack`模块来替换了原有的`nf_conntrack_ipv4`模块
>
> (**Notes**: use `nf_conntrack` instead of `nf_conntrack_ipv4` for Linux kernel 4.19 and later)

```shell
sudo yum install ipset ipvsadm -y


modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack


cat <<EOF | sudo tee /etc/modules-load.d/ipvs.conf
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
EOF

$ lsmod | grep -e ip_vs -e nf_conntrack
nf_conntrack_netlink    49152  0
nfnetlink              20480  2 nf_conntrack_netlink
ip_vs_sh               16384  0
ip_vs_wrr              16384  0
ip_vs_rr               16384  0
ip_vs                 159744  6 ip_vs_rr,ip_vs_sh,ip_vs_wrr
nf_conntrack          159744  5 xt_conntrack,nf_nat,nf_conntrack_netlink,xt_MASQUERADE,ip_vs
nf_defrag_ipv4         16384  1 nf_conntrack
nf_defrag_ipv6         24576  2 nf_conntrack,ip_vs
libcrc32c              16384  4 nf_conntrack,nf_nat,xfs,ip_vs
$ cut -f1 -d " "  /proc/modules | grep -e ip_vs -e nf_conntrack
nf_conntrack_netlink
ip_vs_sh
ip_vs_wrr
ip_vs_rr
ip_vs
nf_conntrack


```

[](#2-1-安装containerd "2.1 安装containerd")2.1 安装 containerd[](#2-1-安装containerd)
------------------------------------------------------------------------------

详细的官方文档可以参考[这里](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)，由于在刚发布的 1.24 版本中移除了`docker-shim`，因此安装的`版本≥1.24`的时候需要注意`容器运行时`的选择。这里我们安装的版本为最新的 1.26，因此我们不能继续使用 docker，这里我们将其换为 [containerd](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd)

### [](#修改Linux内核参数 "修改Linux内核参数")修改 Linux 内核参数[](#修改Linux内核参数)

```shell
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter


cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF


sudo sysctl --system


```

### [](#安装containerd "安装containerd")安装 containerd[](#安装containerd)

centos7 比较方便的部署方式是利用已有的 yum 源进行安装，这里我们可以使用 docker 官方的 yum 源来安装`containerd`

```shell
sudo yum install -y yum-utils device-mapper-persistent-data lvm2

sudo yum-config-manager 


yum list containerd.io 


yum install containerd.io -y


sudo systemctl start containerd


sudo systemctl enable 


```

### [](#关于CRI "关于CRI")关于 CRI[](#关于CRI)

官方表示，对于 k8s 来说，不需要安装`cri-containerd`，并且该功能会在后面的 2.0 版本中废弃。

> **FAQ**: For Kubernetes, do I need to download `cri-containerd-(cni-)<VERSION>-<OS-<ARCH>.tar.gz` too?
>
> **Answer**: No.
>
> As the Kubernetes CRI feature has been already included in `containerd-<VERSION>-<OS>-<ARCH>.tar.gz`, you do not need to download the `cri-containerd-....` archives to use CRI.
>
> The `cri-containerd-...` archives are [deprecated](https://github.com/containerd/containerd/blob/main/RELEASES.md#deprecated-features), do not work on old Linux distributions, and will be removed in containerd 2.0.

### [](#安装cni-plugins "安装cni-plugins")安装 cni-plugins[](#安装cni-plugins)

使用 yum 源安装的方式会把 runc 安装好，但是并不会安装 cni-plugins，因此这部分还是需要我们自行安装。

> The `containerd.io` package contains runc too, but does not contain CNI plugins.

我们直接在 [github 上面](https://github.com/containernetworking/plugins/releases)找到系统对应的架构版本，这里为 amd64，然后解压即可。


```shell
$ wget https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz
$ wget https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz.sha512
$ sha512sum -c cni-plugins-linux-amd64-v1.1.1.tgz.sha512


$ mkdir -p /opt/cni/bin
$ tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.1.1.tgz
```

[](#2-2-配置cgroup-drivers "2.2 配置cgroup drivers")2.2 配置 cgroup drivers[](#2-2-配置cgroup-drivers)
----------------------------------------------------------------------------------------------

CentOS7 使用的是`systemd`来初始化系统并管理进程，初始化进程会生成并使用一个 root 控制组 (`cgroup`), 并充当 `cgroup` 管理器。 `Systemd` 与 `cgroup` 集成紧密，并将为每个 `systemd` 单元分配一个 `cgroup`。 我们也可以配置`容器运行时`和 `kubelet` 使用 `cgroupfs`。 连同 `systemd` 一起使用 `cgroupfs` 意味着将有两个不同的 `cgroup 管理器`。而当一个系统中同时存在 cgroupfs 和 systemd 两者时，容易变得不稳定，因此最好更改设置，令容器运行时和 kubelet 使用 `systemd` 作为 `cgroup` 驱动，以此使系统更为稳定。 对于`containerd`, 需要设置配置文件`/etc/containerd/config.toml`中的 `SystemdCgroup` 参数。

> 参考 k8s 官方的说明文档：
>
> [https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd-systemd](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd-systemd)

```toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
```

接下来我们开始配置 containerd 的 cgroup driver

```shell
$ containerd config default | grep SystemdCgroup
            SystemdCgroup = false


$ cat /etc/containerd/config.toml | egrep -v "^#|^$"
disabled_plugins = ["cri"]


$ mv /etc/containerd/config.toml /etc/containerd/config.toml.origin
$ containerd config default > /etc/containerd/config.toml

$ sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
$ systemctl restart containerd

$ containerd config dump | grep SystemdCgroup
            SystemdCgroup = true




$ systemctl status containerd -l
May 12 09:57:31 tiny-kubeproxy-free-master-18-1.k8s.tcinternal containerd[5758]: time="2022-05-12T09:57:31.100285056+08:00" level=error msg="failed to load cni during init, please check CRI plugin status before setting up network for pods" error="cni config load failed: no network config found in /etc/cni/net.d: cni plugin not initialized: failed to load cni config"


```

[](#2-3-关于kubelet的cgroup-driver "2.3 关于kubelet的cgroup driver")2.3 关于 kubelet 的 cgroup driver[](#2-3-关于kubelet的cgroup-driver)
----------------------------------------------------------------------------------------------------------------------------

k8s 官方有[详细的文档](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/configure-cgroup-driver/)介绍了如何设置 kubelet 的`cgroup driver`，需要特别注意的是，在 1.22 版本开始，如果没有手动设置 kubelet 的 cgroup driver，那么默认会设置为 systemd

> **Note:** In v1.22, if the user is not setting the `cgroupDriver` field under `KubeletConfiguration`, `kubeadm` will default it to `systemd`.

一个比较简单的指定 kubelet 的`cgroup driver`的方法就是在`kubeadm-config.yaml`加入`cgroupDriver`字段

```yaml
kind: ClusterConfiguration
apiVersion: kubeadm.k8s.io/v1beta3
kubernetesVersion: v1.21.0
---
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: systemd
```

我们可以直接查看 configmaps 来查看初始化之后集群的 kubeadm-config 配置。

```shell
$ kubectl describe configmaps kubeadm-config -n kube-system
Name:         kubeadm-config
Namespace:    kube-system
Labels:       <none>
Annotations:  <none>

Data
====
ClusterConfiguration:
----
apiServer:
  extraArgs:
    authorization-mode: Node,RBAC
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: v1.23.6
networking:
  dnsDomain: cali-cluster.tclocal
  serviceSubnet: 10.88.0.0/18
scheduler: {}

BinaryData
====

Events:  <none>


```

当然因为我们需要安装的版本高于 1.22.0 并且使用的就是 systemd，因此可以不用再重复配置。

> 对应的官方文档可以参考这里
>
> [https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl)

kube 三件套就是`kubeadm`、`kubelet` 和 `kubectl`，三者的具体功能和作用如下：

*   `kubeadm`：用来初始化集群的指令。
*   `kubelet`：在集群中的每个节点上用来启动 Pod 和容器等。
*   `kubectl`：用来与集群通信的命令行工具。

需要注意的是：

*   `kubeadm`不会帮助我们管理`kubelet`和`kubectl`，其他两者也是一样的，也就是说这三者是相互独立的，并不存在谁管理谁的情况；
*   `kubelet`的版本必须小于等于`API-server`的版本，否则容易出现兼容性的问题；
*   `kubectl`并不是集群中的每个节点都需要安装，也并不是一定要安装在集群中的节点，可以单独安装在自己本地的机器环境上面，然后配合`kubeconfig`文件即可使用`kubectl`命令来远程管理对应的 k8s 集群；

CentOS7 的安装比较简单，我们直接使用官方提供的`yum`源即可。需要注意的是这里需要设置`selinux`的状态，但是前面我们已经关闭了 selinux，因此这里略过这步。

```shell
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF


cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF


sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes


sed -i 's/repo_gpgcheck=1/repo_gpgcheck=0/g' /etc/yum.repos.d/kubernetes.repo

sudo yum install -y kubelet kubeadm kubectl --nogpgcheck --disableexcludes=kubernetes


sudo yum list --nogpgcheck kubelet kubeadm kubectl --showduplicates --disableexcludes=kubernetes


sudo systemctl enable --now kubelet


```

[](#4-0-etcd高可用 "4.0 etcd高可用")4.0 etcd 高可用[](#4-0-etcd高可用)
----------------------------------------------------------

etcd 高可用架构参考[这篇官方文档](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/)，主要可以分为堆叠 etcd 方案和外置 etcd 方案，两者的区别就是 etcd 是否部署在 apiserver 所在的 node 机器上面，这里我们主要使用的是堆叠 etcd 部署方案。

![](https://resource.tinychen.com/202212091606423.png)

![](https://resource.tinychen.com/202212091606949.png)

[](#4-1-apiserver高可用 "4.1 apiserver高可用")4.1 apiserver 高可用[](#4-1-apiserver高可用)
------------------------------------------------------------------------------

apisever 高可用配置参考[这篇官方文档](https://github.com/kubernetes/kubeadm/blob/main/docs/ha-considerations.md#high-availability-considerations)。目前 apiserver 的高可用比较主流的[官方推荐方案](https://github.com/kubernetes/kubeadm/blob/main/docs/ha-considerations.md#keepalived-and-haproxy)是使用 keepalived 和 haproxy，由于 centos7 自带的版本较旧，重新编译又过于麻烦，因此我们可以参考官方给出的[静态 pod 的部署方式](https://github.com/kubernetes/kubeadm/blob/main/docs/ha-considerations.md#option-2-run-the-services-as-static-pods)，提前将相关的配置文件放置到`/etc/kubernetes/manifests`目录下即可 (需要提前手动创建好目录)。官方表示对于我们这种堆叠部署控制面 master 节点和 etcd 的方式而言这是一种优雅的解决方案。

> This is an elegant solution, in particular with the setup described under [Stacked control plane and etcd nodes](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/#stacked-control-plane-and-etcd-nodes).

首先我们需要准备好三台 master 节点上面的 keepalived 配置文件和 haproxy 配置文件：

```shell
! /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
}
vrrp_script check_apiserver {
  script "/etc/keepalived/check_apiserver.sh"
  interval 3
  weight -2
  fall 10
  rise 2
}

vrrp_instance VI_1 {
    state ${STATE}
    interface ${INTERFACE}
    virtual_router_id ${ROUTER_ID}
    priority ${PRIORITY}
    authentication {
        auth_type PASS
        auth_pass ${AUTH_PASS}
    }
    virtual_ipaddress {
        ${APISERVER_VIP}
    }
    track_script {
        check_apiserver
    }
}


```

实际上我们需要区分三台控制面节点的状态

```shell
! /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
    router_id CALICO_MASTER_90_1
}
vrrp_script check_apiserver {
  script "/etc/keepalived/check_apiserver.sh"
  interval 3
  weight -2
  fall 10
  rise 2
}

vrrp_instance calico_ha_apiserver_10_31_90_0 {
    state MASTER
    interface eth0
    virtual_router_id 90
    priority 100
    authentication {
        auth_type PASS
        auth_pass pass@77
    }
    virtual_ipaddress {
        10.31.90.0
    }
    track_script {
        check_apiserver
    }
}


```

```shell
! /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
    router_id CALICO_MASTER_90_2
}
vrrp_script check_apiserver {
  script "/etc/keepalived/check_apiserver.sh"
  interval 3
  weight -2
  fall 10
  rise 2
}

vrrp_instance calico_ha_apiserver_10_31_90_0 {
    state BACKUP
    interface eth0
    virtual_router_id 90
    priority 99
    authentication {
        auth_type PASS
        auth_pass pass@77
    }
    virtual_ipaddress {
        10.31.90.0
    }
    track_script {
        check_apiserver
    }
}


```

```shell
! /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
    router_id CALICO_MASTER_90_3
}
vrrp_script check_apiserver {
  script "/etc/keepalived/check_apiserver.sh"
  interval 3
  weight -2
  fall 10
  rise 2
}

vrrp_instance calico_ha_apiserver_10_31_90_0 {
    state BACKUP
    interface eth0
    virtual_router_id 90
    priority 98
    authentication {
        auth_type PASS
        auth_pass pass@77
    }
    virtual_ipaddress {
        10.31.90.0
    }
    track_script {
        check_apiserver
    }
}


```

这是 haproxy 的配置文件模板：

```
global
    log /dev/log local0
    log /dev/log local1 notice
    daemon





defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 1
    timeout http-request    10s
    timeout queue           20s
    timeout connect         5s
    timeout client          20s
    timeout server          20s
    timeout http-keep-alive 10s
    timeout check           10s




frontend apiserver
    bind *:${APISERVER_DEST_PORT}
    mode tcp
    option tcplog
    default_backend apiserver




backend apiserver
    option httpchk GET /healthz
    http-check expect status 200
    mode tcp
    option ssl-hello-chk
    balance     roundrobin
        server ${HOST1_ID} ${HOST1_ADDRESS}:${APISERVER_SRC_PORT} check


```

这是 keepalived 的检测脚本，注意这里的`${APISERVER_VIP}`和`${APISERVER_DEST_PORT}`要替换为集群的实际 VIP 和端口

```shell
#!/bin/sh
APISERVER_VIP="10.31.90.0"
APISERVER_DEST_PORT="8443"

errorExit() {
    echo "*** $*" 1&gt;&amp;2
    exit 1
}

curl --silent --max-time 2 --insecure https://localhost:${APISERVER_DEST_PORT}/ -o /dev/null || errorExit "Error GET https://localhost:${APISERVER_DEST_PORT}/"
if ip addr | grep -q ${APISERVER_VIP}; then
    curl --silent --max-time 2 --insecure https://${APISERVER_VIP}:${APISERVER_DEST_PORT}/ -o /dev/null || errorExit "Error GET https://${APISERVER_VIP}:${APISERVER_DEST_PORT}/"
fi
```

这是 keepalived 的部署文件`/etc/kubernetes/manifests/keepalived.yaml`，注意这里的配置文件路径要和上面的对应一致。

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  name: keepalived
  namespace: kube-system
spec:
  containers:
  - image: osixia/keepalived:2.0.17
    name: keepalived
    resources: {}
    securityContext:
      capabilities:
        add:
        - NET_ADMIN
        - NET_BROADCAST
        - NET_RAW
    volumeMounts:
    - mountPath: /usr/local/etc/keepalived/keepalived.conf
      name: config
    - mountPath: /etc/keepalived/check_apiserver.sh
      name: check
  hostNetwork: true
  volumes:
  - hostPath:
      path: /etc/keepalived/keepalived.conf
    name: config
  - hostPath:
      path: /etc/keepalived/check_apiserver.sh
    name: check
status: {}


```

这是 haproxy 的部署文件`/etc/kubernetes/manifests/haproxy.yaml`，注意这里的配置文件路径要和上面的对应一致，且`${APISERVER_DEST_PORT}`要换成我们对应的 apiserver 的端口，这里我们改为 8443，避免和原有的 6443 端口冲突

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: haproxy
  namespace: kube-system
spec:
  containers:
  - image: haproxy:2.1.4
    name: haproxy
    livenessProbe:
      failureThreshold: 8
      httpGet:
        host: localhost
        path: /healthz

        port: 8443
        scheme: HTTPS
    volumeMounts:
    - mountPath: /usr/local/etc/haproxy/haproxy.cfg
      name: haproxyconf
      readOnly: true
  hostNetwork: true
  volumes:
  - hostPath:
      path: /etc/haproxy/haproxy.cfg
      type: FileOrCreate
    name: haproxyconf
status: {}


```

[](#4-2-编写配置文件 "4.2 编写配置文件")4.2 编写配置文件[](#4-2-编写配置文件)
-----------------------------------------------------

在集群中所有节点都执行完上面的操作之后，我们就可以开始创建 k8s 集群了。**因为我们这次需要进行高可用部署，所以初始化的时候先挑任意一台 master 控制面节点进行操作即可。**

```shell
$ kubeadm config images list
registry.k8s.io/kube-apiserver:v1.26.0
registry.k8s.io/kube-controller-manager:v1.26.0
registry.k8s.io/kube-scheduler:v1.26.0
registry.k8s.io/kube-proxy:v1.26.0
registry.k8s.io/pause:3.9
registry.k8s.io/etcd:3.5.6-0
registry.k8s.io/coredns/coredns:v1.9.3
$ kubeadm config print init-defaults > kubeadm-calico-ha.conf
```

*   考虑到大多数情况下国内的网络无法使用谷歌的镜像源 (1.25 版本开始从`k8s.gcr.io`换为`registry.k8s.io`)，我们可以直接在配置文件中修改`imageRepository`参数为阿里的镜像源`registry.aliyuncs.com/google_containers`
*   `kubernetesVersion`字段用来指定我们要安装的 k8s 版本
*   `localAPIEndpoint`参数需要修改为我们的 master 节点的 IP 和端口，初始化之后的 k8s 集群的 apiserver 地址就是这个
*   `criSocket`从 1.24.0 版本开始已经默认变成了`containerd`
*   `podSubnet`、`serviceSubnet`和`dnsDomain`两个参数默认情况下可以不用修改，这里我按照自己的需求进行了变更
*   `nodeRegistration`里面的`name`参数修改为对应 master 节点的`hostname`
*   `controlPlaneEndpoint`参数配置的才是我们前面配置的集群高可用 apiserver 的地址
*   新增配置块使用 ipvs，具体可以参考[官方文档](https://github.com/kubernetes/kubernetes/blob/master/pkg/proxy/ipvs/README.md#cluster-created-by-kubeadm)

```yaml
apiVersion: kubeadm.k8s.io/v1beta3
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 10.31.90.1
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
  imagePullPolicy: IfNotPresent
  name: k8s-calico-master-10-31-90-1.tinychen.io
  taints: null
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: 1.26.0
controlPlaneEndpoint: "k8s-calico-apiserver.tinychen.io:8443"
networking:
  dnsDomain: cali-cluster.tclocal
  serviceSubnet: 10.33.128.0/18
  podSubnet: 10.33.0.0/17
scheduler: {}
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs


```

[](#4-3-初始化集群 "4.3 初始化集群")4.3 初始化集群[](#4-3-初始化集群)
-------------------------------------------------

此时我们再查看对应的配置文件中的镜像版本，就会发现已经变成了对应阿里云镜像源的版本

```yaml
$ kubeadm config images list --config kubeadm-calico-ha.conf
registry.aliyuncs.com/google_containers/kube-apiserver:v1.26.0
registry.aliyuncs.com/google_containers/kube-controller-manager:v1.26.0
registry.aliyuncs.com/google_containers/kube-scheduler:v1.26.0
registry.aliyuncs.com/google_containers/kube-proxy:v1.26.0
registry.aliyuncs.com/google_containers/pause:3.9
registry.aliyuncs.com/google_containers/etcd:3.5.6-0
registry.aliyuncs.com/google_containers/coredns:v1.9.3


$ kubeadm config images pull --config kubeadm-calico-ha.conf
[config/images] Pulled registry.aliyuncs.com/google_containers/kube-apiserver:v1.26.0
[config/images] Pulled registry.aliyuncs.com/google_containers/kube-controller-manager:v1.26.0
[config/images] Pulled registry.aliyuncs.com/google_containers/kube-scheduler:v1.26.0
[config/images] Pulled registry.aliyuncs.com/google_containers/kube-proxy:v1.26.0
[config/images] Pulled registry.aliyuncs.com/google_containers/pause:3.9
[config/images] Pulled registry.aliyuncs.com/google_containers/etcd:3.5.6-0
[config/images] Pulled registry.aliyuncs.com/google_containers/coredns:v1.9.3


$ kubeadm init --config kubeadm-calico-ha.conf  --upload-certs
[init] Using Kubernetes version: v1.26.0
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
...此处略去一堆输出...


```

当我们看到下面这个输出结果的时候，我们的集群就算是初始化成功了。

```shell
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join k8s-calico-apiserver.tinychen.io:8443 
        
        

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join k8s-calico-apiserver.tinychen.io:8443 
        


```

接下来我们在剩下的两个 master 节点上面执行上面输出的命令，注意要执行带有`--control-plane --certificate-key`这两个参数的命令，其中`--control-plane`参数是确定该节点为 master 控制面节点，而`--certificate-key`参数则是把我们前面初始化集群的时候通过`--upload-certs`上传到 k8s 集群中的证书下载下来使用。

```shell
This node has joined the cluster and a new control plane instance was created:
Certificate signing request was sent to apiserver and approval was received.
The Kubelet was informed of the new secure connection details.
Control plane label and taint were applied to the new node.
The Kubernetes control plane instances scaled up.
A new etcd member was added to the local/stacked etcd cluster.
To start administering your cluster from this node, you need to run the following as a regular user:

    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
Run 'kubectl get nodes' to see this node join the cluster.
```

最后再对剩下的三个 worker 节点执行普通的加入集群命令，当看到下面的输出的时候说明节点成功加入集群了。

```
This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.
Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

如果不小心没保存初始化成功的输出信息，或者是以后还需要新增节点也没有关系，我们可以使用 kubectl 工具查看或者生成 token

```shell
# 查看现有的token列表
$ kubeadm token list
TOKEN                     TTL         EXPIRES                USAGES                   DESCRIPTION                                                EXTRA GROUPS
abcdef.0123456789abcdef   23h         2022-12-09T08:14:37Z   authentication,signing   <none>                                                     system:bootstrappers:kubeadm:default-node-token
dss91p.3r5don4a3e9r2f29   1h          2022-12-08T10:14:36Z   <none>                   Proxy for managing TTL for the kubeadm-certs secret        <none>

# 如果token已经失效，那就再创建一个新的token
$ kubeadm token create
8hmoux.jabpgvs521r8rsqm

$ kubeadm token list
TOKEN                     TTL         EXPIRES                USAGES                   DESCRIPTION                                                EXTRA GROUPS
8hmoux.jabpgvs521r8rsqm   23h         2022-12-09T08:29:29Z   authentication,signing   <none>                                                     system:bootstrappers:kubeadm:default-node-token
abcdef.0123456789abcdef   23h         2022-12-09T08:14:37Z   authentication,signing   <none>                                                     system:bootstrappers:kubeadm:default-node-token
dss91p.3r5don4a3e9r2f29   1h          2022-12-08T10:14:36Z   <none>                   Proxy for managing TTL for the kubeadm-certs secret        <none>

# 如果找不到
$ openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
cc68219b233262d8834ad5d6e96166be487c751b53fb9ec19a5ca3599b538a33


```

[](#4-4-配置kubeconfig "4.4 配置kubeconfig")4.4 配置 kubeconfig[](#4-4-配置kubeconfig)
------------------------------------------------------------------------------

刚初始化成功之后，我们还没办法马上查看 k8s 集群信息，需要配置 kubeconfig 相关参数才能正常使用 kubectl 连接 apiserver 读取集群信息。

```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
export KUBECONFIG=/etc/kubernetes/admin.conf

echo "source <(kubectl completion bash)" >> ~/.bashrc
```

> 前面我们提到过`kubectl`不一定要安装在集群内，实际上只要是任何一台能连接到`apiserver`的机器上面都可以安装`kubectl`并且根据步骤配置`kubeconfig`，就可以使用`kubectl`命令行来管理对应的 k8s 集群。

配置完成后，我们再执行相关命令就可以查看集群的信息了，但是此时节点的状态还是`NotReady`，接下来就需要部署 CNI 了。

```shell
$ kubectl cluster-info
Kubernetes control plane is running at https://k8s-calico-apiserver.tinychen.io:8443
CoreDNS is running at https://k8s-calico-apiserver.tinychen.io:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

$ kubectl get nodes -o wide
NAME                                       STATUS     ROLES           AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION                CONTAINER-RUNTIME
k8s-calico-master-10-31-90-1.tinychen.io   NotReady   control-plane   7m55s   v1.26.0   10.31.90.1    <none>        CentOS Linux 7 (Core)   3.10.0-1160.62.1.el7.x86_64   containerd://1.6.14
k8s-calico-master-10-31-90-2.tinychen.io   NotReady   control-plane   4m44s   v1.26.0   10.31.90.2    <none>        CentOS Linux 7 (Core)   3.10.0-1160.62.1.el7.x86_64   containerd://1.6.14
k8s-calico-master-10-31-90-3.tinychen.io   NotReady   control-plane   2m44s   v1.26.0   10.31.90.3    <none>        CentOS Linux 7 (Core)   3.10.0-1160.62.1.el7.x86_64   containerd://1.6.14
k8s-calico-worker-10-31-90-4.tinychen.io   NotReady   <none>          2m9s    v1.26.0   10.31.90.4    <none>        CentOS Linux 7 (Core)   3.10.0-1160.62.1.el7.x86_64   containerd://1.6.14
k8s-calico-worker-10-31-90-5.tinychen.io   NotReady   <none>          91s     v1.26.0   10.31.90.5    <none>        CentOS Linux 7 (Core)   3.10.0-1160.62.1.el7.x86_64   containerd://1.6.14
k8s-calico-worker-10-31-90-6.tinychen.io   NotReady   <none>          63s     v1.26.0   10.31.90.6    <none>        CentOS Linux 7 (Core)   3.10.0-1160.62.1.el7.x86_64   containerd://1.6.14

$ kubectl get pods -A -o wide
NAMESPACE     NAME                                                               READY   STATUS    RESTARTS   AGE     IP           NODE                                       NOMINATED NODE   READINESS GATES
kube-system   coredns-5bbd96d687-l84hq                                           0/1     Pending   0          8m11s   <none>       <none>                                     <none>           <none>
kube-system   coredns-5bbd96d687-wbmdq                                           0/1     Pending   0          8m11s   <none>       <none>                                     <none>           <none>
kube-system   etcd-k8s-calico-master-10-31-90-1.tinychen.io                      1/1     Running   0          8m10s   10.31.90.1   k8s-calico-master-10-31-90-1.tinychen.io   <none>           <none>
kube-system   etcd-k8s-calico-master-10-31-90-2.tinychen.io                      1/1     Running   0          4m51s   10.31.90.2   k8s-calico-master-10-31-90-2.tinychen.io   <none>           <none>
kube-system   etcd-k8s-calico-master-10-31-90-3.tinychen.io                      1/1     Running   0          3m      10.31.90.3   k8s-calico-master-10-31-90-3.tinychen.io   <none>           <none>
kube-system   haproxy-k8s-calico-master-10-31-90-1.tinychen.io                   1/1     Running   0          8m10s   10.31.90.1   k8s-calico-master-10-31-90-1.tinychen.io   <none>           <none>
kube-system   haproxy-k8s-calico-master-10-31-90-2.tinychen.io                   1/1     Running   0          4m45s   10.31.90.2   k8s-calico-master-10-31-90-2.tinychen.io   <none>           <none>
kube-system   haproxy-k8s-calico-master-10-31-90-3.tinychen.io                   1/1     Running   0          3m      10.31.90.3   k8s-calico-master-10-31-90-3.tinychen.io   <none>           <none>
kube-system   keepalived-k8s-calico-master-10-31-90-1.tinychen.io                1/1     Running   0          8m10s   10.31.90.1   k8s-calico-master-10-31-90-1.tinychen.io   <none>           <none>
kube-system   keepalived-k8s-calico-master-10-31-90-2.tinychen.io                1/1     Running   0          4m57s   10.31.90.2   k8s-calico-master-10-31-90-2.tinychen.io   <none>           <none>
kube-system   keepalived-k8s-calico-master-10-31-90-3.tinychen.io                1/1     Running   0          3m1s    10.31.90.3   k8s-calico-master-10-31-90-3.tinychen.io   <none>           <none>
kube-system   kube-apiserver-k8s-calico-master-10-31-90-1.tinychen.io            1/1     Running   0          8m9s    10.31.90.1   k8s-calico-master-10-31-90-1.tinychen.io   <none>           <none>
kube-system   kube-apiserver-k8s-calico-master-10-31-90-2.tinychen.io            1/1     Running   0          4m43s   10.31.90.2   k8s-calico-master-10-31-90-2.tinychen.io   <none>           <none>
kube-system   kube-apiserver-k8s-calico-master-10-31-90-3.tinychen.io            1/1     Running   0          3m      10.31.90.3   k8s-calico-master-10-31-90-3.tinychen.io   <none>           <none>
kube-system   kube-controller-manager-k8s-calico-master-10-31-90-1.tinychen.io   1/1     Running   0          8m10s   10.31.90.1   k8s-calico-master-10-31-90-1.tinychen.io   <none>           <none>
kube-system   kube-controller-manager-k8s-calico-master-10-31-90-2.tinychen.io   1/1     Running   0          4m58s   10.31.90.2   k8s-calico-master-10-31-90-2.tinychen.io   <none>           <none>
kube-system   kube-controller-manager-k8s-calico-master-10-31-90-3.tinychen.io   1/1     Running   0          3m      10.31.90.3   k8s-calico-master-10-31-90-3.tinychen.io   <none>           <none>
kube-system   kube-proxy-9x6gc                                                   1/1     Running   0          108s    10.31.90.5   k8s-calico-worker-10-31-90-5.tinychen.io   <none>           <none>
kube-system   kube-proxy-jnfqm                                                   1/1     Running   0          8m10s   10.31.90.1   k8s-calico-master-10-31-90-1.tinychen.io   <none>           <none>
kube-system   kube-proxy-kb2d5                                                   1/1     Running   0          80s     10.31.90.6   k8s-calico-worker-10-31-90-6.tinychen.io   <none>           <none>
kube-system   kube-proxy-n5g6b                                                   1/1     Running   0          5m1s    10.31.90.2   k8s-calico-master-10-31-90-2.tinychen.io   <none>           <none>
kube-system   kube-proxy-tsqz8                                                   1/1     Running   0          2m26s   10.31.90.4   k8s-calico-worker-10-31-90-4.tinychen.io   <none>           <none>
kube-system   kube-proxy-wcgch                                                   1/1     Running   0          3m1s    10.31.90.3   k8s-calico-master-10-31-90-3.tinychen.io   <none>           <none>
kube-system   kube-scheduler-k8s-calico-master-10-31-90-1.tinychen.io            1/1     Running   0          8m10s   10.31.90.1   k8s-calico-master-10-31-90-1.tinychen.io   <none>           <none>
kube-system   kube-scheduler-k8s-calico-master-10-31-90-2.tinychen.io            1/1     Running   0          4m51s   10.31.90.2   k8s-calico-master-10-31-90-2.tinychen.io   <none>           <none>
kube-system   kube-scheduler-k8s-calico-master-10-31-90-3.tinychen.io            1/1     Running   0          3m      10.31.90.3   k8s-calico-master-10-31-90-3.tinychen.io   <none>           <none>


```

[](#5-1-部署calico "5.1 部署calico")5.1 部署 calico[](#5-1-部署calico)
--------------------------------------------------------------

CNI 的部署我们参考官网的自建 K8S [部署教程](https://projectcalico.docs.tigera.io/getting-started/kubernetes/self-managed-onprem/)，官网主要给出了两种[部署方式](https://projectcalico.docs.tigera.io/getting-started/kubernetes/self-managed-onprem/onpremises)，分别是通过`Calico operator`和`Calico manifests`来进行部署和管理`calico`，operator 是通过 deployment 的方式部署一个 calico 的 operator 到集群中，再用其来管理 calico 的安装升级等生命周期操作。manifests 则是将相关都使用 yaml 的配置文件进行管理，这种方式管理起来相对前者比较麻烦，但是对于高度自定义的 K8S 集群有一定的优势。

这里我们使用 operator 的方式进行部署。

首先我们把需要用到的两个部署文件下载到本地。

```shell
curl https://raw.githubusercontent.com/projectcalico/calico/v3.24.5/manifests/tigera-operator.yaml -O
curl https://raw.githubusercontent.com/projectcalico/calico/v3.24.5/manifests/custom-resources.yaml -O
```

随后我们修改`custom-resources.yaml`里面的 pod ip 段信息和划分子网的大小。

```yaml
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:

  calicoNetwork:

    ipPools:
    - blockSize: 24
      cidr: 10.33.0.0/17
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()

---



apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}


```

最后我们直接部署

```shell
kubectl create -f tigera-operator.yaml
kubectl create -f custom-resources.yaml
```

此时部署完成之后我们应该可以看到所有的 pod 和 node 都已经处于正常工作状态。接下来我们进入高级配置阶段

[](#5-2-安装calicoctl "5.2 安装calicoctl")5.2 安装 calicoctl[](#5-2-安装calicoctl)
--------------------------------------------------------------------------

接下来我们就要[部署 calicoctl](https://projectcalico.docs.tigera.io/maintenance/clis/calicoctl/install) 来帮助我们管理 calico 的相关配置，为了使用 Calico 的许多功能，需要 calicoctl 命令行工具。它用于管理 Calico 策略和配置，以及查看详细的集群状态。

> The `calicoctl` command line tool is required in order to use many of Calico’s features. It is used to manage Calico policies and configuration, as well as view detailed cluster status.

这里我们可以直接使用二进制部署安装

```shell
curl -L https://github.com/projectcalico/calico/releases/download/v3.24.5/calicoctl-linux-amd64 -o /usr/local/bin/calicoctl
chmod +x /usr/local/bin/calicoctl
```

至于配置也比较简单，因为我们这里使用的是直接连接 apiserver 的方式，所以直接配置环境变量即可

```shell
export CALICO_DATASTORE_TYPE=kubernetes
export CALICO_KUBECONFIG=~/.kube/config
calicoctl get workloadendpoints -A
calicoctl node status
```

[](#5-3-配置BGP "5.3 配置BGP")5.3 配置 BGP[](#5-3-配置BGP)
--------------------------------------------------

一般来说，calico 的 [BGP 拓扑](https://projectcalico.docs.tigera.io/networking/bgp)可以分为三种配置：

*   **Full-mesh（全网状连接）**：启用 BGP 后，Calico 的默认行为是创建内部 BGP (iBGP) 连接的**全网状**连接，其中每个节点相互对等。这允许 Calico 在任何 L2 网络上运行，无论是公共云还是私有云，或者是[配置](https://projectcalico.docs.tigera.io/networking/vxlan-ipip)了基于 IPIP 的 overlays 网络。**Calico 不将 BGP 用于 VXLAN overlays 网络。**全网状结构非常适合 100 个或更少节点的中小型部署，但在规模明显更大的情况下，全网状结构的效率会降低，calico 建议使用路由反射器（Route reflectors）。
    
*   **Route reflectors（路由反射器）**：要构建大型内部 BGP (iBGP) 集群，可以使用 **BGP 路由反射器**来减少每个节点上使用的 BGP 对等体的数量。在这个模型中，一些节点充当路由反射器，并被配置为在它们之间建立一个完整的网格。然后将其他节点配置为与这些路由反射器的子集对等（通常为 2 个用于冗余），与全网状相比减少了 BGP 对等连接的总数。
    
*   **Top of Rack (ToR)**：在**本地部署**中，我们可以直接让 calico 和物理网络基础设施建立 BGP 连接，一般来说这需要先把 calico 默认自带的 Full-mesh 配置禁用掉，然后将 calico 和本地的 L3 ToR 路由建立连接。当整个自建集群的规模很大的时候（通常仅当每个 L2 域中的节点数大于 100 时），还可以考虑在每个机架内使用 BGP 的路由反射器（Route reflectors）。
    
    要深入了解常见的本地部署模型，请参阅 [Calico over IP Fabrics](https://projectcalico.docs.tigera.io/reference/architecture/design/l2-interconnect-fabric)。
    

我们这里只是一个小规模的测试集群（6 节点），暂时用不上路由反射器这类复杂的配置，因此我们参考第三种 TOR 的模式，让 node 直接和我们测试网络内的 L3 路由器建立 BGP 连接即可。

在刚初始化的情况下，我们的 calico 是还没有创建`BGPConfiguration`，此时我们需要先手动创建，并且禁用`nodeToNodeMesh`配置，同时还需要借助 calico 将集群的`ClusterIP`和`ExternalIP`都发布出去。

```shell
$ cat calico-bgp-configuration.yaml
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  logSeverityScreen: Info
  nodeToNodeMeshEnabled: false
  asNumber: 64517
  serviceClusterIPs:
  - cidr: 10.33.128.0/18
  serviceExternalIPs:
  - cidr: 10.33.192.0/18
  listenPort: 179
  bindMode: NodeIP
  communities:
  - name: bgp-large-community
    value: 64517:300:100
  prefixAdvertisements:
  - cidr: 10.33.0.0/17
    communities:
    - bgp-large-community
    - 64517:120


```

另一个就是需要准备`BGPPeer`的配置，可以同时配置一个或者多个，下面的示例配置了两个`BGPPeer`，并且 ASN 号各不相同。其中`keepOriginalNextHop`默认是不配置的，这里特别配置为`true`，确保通过 BGP 宣发 pod IP 段路由的时候只宣发对应的 node，而不是针对 podIP 也开启 ECMP 功能。详细的配置可以参考[官方文档](https://projectcalico.docs.tigera.io/reference/resources/bgppeer)

```shell
$ cat calico-bgp-peer.yaml
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: openwrt-peer
spec:
  peerIP: 10.31.254.253
  keepOriginalNextHop: true
  asNumber: 64512
---
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: tiny-unraid-peer
spec:
  peerIP: 10.31.100.100
  keepOriginalNextHop: true
  asNumber: 64516


```

配置完成之后我们直接部署即可，这时候集群默认的 node-to-node-mesh 就已经被我们禁用，此外还可以看到我们配置的两个 BGPPeer 已经顺利建立连接并发布路由了。

```shell
$ kubectl create -f calico-bgp-configuration.yaml
$ kubectl create -f calico-bgp-peer.yaml

$ calicoctl node status
Calico process is running.

IPv4 BGP status
+
| PEER ADDRESS  | PEER TYPE | STATE |  SINCE   |    INFO     |
+
| 10.31.254.253 | global    | up    | 08:03:49 | Established |
| 10.31.100.100 | global    | up    | 08:12:01 | Established |
+

IPv6 BGP status
No IPv6 peers found.


```

目前市面上开源的 K8S-LoadBalancer 主要就是 [MetalLB](https://tinychen.com/20220519-k8s-06-loadbalancer-metallb/)、[OpenELB](https://tinychen.com/20220523-k8s-07-loadbalancer-openelb/) 和 [PureLB](https://tinychen.com/20220524-k8s-08-loadbalancer-purelb/) 这三种，三者的工作原理和使用教程我都写文章分析过，针对目前这种使用场景，我个人认为最合适的是使用 PureLB，因为他的组件高度模块化，并且可以自由选择实现 ECMP 模式的路由协议和软件（MetalLB 和 OpenELB 都是自己通过 gobgp 实现的 BGP 协议），能更好的和我们前面的 calico BGP 模式组合在一起，借助 calico 自带的 BGP 配置把 LoadBalancer IP 发布到集群外。

关于 purelb 的详细工作原理和部署使用方式可以参考我之前写的[这篇文章](https://tinychen.com/20220524-k8s-08-loadbalancer-purelb/)，这里不再赘述。

![](https://resource.tinychen.com/202205241423207.png)

*   **Allocator**：用来监听 API 中的`LoadBalancer`类型服务，并且负责分配 IP。
*   **LBnodeagent**： 作为`daemonset`部署到每个可以暴露请求并吸引流量的节点上，并且负责监听服务的状态变化同时负责把 VIP 添加到本地网卡或者是虚拟网卡
*   **KubeProxy**：k8s 的内置组件，并非是 PureLB 的一部分，但是 PureLB 依赖其进行正常工作，当对 VIP 的请求达到某个具体的节点之后，需要由 kube-proxy 来负责将其转发到对应的 pod

因为我们此前已经配置了 calico 的 BGP 模式，并且会由它来负责 BGP 宣告的相关操作，因此在这里我们直接使用 purelb 的 BGP 模式，并且不需要自己再额外部署 bird 或 frr 来进行 BGP 路由发布，同时也不需要`LBnodeagent`组件来帮助暴露并吸引流量，只需要`Allocator`帮助我们完成 LoadBalancerIP 的分配操作即可。

```shell
$ wget https://gitlab.com/api/v4/projects/purelb%2Fpurelb/packages/generic/manifest/0.0.1/purelb-complete.yaml


$ kubectl apply -f purelb-complete.yaml
$ kubectl apply -f purelb-complete.yaml


$ kubectl delete ds -n purelb lbnodeagent


$ cat purelb-ipam.yaml
apiVersion: purelb.io/v1
kind: ServiceGroup
metadata:
  name: bgp-ippool
  namespace: purelb
spec:
  local:
    v4pool:
      subnet: '10.33.192.0/18'
      pool: '10.33.192.0-10.33.255.254'
      aggregation: /32
$ kubectl apply -f purelb-ipam.yaml
$ kubectl get sg -n purelb
NAME         AGE
bgp-ippool   64s


```

到这里我们的 PureLB 就部署完了，相比完整的 ECMP 模式要**少部署了路由协议软件**和 额外删除了`lbnodeagent`**，接下来可以开始测试了。

集群部署完成之后我们在 k8s 集群中部署一个 nginx 测试一下是否能够正常工作。首先我们创建一个名为`nginx-quic`的命名空间（`namespace`），然后在这个命名空间内创建一个名为`nginx-quic-deployment`的`deployment`用来部署 pod，最后再创建一个`service`用来暴露服务，这里我们同时使用`nodeport`和`LoadBalancer`两种方式来暴露服务，并且其中一个`LoadBalancer`的服务还要指定`LoadBalancerIP`方便我们测试。

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: nginx-quic

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-quic-deployment
  namespace: nginx-quic
spec:
  selector:
    matchLabels:
      app: nginx-quic
  replicas: 4
  template:
    metadata:
      labels:
        app: nginx-quic
    spec:
      containers:
      - name: nginx-quic
        image: tinychen777/nginx-quic:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80

---

apiVersion: v1
kind: Service
metadata:
  name: nginx-headless-service
  namespace: nginx-quic
spec:
  selector:
    app: nginx-quic
  clusterIP: None

---

apiVersion: v1
kind: Service
metadata:
  name: nginx-quic-service
  namespace: nginx-quic
spec:
  externalTrafficPolicy: Cluster
  selector:
    app: nginx-quic
  ports:
  - protocol: TCP
    port: 8080 
    targetPort: 80 
    nodePort: 30088 
  type: NodePort

---

apiVersion: v1
kind: Service
metadata:
  name: nginx-clusterip-service
  namespace: nginx-quic
spec:
  selector:
    app: nginx-quic
  ports:
  - protocol: TCP
    port: 8080 
    targetPort: 80 
  type: ClusterIP

---

apiVersion: v1
kind: Service
metadata:
  annotations:
    purelb.io/service-group: bgp-ippool
  name: nginx-lb-service
  namespace: nginx-quic
spec:
  allocateLoadBalancerNodePorts: true
  externalTrafficPolicy: Cluster
  internalTrafficPolicy: Cluster
  selector:
    app: nginx-quic
  ports:
  - protocol: TCP
    port: 80 
    targetPort: 80 
  type: LoadBalancer
  loadBalancerIP: 10.33.192.80

---

apiVersion: v1
kind: Service
metadata:
  annotations:
    purelb.io/service-group: bgp-ippool
  name: nginx-lb2-service
  namespace: nginx-quic
spec:
  allocateLoadBalancerNodePorts: true
  externalTrafficPolicy: Cluster
  internalTrafficPolicy: Cluster
  selector:
    app: nginx-quic
  ports:
  - protocol: TCP
    port: 80 
    targetPort: 80 
  type: LoadBalancer


```

部署完成之后我们检查各项服务的状态

```shell
$ kubectl get svc -n nginx-quic -o wide
NAME                      TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)          AGE     SELECTOR
nginx-clusterip-service   ClusterIP      10.33.141.36    <none>         8080/TCP         2d22h   app=nginx-quic
nginx-headless-service    ClusterIP      None            <none>         <none>           2d22h   app=nginx-quic
nginx-lb-service          LoadBalancer   10.33.151.137   10.33.192.80   80:30167/TCP     2d22h   app=nginx-quic
nginx-lb2-service         LoadBalancer   10.33.154.206   10.33.192.0    80:31868/TCP     2d22h   app=nginx-quic
nginx-quic-service        NodePort       10.33.150.169   <none>         8080:30088/TCP   2d22h   app=nginx-quic
$ kubectl get pods -n nginx-quic -o wide
NAME                                     READY   STATUS    RESTARTS   AGE     IP           NODE                                       NOMINATED NODE   READINESS GATES
nginx-quic-deployment-5d7d9559dd-2f4kx   1/1     Running   0          2d22h   10.33.26.2   k8s-calico-worker-10-31-90-4.tinychen.io   <none>           <none>
nginx-quic-deployment-5d7d9559dd-8gm7s   1/1     Running   0          2d22h   10.33.93.3   k8s-calico-worker-10-31-90-6.tinychen.io   <none>           <none>
nginx-quic-deployment-5d7d9559dd-jwhth   1/1     Running   0          2d22h   10.33.93.2   k8s-calico-worker-10-31-90-6.tinychen.io   <none>           <none>
nginx-quic-deployment-5d7d9559dd-qxhqh   1/1     Running   0          2d22h   10.33.12.2   k8s-calico-worker-10-31-90-5.tinychen.io   <none>           <none>
```

随后我们分别在集群内外的机器进行测试，分别访问 podIP 、clusterIP 和 loadbalancerIP。

```shell
root@tiny-unraid:~
10.31.100.100:43240

root@tiny-unraid:~
10.31.90.5:52758

root@tiny-unraid:~
10.31.90.5:7319

root@tiny-unraid:~
10.31.90.5:38170



[root@k8s-calico-master-10-31-90-1 ~]
10.31.90.1:40222
[root@k8s-calico-master-10-31-90-1 ~]
10.31.90.1:50773
[root@k8s-calico-master-10-31-90-1 ~]
10.31.90.1:19219
[root@k8s-calico-master-10-31-90-1 ~]
10.31.90.1:22346



[root@nginx-quic-deployment-5d7d9559dd-8gm7s /]
10.33.93.3:39560
[root@nginx-quic-deployment-5d7d9559dd-8gm7s /]
10.33.93.3:58160
[root@nginx-quic-deployment-5d7d9559dd-8gm7s /]
10.31.90.6:34183
[root@nginx-quic-deployment-5d7d9559dd-8gm7s /]
10.31.90.6:64266


```

最后检测一下路由器端的情况，可以看到对应的 podIP、clusterIP 和 loadbalancerIP 段路由

```shell
B>* 10.33.5.0/24 [20/0] via 10.31.90.1, eth0, weight 1, 2d19h22m
B>* 10.33.12.0/24 [20/0] via 10.31.90.5, eth0, weight 1, 2d19h22m
B>* 10.33.23.0/24 [20/0] via 10.31.90.2, eth0, weight 1, 2d19h22m
B>* 10.33.26.0/24 [20/0] via 10.31.90.4, eth0, weight 1, 2d19h22m
B>* 10.33.57.0/24 [20/0] via 10.31.90.3, eth0, weight 1, 2d19h22m
B>* 10.33.93.0/24 [20/0] via 10.31.90.6, eth0, weight 1, 2d19h22m
B>* 10.33.128.0/18 [20/0] via 10.31.90.1, eth0, weight 1, 00:00:20
  *                       via 10.31.90.2, eth0, weight 1, 00:00:20
  *                       via 10.31.90.3, eth0, weight 1, 00:00:20
  *                       via 10.31.90.4, eth0, weight 1, 00:00:20
  *                       via 10.31.90.5, eth0, weight 1, 00:00:20
  *                       via 10.31.90.6, eth0, weight 1, 00:00:20
B>* 10.33.192.0/18 [20/0] via 10.31.90.1, eth0, weight 1, 2d19h21m
  *                       via 10.31.90.2, eth0, weight 1, 2d19h21m
  *                       via 10.31.90.3, eth0, weight 1, 2d19h21m
  *                       via 10.31.90.4, eth0, weight 1, 2d19h21m
  *                       via 10.31.90.5, eth0, weight 1, 2d19h21m
  *                       via 10.31.90.6, eth0, weight 1, 2d19h21m


```

到这里整个 K8S 集群就部署完成了。