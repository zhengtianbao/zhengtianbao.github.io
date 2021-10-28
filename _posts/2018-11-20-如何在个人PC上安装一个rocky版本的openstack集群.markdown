---
layout: post
title: "如何在个人PC上安装一个rocky版本的openstack集群"
date:  2018-11-20 14:10:37
categories: Openstack
---

## 如何在个人PC上安装一个rocky版本的openstack集群

### 前言

最近在做高校大数据实验室项目过程中发现，就场景而言相比较容器老师更容易接受虚拟机的模式，例如想让学生搭个hadoop集群，然后在此基础上做实验，就需要有一个直观的网络拓扑让学生理解。kubernetes只暴露应用入口的设计原则显然不适合。因此，kubernetes on openstack 兴许是个好的解决方案。

openstack已经发布到rocky版本了，各个核心组件已经相当稳定，安装部署也很成熟，这个从我这次安装过程中就能发现，配置文档齐全，小问题google一下也能找到类似解决方案。

这次安装实验性质为主，可用资源有限只有一台个人PC，all in one虽然易于部署，但是出于功能演示的考虑（例如虚拟机热迁移），还是需要多节点部署，因此，openstack on vm，嵌套虚拟化试玩一下。

### 硬件配置

1台普通家用PC，配置如下：

cpu：Intel(R) Core(TM) i7-7700 CPU @ 3.60GHz

内存：32G

硬盘：1T

网卡：100Mb/s 单网口

### 规划

物理机操作系统安装CentOS7.4，qemu-kvm虚拟化软件，libvirt虚拟化管理软件，搭建网桥br0，将物理网卡enp2s0桥接到br0上，搭建网桥br-internal，模拟二层交换机，openstack 各个节点租户内网走br-ineternal。

创建两台虚拟机，配置为8 vcpus，16G，40G硬盘。安装ubuntu18.04，分别为控制节点以及计算节点。控制节点配置3个网卡：分别为管理网，虚拟机间通信的内网，外网。计算节点配置2个网卡：分别为管理网，虚拟机间通信的内网。

openstack 网络模式选择bridge vlan。存储选择ceph（已有ceph集群）。

TODO:

增加网络拓扑图

### 物理机部署

#### 1. 安装系统

BIOS中开启intel CPU虚拟化支持。

操作系统镜像版本为 `CentOS-7-x86_64-Minimal-1708.iso` 。

安装完毕后关闭selinux，删除swap分区，重启。

修改网卡配置文件，桥接物理网卡enp2s0到br0 ，以及创建网卡br-internal，类型为bridge。

#### 2. 升级内核

安装完毕后默认内核版本为 `3.10.0-693.el7.x86_64` ，删除openstack嵌套虚拟机时会产生虚拟机hang住的现象，详见下文问题1。因此需要升级到最新内核。

```
# 载入公钥
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
# 安装ELRepo
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
# 载入elrepo-kernel元数据
yum --disablerepo=\* --enablerepo=elrepo-kernel repolist
# 查看可用的rpm包
yum --disablerepo=\* --enablerepo=elrepo-kernel list kernel*
# 安装最新版本的kernel
yum --disablerepo=\* --enablerepo=elrepo-kernel install -y kernel-ml.x86_64
# 重启
reboot
# 删除旧版本工具包
yum remove kernel-tools-libs.x86_64 kernel-tools.x86_64
# 安装新版本工具包
yum --disablerepo=\* --enablerepo=elrepo-kernel install -y kernel-ml-tools.x86_64
```

#### 3. 开启嵌套虚拟化支持

```
# 查看是否支持嵌套虚拟化 N不支持 Y支持
cat /sys/module/kvm_intel/parameters/nested 
N
# 如果为N 需要手动开启
cat << EOF > /etc/modprobe.d/kvm-nested.conf
options kvm-intel nested=1
options kvm-intel enable_shadow_vmcs=1
options kvm-intel enable_apicv=1
options kvm-intel ept=1
EOF
# 重载kvm 内核模块
modprobe -r kvm_intel
modprobe -a kvm_intel
```

#### 4. 创建openstack虚拟机

```
# 安装qemu-kvm libvirt
yum install -y qemu-kvm libvirt
# 创建虚拟机基础镜像
qemu-img create -f qcow2 /var/lib/libvirt/images/ubuntu18.04.img 40G  
virt-install --name ubuntu18.04 --ram 4096 --disk path=/var/lib/libvirt/images/ubuntu18.04.img,format=qcow2,bus=virtio --vcpus 2 --os-type linux  --network bridge:br0,model=virtio --graphics vnc,listen=0.0.0.0 --console pty,target_type=serial --cdrom /opt/ubuntu-18.04.1-live-server-amd64.iso
# 安装完毕后clone虚拟机
virt-clone --original ubuntu18.04 --name controller --file /var/lib/libvirt/images/controller.img
virt-clone --original ubuntu18.04 --name computer --file /var/lib/libvirt/images/computer.img

```

编辑controller虚拟机配置

```
virsh edit controller
```

```xml
<domain type='kvm' id='3'>                 
  <name>controller</name>    
  <uuid>a3cb00c6-248e-402a-b470-3db49ab04b6b</uuid>                              
  <memory unit='KiB'>16777216</memory>      
  <currentMemory unit='KiB'>16777216</currentMemory>    
  <vcpu placement='static'>8</vcpu>  
  <resource>                 
    <partition>/machine</partition>                                              
  </resource>                   
  <os>                               
    <type arch='x86_64' machine='pc-i440fx-rhel7.0.0'>hvm</type>
    <boot dev='hd'/>                                                             
  </os>                            
  <features>                                          
    <acpi/>                
    <apic/>                                   
  </features>                       
  <cpu mode='custom' match='exact' check='full'>
    <model fallback='forbid'>Skylake-Client</model>          
    <vendor>Intel</vendor> 
    <feature policy='disable' name='ds'/>
    <feature policy='disable' name='acpi'/>
    <feature policy='require' name='ss'/>                                        
    <feature policy='disable' name='ht'/>
    <feature policy='disable' name='tm'/>
    <feature policy='disable' name='pbe'/>
    <feature policy='disable' name='dtes64'/>
    <feature policy='disable' name='monitor'/>
    <feature policy='disable' name='ds_cpl'/>                           
    <feature policy='require' name='vmx'/>
    <feature policy='disable' name='smx'/>                                       
    <feature policy='disable' name='est'/>
    <feature policy='disable' name='tm2'/>                                       
    <feature policy='disable' name='xtpr'/>
    <feature policy='disable' name='pdcm'/>             
    <feature policy='disable' name='osxsave'/>
    <feature policy='disable' name='tsc_adjust'/>
    <feature policy='require' name='clflushopt'/>
    <feature policy='require' name='pdpe1gb'/>
    <feature policy='require' name='hypervisor'/>
    <feature policy='disable' name='arat'/>
  </cpu>
  <clock offset='utc'>
    <timer name='rtc' tickpolicy='catchup'/>
    <timer name='pit' tickpolicy='delay'/>
    <timer name='hpet' present='no'/>
  </clock>
  <on_poweroff>destroy</on_poweroff>
  <on_reboot>restart</on_reboot>
  <on_crash>destroy</on_crash>
  <pm>
    <suspend-to-mem enabled='no'/>
    <suspend-to-disk enabled='no'/>
  </pm>
  <devices>
    <emulator>/usr/libexec/qemu-kvm</emulator>
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2'/>
      <source file='/var/lib/libvirt/images/controller.img'/>
      <backingStore/>
      <target dev='vda' bus='virtio'/>
      <alias name='virtio-disk0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x05' function='0x0'/>
    </disk>
    <disk type='file' device='cdrom'>
      <target dev='hda' bus='ide'/>
      <readonly/>
      <alias name='ide0-0-0'/>
      <address type='drive' controller='0' bus='0' target='0' unit='0'/>
    </disk>
    <controller type='usb' index='0' model='ich9-ehci1'>
      <alias name='usb'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x7'/>
    </controller>
    <controller type='usb' index='0' model='ich9-uhci1'>
      <alias name='usb'/>
      <master startport='0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x0' multifunction='on'/>
          </controller>
    <controller type='usb' index='0' model='ich9-uhci2'>
      <alias name='usb'/>
      <master startport='2'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x1'/>
    </controller>
    <controller type='usb' index='0' model='ich9-uhci3'>
      <alias name='usb'/>
      <master startport='4'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x2'/>
    </controller>
    <controller type='ide' index='0'>
      <alias name='ide'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x01' function='0x1'/>
    </controller>
    <controller type='pci' index='0' model='pci-root'>
      <alias name='pci.0'/>
    </controller>
    <interface type='bridge'>
      <mac address='52:54:00:b8:37:0c'/>
      <source bridge='br0'/>
      <target dev='vnet0'/>
      <model type='virtio'/>
      <alias name='net0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
    </interface>
    <interface type='bridge'>
      <mac address='52:54:00:b8:37:0d'/>
      <source bridge='br0'/>
      <target dev='vnet1'/>
      <model type='virtio'/>
      <alias name='net1'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x08' function='0x0'/>
    </interface>
    <interface type='bridge'>
      <mac address='52:54:00:b8:37:0e'/>
      <source bridge='br-internal'/>
      <target dev='vnet2'/>
      <model type='virtio'/>
      <alias name='net2'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x07' function='0x0'/>
    </interface>
    <serial type='pty'>
      <source path='/dev/pts/1'/>
      <target type='isa-serial' port='0'>
        <model name='isa-serial'/>
      </target>
      <alias name='serial0'/>
    </serial>
    <console type='pty' tty='/dev/pts/1'>
      <source path='/dev/pts/1'/>
      <target type='serial' port='0'/>
      <alias name='serial0'/>
    </console>
    <input type='mouse' bus='ps2'>
      <alias name='input0'/>
    </input>
    <input type='keyboard' bus='ps2'>
      <alias name='input1'/>
    </input>
    <graphics type='vnc' port='5900' autoport='yes' listen='0.0.0.0'>
      <listen type='address' address='0.0.0.0'/>
    </graphics>
    <video>
      <model type='cirrus' vram='16384' heads='1' primary='yes'/>
      <alias name='video0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x0'/>
    </video>
    <memballoon model='virtio'>
      <alias name='balloon0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x06' function='0x0'/>
    </memballoon>
  </devices>
  <seclabel type='dynamic' model='dac' relabel='yes'>
    <label>+107:+107</label>
    <imagelabel>+107:+107</imagelabel>
  </seclabel>
</domain>
```

编辑computer

```
virsh edit computer
```

```xml
<domain type='kvm' id='4'>                 
  <name>computer</name>                                                          
  <uuid>235506f0-8f92-45c2-813f-11306a995451</uuid>
  <memory unit='KiB'>16777216</memory>      
  <currentMemory unit='KiB'>16777216</currentMemory>
  <vcpu placement='static'>8</vcpu>                                              
  <resource>     
    <partition>/machine</partition>                   
  </resource>                   
  <os>                        
    <type arch='x86_64' machine='pc-i440fx-rhel7.0.0'>hvm</type>
    <boot dev='hd'/>                    
  </os>                            
  <features>               
    <acpi/>                 
    <apic/>                                   
  </features>                                                                    
  <cpu mode='custom' match='exact' check='full'>
    <model fallback='forbid'>Skylake-Client</model>        
    <vendor>Intel</vendor>              
    <feature policy='disable' name='ds'/>
    <feature policy='disable' name='acpi'/>
    <feature policy='require' name='ss'/>                                        
    <feature policy='disable' name='ht'/>
    <feature policy='disable' name='tm'/>
    <feature policy='disable' name='pbe'/>
    <feature policy='disable' name='dtes64'/>
    <feature policy='disable' name='monitor'/>
    <feature policy='disable' name='ds_cpl'/>
    <feature policy='require' name='vmx'/>
    <feature policy='disable' name='smx'/>
    <feature policy='disable' name='est'/>
    <feature policy='disable' name='tm2'/>
    <feature policy='disable' name='xtpr'/>
    <feature policy='disable' name='pdcm'/>
    <feature policy='disable' name='osxsave'/>
    <feature policy='disable' name='tsc_adjust'/>
    <feature policy='require' name='clflushopt'/>
    <feature policy='require' name='pdpe1gb'/>
    <feature policy='require' name='hypervisor'/>
    <feature policy='disable' name='arat'/>
  </cpu>
  <clock offset='utc'>
    <timer name='rtc' tickpolicy='catchup'/>
    <timer name='pit' tickpolicy='delay'/>
    <timer name='hpet' present='no'/>
  </clock>
  <on_poweroff>destroy</on_poweroff>
  <on_reboot>restart</on_reboot>
  <on_crash>destroy</on_crash>
  <pm>
    <suspend-to-mem enabled='no'/>
    <suspend-to-disk enabled='no'/>
  </pm>
  <devices>
    <emulator>/usr/libexec/qemu-kvm</emulator>
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2'/>
      <source file='/var/lib/libvirt/images/computer.img'/>
      <backingStore/>
      <target dev='vda' bus='virtio'/>
      <alias name='virtio-disk0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x05' function='0x0'/>
    </disk>
    <controller type='usb' index='0' model='ich9-ehci1'>
      <alias name='usb'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x7'/>
    </controller>
    <controller type='usb' index='0' model='ich9-uhci1'>
      <alias name='usb'/>
      <master startport='0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x0' multifunction='on'/>
    </controller>
    <controller type='usb' index='0' model='ich9-uhci2'>
      <alias name='usb'/>
      <master startport='2'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x1'/>
    </controller>
    <controller type='usb' index='0' model='ich9-uhci3'>
      <alias name='usb'/>
      <master startport='4'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x2'/>
    </controller>
    <controller type='ide' index='0'>
      <alias name='ide'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x01' function='0x1'/>
    </controller>
    <controller type='pci' index='0' model='pci-root'>
      <alias name='pci.0'/>
    </controller>
    <interface type='bridge'>
      <mac address='52:54:00:e6:40:58'/>
      <source bridge='br0'/>
      <target dev='vnet3'/>
      <model type='virtio'/>
      <alias name='net0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
    </interface>
    <interface type='bridge'>
      <mac address='52:54:00:36:3c:b3'/>
      <source bridge='br-internal'/>
      <target dev='vnet4'/>
      <model type='virtio'/>
      <alias name='net1'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x07' function='0x0'/>
    </interface>
    <serial type='pty'>
      <source path='/dev/pts/2'/>
      <target type='isa-serial' port='0'>
        <model name='isa-serial'/>
      </target>
      <alias name='serial0'/>
    </serial>
    <console type='pty' tty='/dev/pts/2'>
      <source path='/dev/pts/2'/>
      <target type='serial' port='0'/>
      <alias name='serial0'/>
    </console>
    <input type='mouse' bus='ps2'>
      <alias name='input0'/>
    </input>
    <input type='keyboard' bus='ps2'>
      <alias name='input1'/>
    </input>
    <graphics type='vnc' port='5901' autoport='yes' listen='0.0.0.0'>
      <listen type='address' address='0.0.0.0'/>
    </graphics>
    <video>
      <model type='cirrus' vram='16384' heads='1' primary='yes'/>
      <alias name='video0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x0'/>
    </video>
    <memballoon model='virtio'>
      <alias name='balloon0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x06' function='0x0'/>
    </memballoon>
  </devices>
  <seclabel type='dynamic' model='dac' relabel='yes'>
    <label>+107:+107</label>
    <imagelabel>+107:+107</imagelabel>
  </seclabel>
</domain>
```

注意CPU中需要配置vmx以支持嵌套虚拟化。

```
# 启动虚拟机
virsh start controller
virsh start computer
```

### 控制节点部署

#### 1. 网络配置

ubuntu18.04 改由netplan来管理网络配置

但是重启后，netplan无法将未配置ip地址的网卡自动设置为up，详见：

https://bugs.launchpad.net/netplan/+bug/1763608

因此增加一个workaround的方法，配置网卡静态ipv6的地址

修改配置文件`/etc/netplan/50-cloud-init.yaml`

```yaml
network:
    ethernets:
        ens3:
            addresses:
            - 192.168.1.11/24
            dhcp4: false
            gateway4: 192.168.1.1
            nameservers:
                addresses:
                - 192.168.1.204
                search: []
        ens7:
            addresses:
            - fe80::10/128
        ens8:
            addresses:
            - fe80::11/128
    version: 2
```

```
# 使配置生效
netplan apply
```

#### 2. 安装devstack

```
# 创建stack用户
useradd -s /bin/bash -d /opt/stack -m stack
echo "stack ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/stack
su - stack
# 下载rocky版本源码
git clone https://git.openstack.org/openstack-dev/devstack -b stable/rocky
```

#### 3. 配置pip源

```
mkdir /root/.pip
cat << EOF > /root/.pip/pip.conf
[global]
index-url = https://pypi.douban.com/simple
download_cache = ~/.cache/pip
[install]
use-mirrors = true
mirrors = http://pypi.douban.com/
EOF

mkdir /opt/stack/.pip
cp /root/.pip/pip.conf /opt/stack/.pip/pip.conf
chown -R stack /opt/stack/.pip
```

#### 4. 配置local.conf

```
stack@controller:~/devstack$ cat local.conf |grep -v "^#" |grep -v "^$"
[[local|localrc]]
ADMIN_PASSWORD=admin
DATABASE_PASSWORD=123456
RABBIT_PASSWORD=123456
SERVICE_PASSWORD=123456
HOST_IP=192.168.1.11
LOGFILE=$DEST/logs/stack.sh.log
LOGDAYS=2
LOG_COLOR=True
MULTI_HOST=True
GIT_BASE=http://git.trystack.cn
NOVNC_REPO=http://git.trystack.cn/kanaka/noVNC.git
SPICE_REPO=http://git.trystack.cn/git/spice/spice-html5.git
disable_service n-net
enable_service q-svc,q-agt,q-dhcp,q-l3,q-meta,neutron,q-lbaas,q-fwaas
Q_AGENT=linuxbridge
ENABLE_TENANT_VLANS=True
TENANT_VLAN_RANGE=3001:4000
PHYSICAL_NETWORK=default

```

#### 5. 安装服务

```
./stack.sh
```



### 计算节点部署

#### 1. 网络配置

修改配置文件`/etc/netplan/50-cloud-init.yaml`

```yaml
network:
    ethernets:
        ens3:
            addresses:
            - 192.168.1.12/24
            dhcp4: false
            gateway4: 192.168.1.1
            nameservers:
                addresses:
                - 192.168.1.204
                search: []
        ens7:
            addresses:
            - fe80::12/128
    version: 2
```

```
# 使配置生效
netplan apply
```

#### 2. 安装devstack

```
# 创建stack用户
useradd -s /bin/bash -d /opt/stack -m stack
echo "stack ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/stack
su - stack
# 下载rocky版本源码
git clone https://git.openstack.org/openstack-dev/devstack -b stable/rocky
```

#### 3. 配置pip源

```
mkdir /root/.pip
cat << EOF > /root/.pip/pip.conf
[global]
index-url = https://pypi.douban.com/simple
download_cache = ~/.cache/pip
[install]
use-mirrors = true
mirrors = http://pypi.douban.com/
EOF

mkdir /opt/stack/.pip
cp /root/.pip/pip.conf /opt/stack/.pip/pip.conf
chown -R stack /opt/stack/.pip
```

#### 4. 配置loal.conf

```
stack@computer:~/devstack$ cat local.conf |grep -v "^#" |grep -v "^$"
[[local|localrc]]
ADMIN_PASSWORD=admin
DATABASE_PASSWORD=123456
RABBIT_PASSWORD=123456
SERVICE_PASSWORD=123456
HOST_IP=192.168.1.12
LOGFILE=$DEST/logs/stack.sh.log
LOGDAYS=2
LOG_COLOR=True
MULTI_HOST=True
SERVICE_HOST=192.168.1.11
MYSQL_HOST=$SERVICE_HOST
RABBIT_HOST=$SERVICE_HOST
GLANCE_HOSTPORT=$SERVICE_HOST:9292
Q_HOST=$SERVICE_HOST
KEYSTONE_AUTH_HOST=$SERVICE_HOST
KEYSTONE_SERVICE_HOST=$SERVICE_HOST
ENABLED_SERVICES=n-cpu,q-agt,neutron,placement-api,placement-client
Q_AGENT=linuxbridge
ENABLE_TENANT_VLANS=True
TENANT_VLAN_RANGE=3001:4000
PHYSICAL_NETWORK=default
NOVA_VNC_ENABLED=True
NOVNCPROXY_URL="http://$SERVICE_HOST:6080/vnc_auto.html"
VNCSERVER_LISTEN=$HOST_IP
VNCSERVER_PROXYCLIENT_ADDRESS=$VNCSERVER_LISTEN
GIT_BASE=http://git.trystack.cn
NOVNC_REPO=http://git.trystack.cn/kanaka/noVNC.git
SPICE_REPO=http://git.trystack.cn/git/spice/spice-html5.git
```

#### 4. 安装服务

```
./stack.sh
```



**注意：安装完毕后登录dashboard清理默认创建的网络，存储，镜像等资源。**

### 配置neutron服务

这里neutron二层采用linuxbridge + vlan 模式，linuxbridge相对openvswitch成熟简单，鉴于测试环境用vlan足以子网数量，同时避免了vxlan的性能损耗。

计算节点 & 控制节点

/etc/neutron/l3_agent.ini

```
# DEFAULT 段下interface_driver配置为linuxbridge
[DEFAULT]
interface_driver = linuxbridge
```

/etc/neutron/plugins/ml2/ml2_conf.ini

```
# ml2 段下配置
[ml2]
tenant_network_types = vlan
extension_drivers = port_security
mechanism_drivers = linuxbridge

# ml2_type_flat 设置外网
[ml2_type_flat]
flat_networks = external

# ml2_type_vlan
[ml2_type_vlan]
network_vlan_ranges = default:3001:4000

# 设置linux_bridge配置
[linux_bridge]
physical_interface_mappings = default:ens7,external:ens8

```

重启服务

```
systemctl restart devstack@q-*
```



### ceph集成（可选）

默认devstack安装的是lvm本地存储，考虑到实际生产环境中大多选用ceph作为统一存储，因此配置cinder volume-type为ceph。

#### 1. ceph配置

```
# 创建pool
ceph osd pool create volumes 32
ceph osd pool create vms 32
ceph osd pool create images 32
# 设置备份数
ceph osd pool set volumes size 1
ceph osd pool set vms size 1
ceph osd pool set images size 1
# 初始化pool
rbd pool init volumes
rbd pool init vms
rbd pool init images
```

复制/etc/ceph/ceph.conf 到控制节点

#### 2. cinder 配置

控制节点

/etc/cinder/cinder.conf

```
# DEFAULT 段
[DEFAULT]
default_volume_type = ceph
enabled_backends = ceph
# 新建ceph段
[ceph]
volume_driver = cinder.volume.drivers.rbd.RBDDriver
volume_backend_name = ceph
rbd_pool = volumes
rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_flatten_volume_from_snapshot = false
rbd_max_clone_depth = 5
rbd_store_chunk_size = 4
rados_connect_timeout = -1

```

#### 3. nova 配置

控制节点 & 计算节点

/etc/nova/nova-cpu.conf

```
# libvirt 段
[libvirt]
live_migration_uri = qemu+tcp://root@%s/system
cpu_mode = none
virt_type = kvm
images_rbd_pool=vms
images_type=rbd
images_rbd_ceph_conf=/etc/ceph/ceph.conf
disk_cachemodes="network=writeback"
inject_password = false
inject_key = false
inject_partition = -2
live_migration_flag="VIR_MIGRATE_UNDEFINE_SOURCE,VIR_MIGRATE_PEER2PEER,VIR_MIGRATE_LIVE,VIR_MIGRATE_PERSIST_DEST,VIR_MIGRATE_TUNNELLED"

```

#### 4. glance 配置

控制节点

/etc/glance/glance-api.conf

```
# glance_store 段 配置后端rbd
[glance_store]
#filesystem_store_datadir = /opt/stack/data/glance/images/
stores = rbd
default_store = rbd
rbd_store_pool = images
#rbd_store_user = glance
rbd_store_ceph_conf = /etc/ceph/ceph.conf
rbd_store_chunk_size = 8
# 去除glance缓存
[paste_deploy]
#flavor = keystone+cachemanagement
flavor = keystone
# default 下增加显示多路径
[DEFAULT]
show_multiple_locations = True
show_image_direct_url = True
```

glance镜像需要重传，为了发挥ceph存储cow优势，秒级创建虚拟机，需要将镜像格式转换为raw格式。

```
qemu-img convert -f {source-format} -O {output-format} {source-filename} {output-filename}
qemu-img convert -f qcow2 -O raw cirros-0.4.0-x86_64-disk.img cirros-0.4.0-x86_64-disk.raw
```


### 安装过程中遇到的问题

#### 1. 删除虚拟机导致宿主机hang住

假设实际的物理PC为layer1 ，物理PC中创建的虚拟机为layer2 ，layer2虚拟机中再建的虚拟机为layer3。

表现为layer2删除layer3虚拟机时将会导致layer2虚拟机hang住，只能在layer1强制destroy layer2虚拟机再开机。

查看layer3虚拟机日志

```
2018-11-16T08:51:10.701589Z qemu-system-x86_64: warning: host doesn't support requested feature: CPUID.80000001H:ECX.svm [bit 2]
```

猜测是CPU不支持该特性，于是升级内核版本

升级内核前：

```
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc art arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc aperfmperf eagerfpu pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 fma cx16 xtpr pdcm pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch intel_pt tpr_shadow vnmi flexpriority ept vpid fsgsbase tsc_adjust bmi1 hle avx2 smep bmi2 erms invpcid rtm mpx rdseed adx smap clflushopt xsaveopt xsavec xgetbv1 dtherm ida arat pln pts hwp hwp_notify hwp_act_window hwp_epp
```

升级内核后：

```
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc art arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc cpuid aperfmperf tsc_known_freq pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single pti tpr_shadow vnmi flexpriority ept vpid ept_ad fsgsbase tsc_adjust bmi1 hle avx2 smep bmi2 erms invpcid rtm mpx rdseed adx smap clflushopt intel_pt xsaveopt xsavec xgetbv1 xsaves dtherm ida arat pln pts hwp hwp_notify hwp_act_window hwp_epp
```

显然多了很多新的flags，例如 `cpuid_fault`

#### 2. 虚拟机无法调度到计算节点 

新增加的计算节点没有加入到nova cell中，需要执行以下命令

```
nova-manage cell_v2 discover_hosts
```

#### 3. 虚拟机热迁移失败

##### 3.1 主机域名解析

nova-cpu.conf中定义了

```
live_migration_uri = qemu+tcp://root@%s/system
```

通过域名进行通信，需要各个节点配置DNS server

##### 3.2 libvirt 开启tcp端口

/etc/default/libvirtd

```
start_libvirtd="yes"
libvirtd_opts="-d -l"
```

/etc/libvirt/libvirtd.conf

```
listen_tls = 0
listen_tcp = 1
unix_sock_group = "libvirt"
unix_sock_ro_perms = "0777"
unix_sock_rw_perms = "0770"
auth_unix_ro = "none"
auth_unix_rw = "none"
auth_tcp = "none"
```



