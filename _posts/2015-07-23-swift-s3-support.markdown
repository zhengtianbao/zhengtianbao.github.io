---
layout: post
title:  "swift s3 support"
date:   2015-07-23 20:46:53
categories: Swift S3
---


## 配置swift启用s3 middleware

swift3 middleware: [https://github.com/stackforge/swift3](https://github.com/stackforge/swift3)

### 1. 安装

{% highlight text %}
# python setup.py egg_info
# pip install -r swift3.egg-info/requires.txt -i http://pypi.douban.com/simple
# python setup.py install
{% endhighlight %}


### 2. 配置`vim /etc/swift/proxy-server.conf` *pipeline*

在`authtoken`前加`swift3 s3token`两个*filter*

    Was::

        [pipeline:main]
        pipeline = catch_errors cache authtoken keystone proxy-server

    Change To::

        [pipeline:main]
        pipeline = catch_errors cache swift3 s3token authtoken keystoneauth proxy-server

    To support Multipart Upload::

        [pipeline:main]
        pipeline = catch_errors cache swift3 s3token authtoken keystoneauth slo proxy-server

### 3. 加上这两个*filter*

    [filter:swift3]
    use = egg:swift3#swift3

    [filter:s3token]
    paste.filter_factory = keystoneclient.middleware.s3_token:filter_factory
    auth_port = 35357
    auth_host = 10.160.0.31
    auth_protocol = http
    cache = swift.cache

注意s3token中的*paste.filter_factory*, 较老的版本可能需要配置为: `keystone.middleware.s3_token:filter_factory`

### 4. 重启swift-proxy服务

{% highlight text %}
# systemctl restart openstack-swift-proxy
{% endhighlight %}

## 测试swift s3

### 1. 为指定_project(tenant)_的_user_创建_ec2-credentials_

{% highlight text %}
# keystone ec2-credentials-create --user-id <user-id> --tenant-id <tenant-id>
# keystone ec2-credentials-list --user-id <user-id>
+----------------------------------+----------------------------------+----------------------------------+
|              tenant              |              access              |              secret              |
+----------------------------------+----------------------------------+----------------------------------+
| 1d49717073724f61a66918841071767d | fd8d0b386373480b964e321be35dc754 | d63262799c14431b88a408e0a88bd3e1 |
+----------------------------------+----------------------------------+----------------------------------+
{% endhighlight %}

获取到`aws_access_key_id`和它的`aws_secret_access_key`

### 2. 通过AWS提供的[python boto interface](https://boto.readthedocs.org/en/latest/s3_tut.html)来测试

{% highlight python %}
>>> import boto
>>> from boto.s3.connection import S3Connection
>>> c = S3Connection(
...    aws_access_key_id='fd8d0b386373480b964e321be35dc754',
...    aws_secret_access_key='d63262799c14431b88a408e0a88bd3e1',
...    port=8080,
...    host='10.160.0.3',
...    is_secure=False,
...    calling_format=boto.s3.connection.OrdinaryCallingFormat())
>>> c.get_all_buckets()
[<Bucket: 1st>, <Bucket: 2nd>, <Bucket: media.mydomain.com>, <Bucket: media.yourdomain.com>]
>>> test_b = c.create_bucket('test') # 创建一个bucket
>>> test_b.get_all_keys() # 获取bucket中的所有key
[]
>>> from boto.s3.key import Key
>>> k=Key(test_b,'hello') # 在test_b中创建一个名为hello的key
>>> k.set_contents_from_filename('/root/hello.txt') # 从文件中将内容上传到key
11
>>> k.get_contents_as_string() # 读取key的内容
'helloworld\n'
>>> k.delete()
<Key: test,hello>
>>> test_b.get_all_keys()
[]
>>> c.delete_bucket('test')
>>> c.get_all_buckets()
[<Bucket: 1st>, <Bucket: 2nd>, <Bucket: media.mydomain.com>, <Bucket: media.yourdomain.com>]
>>> 
{% endhighlight %}

## S3FS-Fuse

测试使用[s3fs](https://github.com/s3fs-fuse/s3fs-fuse)挂载swift bucket到本地

### 1. 安装fuse

{% highlight text %}
# yum install fuse fuse-devel
# modprobe fuse
# pkg-config --modversion fuse
2.9.2
{% endhighlight %}

### 2. 安装s3fs

{% highlight text %}
# git clone https://github.com/s3fs-fuse/s3fs-fuse.git
# cd s3fs-fuse
# ./autogen.sh
# ./configure
# make && make install
{% endhighlight %}

### 3. 创建s3fs credential file

格式为bucketName:accessKeyId:secretAccessKey

{% highlight text %}
# cat << EOT > ~/.passwd-s3fs
> 1st:fd8d0b386373480b964e321be35dc754:d63262799c14431b88a408e0a88bd3e1
> EOT
# chmod 600 ~/.passwd-s3fs
# cat ~/.passwd-s3fs 
1st:fd8d0b386373480b964e321be35dc754:d63262799c14431b88a408e0a88bd3e1
{% endhighlight %}

### 4. 测试挂载bucket 1st到/mnt/s3下

{% highlight text %}
# s3fs 1st /mnt/s3 -o url=http://10.160.0.3:8080/ -o use_path_request_style -o sigv2 
# ls /mnt/s3/
1.png  1.txt  2.txt  333.txt  sample
{% endhighlight %}

注意参数`use_path_request_style`和`sigv2`, 可以通过加参数*-d -f*输出debug信息


## S3 Browser

Windows环境下连接Amazon S3的客户端, 注意连接的时候*NOT* use HTTPS transfer.

效果如下所示:

![S3 Browser](/images/s3browser.png)


## 参考链接:

<http://blog.csdn.net/anhuidelinger/article/details/9749973>

<http://www.cnblogs.com/yuxc/archive/2012/05/12/2497385.html>

<http://www.itisopen.net/2011/12/CentOS_and_s3fs/>

<https://github.com/sagara177/s3fs-swift>


