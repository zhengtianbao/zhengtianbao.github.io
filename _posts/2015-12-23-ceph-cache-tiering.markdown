---
layout: post
title:  "ceph cache tiering"
date:   2015-12-23 15:41:53
categories: Ceph
---


### 配置说明

普通的SATA盘的读写性能在75MB/s左右, SAS盘是它的两倍150MB/s左右.

选用的SSD以[Intel SSD DC P3700 400G](http://www.intel.com/content/www/us/en/solid-state-drives/intel-ssd-dc-family-for-pcie.html)为例, 其顺序读写为 2700/1080 MB/s.

一般而言, 为了提高读写性能ceph的journal建议放在SSD盘上, journal的`flush interval`默认配置是5s, 配小了体现不出journal层提升的速度, 配大了又会导致数据落盘(osd)慢.

常用的配置是1块SSD做3个osd(3个SATA盘)的journal:

{% highlight text %}
1080 MB ≈ 75MB/s * 5s * 3 = 1125MB/s
{% endhighlight %}

基本上达到了SSD盘的写最大值, 当然, 这是满负荷测试写性能的峰值, 实际情况不会出现一直跑满.

为了保证数据安全性, 做journal的SSD一般会做*RAID1*.

这样也就是说: 2 * SSD (400G) 配 3 * SATA (1T)

如果有4块SSD就做*RAID10*, 这样就相当与有800G的可用空间, 读写速度都乘以2, 可以抗6块osd(这里不考虑网络带宽情况).

一般每个osd分配6G空间做journal就足够了, 假如节点上有3个osd, 就只需要分配18G的空间.

这样就产生一个问题, SSD盘作为一种昂贵的资源, 剩余的空间如何利用? 那为什么不用小一点的SSD, 答案是因为小的SSD盘性能上差很多.

一个行之有效的方案是将SSD剩下的空间也独立出来作为osd, 因为如之前所说SSD盘实际生产环境中不可能完全跑满, 有资源剩余.

如何将这个osd作为高性能型的盘, 参见我之前的文章:

[cinder multi-backend support with ceph](http://zhengtianbao.com/ceph/cinder/2014/12/22/cinder-multi-backend-support-with-ceph.html)


### cache tiering

cache tiering作为缓存层能在一定程度上提升普通pool的读写, 可以理解为将SSD作为SATA的缓存层来用, 在不繁忙的情况下是完全OK的.

如何使用cache tiring:

<http://docs.ceph.com/docs/master/rados/operations/cache-tiering/>

thanks:)

### 参考链接:

<http://www.sebastien-han.fr/blog/2014/06/10/ceph-cache-pool-tiering-scalable-cache/>
