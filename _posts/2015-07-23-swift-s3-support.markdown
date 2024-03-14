---
title: "swift s3 support"
date: 2015-07-23 20:46:53
categories: ["2015"]
tags: [swift]
---

本文记录了如何配置 swift 使其支持 S3 接口协议，以及测试相关工具兼容性的过程。

## 配置 swift 启用 s3 middleware

swift3 middleware：[https://github.com/stackforge/swift3](https://github.com/stackforge/swift3)

### 1. 安装

```
# python setup.py egg_info
# pip install -r swift3.egg-info/requires.txt -i http://pypi.douban.com/simple
# python setup.py install
```

### 2. 配置 /etc/swift/proxy-server.conf pipeline

在 `authtoken` 前加 `swift3 s3token` 两个 **filter**，参考注释说明：

```
    Was::

        [pipeline:main]
        pipeline = catch_errors cache authtoken keystone proxy-server

    Change To::

        [pipeline:main]
        pipeline = catch_errors cache swift3 s3token authtoken keystoneauth proxy-server

    To support Multipart Upload::

        [pipeline:main]
        pipeline = catch_errors cache swift3 s3token authtoken keystoneauth slo proxy-server
```

### 3. 加上这两个 filter

```
[filter:swift3]
use = egg:swift3#swift3

[filter:s3token]
paste.filter_factory = keystoneclient.middleware.s3_token:filter_factory
auth_port = 35357
auth_host = 10.160.0.31
auth_protocol = http
cache = swift.cache
```

注意：s3token 中的 **paste.filter_factory**，较老的版本可能需要配置为：`keystone.middleware.s3_token:filter_factory`

### 4. 重启 swift-proxy 服务

```
# systemctl restart openstack-swift-proxy
```

## 测试 swift s3

### 1. 为指定 _project(tenant)_ 的 _user_ 创建 _ec2-credentials_

```
# keystone ec2-credentials-create --user-id <user-id> --tenant-id <tenant-id>
# keystone ec2-credentials-list --user-id <user-id>
+----------------------------------+----------------------------------+----------------------------------+
|              tenant              |              access              |              secret              |
+----------------------------------+----------------------------------+----------------------------------+
| 1d49717073724f61a66918841071767d | fd8d0b386373480b964e321be35dc754 | d63262799c14431b88a408e0a88bd3e1 |
+----------------------------------+----------------------------------+----------------------------------+
```

获取到 `aws_access_key_id` 和它的 `aws_secret_access_key`

### 2. 通过 AWS 提供的 [python boto interface](https://boto.readthedocs.org/en/latest/s3_tut.html) 来测试

```
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
>>> test_b = c.create_bucket('test') # 创建一个 bucket
>>> test_b.get_all_keys() # 获取 bucket 中的所有 key
[]
>>> from boto.s3.key import Key
>>> k=Key(test_b,'hello') # 在 test_b 中创建一个名为 hello 的 key
>>> k.set_contents_from_filename('/root/hello.txt') # 从文件中将内容上传到 key
11
>>> k.get_contents_as_string() # 读取 key 的内容
'helloworld\n'
>>> k.delete()
<Key: test,hello>
>>> test_b.get_all_keys()
[]
>>> c.delete_bucket('test')
>>> c.get_all_buckets()
[<Bucket: 1st>, <Bucket: 2nd>, <Bucket: media.mydomain.com>, <Bucket: media.yourdomain.com>]
>>> 
```

## S3FS-Fuse

测试使用 [s3fs](https://github.com/s3fs-fuse/s3fs-fuse) 挂载 swift bucket 到本地

### 1. 安装 fuse

```
# yum install fuse fuse-devel
# modprobe fuse
# pkg-config --modversion fuse
2.9.2
```

### 2. 安装 s3fs

```
# git clone https://github.com/s3fs-fuse/s3fs-fuse.git
# cd s3fs-fuse
# ./autogen.sh
# ./configure
# make && make install
```

### 3. 创建 s3fs credential file

格式为：bucketName:accessKeyId:secretAccessKey

```
# cat << EOT > ~/.passwd-s3fs
> 1st:fd8d0b386373480b964e321be35dc754:d63262799c14431b88a408e0a88bd3e1
> EOT
# chmod 600 ~/.passwd-s3fs
# cat ~/.passwd-s3fs 
1st:fd8d0b386373480b964e321be35dc754:d63262799c14431b88a408e0a88bd3e1
```

### 4. 测试挂载 bucket 1st 到 /mnt/s3 下

```
# s3fs 1st /mnt/s3 -o url=http://10.160.0.3:8080/ -o use_path_request_style -o sigv2 
# ls /mnt/s3/
1.png  1.txt  2.txt  333.txt  sample
```

注意：参数 `use_path_request_style` 和 `sigv2`，可以通过加参数 *-d -f* 输出 debug 信息

## S3 Browser

Windows 环境下连接 Amazon S3 的客户端，注意连接的时候 *NOT* use HTTPS transfer。

效果如下所示:

![S3 Browser](/images/s3browser.png)

## 参考链接

<http://blog.csdn.net/anhuidelinger/article/details/9749973>

<http://www.cnblogs.com/yuxc/archive/2012/05/12/2497385.html>

<http://www.itisopen.net/2011/12/CentOS_and_s3fs/>

<https://github.com/sagara177/s3fs-swift>
