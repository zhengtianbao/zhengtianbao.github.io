---
title: "ceph cache tiering"
date: 2015-12-23 15:41:53
categories: ["2015"]
tags: [ceph]
---

本文介绍 ceph cache tiering 的使用场景。

## 配置策略说明

普通的 SATA 盘的读写性能在 75MB/s 左右，SAS盘是它的两倍 150MB/s 左右。

选用的 SSD 以 **Intel SSD DC P3700 400G** 为例，其顺序读写为 2700/1080 MB/s。

一般而言，为了提高读写性能 ceph 的 journal 建议放在 SSD 盘上，journal 的 `flush interval` 默认配置是 5s，配小了体现不出 journal 层提升的速度，配大了又会导致数据落盘（osd）慢。

常用的配置是 1 块 SSD 做 3 个 osd（3 个 SATA 盘）的 journal：

```
1080MB ≈ 75MB/s * 5s * 3 = 1125MB/s
```

基本上达到了 SSD 盘的写最大值，当然，这是满负荷测试写性能的峰值，实际情况不会出现一直跑满。

为了保证数据安全性，做 journal 的 SSD 一般会做 **RAID1**。

这样也就是说：2 * SSD（400G）配 3 * SATA（1T）

如果有 4 块 SSD 就做 **RAID10** ，这样就相当与有 800G 的可用空间，读写速度都乘以 2，可以抗 6 块 osd（这里不考虑网络带宽情况）。

一般每个 osd 分配 6G 空间做 journal 就足够了，假如节点上有 3 个 osd，就只需要分配 18G 的空间。

这样就产生一个问题，SSD 盘作为一种昂贵的资源，剩余的空间如何利用？那为什么不用小一点的 SSD，答案是因为小的 SSD 盘性能上差很多。

一个可行有效的方案是将 SSD 剩下的空间也独立出来作为 osd，因为如之前所说 SSD 盘实际生产环境中不可能完全跑满，有资源剩余。

如何将这个 osd 作为高性能型的盘，参见我之前的文章：

[cinder multi-backend support with ceph](https://zhengtianbao.com/ceph/cinder/2014/12/22/cinder-multi-backend-support-with-ceph.html)

## cache tiering

cache tiering 作为缓存层能在一定程度上提升普通 pool 的读写，可以理解为将 SSD 作为 SATA 的缓存层来用，在不繁忙的情况下是完全 OK 的.

如何使用 cache tiring：<http://docs.ceph.com/docs/master/rados/operations/cache-tiering/>

thanks :)

## 参考链接

<http://www.sebastien-han.fr/blog/2014/06/10/ceph-cache-pool-tiering-scalable-cache/>
