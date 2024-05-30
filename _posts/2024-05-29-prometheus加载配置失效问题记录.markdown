---
layout: post
title: "Prometheus 加载配置部分缺失问题记录"
date: 2024-05-29 15:05:31 +0800
categories: ["2024"]
tags: [prometheus]
---

## 问题描述

prometheus 在执行配置文件热加载的时候出现了部分配置没有加载的现象。

主配置文件内容为：

```yaml
global:
  scrape_interval:     15s
  evaluation_interval: 15s
  scrape_timeout:      15s

alerting:
  alertmanagers:
  - static_configs:
    - targets:
      - 12.99.78.69:9093
      - 12.99.78.70:9093

rule_files:
  - '../rules/*rules.yml'

scrape_configs:
  - job_name: '核心业务系统'
    metrics_path: /actuator/prometheus
    file_sd_configs:
    - files:
      - '核心业务系统.yml'
  - job_name: '电销系统'
    metrics_path: /actuator/prometheus
    file_sd_configs:
    - files:
      - '电销系统.yml'
  - job_name: '收单服务系统'
    metrics_path: /actuator/prometheus
    file_sd_configs:
    - files:
      - '收单服务系统.yml'
   # 省略其他相同格式的系统 job 配置
```

主配置文件中通过 file_sd_config 指定需要采集系统的 target，以核心业务系统举例，它的配置为：

核心业务系统.yml

```yaml
- targets:
  - 12.103.63.160:14600
  labels:
    system: 核心系统
    application: app-ltcb-task-r00
    module: RDCN00
    env: sit
- targets:
  - 12.103.6.179:10400
  labels:
    system: 核心系统
    application: app-taae-onl-a00
    module: 核心子系统
    env: uat
- targets:
  - 12.103.52.221:13402
  - 12.103.52.222:13402
  labels:
    system: 核心系统
    application: app-ltpb-onl-r02
    module: 冲销查证
    env: uat
# 省略其他配置
```

这样配置带来的好处是显而易见的，主配置文件中的 job 只跟系统关联，系统下的需要采集的应用，都能通过自定义标签区分开来。

为了方便对接入 prometheus 监控系统进行管理，我们开发了一套系统：主要就是维护系统、应用与配置文件的对应关系。通过前端表单填写接入信息生成对应的配置文件，然后请求 prometheus 的 reload 接口，在不需要重启 prometheus 服务的情况下热加载配置。

注：配置热加载需要在 promethues 启动参数中添加启动参数 `--web.enable-lifecycle`，发送以下请求触发热加载：

```
curl -X POST http://prometheus:9090/-/reload
```

没想到的是随着接入系统的不断增加，发现有些时候配置文件已经更新，但是执行 reload 操作后 prometheus 采集的 target 却没有生成，因为这个现象是偶发的，开始以为是误操作导致的，后来频率越来越高才发觉问题没那么简单。

## 排查过程

因为是配置文件过多导致出现了问题，因此编写脚本生成大量的配置文件来模拟真实环境。

```shell
#!/bin/bash

function append_conf() {
cat <<EOF >> prometheus.yml
  - job_name: system$1
    metrics_path: /actuator/prometheus
    file_sd_configs:
    - files:
      - system$1.yml
EOF
}

function gen_conf() {
cat <<EOF > system$1.yml
- targets:
  - 12.103.52.$i:13402
  labels:
    system: system$1
    application: system$1-app-A
    module: moduleA
    env: sit
EOF
}

function reload() {
  curl -X POST http://127.0.0.1:9090/-/reload
  sleep 5
}

function get_prom_target() {
  echo $(curl -s http://127.0.0.1:9090/api/v1/targets | jq ' .data.activeTargets | length ')
}

for i in {0..9}; do
  for j in {1..10}; do
    append_conf $(($i*10+$j))
    gen_conf $(($i*10+$j))
  done
  reload
  prom_target=$(get_prom_target)
  echo ${prom_target}
  if (( ${prom_target} != $(($i*10+10)) )); then
    echo "prometheus scrape target count ${prom_target} not match $(($i*10+10)) at round $(($i+1))"
    break
  fi
done
```

解释下这个脚本：

1. 生成 prometheus 主配置文件中的 job
2. 生成对应系统的 yaml 配置文件
3. 每执行 10 次之后执行 reload 操作
4. 比较 prometheus 抓取 target 数量是否正确

执行后产生输出：

```
$ ./gen.sh
10
20
30
40
50
60
69
prometheus scrape target count 69 not match 70 at round 7
```

在第 7 轮的时候，也就是添加了 70 个 job 的时候就出现了目标采集数量缺失的现象。

查看 prometheus 日志，发现报错信息：

```
level=error ts=2024-05-30T07:39:42.592Z caller=file.go:225 component="discovery manager scrape" discovery=file msg="Error adding file watcher" err="too many open files"
```

"too many open files" 好熟悉的错误，难道是文件描述符不够用了？

尝试增大 `fs.file-max` 参数值后发现这个问题依然存在，看来报错信息过于简单，只能从代码入手。

file.go：

```go
// Run implements the Discoverer interface.
func (d *Discovery) Run(ctx context.Context, ch chan<- []*targetgroup.Group) {
	watcher, err := fsnotify.NewWatcher()
	if err != nil {
		level.Error(d.logger).Log("msg", "Error adding file watcher", "err", err)
		return
	}
	// ...
}
```

可以看出该报错是调用 fsnotify 库的 `NewWatcher()` 出错导致的

inotify.go：

```go
// NewWatcher establishes a new watcher with the underlying OS and begins waiting for events.
func NewWatcher() (*Watcher, error) {
	// Create inotify fd
	fd, errno := unix.InotifyInit1(unix.IN_CLOEXEC)
	if fd == -1 {
		return nil, errno
	}
	// ...
}
```

`NewWatcher()` 执行了系统调用，查看说明文档：

> The `fs.inotify.max_user_watches` sysctl variable specifies the upper limit for the number of watches per user, and `fs.inotify.max_user_instances` specifies the maximum number of inotify instances per user. Every Watcher you create is an "instance", and every path you add is a "watch".

有两个系统变量影响了 inotfiy 的数量，可以在 `/proc/sys/fs/inotify/` 路径下查看：

```
$ cat /proc/sys/fs/inotify/max_user_watches 
8192
$ cat /proc/sys/fs/inotify/max_user_instances 
128
```

查看 prometheus 已占用的 `instance`

```
$ lsof -p $(pidof prometheus) |grep notify |wc -l
68
```

虽然小于 `fs.inotify.max_user_instances` 定义的 128，但是这个值是针对用户级别的，可能有其他进程也有占用。

调整大小为 1024

```
$ sudo sysctl fs.inotify.max_user_instances=1024
fs.inotify.max_user_instances = 1024
```

调整后调整测试代码，增加轮数：

```
$ ./gen.sh
10
20
# skip output
200
209
prometheus scrape target count 209 not match 210 at round 21
```

增加到 200 多个才出现报错信息，说明是有效果的。

完。
