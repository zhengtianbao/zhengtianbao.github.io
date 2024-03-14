---
title: "How Manila Communicate With VM"
date: 2016-01-29 17:42:53
categories: ["2016"]
tags: [manila, trove, sahara]
---

上周重装了 Manila 服务，这次就顺便整理了下 Trove，Manila，Sahara，Murano 几个项目与虚机通信的方式。

## Trove

Trove，Murano 在虚拟机镜像内部预先安装 agent，随着虚拟机开机服务启动，通过 rabbitmq 等待消息，获取到消息后在虚拟机内部执行相关操作，需要虚拟机与管理网连通，否则 rabbitmq 不通。

## Manila

Manila 本身会创建一个网络 manila_service_network，用户指定需要使用 NFS 的子网，如 net1，192.168.10.0/28，manila 做的事情就是：

1. 创建一个 manila_service_network 的子网 routed_to_net1，10.254.0.0/28
2. 连接 routed_to_net1 和 net1 到同一个 router 上，使这两个网络能够通信
3. 在 manila-share 节点创建一个 port，通过 ovs 在 br-int 上创建 tap 设备，并为其绑定 port 和 mac
4. 在 manila-share 节点添加到 routed_to_net1 的路由，如：10.254.0.0/28 dev tapdd155115-f7  proto kernel  scope link  src 10.254.0.3
5. 创建虚机（通过 nic 参数指定 port），该虚机在 routed_to_net1 内分配了一个地址，创在某个 manila-share 节点上
6. Neutron 在 manila-share（nova-compute）节点上分配网络，具体执行第 3 步操作，两个 tap 具有相同的 tag 号，这样 manila-share 就可以与虚机通信，通过 ssh 执行命令

## Sahara

Sahara 支持 3 种可选配置：

1. 指定 floatingip，这样创建一个 cluster 所有的虚机都要有 floatingip，sahara-engine 直接通过 floatingip 与虚机通信
2. 不分配 floatingip，这时 sahara-engine 会要求使用 proxy 与虚拟机 fixedip 通信，proxy_command='ssh relay-machine-{tenant_id} nc {host} {port}' 通过一台能与虚机网络通信的跳板机实现（kilo以上版本支持，跳板机需要手动配置）
3. 不使用 floatingip，使用 namespaces，需要 Neutron/DVR 支持，直接在计算节点通过命令 ip netns exec qrouter-{id} nc {fixedip} 22 做 proxy 与虚机的 fixedip 通信，通过ssh执行命令

## 总结

Trove，Murano 使用 rabbitmq 安全性存在问题，Manila 需要维护自己的管理网络，Sahara 使用floatingip 不现实，而使用 namespaces 则需要Neutron/DVR支持。

过完年回来更新我对 liberty 版本 Trove 通信方式的修改。

完:)

## 参考链接

<http://mytrix.me/2015/01/network-part-of-manila/>
