---
layout: post
title:  "Libvirt 创建整机快照"
date:   2018-07-16 17:06:53
categories: Libvirt
---

最近在尝试使用`ansible`部署`ceph`作为对象存储使用，因为没有部署`openstack`，所以只能手动对虚拟机进行快照以及回滚操作了。但是发现`virsh`没有对挂载盘进行`snapshot`相关的操作。只能对虚拟机整机级别的快照回滚。回忆`cinder`对`volume`的快照，取决于后端存储是否支持，例如`ceph`对`rbd`就提供了`snapshot`的操作。这篇文章基于`libvirt`本身进行快照操作。

### 1. 创建硬盘

```
# qemu-img create -f qcow2 /var/lib/libvirt/images/centos7-ceph-xfs.disk 100G -o preallocation=full
```

注：支持快照功能需要硬盘为qcow2格式。

### 2. 挂载硬盘

```
# virsh attach-disk --domain centos7-test /var/lib/libvirt/images/centos7-ceph-xfs.disk --target vdb --persistent --config
# virsh shutdown centos7-test
```

注： 挂载之后还需要修改xml配置文件，默认生成的依然是raw格式。

### 3. 修改xml配置

```
# virsh edit centos7-test
<disk type='file' device='disk'>
      <driver name='qemu' type='qcow2'/>
      <source file='/var/lib/libvirt/images/centos7-ceph-xfs.disk'/>
      <target dev='vdb' bus='virtio'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x08' function='0x0'/>
</disk>
# virsh start centos7-test
```

### 4. 创建快照

```
# virsh snapshot-create centos7-test
```

