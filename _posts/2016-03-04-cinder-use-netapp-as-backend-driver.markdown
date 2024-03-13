---
layout: post
title:  "Cinder use Netapp as backend driver"
date:   2016-03-04 14:58:53
categories: Cinder Netapp
---

本文主要介绍 cinder 对接 Netapp 商业存储做为后端存储的配置过程。

## cinder配置

### 安装 nfs-utils

cinder-volume 节点以及 nova-compute 节点需要预先安装 nfs-utils，并启动相关服务

```
yum install nfs-utils
systemctl enable rpcbind
systemctl enable nfs-server
systemctl enable nfs-lock
systemctl enable nfs-idmap
systemctl start rpcbind
systemctl start nfs-server
systemctl start nfs-lock
systemctl start nfs-idmap
```

### 编辑 /etc/cinder/cinder.conf

创建一个段（segment）配置如下：

```
[netapp]
volume_driver = cinder.volume.drivers.netapp.dataontap.nfs_cmode.NetAppCmodeNfsDriver
netapp_storage_family = ontap_cluster
netapp_storage_protocol = nfs
netapp_login=$(cluster_managerment_username)
netapp_password=$(cluster_managerment_password)
netapp_server_hostname=$(cluster_host_ip)
netapp_server_port=80
netapp_vserver = svm1
nfs_shares_config = /etc/cinder/nfs_shares
volume_backend_name=netapp
```

编辑 [DEFAULT] 段下的配置项：

```
enabled_backends = netapp
```

如果要启动多个 backends 就在后面加上 netapp 即可，如：

```
enabled_backends=lvm,nfs
```

### 新建文件 /etc/cinder/nfs_shares

格式为：NFSServerIP:命名空间，如下：

```
192.168.131.21:/vol_04032016_151540
```

### 重启 cinder-volume 服务

```
# systemctl restart openstack-cinder-volume
```

### 创建 volume type 使用 NFS

```
# cinder type-create netapp
# cinder type-key netapp set volume_backend_name=netapp
# cinder type-list
+--------------------------------------+---------+-------------+-----------+
|                  ID                  |   Name  | Description | Is_Public |
+--------------------------------------+---------+-------------+-----------+
| 44ada0a5-144a-4115-addd-c4612140a143 | netapp  |      -      |    True   |
| c683ea54-3a4e-4733-b0b9-f3ba2b421717 |  iscsi  |      -      |    True   |
| d6740a08-2a80-4a7d-8978-cbd228cd3263 | netapp2 |      -      |    True   |
+--------------------------------------+---------+-------------+-----------+
# cinder extra-specs-list
+--------------------------------------+---------+--------------------------------------+
|                  ID                  |   Name  |             extra_specs              |
+--------------------------------------+---------+--------------------------------------+
| 44ada0a5-144a-4115-addd-c4612140a143 | netapp  | {u'volume_backend_name': u'netapp '} |
| c683ea54-3a4e-4733-b0b9-f3ba2b421717 |  iscsi  |   {u'volume_backend_name': u'lvm'}   |
| d6740a08-2a80-4a7d-8978-cbd228cd3263 | netapp2 | {u'volume_backend_name': u'netapp2'} |
+--------------------------------------+---------+--------------------------------------+
```

### 创建 volume 测试指定 volume_type 为 netapp

```
# cinder create --volume_type netapp --display_name nfsvolume 1
```

## 参考链接：

<https://access.redhat.com/articles/1323213>
