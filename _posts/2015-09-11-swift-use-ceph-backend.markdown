---
title: "swift object store use ceph backend"
date: 2015-09-11 14:46:53
categories: ["2015"]
tags: [swift, ceph]
---

本文记录了 swift 使用 ceph 作为对象存储方案的原因以及一些问题。

Swift 作为对象存储本身是提供 replication 设置的，通常会设置为 3 份，这里不考虑 replicas 设置为 1 的情况。

```
[root@swift ~]# swift-ring-builder /etc/swift/object.ring.gz 
/etc/swift/object.ring.gz, build version 3
262144 partitions, 3.000000 replicas, 1 regions, 3 zones, 3 devices, 0.00 balance
The minimum number of hours before a partition can be reassigned is 1
Devices:    id  region  zone      ip address  port  replication ip  replication port      name weight partitions balance meta
             0       1     1      10.160.0.3  6000      10.160.0.3              6000 keystonedev 100.00     262144    0.00 
             1       1     2     10.160.0.41  6000     10.160.0.41              6000 keystonedev 100.00     262144    0.00 
             2       1     3     10.160.0.42  6000     10.160.0.42              6000 keystonedev 100.00     262144    0.00 
```

Swift 本身提供 *local disk backend* 和 *In-memory backend* 两种方式，前者是可以用作生产的。

**Q1：为什么使用 ceph 作为 backend？**

考虑到我们的环境中已经使用了 ceph 作为块存储，同时 ceph 也可以通过 rados 提供对象存储，如果 swift 使用 ceph 作为 backend 就可以节省下 swift 本身的硬盘。

**Q2: 为什么不直接使用 ceph 的对象存储？**

因为 ceph 对象存储对 keystone 认证支持不好，因此需要使用 swift 结合 ceph 的方式。

stackforge 有提供 swift-ceph-backend 项目做支持：<https://github.com/stackforge/swift-ceph-backend>

这部分代码只是提供 backend 的实现，而且它默认设置 replicas 为 1，因为备份数由后端的 ceph 保证，但是如果 replicas 为 1，假设通过 ring 找到的 obj 所在的物理机恰巧 down 了，那就获取不到数据了，这个是没法保证 HA 的，这是社区代码没考虑的。

如果 replicas 设置为 3，那么就会存在以下问题：

1. `PUT/POST` object 的时候会通过 rados 上传相同文件至少 2 次，重复操作
2. `DELETE` object 的时候有一定几率失败，因为至少需要 `(3 // 2) + 1` 个成功才算成功，然而 ceph 中只能删除 1 次，因此会导致只成功删除 1 次，其余 2 次失败

因此需要修改 **proxy/controller/obj.py** 下的 `ObjectController` 的 `PUT` 和 `DELETE` 方法，代码略。

## 参考链接

<https://software.intel.com/en-us/blogs/2015/02/03/using-multiple-backends-in-openstack-swift>
