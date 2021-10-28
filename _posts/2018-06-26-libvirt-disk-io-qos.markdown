---
layout: post
title: "Libvirt 磁盘IO限速"
date:  2018-06-26 16:41:20
categories: Libvirt Qemu KVM
---

随着[我们公司](http://jfbrother.com)开发业务的不断增重，Dell服务器上的虚拟机日益增多，而虚拟机磁盘却都压在物理服务器单块raid磁盘上共享IO。考虑到某台虚拟机将要部署爬虫业务，可能占用大量IO，因此防患与未然，限制其虚拟机的磁盘IO，以免影响其他虚拟机或者宿主机的正常读写。

鉴于我的从业经验以及公司规模，部署的是KVM虚拟化，通过Libvirt管理虚机，网络是桥接模式的flat network，存储也是最简单的本地LVM，很简单，没有上OpenStack，然而底层原理都是一致的。OpenStack中的Cinder可以对虚拟机的每块磁盘IO进行限速，包括：`read/s` `write/s` `iops`等颗粒度，因此，KVM也能支持磁盘IO限速。

## qemu-kvm 磁盘限速参数

```
# qemu-kvm -drive file=/var/lib/libvirt/images/testvm.img,format=qcow2,if=none,id=drive-virtio-disk0,throttling.bps-read=10000000,throttling.bps-write=10000000
```

参数说明：

```
file: 硬盘路径
throttling.bps-read: 读字节每秒
throttling.bps-write: 写字节每秒
# 其他参数
throttling.bps-total: 读写总字节每秒
throttling.iops-total: 总iops
throttling.iops-read：读iops
throttling.iops-write：写iops
```

qemu-kvm是底层实现，这里通过libvirt管理虚拟机，因此通过virsh命令来做磁盘IO限速。

## Libvirt 磁盘IO限速配置  

```
# 查看虚拟机硬盘
# virsh domblklist testvm
Target     Source
------------------------------------------------
vda        /var/lib/libvirt/images/testvm.img
hda        -
# 对vda硬盘进行限速 读写都为10MB/s
# virsh blkdeviotune testvm vda --read-bytes-sec 10000000 --live
# virsh blkdeviotune testvm vda --write-bytes-sec 10000000 --live
```

`--live`参数是立即生效，然而实际使用过程中发现无法生效，还是得修改虚机xml配置文件。

```
# virsh shutdown testvm
# virsh edit testvm
```

修改xml文件disk中增加`<iotune>` `</iotune>`属性。

```
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2'/>
      <source file='/var/lib/libvirt/images/testvm.img'/>
      <target dev='vda' bus='virtio'/>
      <iotune>
        <read_bytes_sec>10000000</read_bytes_sec>
        <write_bytes_sec>10000000</write_bytes_sec>
      </iotune>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x06' function='0x0'/>
    </disk>
```

## 虚拟机磁盘测速

```
# 写
# dd if=/dev/zero of=test.out bs=512 count=1000000 oflag=direct
# 读
# dd if=test.out of=/dev/null bs=512 count=1000000
# 在dd命令执行过程中通过dstat命令查看读写
# dstat -d 2
-dsk/total-
 read  writ
   0  9988k
# 符合10MB/s的读写预期
```

