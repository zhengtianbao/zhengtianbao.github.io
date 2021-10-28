---
layout: post
title:  "swift set container quotas"
date:   2015-12-04 13:17:20
categories: Swift
---


### 开启swift container quotas

设置container quotas的入口是swift-proxy, 因此需要修改proxy-server.conf

{% highlight text %}
[pipeline:main]
pipeline = ... account_quotas container_quotas proxy-server

[filter:account_quotas]
use = egg:swift#account_quotas

[filter:container_quotas]
use = egg:swift#container_quotas
{% endhighlight %}

两个egg可以在setup.cfg中找到对应的入口位置:

{% highlight text %}
[entry_points]
...

paste.filter_factory =
    ...
    container_quotas = swift.common.middleware.container_quotas:filter_factory
    account_quotas = swift.common.middleware.account_quotas:filter_factory


{% endhighlight %}

修改完后只需要重启*swift-proxy-server*


### API使用说明

1. 设置quota:

    swift -A <auth_url> -U <user> -K <password> post <container> -m quota-bytes:<value>

2. 查看某个container的quota:

    swift -A <auth_url> -U <user> -K <password> stat <container>

3. 删除某个container的quota:

    swift -A <auth_url> -U <user> -K <password> post <container> -m quota-bytes:

    {% highlight text %}
    [root@swift ~]# swift post test -m quota-bytes:5368709120
    [root@swift ~]# swift stat test
             Account: 1d49717073724f61a66918841071767d
           Container: test
             Objects: 5
               Bytes: 5912
            Read ACL:
           Write ACL:
             Sync To:
            Sync Key:
    Meta Quota-Count: 10000
    Meta Quota-Bytes: 5368709120
       Accept-Ranges: bytes
    X-Storage-Policy: Policy-0
         X-Timestamp: 1436148388.51591
          X-Trans-Id: txdc240e7856da4319bfffa-005660f3be
        Content-Type: text/plain; charset=utf-8
    [root@swift ~]# swift post test -m quota-bytes:

    {% endhighlight %}

4. 上传文件当容量不足时将返回错误码 `413`

    {% highlight text %}
    [root@swift ~]# swift upload hello 3.cap 
    Object PUT failed: http://10.160.0.3:8080/v1/1d49717073724f61a66918841071767d/hello/3.cap 413 Request Entity Too Large   Upload exceeds quota
    {% endhighlight %}


### 参考链接:

<http://docs.openstack.org/juno/config-reference/content/object-storage-account-quotas.html>

<https://swiftstack.com/docs/admin/middleware/container_quotas.html>

