---
layout: post
title:  "Ceph Radosgw install Guide"
date:   2016-05-31 14:30:53
categories: Ceph Radosgw
---

radosgw 是建立在 librados 之上的对象存储网关，和 openstack swift 一样用来存放 object。同时它支持两种API：s3 和 swift。就是说 radosgw 可以替代 swift 作为 openstack 的对象存储（object storage）。

本文使用的版本是 `jewel` `10.2.1`，下载地址：<http://download.ceph.com/rpm-jewel/el7/x86_64/>

以 `radosgw.Control-1` 单节点部署结合 `keystone` 为例进行说明：

## 安装 radosgw

```
yum install -y ceph-radosgw
```

## 修改 /etc/ceph/ceph.conf

最后面添加如下:

```
[client.radosgw.Control-1]
# 主机名
host = Control-1
rgw socket path = ""
# 启动端口
rgw_frontends = civetweb port=8080
# keystone地址
rgw keystone url = http://10.15.2.113:5000
# keystone admin用户
rgw keystone admin user = admin
rgw keystone admin password = admin
rgw keystone admin project = admin
rgw keystone admin domain = default
rgw keystone api version = 3
rgw keystone accepted roles = SwiftOperator,admin,_member_，project_admin，member2
rgw keystone token cache size = 500
rgw keystone revocation interval = 500
# 设置使用keystone作为radosgw认证
rgw s3 auth use keystone = true
# keystone没有开启ssl设置为false
rgw keystone verify ssl = false
# 如果启用了cephx认证需要指定keyring
#keyring = /etc/ceph/ceph.client.radosgw.Control-1.keyring
```

## 生成 keyring（如果没有 cephx，可跳过）

```
ceph auth get-or-create client.radosgw.Control-1 mon 'allow *' mds 'allow *' osd 'allow *' -o /etc/ceph/ceph.client.radosgw.Control-1.keyring
```

## 设置 keystone endpoint

```
openstack service create --name swift object-store
openstack endpoint create --region RegionOne swift public http://192.168.130.100:8080/swift/v1
openstack endpoint create --region RegionOne swift admin http://192.168.130.100:8080/swift/v1
openstack endpoint create --region RegionOne swift internal  http://192.168.130.100:8080/swift/v1
openstack service create --name swift_s3 s3
openstack endpoint create --region RegionOne swift_s3 public  http://192.168.130.100:8080
openstack endpoint create --region RegionOne swift_s3 admin http://192.168.130.100:8080
openstack endpoint create --region RegionOne swift_s3 internal http://192.168.130.100:8080
```

## 添加服务自启动

```
systemctl enable ceph-radosgw@radosgw.Control-1
systemctl start ceph-radosgw@radosgw.Control-1
```

## 测试

```
swift -V3 list
```

## 参考链接：

<http://docs.ceph.com/docs/hammer/radosgw/keystone/>
