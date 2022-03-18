---
layout: post
title: "Tanzu安装测试"
date: 2022-03-17 09:21:22
categories: Tanzu Kubernetes
---

![log](/images/tanzu.png)

本文记录tanzu community版本在vSphere7环境的安装部署以及功能测试。

## 环境信息

因为资源受限，本文将在一台物理服务器上部署所有TKG(Tanzu Kubernetes Grid)环境。
cpu: 8核
内存: 40G
硬盘: 500G
网卡: 千兆网卡enp0s31f6
系统: Ubuntu 20.04 LTS

通过kvm创建ESXi虚拟机，注意: 需要开启嵌套虚拟化支持

## 安装ESXi

kvm定义ESXi专用网络br0，IP地址规划为192.168.100.0/24

```xml
<network>
  <name>br0</name>
  <uuid>0a4a04ba-81ba-412c-b55a-404673c17728</uuid>
  <forward mode='nat'>
    <nat>
      <port start='1024' end='65535'/>
    </nat>
  </forward>
  <bridge name='virbr1' stp='on' delay='0'/>
  <mac address='52:54:00:64:bd:87'/>
  <domain name='br0'/>
  <ip address='192.168.100.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.100.128' end='192.168.100.254'/>
    </dhcp>
  </ip>
</network>
```

创建出来的虚拟网桥为virbr1

```
zhengtianbao@thinkpad:~/esxi$ brctl show
bridge name	bridge id		STP enabled	interfaces	
docker0		8000.02428ab3c6fa	no		
virbr0		8000.52540086bf1f	yes		virbr0-nic

```

定义ESXi虚拟机配置

```xml
<domain type='kvm' id='2'>
  <name>esxi</name>
  <uuid>e79a4518-d020-4c61-a1ac-6a7eeaf1f531</uuid>
  <memory unit='KiB'>37748736</memory>
  <currentMemory unit='KiB'>37748736</currentMemory>
  <vcpu placement='static'>8</vcpu>
  <resource>
    <partition>/machine</partition>
  </resource>
  <os>
    <type arch='x86_64' machine='pc-i440fx-focal'>hvm</type>
    <boot dev='hd'/>
  </os>
  <features>
    <acpi/>
    <apic/>
    <vmport state='off'/>
  </features>
  <cpu mode='host-passthrough' check='none'/>
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
    <emulator>/usr/bin/qemu-system-x86_64</emulator>
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2'/>
      <source file='/var/lib/libvirt/images/esxi.qcow2' index='3'/>
      <backingStore/>
      <target dev='sda' bus='sata'/>
      <alias name='sata0-0-0'/>
      <address type='drive' controller='0' bus='0' target='0' unit='0'/>
    </disk>
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2'/>
      <source file='/var/lib/libvirt/images/esxi-1.qcow2' index='2'/>
      <backingStore/>
      <target dev='sdb' bus='sata'/>
      <alias name='sata0-0-1'/>
      <address type='drive' controller='0' bus='0' target='0' unit='1'/>
    </disk>
    <disk type='file' device='cdrom'>
      <driver name='qemu'/>
      <target dev='hdb' bus='ide'/>
      <readonly/>
      <alias name='ide0-0-1'/>
      <address type='drive' controller='0' bus='0' target='0' unit='1'/>
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
    <controller type='pci' index='0' model='pci-root'>
      <alias name='pci.0'/>
    </controller>
    <controller type='ide' index='0'>
      <alias name='ide'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x01' function='0x1'/>
    </controller>
    <controller type='sata' index='0'>
      <alias name='sata0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x05' function='0x0'/>
    </controller>
    <interface type='network'>
      <mac address='52:54:00:f6:9e:39'/>
      <source network='br0' portid='73972b34-3739-4f99-9b93-7780c6e3c735' bridge='virbr1'/>
      <target dev='vnet0'/>
      <model type='e1000e'/>
      <alias name='net0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
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
    <graphics type='vnc' port='5900' autoport='yes' listen='127.0.0.1'>
      <listen type='address' address='127.0.0.1'/>
    </graphics>
    <video>
      <model type='qxl' ram='65536' vram='65536' vgamem='16384' heads='1' primary='yes'/>
      <alias name='video0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x0'/>
    </video>
    <redirdev bus='usb' type='spicevmc'>
      <alias name='redir0'/>
      <address type='usb' bus='0' port='1'/>
    </redirdev>
    <redirdev bus='usb' type='spicevmc'>
      <alias name='redir1'/>
      <address type='usb' bus='0' port='2'/>
    </redirdev>
    <memballoon model='virtio'>
      <alias name='balloon0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x06' function='0x0'/>
    </memballoon>
  </devices>
  <seclabel type='dynamic' model='apparmor' relabel='yes'>
    <label>libvirt-e79a4518-d020-4c61-a1ac-6a7eeaf1f531</label>
    <imagelabel>libvirt-e79a4518-d020-4c61-a1ac-6a7eeaf1f531</imagelabel>
  </seclabel>
  <seclabel type='dynamic' model='dac' relabel='yes'>
    <label>+64055:+108</label>
    <imagelabel>+64055:+108</imagelabel>
  </seclabel>
</domain>
```

这里给ESXi配了两块qcow格式的100G的盘

注意: CPU mode需要配置为`host-passthrough`，网卡 model 需要配置为`e1000e`

等待ESXi虚拟机启动，按提示安装完毕后，上传[tinycore](http://tinycorelinux.net/13.x/x86/release/TinyCore-current.iso) 镜像创建虚拟机，测试网络联通性没问题即可进行下一步


## 安装vCenter

本文vCenter部署在ESXi上，按提示部署， 需要注意的是在stage1中配置完毕后需要通过SSH连到vCenter虚拟机中加一条域名解析, 否则会安装进度条会卡住

/etc/hosts

```
$vCenter_IP localhost 
```

完成安装之后，建议先对当前虚拟机打一个快照，这样如果第二阶段由于配置问题安装失败，可以重新尝试。


## 安装Tanzu

### 下载tanzu CLI

从 https://github.com/vmware-tanzu/community-edition/releases 下载合适版本

解压后执行 `install.sh`

### 安装management Cluster

```
zhengtianbao@thinkpad:~$ tanzu management-cluster create --ui

Validating the pre-requisites...
Serving kickstart UI at http://127.0.0.1:8080
Identity Provider not configured. Some authentication features won't work.
WARNING: key is not a string: %!s(int=1)WARNING: key is not a string: %!s(int=1)Validating configuration...
web socket connection established
sending pending 2 logs to UI
Using infrastructure provider vsphere:v0.7.10
Generating cluster configuration...
WARNING: key is not a string: %!s(int=1)WARNING: key is not a string: %!s(int=1)Setting up bootstrapper...
Bootstrapper created. Kubeconfig: /home/zhengtianbao/.kube-tkg/tmp/config_UHwCO6e7
Installing providers on bootstrapper...
Fetching providers
Installing cert-manager Version="v1.1.0"
Waiting for cert-manager to be available...
Installing Provider="cluster-api" Version="v0.3.23" TargetNamespace="capi-system"
Installing Provider="bootstrap-kubeadm" Version="v0.3.23" TargetNamespace="capi-kubeadm-bootstrap-system"
Installing Provider="control-plane-kubeadm" Version="v0.3.23" TargetNamespace="capi-kubeadm-control-plane-system"
Installing Provider="infrastructure-vsphere" Version="v0.7.10" TargetNamespace="capv-system"
Start creating management cluster...
cluster control plane is still being initialized: WaitingForControlPlane
cluster control plane is still being initialized: ScalingUp
cluster control plane is still being initialized: WaitingForKubeadmInit
Saving management cluster kubeconfig into /home/zhengtianbao/.kube/config
Installing providers on management cluster...
Fetching providers
Installing cert-manager Version="v1.1.0"
Waiting for cert-manager to be available...
Installing Provider="cluster-api" Version="v0.3.23" TargetNamespace="capi-system"
Installing Provider="bootstrap-kubeadm" Version="v0.3.23" TargetNamespace="capi-kubeadm-bootstrap-system"
Installing Provider="control-plane-kubeadm" Version="v0.3.23" TargetNamespace="capi-kubeadm-control-plane-system"
Installing Provider="infrastructure-vsphere" Version="v0.7.10" TargetNamespace="capv-system"
Waiting for the management cluster to get ready for move...
Waiting for addons installation...
Moving all Cluster API objects from bootstrap cluster to management cluster...
Performing move...
Discovering Cluster API objects
Moving Cluster API objects Clusters=1
Creating objects in the target cluster
Deleting objects from the source cluster
Waiting for additional components to be up and running...
Waiting for packages to be up and running...
You can now access the management cluster mgmt by running 'kubectl config use-context mgmt-admin@mgmt'

Management cluster created!


You can now create your first workload cluster by running the following:

  tanzu cluster create [name] -f [file]
```

查看管理集群信息

```
zhengtianbao@thinkpad:~$ tanzu management-cluster get
  NAME  NAMESPACE   STATUS   CONTROLPLANE  WORKERS  KUBERNETES        ROLES       
  mgmt  tkg-system  running  1/1           1/1      v1.21.5+vmware.1  management  


Details:

NAME                                                     READY  SEVERITY  REASON  SINCE  MESSAGE
/mgmt                                                    True                     5m54s         
├─ClusterInfrastructure - VSphereCluster/mgmt            True                     5m58s         
├─ControlPlane - KubeadmControlPlane/mgmt-control-plane  True                     5m54s         
│ └─Machine/mgmt-control-plane-vz847                     True                     5m56s         
└─Workers                                                                                       
  └─MachineDeployment/mgmt-md-0                                                                 
    └─Machine/mgmt-md-0-cd469cfc6-nktfb                  True                     5m57s         


Providers:

  NAMESPACE                          NAME                    TYPE                    PROVIDERNAME  VERSION  WATCHNAMESPACE  
  capi-kubeadm-bootstrap-system      bootstrap-kubeadm       BootstrapProvider       kubeadm       v0.3.23                  
  capi-kubeadm-control-plane-system  control-plane-kubeadm   ControlPlaneProvider    kubeadm       v0.3.23                  
  capi-system                        cluster-api             CoreProvider            cluster-api   v0.3.23                  
  capv-system                        infrastructure-vsphere  InfrastructureProvider  vsphere       v0.7.10   

```

配置kubeconfig使用管理集群

```
zhengtianbao@thinkpad:~$ tanzu management-cluster kubeconfig get mgmt --admin
Credentials of cluster 'mgmt' have been saved 
You can now access the cluster by running 'kubectl config use-context mgmt-admin@mgmt'
zhengtianbao@thinkpad:~$ kubectl config use-context mgmt-admin@mgmt
Switched to context "mgmt-admin@mgmt".
zhengtianbao@thinkpad:~$ kubectl get nodes
NAME                        STATUS   ROLES                  AGE   VERSION
mgmt-control-plane-vz847    Ready    control-plane,master   17m   v1.21.5+vmware.1
mgmt-md-0-cd469cfc6-nktfb   Ready    <none>                 16m   v1.21.5+vmware.1
```


### 安装workload集群

利用managerment集群的配置文件创建workload集群
```
zhengtianbao@thinkpad:~$ cp ./.config/tanzu/tkg/clusterconfigs/cjxfqluc5t.yaml ~/.config/tanzu/tkg/clusterconfigs/workload1.yaml
zhengtianbao@thinkpad:~$ tanzu cluster list
  NAME  NAMESPACE  STATUS  CONTROLPLANE  WORKERS  KUBERNETES  ROLES  PLAN  
```

需要修改workload集群的配置, 参考： <https://tanzucommunityedition.io/docs/latest/aws-wl-template/>

```
zhengtianbao@thinkpad:~$ vim ~/.config/tanzu/tkg/clusterconfigs/workload1.yaml 
```

```
CLUSTER_CIDR: 100.96.0.0/11
CLUSTER_NAME: my-workload-cluster
CLUSTER_PLAN: dev
```

创建workload集群
```
zhengtianbao@thinkpad:~$ tanzu cluster create k8s-dev --file ~/.config/tanzu/tkg/clusterconfigs/workload1.yaml 
Validating configuration...
Warning: Pinniped configuration not found. Skipping pinniped configuration in workload cluster. Please refer to the documentation to check if you can configure pinniped on workload cluster manually
Creating workload cluster 'k8s-dev'...
Waiting for cluster to be initialized...
cluster control plane is still being initialized: WaitingForControlPlane
cluster control plane is still being initialized: ScalingUp
Waiting for cluster nodes to be available...
Waiting for addons installation...
Waiting for packages to be up and running...

Workload cluster 'k8s-dev' created

zhengtianbao@thinkpad:~$ tanzu cluster list
  NAME     NAMESPACE  STATUS   CONTROLPLANE  WORKERS  KUBERNETES        ROLES   PLAN  
  k8s-dev  default    running  1/1           1/1      v1.21.5+vmware.1  <none>  dev   
```

### 管理workload集群

```
zhengtianbao@thinkpad:~$ tanzu cluster kubeconfig get k8s-dev --admin
Credentials of cluster 'k8s-dev' have been saved 
You can now access the cluster by running 'kubectl config use-context k8s-dev-admin@k8s-dev'
zhengtianbao@thinkpad:~$ kubectl config use-context k8s-dev-admin@k8s-dev
Switched to context "k8s-dev-admin@k8s-dev".
zhengtianbao@thinkpad:~$ kubectl get pods --all-namespaces
NAMESPACE      NAME                                                    READY   STATUS    RESTARTS   AGE
kube-system    antrea-agent-kqdll                                      2/2     Running   0          8m45s
kube-system    antrea-agent-z4tr8                                      2/2     Running   0          8m45s
kube-system    antrea-controller-59dcfbf9d7-fb5c8                      1/1     Running   0          8m45s
kube-system    coredns-657879bf57-6jhrt                                1/1     Running   0          12m
kube-system    coredns-657879bf57-rbd49                                1/1     Running   0          12m
kube-system    etcd-k8s-dev-control-plane-dsctr                        1/1     Running   0          11m
kube-system    kube-apiserver-k8s-dev-control-plane-dsctr              1/1     Running   0          11m
kube-system    kube-controller-manager-k8s-dev-control-plane-dsctr     1/1     Running   0          12m
kube-system    kube-proxy-mh9fn                                        1/1     Running   0          10m
kube-system    kube-proxy-nk29g                                        1/1     Running   0          12m
kube-system    kube-scheduler-k8s-dev-control-plane-dsctr              1/1     Running   0          11m
kube-system    kube-vip-k8s-dev-control-plane-dsctr                    1/1     Running   0          11m
kube-system    metrics-server-5f6c96db75-fpvc7                         1/1     Running   0          8m56s
kube-system    vsphere-cloud-controller-manager-8xrz7                  1/1     Running   0          8m53s
kube-system    vsphere-csi-controller-5f4f98d64c-2dw5n                 6/6     Running   0          7m46s
kube-system    vsphere-csi-node-c7lrc                                  3/3     Running   0          7m46s
kube-system    vsphere-csi-node-t5qt9                                  3/3     Running   0          7m46s
tanzu-system   secretgen-controller-658fc49779-g6ptq                   1/1     Running   0          8m53s
tkg-system     kapp-controller-85fdbc6b5b-29f7k                        1/1     Running   0          11m
tkg-system     tanzu-capabilities-controller-manager-7959d6b44-kxdr6   1/1     Running   0          11m

```

### 安装multus-cni网络插件

配置repository

```
zhengtianbao@thinkpad:~$ tanzu package repository list
/ Retrieving repositories... 
  NAME  REPOSITORY  TAG  STATUS  DETAILS  
zhengtianbao@thinkpad:~$ tanzu package repository add tce-repo --url projects.registry.vmware.com/tce/main:0.10.0 --namespace tanzu-package-repo-global
/ Adding package repository 'tce-repo' 


- Waiting for 'PackageRepository' reconciliation for 'tce-repo' 
- 'PackageRepository' resource install status: Reconciling 

Added package repository 'tce-repo' in namespace 'tanzu-package-repo-global'
zhengtianbao@thinkpad:~$ tanzu package repository list --namespace tanzu-package-repo-global
/ Retrieving repositories... 
  NAME      REPOSITORY                             TAG     STATUS               DETAILS  
  tce-repo  projects.registry.vmware.com/tce/main  0.10.0  Reconcile succeeded           
zhengtianbao@thinkpad:~$ tanzu package available list
- Retrieving available packages... 
  NAME                                           DISPLAY-NAME        SHORT-DESCRIPTION                                                                                                             LATEST-VERSION  
  cert-manager.community.tanzu.vmware.com        cert-manager        Certificate management                                                                                                        1.6.1           
  contour.community.tanzu.vmware.com             contour             An ingress controller                                                                                                         1.19.1          
  external-dns.community.tanzu.vmware.com        external-dns        This package provides DNS synchronization functionality.                                                                      0.10.0          
  fluent-bit.community.tanzu.vmware.com          fluent-bit          Fluent Bit is a fast Log Processor and Forwarder                                                                              1.7.5           
  gatekeeper.community.tanzu.vmware.com          gatekeeper          policy management                                                                                                             3.7.0           
  grafana.community.tanzu.vmware.com             grafana             Visualization and analytics software                                                                                          7.5.7           
  harbor.community.tanzu.vmware.com              harbor              OCI Registry                                                                                                                  2.3.3           
  knative-serving.community.tanzu.vmware.com     knative-serving     Knative Serving builds on Kubernetes to support deploying and serving of applications and functions as serverless containers  1.0.0           
  local-path-storage.community.tanzu.vmware.com  local-path-storage  This package provides local path node storage and primarily supports RWO AccessMode.                                          0.0.20          
  multus-cni.community.tanzu.vmware.com          multus-cni          This package provides the ability for enabling attaching multiple network interfaces to pods in Kubernetes                    3.7.1           
  prometheus.community.tanzu.vmware.com          prometheus          A time series database for your metrics                                                                                       2.27.0          
  velero.community.tanzu.vmware.com              velero              Disaster recovery capabilities                                                                                                1.6.3           
  whereabouts.community.tanzu.vmware.com         whereabouts         A CNI IPAM plugin that assigns IP addresses cluster-wide                                                                      0.5.0           
zhengtianbao@thinkpad:~$ tanzu package available list multus-cni.community.tanzu.vmware.com
/ Retrieving package versions for multus-cni.community.tanzu.vmware.com... 
  NAME                                   VERSION  RELEASED-AT                    
  multus-cni.community.tanzu.vmware.com  3.7.1    2021-06-05 02:00:00 +0800 CST  
```

安装multus-cni

```
zhengtianbao@thinkpad:~$ tanzu package install multus-cni --package-name multus-cni.community.tanzu.vmware.com --version 3.7.1 
/ Installing package 'multus-cni.community.tanzu.vmware.com' 
| Getting package metadata for 'multus-cni.community.tanzu.vmware.com' 
| Creating service account 'multus-cni-default-sa' 
| Creating cluster admin role 'multus-cni-default-cluster-role' 
| Creating cluster role binding 'multus-cni-default-cluster-rolebinding' 
| Creating package resource 
/ Waiting for 'PackageInstall' reconciliation for 'multus-cni' 
/ 'PackageInstall' resource install status: Reconciling 


 Added installed package 'multus-cni'
zhengtianbao@thinkpad:~$ tanzu package installed list
/ Retrieving installed packages... 
  NAME        PACKAGE-NAME                           PACKAGE-VERSION  STATUS               
  multus-cni  multus-cni.community.tanzu.vmware.com  3.7.1            Reconcile succeeded 
```


## 测试

创建multus CRD


```
zhengtianbao@thinkpad:~$ cat multus-cni-crd.yaml 
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan-conf
spec:
  config: '{
      "cniVersion": "0.3.0",
      "type": "macvlan",
      "master": "eth0",
      "mode": "bridge",
      "ipam": {
        "type": "host-local",
        "subnet": "192.168.100.0/24",
        "rangeStart": "192.168.100.20",
        "rangeEnd": "192.168.100.30",
        "routes": [
          { "dst": "0.0.0.0/0" }
        ],
        "gateway": "192.168.100.1"
      }
    }'

```
注意: 这里的eth0是workload集群node节点上的eth0网卡

创建测试容器

```
zhengtianbao@thinkpad:~$ cat my-multi-cni-pod.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: sample-pod
  annotations:
    #k8s.v1.cni.cncf.io/networks: macvlan-conf
    k8s.v1.cni.cncf.io/networks: '[{
      "name": "macvlan-conf",
      "default-route": ["192.168.100.1"]
    }]'
spec:
  containers:
  - name: sample-pod
    command: ["/bin/sh", "-c", "trap : TERM INT; sleep infinity & wait"]
    image: docker.io/library/busybox
    imagePullPolicy: IfNotPresent

```
annotations中的 macvlan-conf 为上面NetworkAttachmentDefinition定义的名字
更详细的用法参见： <https://github.com/k8snetworkplumbingwg/multus-cni/blob/master/docs/quickstart.md>

注意: 因为是macvlan的方式，所以需要配置vswitch的网络为混杂模式。另外，pod无法与所在主机IP通信，因为macvlan的父设备只负责接收包，而不会处理包。




## 参考链接

esxi部署: <https://itsjustbytes.wordpress.com/2020/12/14/kvm-esxi-7-as-a-nested-cluster>

vCenter部署: <https://itsjustbytes.wordpress.com/2021/01/01/install-vcsa-in-nested-vsphere-hosts>

vCenter部署踩坑: <https://segmentfault.com/a/1190000039928498>

tanzu部署: <https://tanzucommunityedition.io/docs/latest/getting-started/>

