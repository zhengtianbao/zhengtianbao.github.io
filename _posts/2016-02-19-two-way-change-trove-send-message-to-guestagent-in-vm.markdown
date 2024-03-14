---
title: "Two way change Trove send message to guestagent in VM"
date: 2016-02-19 14:58:53
categories: ["2016"]
tags: [trove]
---

上篇文章介绍了 Trove 与虚机通信的方式是通过 rabbitmq 实现：

`trove-taskmanager` 或者 `trove-api` 作为 **publish** 发送消息到 **queue** 中，随着实例（instance）的启动，虚机内部 `trove-guestagent` 以 service 运行，同时作为 **consumer** 接收到 **queue** 里的消息，执行对应的 action，返回结果给 **queue**。

这里面有个问题就是要求虚机能访问到 rabbitmq，必须要与管理网打通。我们认为这样做存在安全隐患，因此对 Trove 的通信机制进行了修改。

考虑到 `openstack` 服务管理网段无法直接与虚机通信，因此如果需要调用虚机内部执行命令，只能在该虚机所在的宿主机（nova-compute node）通过某种方式连到虚机内部执行。

1. 通过 KVM/QEMU 自带的 guest agent：在虚机内部创建一个可读写设备，而在宿主机表现为一个 socket 文件，宿主机对该文件读写，虚机内部监听设备获取到可读消息内容后执行相关操作，完成通信
2. 通过 namespaces 与虚机通信，走 HTTP 协议，宿主机上通过命令 `ip netns exec qrouter-XXXX nc {vm_fixed_ip} {guestagent-port}` 将 HTTP 请求 adapter 到对应的namespaces（qrouter-XXXX）下，虚机内部 guestagent 作为 HTTPServer 接受请求，分发处理，返回结果

测试两种方案都是可行的，不过第一种需要修改 `nova` 在创建 `trove` 数据库实例的时候增加 socket 文件，第二种需要 `Neutron` DVR 支持。

两种都需要在 `nova-compute` 节点上增加一个 trove 服务，转发 `trove-taskmanager` 或者 `trove-api` 发过来的消息，通过两种方式里的某一种发送给虚机里的 `guestagent`，而 `guestagent` 自然也有不同的实现方式。

还有一个不同点在与虚机内部主动发消息给 `trove-conductor` 的情况，第一种比较简单只要在宿主机上监听 socket 文件就成，第二种虚机内部没法主动发消息给宿主机，只能通过增加 API 接口在宿主机上轮询获取状态。

两种方式各有利弊，具体实现略过，参考的项目分别为 [python-negotiator](https://github.com/xolox/python-negotiator) 和 [sahara](https://github.com/openstack/sahara)。
