---
layout: post
title:  "ceph calamari install"
date:   2015-07-10 20:46:53
categories: Ceph Calamari
---

本文主要记录了安装 ceph calamari 过程中碰到的问题以及解决方案。

## 节点服务分配

| node        | ip          | server                                  |
| ----------- | ----------- | --------------------------------------- |
| ceph-1      | 10.160.0.41 | osd.0 mon.0 Salt-minion Diamond         |
| ceph-2      | 10.160.0.42 | osd.1 mon.1 Salt-minion Diamond         |
| ceph-3      | 10.160.0.43 | osd.2 mon.2 Salt-minion Diamond         |
|             |             | Salt-master Calamari-server Apache      |
|             |             | Graphite(carbon，whisper，grahpite-web) |
|             |             | Calamari-client                         |
| cns-5       | 10.160.0.55 | osd.3 Salt-minion Diamond               |

节点说明：

- 共 4 个 osd 节点，3 个 mon 节点，所有 osd，mon 节点都安装 Salt-minion，Diamond
- Salt-master 远程调用目标 minion 执行 ceph 命令，如：ceph osd dump
- Diamond 收集 ceph 集群以及节点磁盘，网络等信息发送给 Graphite
- Graphite 由三个组件构成，carbon 接受 Diamond 发送的数据，whisper 类似数据库持久化数据，graphite-web 借助 django，apache 等提供web服务
- Calamari-server 调用 salt 获取 ceph 集群信息，对外暴露 REST API
- Calamari-client 是 nodejs 写的一个前端页面，调用 Calamari-server 以及Graphite 的 API 获取数据展示图表

## 安装

### 1. 在所有 ceph 节点安装 salt-minion

```
# yum install salt-minion
```

注意：如果有错误后面提示 cthulhu 服务起不来参考：<https://github.com/ceph/calamari/issues/285>

可能是 salt 版本过高导致，需要降低版本到 2014.7.5

```
# wget ftp://195.220.108.108/linux/fedora/linux/releases/22/Everything/x86_64/os/Packages/s/salt-2014.7.5-1.fc22.noarch.rpm
# rpm -Uvh --oldpackage salt-minion-2014.7.5-1.fc22.noarch.rpm 
# salt-minion --version
salt-minion 2014.7.5 (Helium)
```

编辑 /etc/salt/minion.d/calamari.conf

```
master: {fqdn}
```

{fqdn} 对应 Salt-master 的域名，测试环境是 10.160.0.43

重启服务：

```
# service salt-minion restart
```

### 2. 在所有 ceph 节点安装 Diamond

```
# git clone https://github.com/ceph/Diamond
# cd Diamond
# git checkout origin/calamari
# make rpm
# yum localinstall diamond-3.4.67-0.noarch.rpm
```

编辑 /etc/diamond/diamond.conf，设置 hostname_method = hostname_short

重启服务：

```
# /etc/init.d/diamond restart
```

### 3. 安装 Calamari-server

```
# yum install postgresql-server salt-master salt-minion supervisor
# git clone https://github.com/ceph/calamari.git
# cd calamari && ./build-rpm.sh
# yum localinstall ../rpmbuild/RPMS/x86_64/calamari-server-<version>.rpm
# calamari-ctl initialize
[INFO] Loading configuration..
[INFO] Starting/enabling salt...
[INFO] Starting/enabling postgres...
[INFO] Updating database...
[INFO] Initializing web interface...
[INFO] Starting/enabling services...
[INFO] Restarting services...
[INFO] Complete.
```

配置 saltstack 认证：

```
# salt-key -L
# salt-key -A
# salt '*' test.ping
ceph-1:
    True
ceph-2:
    True
ceph-3:
    True
cns-5:
    True
#salt '*' ceph.get_heartbeats
```

查看启动的服务：

```
# supervisorctl status
carbon-cache                     RUNNING    pid 7745，uptime 2:03:54
cthulhu                          RUNNING    pid 14432，uptime 1:05:01
```

至此，Calamari-server 安装完毕，调用API：

<http://ceph.com/calamari/docs/calamari_rest/resources/resources.html>

运行 calamari/tests/http_client.py 验证：

```
python http_client.py -u http://10.160.0.43/api/v2/ --user root --pass 123456
```

注意：调用 api/v2/cluster/{fsid}/cli 如果失败返回 500 错误，可能是 salt 的配置问题

因为 Calamari-server 是以 apache 身份运行的，没有权限，需要修改 /etc/salt/master.d/calamari.conf

```
file_roots:
  base:
      - /opt/calamari/salt/salt/

pillar_roots:
  base:
      - /opt/calamari/salt/pillar/

reactor:
  - 'salt/minion/*/start':
    - /opt/calamari/salt/reactor/start.sls

# add the Debian，RedHat and SUSE default apache users to
# avoid making this file distro-dependent

client_acl:
  www-data:
    - log_tail.*
  apache:
    - log_tail.*
    - .*
  wwwrun:
    - log_tail.*
```

同时修改一些运行目录的权限：

```
# chmod 755 /var/cache/salt /var/cache/salt/master /var/cache/salt/master/jobs /var/run/salt /var/run/salt/master
```

详见：<http://docs.saltstack.cn/zh_CN/latest/ref/clientacl.html>

## 4. 安装 Calamari-client

```
# yum install npm ruby rubygems
# npm install -g grunt grunt-cli bower grunt-contrib-compass
# gem update --system && gem install compass
# cd calamari-clients
# make build-real
```

npm install 出错解决方案：

```
# npm config set strict-ssl false
# npm config set registry="http://registry.npmjs.org/"
```

详见：<http://stackoverflow.com/questions/21855035/ssl-error-cert-untrusted-while-using-npm-command>

make build-real 出错：

```
Loading "connect_proxy.js" tasks...ERROR
    >> TypeError: Cannot read property 'prototype' of undefined

Looks like I've found the solution in this GitHub issue68 for grunt-connect-proxy.
I found a few fixes whilst this gets patched up，you only need to do one or the other:

Change the source for grunt-connect-proxy in your package.json to this:
"grunt-connect-proxy": "git://github.com/ruiaraujo/grunt-connect-proxy#patch-1",

or run this in your webhook project folder:

rm -rf node_modules/grunt-connect-proxy
npm install eventemitter3@0.1.6
npm install grunt-connect-proxy

As you can see it's an issue with the version of eventemitter3
```

详见：<http://forums.webhook.com/t/cant-run-local-server/643>

phantomjs 由于被墙导致安装失败：

```
# export PHANTOMJS_CDNURL=http://cnpmjs.org/downloads npm install phantomjs
# npm install -g phantomjs
```

编辑 Makefile.sub 文件：

```
grunt --no-color --force build
```

添加 `--force` 参数

build-real 完毕后会在 admin，login，dashboard，manage 目录下生成 dist 目录，将 dist 目录下的内容分别拷贝到 /opt/calamari/webapp/content 目录下的admin，login，dashboard，manage 目录下（需先手动创建）

重启 Calamari-server 服务后，打开浏览器访问：

http://10.160.0.43/dashboard/

http://10.160.0.43/graphite/dashboard/

## 参考链接：

<http://ceph.com/category/calamari/>

<http://ovirt-china.org/mediawiki/index.php/%E5%AE%89%E8%A3%85%E9%83%A8%E7%BD%B2Ceph_Calamari>

<http://blog.csdn.net/wytdahu/article/details/46350909>
