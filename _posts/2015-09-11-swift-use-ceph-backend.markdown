---
layout: post
title:  "swift object store use ceph backend"
date:   2015-09-11 14:46:53
categories: Swift Ceph
---


Swift作为对象存储本身是提供replication设置的, 通常会设置为3份, 这里不考虑replicas设置为1的情况.

{% highlight text %}
[root@swift ~]# swift-ring-builder /etc/swift/object.ring.gz 
/etc/swift/object.ring.gz, build version 3
262144 partitions, 3.000000 replicas, 1 regions, 3 zones, 3 devices, 0.00 balance
The minimum number of hours before a partition can be reassigned is 1
Devices:    id  region  zone      ip address  port  replication ip  replication port      name weight partitions balance meta
             0       1     1      10.160.0.3  6000      10.160.0.3              6000 keystonedev 100.00     262144    0.00 
             1       1     2     10.160.0.41  6000     10.160.0.41              6000 keystonedev 100.00     262144    0.00 
             2       1     3     10.160.0.42  6000     10.160.0.42              6000 keystonedev 100.00     262144    0.00 
{% endhighlight %}

Swift本身提供*local disk backend*和*In-memory backend*两种方式, 前者是可以用作生产的.

**Q1: 为什么使用ceph作为backend?**<br>
考虑到我们的环境中已经使用了ceph作为块存储, 同时ceph也可以通过rados提供对象存储, 如果swift使用ceph作为backend就可以节省下swift本身的硬盘.

**Q2: 为什么不直接使用ceph的对象存储?**<br>
因为ceph对象存储对keystone认证支持不好, 因此需要使用swift结合ceph的方式.

stackforge有提供swift-ceph-backend项目做支持: 
<https://github.com/stackforge/swift-ceph-backend>

这部分代码只是提供backend的实现, 而且它默认设置replicas为1, 因为备份数由后端的ceph保证.<br>
如果replicas为1, 假设通过ring找到的obj所在的物理机恰巧down了, 那就获取不到数据了, 这个是没法保证HA的, 这是社区代码没考虑的.<br>

如果replicas设置为3, 那么就会存在以下问题:

1. `PUT/POST` object的时候会通过rados上传相同文件至少2次, 重复操作.
2. `DELETE` object的时候有一定几率失败, 因为至少需要`(3 // 2) + 1`个成功才算成功, 然而ceph中只能删除一次, 因此会导致只成功删除1次, 其余2次失败.

因此需要修改*proxy/controller/obj.py*下的`ObjectController`的`PUT`和`DELETE`方法, 代码略.


### 参考链接:

<https://software.intel.com/en-us/blogs/2015/02/03/using-multiple-backends-in-openstack-swift>

