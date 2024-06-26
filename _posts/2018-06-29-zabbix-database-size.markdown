---
title: "Zabbix 数据库磁盘占用空间计算"
date:  2018-06-29 15:19:20
categories: ["2018"]
tags: [zabbix]
---

针对线上 Zabbix 所需数据库磁盘占用空间进行了一个简单的计算。

## 计算公式

### 定义：

被监控服务器数为：n（台）

每台被监控服务器监控项为：m（个）

监控项更新间隔平均值约为：f1（秒）

监控项历史数据保留时长为：t1（秒）

监控项趋势存储时间为：t2（秒）

一条MySQL记录占用字节数为：x（B）

### 已知：

趋势数据每小时生成一次，即趋势数据更新间隔 `f2 = 3600s`

### 则：

总监控项为：`n * m`

每秒总监控项为：`n * m / f1`，即每秒插入数据库条数为：`n * m / f1`

监控项历史数据占用总空间为：`n * m * t1 * x / f1`

监控项趋势数据占用总空间为：`n * m * t2 * x / f2`

报警所占用空间可以忽略不计

zabbix 自身占用空间 10M

总计为：

```
(n * m * t1 * x / f1 + n * m * t2 * x / f2)/(1024*1024) + 10 MB
```

## 举例说明

假设共有服务器 `n=15` 台，每台服务器监控项 `m=20`，每个监控项更新间隔 `f1=1min=60s`，监控项历史数据保留时长 `t1=1w=604800s`，监控项趋势存储时间 `t2=1Y=31536000s`，每条 MySQL 记录字节数 `x=128B`，代入公式可得占用总空间为：

```
(15*20*604800*128/60+15*20*31536000*128/3600)/(1024*1024)+10 ≈ 700 MB
```

## 存储空间清理策略

Zabbix 会对监控项历史数据以及趋势数据进行清理，所以占用总空间会维持在一个稳定值。

## 参考链接

[zabbix 官网](https://www.zabbix.com/documentation/3.4/manual/installation/requirements)
