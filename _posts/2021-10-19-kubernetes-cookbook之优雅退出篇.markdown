---
layout: post
title: "kubernetes cookbook之优雅退出篇"
date:  2021-10-19 14:23:34
categories: Kubernetes
---

![log](/images/gracefulshutdown.jpeg)

云原生(cloud native)应用其中有一条就是能够快速迭代，应用不断升级过程中不可避免就是启停操作，删除旧版本，更新新版本，如果应用是部署在kubernetes上面的，那么update deployment过程中kubelet会发送`SIGTERM`信号给容器PID 1的进程，做为应用程序应当捕获这个信号，可以在真正退出之前做一些清理工作，例如通知注册中心自己已消亡。

## 捕获SIGTERM信号

go语言中的`signal`包提供了接受信号的方法

```go

import (
	"context"
	"os"
	"os/signal"
)

var onlyOneSignalHandler = make(chan struct{})
var shutdownHandler chan os.Signal
var shutdownSignals = []os.Signal{os.Interrupt, syscall.SIGTERM}

func SetupSignalContext() context.Context {
	close(onlyOneSignalHandler) // panics when called twice

	shutdownHandler = make(chan os.Signal, 2)

	ctx, cancel := context.WithCancel(context.Background())
	signal.Notify(shutdownHandler, shutdownSignals...) // 当接受到信号时发送给shutdownHandler channel
	go func() {
		<-shutdownHandler // 阻塞
		cancel()
		<-shutdownHandler
		os.Exit(1) // second signal. Exit directly.
	}()

	return ctx
}
```

`SetupSignalContext()`返回给调用者一个`context.Context`对象，那么调用者就可以通过`ctx.Done()` 判断是否接受到退出信号：

```go
ctx := SetupSignalContext()
Run(ctx)

func Run(ctx context.Context) {
	fmt.Printf("hello world!\n")
	select {
	case <-ctx.Done():
		fmt.Println("got shutdown signal")
		break
	}
}
```

## systemd通知

当程序以systemd规范的服务启动时，例如canoe的service配置文件：

```
[Unit]
Description=canoe
Documentation=canoe project

[Service]
ExecStart=/usr/local/bin/canoe
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

那么在执行命令`systemctl start canoe`服务启动后，需要通知systemd服务已经就绪，通信的方法就是发送 `"READY=1"` 到"unixgram"

[go-systemd](https://github.com/coreos/go-systemd) 这个库已经封装好了

```go
// 应用就绪后执行
import "github.com/coreos/go-systemd/v22/daemon"

func main() {
	// do something
	do()
	go daemon.SdNotify(false, "READY=1")
}
```



