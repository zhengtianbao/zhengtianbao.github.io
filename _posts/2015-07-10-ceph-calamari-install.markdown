---
layout: post
title:  "ceph calamari install"
date:   2015-07-10 20:46:53
categories: Ceph Calamari
---


## 节点服务分配

- 4个osd节点, 3个mon节点, 所有osd, mon节点都安装Salt-minion, Diamond
- Salt-master远程调用目标minion执行ceph命令, 如: ceph osd dump
- Diamond收集ceph集群以及节点磁盘, 网络等信息发送给Graphite
- Graphite由三个组件构成, carbon接受Diamond发送的数据, whisper类似数据库持久化数据, graphite-web借助django、apache等提供web服务
- Calamari-server调用salt获取ceph集群信息, 对外暴露REST API
- Calamari-client是nodejs写的一个前端页面, 调用Calamari-server以及Graphite的API获取数据展示图表


| node        | ip          | server                                  |
| ----------- | ----------- | --------------------------------------- |
| ceph-1      | 10.160.0.41 | osd.0 mon.0 Salt-minion Diamond         |
| ceph-2      | 10.160.0.42 | osd.1 mon.1 Salt-minion Diamond         |
| ceph-3      | 10.160.0.43 | osd.2 mon.2 Salt-minion Diamond         |
|             |             | Salt-master Calamari-server Apache      |
|             |             | Graphite(carbon, whisper, grahpite-web) |
|             |             | Calamari-client                         |
| cns-5       | 10.160.0.55 | osd.3 Salt-minion Diamond               |


## 安装


### 1. 在所有ceph节点安装salt-minion

{% highlight text %}
# yum install salt-minion
{% endhighlight %}

注意: 如果有错误后面提示cthulhu服务起不来: <https://github.com/ceph/calamari/issues/285>

可能是salt版本过高导致, 需要降低版本到2014.7.5

{% highlight text %}
# wget ftp://195.220.108.108/linux/fedora/linux/releases/22/Everything/x86_64/os/Packages/s/salt-2014.7.5-1.fc22.noarch.rpm
# rpm -Uvh --oldpackage salt-minion-2014.7.5-1.fc22.noarch.rpm 
# salt-minion --version
salt-minion 2014.7.5 (Helium)
{% endhighlight %}

编辑/etc/salt/minion.d/calamari.conf

{% highlight text %}
master: {fqdn}
{% endhighlight %}

{fqdn}对应Salt-master的域名(10.160.0.43)

重启服务:

{% highlight text %}
# service salt-minion restart
{% endhighlight %}


### 2. 在所有ceph节点安装Diamond

{% highlight text %}
# git clone https://github.com/ceph/Diamond
# cd Diamond
# git checkout origin/calamari
# make rpm
# yum localinstall diamond-3.4.67-0.noarch.rpm
{% endhighlight %}

编辑/etc/diamond/diamond.conf:

设置hostname_method = hostname_short

重启服务:

{% highlight text %}
# /etc/init.d/diamond restart
{% endhighlight %}


### 3. 安装Calamari-server

{% highlight text %}
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
{% endhighlight %}

配置saltstack认证:

{% highlight text %}
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
{% endhighlight %}

查看启动的服务:

{% highlight text %}
# supervisorctl status
carbon-cache                     RUNNING    pid 7745, uptime 2:03:54
cthulhu                          RUNNING    pid 14432, uptime 1:05:01
{% endhighlight %}

至此, Calamari-server安装完毕, 调用API: 

<http://ceph.com/calamari/docs/calamari_rest/resources/resources.html>

运行calamari/tests/http_client.py验证:

{% highlight text %}
python http_client.py -u http://10.160.0.43/api/v2/ --user root --pass 123456
{% endhighlight %}

注意: 调用api/v2/cluster/{fsid}/cli 如果失败返回500错误, 可能是salt的配置问题

因为Calamari-server是以apache身份运行的, 没有权限, 需要修改/etc/salt/master.d/calamari.conf:

{% highlight text %}
file_roots:
  base:
      - /opt/calamari/salt/salt/

pillar_roots:
  base:
      - /opt/calamari/salt/pillar/

reactor:
  - 'salt/minion/*/start':
    - /opt/calamari/salt/reactor/start.sls

# add the Debian, RedHat and SUSE default apache users to
# avoid making this file distro-dependent

client_acl:
  www-data:
    - log_tail.*
  apache:
    - log_tail.*
    - .*
  wwwrun:
    - log_tail.*
{% endhighlight %}

同时修改一些运行目录的权限:

{% highlight text %}
# chmod 755 /var/cache/salt /var/cache/salt/master /var/cache/salt/master/jobs /var/run/salt /var/run/salt/master
{% endhighlight %}

详见: <http://docs.saltstack.cn/zh_CN/latest/ref/clientacl.html>


## 4. 安装Calamari-client

{% highlight text %}
# yum install npm ruby rubygems
# npm install -g grunt grunt-cli bower grunt-contrib-compass
# gem update --system && gem install compass
# cd calamari-clients
# make build-real
{% endhighlight %}

npm install出错:

{% highlight text %}
# npm config set strict-ssl false
# npm config set registry="http://registry.npmjs.org/"
{% endhighlight %}

详见:<http://stackoverflow.com/questions/21855035/ssl-error-cert-untrusted-while-using-npm-command>

make build-real出错:

{% highlight text %}
Loading "connect_proxy.js" tasks...ERROR
    >> TypeError: Cannot read property 'prototype' of undefined

Looks like I've found the solution in this GitHub issue68 for grunt-connect-proxy.
I found a few fixes whilst this gets patched up, you only need to do one or the other:

Change the source for grunt-connect-proxy in your package.json to this:
"grunt-connect-proxy": "git://github.com/ruiaraujo/grunt-connect-proxy#patch-1",

or run this in your webhook project folder:

rm -rf node_modules/grunt-connect-proxy
npm install eventemitter3@0.1.6
npm install grunt-connect-proxy

As you can see it's an issue with the version of eventemitter3
{% endhighlight %}

详见<http://forums.webhook.com/t/cant-run-local-server/643>


phantomjs由于被墙导致安装失败:

{% highlight text %}
# export PHANTOMJS_CDNURL=http://cnpmjs.org/downloads npm install phantomjs
# npm install -g phantomjs
{% endhighlight %}

编辑Makefile.sub文件:

{% highlight text %}
grunt --no-color --force build
{% endhighlight %}

添加`--force`参数


build-real完毕后会在admin, login, dashboard, manage目录下生成dist目录, 将dist目录下的内容分别拷贝到/opt/calamari/webapp/content目录下的admin, login, dashboard, manage目录下(需先手动创建)

重启Calamari-server服务后, 打开浏览器访问:

http://10.160.0.43/dashboard/

http://10.160.0.43/graphite/dashboard/



## 参考链接:

<http://ceph.com/category/calamari/>

<http://ovirt-china.org/mediawiki/index.php/%E5%AE%89%E8%A3%85%E9%83%A8%E7%BD%B2Ceph_Calamari>

<http://blog.csdn.net/wytdahu/article/details/46350909>
