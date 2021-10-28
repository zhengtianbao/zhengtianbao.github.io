---
layout: post
title: "kubernetes flannel 网络模式"
date:  2018-11-06 15:58:23
categories: Kubernetes Flannel
---

## kubernetes flannel 网络模式

最近在研究kuberneters各个节点间容器通信的原理，深感自己网络知识的不足，从简单的flannel开始看起，搜集整理了网上的资料，记录下来。

kubernetes各节点间网络结构如下图：

![](/images/tuopu1.png)


实验环境如下：

|       | IP           | 角色   | 服务                                                         |
| ----- | ------------ | ------ | ------------------------------------------------------------ |
| node1 | 192.168.1.47 | master | etcd kube-controller-manager kube-apiserver kube-scheduler kube-proxy kubelet flanneld |
| node2 | 192.168.1.48 | node   | etcd kube-proxy kubelet flanneld                             |
| node3 | 192.168.1.49 | node   | etcd kube-proxy kubelet flanneld                             |



### host-gw

docker 跨主机通信，可以通过在宿主机手动建立route规则来实现，但是手动添加规则维护起来太过麻烦，而且当主机重启后，规则将会消失，因此交由flanneld来管理主机间的路由规则，这就是host-gw模式。该模式需要主机都在一个二层网络内，路由规则随节点数增加而增加，性能损耗相对较低。

### udp

当宿主机不在同一个二层网络内时，`host-gw` 模式本地静态路由不可达，需要通过三层路由器。

flannel的解决方案是创建一个overlay network，逻辑上属于同一网段，参考vxlan实现：

![](/images/vxlan1.png)

参考下图：

![](/images/tuopu2.png)

flannel创建一个 `overlay network`  其地址为 `100.96.0.0/16` ， node1节点subnet范围为 `100.96.1.1/24` ，node2节点subnet范围为 `100.96.2.1/24` ， node3节点subnet范围为 `100.96.3.1/24` ，因此理论上共支持256个node节点，每个node节点pod可用地址为256个 。

这个例子中的网络：

1.  node节点 实际的物理网络`172.20.32.0/19`，这里是在同一子网中，当然实际上可能是跨三层路由器的不同节点，物理上是可达的。
2. 由flannel创建的overlay network `100.96.0.0/16` 逻辑上的网络，每个pod分配的地址都是属于这个网络。
3. docker0 网桥地址，每个节点分配到地址为 `100.96.x.1/24` ，新建的pod都桥接到该网桥，从该网段分配地址，内部路由因此同主机内容器通信不需要经过三层。

跨主机通信：

![](/images/tuopu3.png)

假设container-1发送TCP包到container-2：

1. `100.96.2.3` 与container-1不在同一子网内，因此包发送给网关 `docker0`  ，宿主机查看路由表，存在记录 `100.96.0.0/16 dev flannel0  proto kernel  scope link  src 100.96.1.0` ，匹配后发送给`flannel0`，（这条route规则是由flannel daemon创建的）`flannel0`是一个`TUN`设备，TUN is a software interface implemented in linux kernel, it can pass raw ip packet between user program and the kernel. It works in two directions:
   - when it write IP packet to the `flannel0` device, the packet will be send to kernel directly, and kernel will route the packet according to its route table
   - when an IP packet arrives to the kernel, and the route tables says it should be routed to `flannel0` device, kernel will send the packet directly to the process which created this device, which is the `flanneld` daemon process
2.  因此该包发送给`flannel0` 等于发送给`flanneld`进程处理，`flanneld`进程拆包发现目的地址为`100.96.2.3` ，通过`etcd`中保存的信息找到目的容器宿主机为`node2` 。 然后`flanneld`在原有包基础上封装一层，将原包作为新UDP包的payload，而新UDP包头的dst ip为 node2, dst port 为8285，即node2上flanneld进程listen的port。UDP包经过三层路由到达node2的flanneld进程。
3. node2上的flanneld进程拆包后将里面的TCP包直接发给`flannel0`设备，当直接发送包给`flannel0`设备时，将会直接转发给kernel，由kernel路由，在node2上存在有`flanneld`进程创建的路由规则`100.96.2.0/24 dev docker0  proto kernel  scope link  src 100.96.2.1` 匹配该规则发送给`docker0`。
4. container-2桥接在`docker0`上，因此最终该包可达。

使用UDP模式可跨三层通信，但是通过UDP封装了一层将会导致额外的开销，同时`TUN`设备在包转发过程中存在用户空间与内核空间切换的过程造成额外开销。

![](/images/udp.png)

### vxlan

vxlan模式是flanneld推荐使用的backend，采用kernel种的vxlan替换了flannel自己实现的udp服务。



### 参考链接：

Host-gw 模式: <https://prefetch.net/blog/2018/02/20/getting-the-flannel-host-gw-working-with-kubernetes/>

UDP模式: <https://blog.laputa.io/kubernetes-flannel-networking-6a1cb1f8ec7c>

VXLAN模式：<http://dockone.io/article/2216>

