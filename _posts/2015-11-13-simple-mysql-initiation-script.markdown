---
layout: post
title:  "Simple MySQL Initiation Script"
date:   2015-10-15 15:46:53
categories: Trove MySQL
---


一个简单的MySQL初始化脚本, 用于配置虚机内部的MySQL服务, 参考了`OpenStack` `Trove`项目里的代码.

基本原理:

1. 脚本放置与镜像中, 基于该镜像创建的虚机启动的时候会运行该脚本.
2. 镜像中MySQL服务默认已安装好, 同时已授权用户*os_admin*拥有超级用户权限.
3. MySQL配置项通过`nova`的[*config-driver*](http://docs.openstack.org/user-guide/cli_config_drive.html)获取.
4. MySQL的*datadir*跑在`cinder`创建的*volume*中, 因此虚机会挂载好一块*volume*.
5. 通过参数*is_master*判断是否需要搭建主从, 如果是从则需要额外配置.



repositroy:

<https://github.com/begonia/MonkeyStick/blob/master/upd/mysql/mysql_init.py>
