---
layout: post
title:  "cinder multi-backend support with ceph"
date:   2014-12-22 20:46:53
categories: Ceph Cinder
---

本文主要记录了将 ceph 中的不同读写性能的存储 pool 作为 cinder 的多种后端存储的过程。

## 服务节点说明

- 为不同虚机分配不同类型的云硬盘，容量型（SATA）和性能型（SSD）
- ceph 设置不同 pool 分配到对应 osd 硬盘的规则
- cinder 设置 multi-backend 支持

| hostname | IP         | volume-type | services     |
|----------|------------|-------------|--------------|
| ceph-1   | 10.180.4.1 | SSD         | mon.0，osd.0 |
| ceph-2   | 10.180.4.2 | SSD         | mon.1，osd.1 |
| ceph-3   | 10.180.4.3 | SATA        | mon.2，osd.2 |
| ceph-4   | 10.180.4.4 | SATA        | osd.3        |


## 修改CRUSHMap

### 1. 从 ceph 集群获取到现有 crushmap

```
[root@ceph-1 ~]# ceph osd getcrushmap -o {compiled-crushmap-filename}
[root@ceph-1 ~]# crushtool -d {compiled-crushmap-filename} -o {decompiled-crushmap-filename}
```

### 2. 编辑配置 crushmap

```
[root@ceph-1 ~]# vim {decompiled-crushmap-filename}
```

配置规则详见：<http://ceph.com/docs/master/rados/operations/crush-map/#crush-map-parameters>

这里的配置如下：

```
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

host ceph-osd-ssd-server1 {
        id -1
        alg straw
        hash 0
        item osd.0 weight 1.00
}

host ceph-osd-ssd-server2 {
        id -2
        alg straw
        hash 0
        item osd.1 weight 1.00
}

host ceph-osd-sata-server1 {
        id -3
        alg straw
        hash 0
        item osd.2 weight 1.00
}


host ceph-osd-sata-server2 {
        id -4
        alg straw
        hash 0
        item osd.3 weight 1.00
}

root ssd {
        id -5
        alg straw
        hash 0
        item ceph-osd-ssd-server1 weight 1.00
        item ceph-osd-ssd-server2 weight 1.00
}

root sata {
        id -6
        alg straw
        hash 0
        item ceph-osd-sata-server1 weight 1.00
        item ceph-osd-sata-server2 weight 1.00
}
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

# buckets
root default {
        id -10          # do not change unnecessarily
        # weight 0.000
        alg straw
        hash 0  # rjenkins1
}

# rules
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
```

### 3. 导入编辑后的新 crushmap 到集群内

```
[root@ceph-1 ~]# crushtool -c {decompiled-crush-map-filename} -o {compiled-crush-map-filename}
[root@ceph-1 ~]# ceph osd setcrushmap -i  {compiled-crushmap-filename}
```

## 创建 pool 并分配 rule

| pool name       | volume type | rule id in crushmap |
|-----------------|-------------|---------------------|
| 180-nova        | SSD         | 1                   |
| 180-glance      | SATA        | 2                   |
| 180-swift       | SATA        | 2                   |
| 180-cinder-ssd  | SSD         | 1                   |
| 180-cinder-sata | SATA        | 2                   |

### 1. 创建 pools

```
[root@ceph-1 ~]# ceph osd pool create 180-nova 256 256 replicated ssd
[root@ceph-1 ~]# ceph osd pool create 180-glance 256 256 replicated sata
[root@ceph-1 ~]# ceph osd pool create 180-swift 256 256 replicated sata
[root@ceph-1 ~]# ceph osd pool create 180-cinder-ssd 256 256 replicated ssd
[root@ceph-1 ~]# ceph osd pool create 180-cinder-sata 256 256 replicated sata
```

### 2. 设置各个 pool 的 rule

```
[root@ceph-1 ~]# ceph osd pool set 180-nova crush_ruleset 1
[root@ceph-1 ~]# ceph osd pool set 180-glance crush_ruleset 2
[root@ceph-1 ~]# ceph osd pool set 180-swift crush_ruleset 2
[root@ceph-1 ~]# ceph osd pool set 180-cinder-ssd crush_ruleset 1
[root@ceph-1 ~]# ceph osd pool set 180-cinder-sata crush_ruleset 2
```

### 3. 设置 ceph 默认存在的 3 个 pool 的 rule 为 sata

```
[root@ceph-1 ~]# ceph osd pool set data crush_ruleset 1
[root@ceph-1 ~]# ceph osd pool set metadata crush_ruleset 1
[root@ceph-1 ~]# ceph osd pool set rbd crush_ruleset 1
```

### 4. 查看各个 pool 是否设置正确

```
[root@ceph-1 ~]# ceph osd tree
# id    weight  type name       up/down reweight
-10     0       root default
-6      3       root sata
-3      2               host ceph-osd-sata-server1
2       1                       osd.2   up      1
-4      1               host ceph-osd-sata-server2
3       1                       osd.3   up      1
-5      2       root ssd
-1      1               host ceph-osd-ssd-server1
0       1                       osd.0   up      1
-2      1               host ceph-osd-ssd-server2
1       1                       osd.1   up      1
```

至此 ceph 部分已经配置完毕，下面需要配置 cinder，分配给 cinder-driver 配置不同的 pool（180-cinder-ssd 和 180-cinder-sata）。


## 配置 cinder

### 1. 修改 cinder 配置文件 /etc/cinder/cinder.conf

在 [default] 部分修改 enabled_backends 配置为：

```
enabled_backends=RBDDriver-1，RBDDriver-2
```

在 cinder.conf 的最后加上两个段：

```
[RBDDriver-1]
rbd_pool=180-cinder-ssd
volume_driver=cinder.volume.drivers.rbd.RBDDriver
volume_backend_name=RBD-SSD

[RBDDriver-2]
rbd_pool=180-cinder-sata
volume_driver=cinder.volume.drivers.rbd.RBDDriver
volume_backend_name=RBD-SATA
```

这样配置的两个 RBDDriver 分别使用 ceph 不同的 rbd_pool。

### 2. 重启 cinder 服务

```
[root@cinderserver-1 cinder]# for s in api scheduler volume; do service openstack-cinder-$s restart; done
```

### 3. 为 cinder 创建 ssd 类型的 type

```
[root@cinderserver-1 cinder]# cinder type-create ssd
```

### 4. 配置 ssd 类型的 type

```
[root@cinderserver-1 cinder]# cinder type-key ssd set volume_backend_name=RBD-SSD
```

### 5. 配置 common 类型的 type 为 sata

```
[root@cinderserver-1 cinder]# cinder type-key common set volume_backend_name=RBD-SATA
```

### 6. 查看 cinder type 状态是否正常分配

```
[root@cinderserver-1 ~]# cinder extra-specs-list
+--------------------------------------+--------+---------------------------------------+
|                  ID                  |  Name  |              extra_specs              |
+--------------------------------------+--------+---------------------------------------+
| 06a9dc8a-d431-4313-b9fb-994311897aa1 |  ssd   |  {u'volume_backend_name': u'RBD-SSD'} |
| d98e86f0-4221-4f0b-bc42-a354a40ac0c2 | common | {u'volume_backend_name': u'RBD-SATA'} |
+--------------------------------------+--------+---------------------------------------+
```

至此 cinder 部分配置完毕，可以通过 cinder client 测试创建不同 type 的 volume。

## 测试

### 1. 创建 ssd 与 sata 类型的 volume

```
[root@cinderserver-1 ~]# cinder create --display-name sata-test --volume-type common 1
[root@cinderserver-1 ~]# cinder create --display-name ssd-test --volume-type ssd 1
[root@cinderserver-1 ~]# cinder --os-username cinder --os-password CONFIG_CINDER_KS_PW --os-tenant-name services --os-auth-url http://10.180.0.33:5000/v2.0 list
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
|                  ID                  |   Status  | Display Name | Size | Volume Type | Bootable | Attached to |
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
| 4f8ccd97-e456-42d0-a5b9-9c5fd8ab72ff | available |  sata-test   |  1   |    common   |  false   |             |
| ab6160a2-af68-463e-b8a8-71f8fefb9b52 | available |   ssd-test   |  1   |     ssd     |  false   |             |
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
```

### 2. 验证是否创建在指定的 ceph pool 下

```
[root@ceph-1 ~]# rados ls -p 180-cinder-sata
rbd_directory
rbd_header.19295a69432a
rbd_id.volume-4f8ccd97-e456-42d0-a5b9-9c5fd8ab72ff
[root@ceph-1 ~]# rados ls -p 180-cinder-ssd
rbd_directory
rbd_header.192c45b6cbfd
rbd_id.volume-ab6160a2-af68-463e-b8a8-71f8fefb9b52
```

至此，cinder 与 ceph 配合使用 multiple backend 测试完毕。


## 其他测试

### 1. 一台机器起多个 osd 情况

```
[root@ceph-3 ~]# df -Th
Filesystem     Type      Size  Used Avail Use% Mounted on
/dev/sda1      xfs       137G  1.8G  135G   2% /
devtmpfs       devtmpfs  7.8G     0  7.8G   0% /dev
tmpfs          tmpfs     7.8G     0  7.8G   0% /dev/shm
tmpfs          tmpfs     7.8G  8.6M  7.8G   1% /run
tmpfs          tmpfs     7.8G     0  7.8G   0% /sys/fs/cgroup
/dev/sdb1      xfs       217G  4.1G  213G   2% /var/lib/ceph/osd/ceph-2
/dev/sdb2      xfs       217G  4.1G  213G   2% /var/lib/ceph/osd/ceph-4
[root@ceph-3 ~]# netstat -tunlp |grep ceph-osd
tcp        0      0 10.180.4.3:6809         0.0.0.0:*               LISTEN      22439/ceph-osd      
tcp        0      0 10.180.4.3:6800         0.0.0.0:*               LISTEN      22365/ceph-osd      
tcp        0      0 10.180.4.3:6801         0.0.0.0:*               LISTEN      22365/ceph-osd      
tcp        0      0 10.180.4.3:6802         0.0.0.0:*               LISTEN      22365/ceph-osd      
tcp        0      0 10.180.4.3:6803         0.0.0.0:*               LISTEN      22365/ceph-osd      
tcp        0      0 10.180.4.3:6804         0.0.0.0:*               LISTEN      22365/ceph-osd      
tcp        0      0 10.180.4.3:6805         0.0.0.0:*               LISTEN      22439/ceph-osd      
tcp        0      0 10.180.4.3:6806         0.0.0.0:*               LISTEN      22439/ceph-osd      
tcp        0      0 10.180.4.3:6807         0.0.0.0:*               LISTEN      22439/ceph-osd      
tcp        0      0 10.180.4.3:6808         0.0.0.0:*               LISTEN      22439/ceph-osd      
[root@ceph-3 ~]# ps -ef |grep ceph-osd
root     22365     1  0 Dec19 ?        00:09:16 ceph-osd -i 2
root     22439     1  0 Dec19 ?        00:09:23 ceph-osd -i 4
root     26412 26134  0 14:16 pts/0    00:00:00 grep --color=auto ceph-osd
[root@ceph-3 ~]# ceph osd tree
# id    weight  type name       up/down reweight
-10     0       root default
-6      3       root sata
-3      2               host ceph-osd-sata-server1
2       1                       osd.2   up      1
4       1                       osd.4   up      1
-4      1               host ceph-osd-sata-server2
3       1                       osd.3   up      1
-5      2       root ssd
-1      1               host ceph-osd-ssd-server1
0       1                       osd.0   up      1
-2      1               host ceph-osd-ssd-server2
1       1                       osd.1   up      1
```

在 ceph-3 机器上起 osd.2 与 osd.4，发现 ceph 分别起了两个进程，监听端口号从 6800 递增，处理请求正常。

### 2. Journal

Journal 可以在 ceph.conf 中配置

```
osd journal size = 4096
osd journal = /var/lib/ceph/osd/ceph-$id/journal
```

推荐指定到ssd上提高性能，同时size按以下规则设置:

```
osd journal size = {2 * (expected throughput * filestore max sync interval)}
```

## 参考链接

<http://blog.csdn.net/juvxiao/article/details/20536117>
