---
layout: post
title: "kickstart网络引导批量部署服务器"
date:  2019-02-13 14:37:21
categories: Kickstart
---

## kickstart PXE部署CentOS7-Minimal

### 1. 配置dnsmasq

```
yum install dnsmasq
```

/etc/dnsmasq.conf

```
domain=jfbrother.centos7.lan
# DNS
server=114.114.114.114
#bind-interfaces
interface=eth0
# DHCP range-leases
dhcp-range=192.168.1.200,192.168.1.250,255.255.255.0,1h
# Gateway
dhcp-option=3,192.168.1.1
# PXE
dhcp-boot=pxelinux.0,pxeserver,192.168.1.177
pxe-prompt="Press F8 for menu.", 60
pxe-service=x86PC, "Install CentOS 7 from network server 192.168.1.177", pxelinux
# tftp
enable-tftp
tftp-root=/var/lib/tftpboot
conf-dir=/etc/dnsmasq.d,.rpmnew,.rpmsave,.rpmorig
```

参数说明:

```
interface – 服务器需要监听并提供服务的网络接口。
bind-interfaces – 取消注释来绑定到该网络接口
domain – 替换为你的域名。
dhcp-range – 替换为你的网络掩码定义的网段。
dhcp-boot – 替换该IP地址为你的网络接口IP地址。
dhcp-option=3,192.168.1.1 – 替换该IP地址为你的网段的网关。
dhcp-option=6,92.168.1.1 – 替换该IP地址为你的DNS服务器IP——可以定义多个IP地址。
server=8.8.4.4 – 这里放置DNS转发服务器IP地址。
dhcp-option=28,10.0.0.255 – 替换该IP地址为网络广播地址——可选项。
dhcp-option=42,0.0.0.0 – 这里放置网络时钟服务器——可选项（0.0.0.0地址表示参考自身）。
pxe-prompt – 保持默认——按F8进入菜单，60秒等待时间。
pxe=service – 使用x86PC作为32为/64位架构，并在字符串引述中输入菜单描述提示。其它类型值可以是：PC98,IAEFI,Alpha,Arcx86,IntelLeanClient,IA32EFI,BCEFI,XscaleEFI和X86-64EFI。
enable-tftp – 启用内建TFTP服务器。
tftp-root – 使用/var/lib/tftpboot——所有网络启动文件所在位置。
```

### 2. 安装syslinux启动加载器

```
yum install syslinux
yum install tftp-server
cp -r /usr/share/syslinux/* /var/lib/tftpboot/
mkdir /var/lib/tftpboot/pxelinux.cfg
touch /var/lib/tftpboot/pxelinux.cfg/default
```

/var/lib/tftpboot/pxelinux.cfg/default

```
default kickstart
prompt 0
timeout 300

menu title ########## PXE Boot Menu ##########

label kickstart 
menu label ^0) Install CentOS 7 x64 with kickstart
kernel centos7/vmlinuz
append initrd=centos7/initrd.img ks=ftp://192.168.1.177/pub/centos7-ks.cfg

label 1
menu label ^1) Install CentOS 7 x64 with ftp Repo
kernel centos7/vmlinuz
append initrd=centos7/initrd.img method=ftp://192.168.1.177/pub devfs=nomount

label 3
menu label ^3) Install CentOS 7 x64 with ftp Repo using VNC
kernel centos7/vmlinuz
append initrd=centos7/initrd.img method=ftp://192.168.1.177/pub devfs=nomount inst.vnc inst.vncpassword=123456

label 4
menu label ^4) Boot from local drive
```

### 3. 添加镜像到PXE服务器

```
mount -o loop CentOS-7-x86_64-Minimal-1708.iso /mnt/
mkdir /var/lib/tftpboot/centos7
cp /mnt/images/pxeboot/vmlinuz  /var/lib/tftpboot/centos7
cp /mnt/images/pxeboot/initrd.img  /var/lib/tftpboot/centos7
```

### 4. 创建ftp服务器作为安装源

```
yum install vsftpd
cp -r /mnt/*  /var/ftp/pub/ 
chmod -R 755 /var/ftp/pub
```

### 5. 编写kickstart文件

/var/ftp/pub/centos7-ks.cfg

```
on=DEVEL
# System authorization information
auth --enableshadow --passalgo=sha512
# Use network installation
url --url="ftp://192.168.1.177/pub"
# Use graphical install
text
# Run the Setup Agent on first boot
firstboot --enable
ignoredisk --only-use=sda
# Keyboard layouts
keyboard --vckeymap=cn --xlayouts='cn'
# System language
lang zh_CN.UTF-8

# Network information
network  --bootproto=dhcp --device=em1 --ipv6=auto --activate
network  --bootproto=dhcp --device=em2 --onboot=off --ipv6=auto
network  --bootproto=dhcp --device=em3 --onboot=off --ipv6=auto
network  --bootproto=dhcp --device=em4 --onboot=off --ipv6=auto
network  --hostname=localhost.localdomain

# Root password
rootpw --iscrypted $6$hDKYCwoiqaqnL15O$vEmkVdcmbnVt6bbLlhFIkIfUq3LDDuiRXEwR.DoCAqzaZslP8Pl3I3YHG7nyX8fF/eNZ/zzMtUf.kiVNRo3MJ1
# System services
services --enabled="chronyd"
# System timezone
timezone Asia/Shanghai
# System bootloader configuration
bootloader --append=" crashkernel=auto" --location=mbr --boot-drive=sda
# Partition clearing information
clearpart --all --initlabel --drives=sda
# Disk partitioning information
part /boot --fstype="ext4" --ondisk=sda --size=1024
part pv.1012 --fstype="lvmpv" --ondisk=sda --size=1 --grow
volgroup centos --pesize=4096 pv.1012
logvol swap  --fstype="swap" --size=4096 --name=swap --vgname=centos
logvol /  --fstype="ext4" --grow --size=1 --name=root --vgname=centos --grow

%packages
@^minimal
@core
chrony
kexec-tools

%end

%addon com_redhat_kdump --enable --reserve-mb='auto'

%end

%anaconda
pwpolicy root --minlen=6 --minquality=1 --notstrict --nochanges --notempty
pwpolicy user --minlen=6 --minquality=1 --notstrict --nochanges --emptyok
pwpolicy luks --minlen=6 --minquality=1 --notstrict --nochanges --notempty
%end
```

磁盘分区

`boot`分区 1G, `swap`分区 4G, 剩余空间都划分给`/`分区


### 6. 启动服务

```
systemctl start dnsmasq
systemctl start vsftpd
firewall-cmd --add-service=ftp --permanent
firewall-cmd --add-service=dns --permanent
firewall-cmd --add-service=dhcp --permanent
## Port for TFTP
firewall-cmd --add-port=69/udp --permanent
## Port for ProxyDHCP
firewall-cmd --add-port=4011/udp --permanent
firewall-cmd --reload
```


## 参考链接:

[安装PXE引导](https://linux.cn/article-4902-1.html#3_1878)

[使用Kickstart实现CentOS自动化安装](http://debugo.com/kickstart-install-centos/)


