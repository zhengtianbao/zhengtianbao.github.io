---
layout: post
title: "kubernetes cookbook之日志篇"
date:  2021-10-17 09:25:29
categories: Kubernetes
---

![log](/images/log.jpeg)

开始新项目之前我总是会习惯先设计好日志模块，这样可以避免在开发过程中代码行中充斥着大量且临时的`print`输出语句。kubernetes的日志模块是C++版本 [google/glog](https://github.com/google/glog) 的Go版本实现，基本实现了原生glog的日志格式，早期版本中是[golang/glog](https://github.com/golang/glog)，目前已经迁移到[klog](https://github.com/kubernetes/klog) 作为日志库，被替换的原因总结下来有以下几条：

- glog 默认会在 `init` 方法中注册 `flag` 参数，所以当程序导入glog库后执行 `flag.Parse()` 会将glog的参数默认导入
- glog 默认将日志写入到文件中，如果没有权限创建日志文件就会报错退出，这在readonly的容器环境中容易出错
- glog没有 **logrotate** 机制

## klog

klog的使用非常简单，举个例子说明：

t.go

```go
package main

import (
	"errors"
	"flag"

	"k8s.io/klog/v2"
)

func main() {
	klog.InitFlags(flag.CommandLine)
	flag.Parse()
	defer klog.Flush()

	err := errors.New("fail")
	klog.Error("This is error message")
	klog.Errorf("Log using Errorf, err: %v", err)
	klog.ErrorS(err, "Log using ErrorS")

	klog.Info("This is info message")
	klog.Infof("This is info message: %v", 12345)
	klog.InfoDepth(1, "This is info message", 12345)

	klog.Warning("This is warning message")
	klog.Warningf("This is warning message: %v", 12345)
	klog.WarningDepth(1, "This is warning message", 12345)

	klog.Error("This is error message")
	klog.Errorf("This is error message: %v", 12345)
	klog.ErrorDepth(1, "This is error message", 12345)

	klog.V(3).Info("LEVEL 3 message")
	klog.V(4).Infof("LEVEL %s message", "4")
	klog.V(5).InfoS("LEVEL 5 message", "level", "5")

	klog.Fatal("This is fatal message")
	klog.Fatalf("This is fatal message: %v", 12345)
	klog.FatalDepth(1, "This is fatal message", 12345)
}
```

执行这个文件，将会得到以下输出：

```
➜  canoe go run t.go -v 5
E1018 11:25:02.181292  274352 t.go:16] This is error message
E1018 11:25:02.181451  274352 t.go:17] Log using Errorf, err: fail
E1018 11:25:02.181483  274352 t.go:18] "Log using ErrorS" err="fail"
I1018 11:25:02.181500  274352 t.go:20] This is info message
I1018 11:25:02.181509  274352 t.go:21] This is info message: 12345
I1018 11:25:02.181520  274352 proc.go:255] This is info message12345
W1018 11:25:02.181530  274352 t.go:24] This is warning message
W1018 11:25:02.181538  274352 t.go:25] This is warning message: 12345
W1018 11:25:02.181548  274352 proc.go:255] This is warning message12345
E1018 11:25:02.181561  274352 t.go:28] This is error message
E1018 11:25:02.181576  274352 t.go:29] This is error message: 12345
E1018 11:25:02.181591  274352 proc.go:255] This is error message12345
I1018 11:25:02.181602  274352 t.go:32] LEVEL 3 message
I1018 11:25:02.181613  274352 t.go:33] LEVEL 4 message
I1018 11:25:02.181626  274352 t.go:34] "LEVEL 5 message" level="5"
F1018 11:25:02.181638  274352 t.go:36] This is fatal message
goroutine 1 [running]:
k8s.io/klog/v2.stacks(0x1)
	/home/zhengtianbao/go/pkg/mod/k8s.io/klog/v2@v2.9.0/klog.go:1026 +0x8a
k8s.io/klog/v2.(*loggingT).output(0x578b40, 0x3, {0x0, 0x0}, 0xc000106000, 0x1, {0x4fc439, 0x578f20}, 0xc00004c480, 0x0)
	/home/zhengtianbao/go/pkg/mod/k8s.io/klog/v2@v2.9.0/klog.go:975 +0x63d
k8s.io/klog/v2.(*loggingT).printDepth(0x0, 0x0, {0x0, 0x0}, {0x0, 0x0}, 0x4c6cef, {0xc00004c480, 0x1, 0x1})
	/home/zhengtianbao/go/pkg/mod/k8s.io/klog/v2@v2.9.0/klog.go:735 +0x1ba
k8s.io/klog/v2.(*loggingT).print(...)
	/home/zhengtianbao/go/pkg/mod/k8s.io/klog/v2@v2.9.0/klog.go:717
k8s.io/klog/v2.Fatal(...)
	/home/zhengtianbao/go/pkg/mod/k8s.io/klog/v2@v2.9.0/klog.go:1494
main.main()
	/home/zhengtianbao/workspace/canoe/t.go:36 +0x948

goroutine 6 [chan receive]:
k8s.io/klog/v2.(*loggingT).flushDaemon(0x0)
	/home/zhengtianbao/go/pkg/mod/k8s.io/klog/v2@v2.9.0/klog.go:1169 +0x6a
created by k8s.io/klog/v2.init.0
	/home/zhengtianbao/go/pkg/mod/k8s.io/klog/v2@v2.9.0/klog.go:420 +0xfb
exit status 255
```

klog提供了四个级别的日志FATAL, ERROR, WARNING, INFO，同时支持`V()`方法定义日志level，例如可以通过命令行参数`-v 5`打印输出level 5及以下的日志信息

## kubernetes中的用法

kubernetes又将日志相关的操作放到了`k8s.io/component-base/logs`库里面：

cmd/kubelet/kubelet.go

```go
package main

import (
	"math/rand"
	"os"
	"time"

	"k8s.io/component-base/logs"
	_ "k8s.io/component-base/logs/json/register" // for JSON log format registration
	_ "k8s.io/component-base/metrics/prometheus/restclient"
	_ "k8s.io/component-base/metrics/prometheus/version" // for version metric registration
	"k8s.io/kubernetes/cmd/kubelet/app"
)

func main() {
	rand.Seed(time.Now().UnixNano())

	command := app.NewKubeletCommand()
	logs.InitLogs()
	defer logs.FlushLogs()

	if err := command.Execute(); err != nil {
		os.Exit(1)
	}
}
```

其它的可以先不管，日志初始化主要是`logs.InitLogs()`

staging/src/k8s.io/component-base/logs/logs.go

```go
// KlogWriter serves as a bridge between the standard log package and the glog package.
type KlogWriter struct{}

// Write implements the io.Writer interface.
func (writer KlogWriter) Write(data []byte) (n int, err error) {
	klog.InfoDepth(1, string(data))
	return len(data), nil
}

// InitLogs initializes logs the way we want for kubernetes.
func InitLogs() {
	log.SetOutput(KlogWriter{})
	log.SetFlags(0)
	// The default glog flush interval is 5 seconds.
	go wait.Forever(klog.Flush, *logFlushFreq)
}
```

将标准库`log`的输出交给`KlogWriter`，其`Write()`方法简单调用了`klog.InfoDepth()`输出日志

```go
func init() {
	klog.InitFlags(flag.CommandLine)
}
```

同时`init()`方法在包导入的时候执行，加载日志相关的命令行参数。

## 继续canoe项目

在上篇的基础上增加日志模块，加在`pkg/component-base/logs`路径下：

```go
package logs

import (
	"flag"
	"log"
	"time"

	"github.com/spf13/pflag"
	"k8s.io/apimachinery/pkg/util/wait"
	"k8s.io/klog/v2"
)

const logFlushFreqFlagName = "log-flush-frequency"

var logFlushFreq = pflag.Duration(logFlushFreqFlagName, 5*time.Second, "Maximum number of seconds between log flushes")

func init() {
	klog.InitFlags(flag.CommandLine)
}

// KlogWriter serves as a bridge between the standard log package and the glog package.
type KlogWriter struct{}

// Write implements the io.Writer interface.
func (writer KlogWriter) Write(data []byte) (n int, err error) {
	klog.InfoDepth(1, string(data))
	return len(data), nil
}

// InitLogs initializes logs the way we want for kubernetes.
func InitLogs() {
	log.SetOutput(KlogWriter{})
	log.SetFlags(0)
	// The default glog flush interval is 5 seconds.
	go wait.Forever(klog.Flush, *logFlushFreq)
}

// FlushLogs flushes logs immediately.
func FlushLogs() {
	klog.Flush()
}
```

修改`cmd/server.go`，简单测试输出**hello world**

```go
package main

import (
	"flag"
	"math/rand"
	"time"

	"github.com/wgnc/canoe/pkg/component-base/logs"
	"k8s.io/klog/v2"
)

func main() {
	rand.Seed(time.Now().UnixNano())

	logs.InitLogs()
	defer logs.FlushLogs()

	flag.Parse()
	klog.Infof("hello world!")
}
```

测试执行：

```
➜  canoe go run cmd/server/server.go -h
Usage of /tmp/go-build3414547347/b001/exe/server:
  -add_dir_header
    	If true, adds the file directory to the header of the log messages
  -alsologtostderr
    	log to standard error as well as files
  -log_backtrace_at value
    	when logging hits line file:N, emit a stack trace
  -log_dir string
    	If non-empty, write log files in this directory
  -log_file string
    	If non-empty, use this log file
  -log_file_max_size uint
    	Defines the maximum size a log file can grow to. Unit is megabytes. If the value is 0, the maximum file size is unlimited. (default 1800)
  -logtostderr
    	log to standard error instead of files (default true)
  -one_output
    	If true, only write logs to their native severity level (vs also writing to each lower severity level)
  -skip_headers
    	If true, avoid header prefixes in the log messages
  -skip_log_headers
    	If true, avoid headers when opening log files
  -stderrthreshold value
    	logs at or above this threshold go to stderr (default 2)
  -v value
    	number for the log level verbosity
  -vmodule value
    	comma-separated list of pattern=N settings for file-filtered logging
➜  canoe go run cmd/server/server.go   
I1018 14:32:04.028080  279442 server.go:19] hello world!

```

目前的目录结构如下所示：

```
➜  canoe tree
.
├── build
│   └── Makefile
├── cmd
│   └── server
│       ├── options
│       │   ├── options.go
│       │   └── validation.go
│       └── server.go
├── go.mod
├── go.sum
├── Makefile -> build/Makefile
├── pkg
│   ├── component-base
│   │   └── logs
│   │       └── logs.go
│   └── server
│       └── server.go
└── README.md

8 directories, 10 files

```

后续继续添加命令行参数解析模块。
