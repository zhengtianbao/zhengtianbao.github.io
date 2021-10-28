---
layout: post
title:  "How Manila Communicate With VM"
date:   2016-01-29 17:42:53
categories: Manila Trove Sahara
---

上周重装了Manila服务, 这次就顺便整理了下Trove, Manila, Sahara, Murano几个项目与虚机通信的方式.

### Trove

Trove, Murano在虚拟机镜像内部预先安装agent, 随着虚拟机开机服务启动, 通过rabbitmq等待消息, 获取到消息后在虚拟机内部执行相关操作, 需要虚拟机与管理网连通, 否则rabbitmq不通.


### Manila

Manila本身会创建一个网络manila_service_network, 用户指定需要使用NFS的子网, 如net1, 192.168.10.0/28, manila做的事情就是:

1. 创建一个manila_service_network的子网routed_to_net1, 10.254.0.0/28 
2. 连接routed_to_net1和net1到同一个router上, 使这两个网络能够通信 
3. 在manila-share节点创建一个port, 通过ovs在br-int上创建tap设备, 并为其绑定port和mac 
4. 在manila-share节点添加到routed_to_net1的路由, 如: 10.254.0.0/28 dev tapdd155115-f7  proto kernel  scope link  src 10.254.0.3
5. 创建虚机(通过nic参数指定port), 该虚机在routed_to_net1内分配了一个地址, 创在某个manila-share节点上
6. Neutron在manila-share(nova-compute)节点上分配网络, 具体执行第3步操作, 两个tap具有相同的tag号, 这样manila-share就可以与虚机通信, 通过ssh执行命令


### Sahara

Sahara支持3种可选配置

1. 指定floatingip, 这样创建一个cluster所有的虚机都要有floatingip, sahara-engine直接通过floatingip与虚机通信
2. 不分配floatingip, 这时sahara-engine会要求使用proxy与虚拟机fixedip通信, proxy_command='ssh relay-machine-{tenant_id} nc {host} {port}' 通过一台能与虚机网络通信的跳板机实现(kilo以上版本支持, 跳板机需要手动配置) 
3. 不使用floatingip, 使用namespaces, 需要Neutron/DVR支持, 直接在计算节点通过命令ip netns exec qrouter-{id} nc {fixedip} 22 做proxy与虚机的fixedip通信, 通过ssh执行命令


### 总结

Trove, Murano使用rabbitmq安全性存在问题, Manila需要维护自己的管理网络, Sahara使用floatingip不现实, 而使用namespaces则需要Neutron/DVR支持. 

过完年回来更新我对liberty版Trove通信方式的修改.

完:)


### 参考链接:

<http://mytrix.me/2015/01/network-part-of-manila/>


