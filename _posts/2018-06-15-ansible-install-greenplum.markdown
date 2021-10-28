---
layout: post
title: "Ansible install Greenplum"
date:  2018-06-15 16:17:29
categories: Ansible Greenplum
---

greenplum官方提供安装工具，只需要先部署master节点就可以通过gpinstll命令分发到各个segment节点。但是各个节点的系统环境参数调优工作需要提前处理。因此使用ansible提前对各个节点进行预处理。

## 节点说明

| IP            | hostname  | role           | 数据盘 |
| ------------- | --------- | -------------- | ------ |
| 192.168.1.213 | mdw       | master         | vdb    |
| 192.168.1.223 | smdw      | standby master | vdb    |
| 192.168.1.233 | sdw1      | segment1       | vdb    |
| 192.168.1.204 | localhost | dns, ntp       |        |


## ansible配置

inventory 配置文件

```
[ntp]
mdw
smdw
sdw1

[dns]
mdw
smdw
sdw1

[greenplum_master]
mdw

[greenplum_standby]
smdw

[greenplum_segments]
sdw1

[greenplums]
mdw
smdw
sdw1
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

dnsmasq.conf

```
address=/mdw/192.168.1.213
address=/smdw/192.168.1.223
address=/sdw1/192.168.1.233
address=/sdw2/192.168.1.243
```

ansible-playbook

```shell
# git clone http://git.jfbrother.com/zhengtianbao/jfbigdata-ansible.git jfbigdata-ansible
# cd jfbigdata-ansible
# ansible-playbook -t ntp,dns,greenplum -i inventory site.yml 
```

至此 ansible预部署完毕

## greenplum master节点操作

```shell
# 添加互信root密码
[root@mdw ~]# gpssh-exkeys -f /home/gpadmin/all_nodes
# 测试是否添加成功
[root@mdw ~]# gpssh -f /home/gpadmin/all_nodes -e "ls -l"
# 安装gpdb到all_nodes
[root@mdw ~]# gpseginstall -f /home/gpadmin/all_nodes -u gpadmin -p 123456
# 切换到gpadmin用户
[root@mdw ~]# su - gpadmin
[gpadmin@mdw ~]$ gpinitsystem -c /home/gpadmin/gpinitsystem_config -s smdw
# 添加访问权限
[gpadmin@mdw ~]$ echo "host     all         gpadmin         0.0.0.0/0       md5" >> /data1/gpmaster/gpseg-1/pg_hba.conf
# 同步pg_hba.conf到smdw standby 节点
[gpadmin@mdw ~]$ gpscp -h smdw -v $MASTER_DATA_DIRECTORY/pg_hba.conf =:$MASTER_DATA_DIRECTORY/
# 重新加载gpdb配置文件
[gpadmin@mdw ~]$ gpstop -u
# 查看集群状态
[gpadmin@mdw ~]$ gpstate -s
# 设置gpadmin远程访问密码
[gpadmin@mdw ~]$ psql postgres gpadmin
psql (8.3.23)
Type "help" for help.

postgres=# alter user gpadmin encrypted password 'gpadmin';
ALTER ROLE
postgres=# \q

```

至此，greenplum集群搭建完毕

## 集群运行中增加segment节点

参见下文:

[greenplum扩容](https://zhengtianbao.com/greenplum/2018/06/20/greenplum%E6%89%A9%E5%AE%B9.html)
