---
layout: post
title: "Add new segment to greenplum cluster"
date:  2018-06-20 18:09:29
categories: Greenplum
---

在生产环境中，因为数据量的增加，存在动态扩容 greenplum 存储容量的需求。

纵向扩容指在现有服务器上增加配置，如增加磁盘，增加内存等，因为在集群初始化部署规划中就已经考虑到了服务器各资源的配置，一般而言多用于测试环境。

横向扩容指增加服务器节点，这在分布式存储系统上是通用解决方案，如 hdfs，ceph 等都能做到横向动态扩容。

本文续接上文：[greenplum 部署](https://zhengtianbao.com/ansible/greenplum/2018/06/15/ansible-install-greenplum.html) 因此环境与上文保持一致，测试增加一台服务器 sdw2 作为segment 节点，服务器列表如下表：

| IP            | hostname  | role           | 数据盘 |
| ------------- | --------- | -------------- | ------ |
| 192.168.1.213 | mdw       | master         | vdb    |
| 192.168.1.223 | smdw      | standby master | vdb    |
| 192.168.1.233 | sdw1      | segment1       | vdb    |
| 192.168.1.204 | localhost | dns, ntp       |        |
| 192.168.1.243 | sdw2      | segment2       | vdb    |

## ansible 脚本配置 sdw2 服务器环境

ansible inventory配置：

```
[ntp]
sdw2

[dns]
sdw2

[greenplum_master]
[greenplum_standby]

[greenplum_segments]
sdw2

[greenplums]
sdw2
```

site.yml

```yaml
- name: configure local DNS server
  hosts: dns
  remote_user: root
  roles:
    - {role: dns, dns_nameservers:['192.168.1.204']}
  tags:
    - dns
    
- name: deploy greenplum
  hosts: greenplums                                                        
  remote_user: root
  roles:
    - greenplum
  tags:
    - greenplum 
```

ansible-playbook

```
# git clone http://git.jfbrother.com/zhengtianbao/jfbigdata-ansible.git jfbigdata-ansible
# cd jfbigdata-ansible
# ansible-playbook -t ntp,dns,greenplum -i inventory site.yml 
```

初始化 sdw2 环境完毕。

## 扩容

登录到 master 节点

```
ssh root@mdw
[root@mdw ~]# cat << EOF > /home/gpadmin/new_segnodes
sdw2
EOF
# ssh key
[root@mdw ~]# gpssh-exkeys -e /home/gpadmin/all_nodes -x /home/gpadmin/newseg_nodes
# 初始化新增segment node
[root@mdw ~]# gpseginstall -f /home/gpadmin/newseg_nodes -u gpadmin -p 123456
# 登录到gpadmin用户
[root@mdw ~]# su - gpadmin
# greenplum创建临时数据库
[gpadmin@mdw ~]$ psql postgres gpadmin
psql (8.3.23)
Type "help" for help.in;

postgres=# create database jfbrother owner gpadmin;
postgres=# \q
# 生成gpexpand配置文件，也可以自己手动配置
[gpadmin@mdw ~]$ gpexpand -f newseg_nodes -D jfbrother
# ！ 注意 这里报错，因为集群已配置为mirror模式，因此需要增加偶数个节点，作为测试手动配置文件，文件规则见下文
[gpadmin@mdw ~]$ cat << EOF > gpexpand_inputfile
sdw2:sdw2:40000:/data1/gpdatap1/gpseg2:7:2:p:41000
sdw2:sdw2:40001:/data1/gpdatap2/gpseg3:8:3:p:41001
sdw2:sdw2:50000:/data1/gpdatam1/gpseg2:9:2:m:51000
sdw2:sdw2:50001:/data1/gpdatam2/gpseg3:10:3:m:51001
EOF
# expand节点
[gpadmin@mdw ~]$ gpexpand -i gpexpand_inputfile -D jfbrother -S -V -v -n 1 -t /tmp
# 查看结果
[gpadmin@mdw ~]$ gpstate
```

## gpexpand 文件规则说明

postgre 中查看现有的配置规则：

```
[gpadmin@mdw ~]$ psql postgres gpadmin
psql (8.3.23)
Type "help" for help.

postgres=# select * from gp_segment_configuration;
 dbid | content | role | preferred_role | mode | status | port  | hostname | address | replication_port 
------+---------+------+----------------+------+--------+-------+----------+---------+------------------
    1 |      -1 | p    | p              | s    | u      |  5432 | mdw      | mdw     |                 
    2 |       0 | p    | p              | s    | u      | 40000 | sdw1     | sdw1    |            41000
    3 |       1 | p    | p              | s    | u      | 40001 | sdw1     | sdw1    |            41001
    4 |       0 | m    | m              | s    | u      | 50000 | sdw1     | sdw1    |            51000
    5 |       1 | m    | m              | s    | u      | 50001 | sdw1     | sdw1    |            51001
    6 |      -1 | m    | m              | s    | u      |  5432 | smdw     | smdw    |                 
```

而 gpexpand 文件每行为：

```
<hostname>:<address>:<port>:<fselocation>:<dbid>:<content>:<preferred_role>:<replication_port>
hostname			主机名
address				类似主机名
port				segment 监听端口
fselocation			segment data 目录，注意是全路径
dbid				gp 集群的唯一 ID，可以到gp_segment_configuration 中获得，必须顺序累加
content				可以到 gp_segment_configuration 中获得，必须顺序累加
prefered_role		角色（p 或 m）（primary，mirror）
replication_port	如果没有 mirror 则不需要（用于 replication 的端口）
```

## 扩容失败回滚操作

```
[gpadmin@mdw ~]$ gpstart -R
[gpadmin@mdw ~]$ gpexpand -r -D jfbrother
[gpadmin@mdw ~]$ gpstart
```

## 参考链接：

<https://yq.aliyun.com/articles/177>

<http://www.dbdream.com.cn/2016/03/02/greenplum%E6%95%B0%E6%8D%AE%E5%BA%93%E6%89%A9%E5%AE%B9-%E5%A2%9E%E5%8A%A0segment/>
