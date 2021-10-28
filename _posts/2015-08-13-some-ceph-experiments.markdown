---
layout: post
title:  "ceph experiments about replication size"
date:   2015-08-13 15:15:47
categories: Ceph
---


在一次ceph环境的部署中, 由于一台机器主板坏了, 导致ceph集群状态异常, 在恢复过程中意外的发现pool replication size的问题, 这里记录下.


下面是模拟的ceph cluster环境:

| node   | ip          | server      |
|--------|-------------|-------------|
| ceph-1 | 10.160.0.41 | osd.0 mon.0 |
| ceph-2 | 10.160.0.42 | osd.1 mon.1 |
| ceph-3 | 10.160.0.43 | osd.2 mon.2 |
| cns-5  | 10.160.0.55 | osd.3 osd.4 |

_cns-5_上的_osd.3_和_osd.4_是由一块硬盘分出来的, 用来模拟ssd和sata盘两种类型:

{% highlight text %}
# parted -a optimal --script /dev/sdb mktable gpt
# parted -a optimal -s /dev/sdb mkpart ceph-ssd 0% 40%
# parted -a optimal -s /dev/sdb mkpart ceph-sata 50% 90%
# mkfs.xfs -f /dev/sdb1
# mkfs.xfs -f /dev/sdb2
# blkid
/dev/sda1: UUID="e1352926-d093-4960-91ab-7b2435b3149d" TYPE="xfs" 
/dev/sdb1: UUID="2a64d0d6-760d-4e85-b09c-d982604e0d4e" TYPE="xfs" PARTLABEL="ceph-ssd" PARTUUID="e223907f-9ab2-4e62-91c4-29720d27b4d0" 
/dev/sdb2: UUID="29a9be1b-0178-4fc8-ae98-4485f561529a" TYPE="xfs" PARTLABEL="ceph-sata" PARTUUID="73503769-ce44-4599-843f-35b07c0820e5"
# ceph osd create e223907f-9ab2-4e62-91c4-29720d27b4d0
# ceph osd create 73503769-ce44-4599-843f-35b07c0820e5
# mount -o rw,noatime,inode64 /dev/sdb1 /var/lib/ceph/osd/ceph-3
# mount -o rw,noatime,inode64 /dev/sdb2 /var/lib/ceph/osd/ceph-4
# ceph-osd -c /etc/ceph/ceph.conf -i 3 --mkfs --osd-uuid e223907f-9ab2-4e62-91c4-29720d27b4d0
# ceph-osd -c /etc/ceph/ceph.conf -i 4 --mkfs --osd-uuid 73503769-ce44-4599-843f-35b07c0820e5
# /etc/init.d/ceph start osd.3
# /etc/init.d/ceph start osd.4
{% endhighlight %}

CRUSHMap:

{% highlight text %}
# begin crush map
tunable choose_local_tries 0
tunable choose_local_fallback_tries 0
tunable choose_total_tries 50
tunable chooseleaf_descend_once 1

# devices
device 0 osd.0
device 1 osd.1
device 2 osd.2
device 3 osd.3
device 4 osd.4

# types
type 0 osd
type 1 host
type 2 chassis
type 3 rack
type 4 row
type 5 pdu
type 6 pod
type 7 room
type 8 datacenter
type 9 region
type 10 root

# buckets
host ceph-1 {
        id -2           # do not change unnecessarily
        # weight 0.100
        alg straw
        hash 0  # rjenkins1
        item osd.0 weight 0.100
}
host ceph-2 {
        id -3           # do not change unnecessarily
        # weight 0.100
        alg straw
        hash 0  # rjenkins1
        item osd.1 weight 0.100
}
host ceph-3 {
        id -4           # do not change unnecessarily
        # weight 0.100
        alg straw
        hash 0  # rjenkins1
        item osd.2 weight 0.100
}
host cns-5-ssd {
        id -5           # do not change unnecessarily
        # weight 0.100
        alg straw
        hash 0  # rjenkins1
        item osd.3 weight 0.050
}
host cns-5-sata {
        id -6           # do not change unnecessarily
        # weight 0.100
        alg straw
        hash 0  # rjenkins1
        item osd.4 weight 0.050
}

root ssd {
        id -7
        alg straw
        hash 0
        item ceph-3 weight 1.00
        item cns-5-ssd weight 1.00
}

root sata {
        id -8
        alg straw
        hash 0
        item ceph-1 weight 1.00
        item ceph-2 weight 1.00
        item cns-5-sata weight 1.00

}
root default {
        id -1           # do not change unnecessarily
        # weight 0.400
        alg straw
        hash 0  # rjenkins1
}

# rules
rule ssd {
        ruleset 1
        type replicated
        min_size 2
        max_size 2
        step take ssd
        step chooseleaf firstn 0 type host
        step emit
}

rule sata {
        ruleset 2
        type replicated
        min_size 2
        max_size 2
        step take sata
        step chooseleaf firstn 0 type host
        step emit
}

rule replicated_ruleset {
        ruleset 0
        type replicated
        min_size 1
        max_size 10
        step take default
        step chooseleaf firstn 0 type host
        step emit
}

# end crush map
{% endhighlight %}

当改变了CRUSHMap, OSD重启后默认不会加入到该location, 需要配置ceph.conf中, 对应的OSD下添加`osd crush location`:

{% highlight text %}
[osd.3]
   osd crush location = "root=ssd host=cns-5-ssd"

[osd.4]
   osd crush location = "root=sata host=cns-5-sata"
{% endhighlight %}

osd tree:

{% highlight text %}
[root@cns-5 ~]# ceph osd tree
# id    weight  type name       up/down reweight
-8      3       root sata
-2      1               host ceph-1
0       0.09999                 osd.0   up      1
-3      1               host ceph-2
1       0.09999                 osd.1   up      1
-6      1               host cns-5-sata
4       0.04999                 osd.4   up      1
-7      2       root ssd
-4      1               host ceph-3
2       0.09999                 osd.2   up      1
-5      1               host cns-5-ssd
3       0.04999                 osd.3   up      1
-1      0       root default
{% endhighlight %}

创建两个_test-ssd_和_test-sata_两个pool, 分别将数据保存到ssd和sata上:

{% highlight text %}
# ceph osd pool create test-sata 256 256 replicated sata
# ceph osd pool create test-ssd 256 256 replicated ssd
# ceph osd pool set test-sata crush_ruleset 2
# ceph osd pool set test-ssd crush_ruleset 1
{% endhighlight %}

{% highlight text %}
[root@cns-5 ~]# ceph osd dump |grep test-
pool 6 'test-ssd' replicated size 2 min_size 1 crush_ruleset 1 object_hash rjenkins pg_num 256 pgp_num 256 last_change 270 flags hashpspool stripe_width 0
pool 7 'test-sata' replicated size 2 min_size 1 crush_ruleset 2 object_hash rjenkins pg_num 256 pgp_num 256 last_change 298 flags hashpspool stripe_width 0
{% endhighlight %}

如你所见, 现在_test-ssd_和_test-sata_的_replicated size_都为_2_, _min size_为_1_;

由此可得_test-ssd_中的数据将会在_osd.2_和_osd.3_中各保存一份;

假设_osd.3_ 由于某些原因down了, 由于_min size_为_1_, 因此只要_osd.2_成功写如数据, _test-ssd_ pool依然能够正常对外工作.

{% highlight text %}
[root@cns-5 ~]# rbd --pool test-ssd ls
bbb
vvvv
www
xxx
[root@cns-5 ~]# /etc/init.d/ceph stop osd.3
=== osd.3 === 
Stopping Ceph osd.3 on cns-5...kill 31448...kill 31448...done
[root@cns-5 ~]# ceph osd tree
# id    weight  type name       up/down reweight
-8      3       root sata
-2      1               host ceph-1
0       0.09999                 osd.0   up      1
-3      1               host ceph-2
1       0.09999                 osd.1   up      1
-6      1               host cns-5-sata
4       0.04999                 osd.4   up      1
-7      2       root ssd
-4      1               host ceph-3
2       0.09999                 osd.2   up      1
-5      1               host cns-5-ssd
3       0.04999                 osd.3   down    1
-1      0       root default
[root@cns-5 ~]# rbd --pool test-ssd ls
bbb
vvvv
www
xxx
[root@cns-5 ~]# rbd --pool test-ssd create test --size 1000
[root@cns-5 ~]# rbd --pool test-ssd ls
bbb
test
vvvv
www
xxx

{% endhighlight %}

结果与猜想的一样, 同理, 在这种情况下_test-sata_也是一样, 这里省略测试结果; 

当_test-sata_的_replicated size_设置为_3_, _min size_设置为_1_的情况下, 按推论应该在_osd.0_, _osd.1_, _osd.4_上都保存一份数据, 当挂了一个或者两个OSD的时候依然正常工作, 然而实际情况却`hang`住了!

{% highlight text %}
[root@cns-5 ~]# ceph osd pool set test-sata size 3
set pool 7 size to 3
[root@cns-5 ~]# ceph osd tree
# id    weight  type name       up/down reweight
-8      3       root sata
-2      1               host ceph-1
0       0.09999                 osd.0   up      1
-3      1               host ceph-2
1       0.09999                 osd.1   up      1
-6      1               host cns-5-sata
4       0.04999                 osd.4   up      1
-7      2       root ssd
-4      1               host ceph-3
2       0.09999                 osd.2   up      1
-5      1               host cns-5-ssd
3       0.04999                 osd.3   up      1
-1      0       root default
[root@cns-5 ~]# ceph health
HEALTH_OK
[root@cns-5 ~]# ceph osd dump |grep test-sata
pool 7 'test-sata' replicated size 3 min_size 1 crush_ruleset 2 object_hash rjenkins pg_num 256 pgp_num 256 last_change 328 flags hashpspool stripe_width 0
[root@cns-5 ~]# rbd --pool test-sata ls
^C                      # hang here!!!
[root@cns-5 ~]# rbd --pool test-ssd ls
bbb
test
vvvv
www
xxx
[root@cns-5 ~]# ceph osd pool set test-sata size 2
set pool 7 size to 2
[root@cns-5 ~]# rbd --pool test-sata ls
test
[root@cns-5 ~]# 

{% endhighlight %}

我不知道这是什么原因导致的, 一个猜测是因为OSD数目过少, CRUSH算法找不到合适的osd导致了这种情况??...


## 20150814

找到了问题的原因: 是由于在CRUSHMap中的_rule_设置的两个属性`min_size`, `max_size`导致的.

- min_size: If a pool makes fewer replicas than this number, CRUSH will *NOT* select this rule.
- max_size: If a pool makes more replicas than this number, CRUSH will *NOT* select this rule.

也就是说当某个pool的_replicated size_只有在区间[min_size, max_size]之间时, CRUSH算法才会选择这个rule.

开始误以为这两个参数和pool下的_replicated size_, _min size_是一个意思, 作为default值配置, 才产生了这个错误.

最后建议rule的`min_size`设为_1_, `max_size`设为_10_.



## 参考链接:

<http://ceph.com/docs/master/rados/operations/pools/#set-the-number-of-object-replicas>

<http://cephnotes.ksperis.com/blog/2015/02/02/crushmap-example-of-a-hierarchical-cluster-map>

<http://www.admin-magazine.com/HPC/Articles/RADOS-and-Ceph-Part-2>

<http://tracker.ceph.com/issues/2454>
