---
title: "kubernetes cookbook 之命令行解析篇"
date: 2021-10-18 19:25:29
categories: ["2021"]
tags: [kubernetes]
---

命令行解析几乎是所有程序的标准功能，go 语言标准库中提供了 `flag` 模块，而 kubernetes 中则使用了 `pflag` 和 `cobra` 来构建。

## flag 标准库

`flag` 包内置了常用的几种参数类型：`string`，`int`，`bool`，`time.Duration`，如果需要自定义类型，例如以 `,` 分割的字符串数组，就需要实现 `flag.Value` 接口。

### 基本类型

基本的使用方法

```go
flagPtr := flag.String("<identifier>", "<defaultOutput>", "<help message>")
```

或者

```go
var flagvar string
func init() {
  flag.StringVar(&flagvar, "<identifier>", "<defaultOutput>", "<help message>")
}
```

例如coffee.go：

```go
package main

import (
    "flag"
    "fmt"
)

func main() {
    // 返回值为指针类型
    wordPtr := flag.String("flavor", "vanilla", "select shot flavor")
    numbPtr := flag.Int("quantity", 2, "quantity of shots")
    boolPtr := flag.Bool("cream", false, "decide if you want cream")

    var order string
    // 通过传递地址来修改变量
    flag.StringVar(&order, "order", "complete", "status of order")
	// 解析命令行参数
    flag.Parse()

    fmt.Println("flavor:", *wordPtr)
    fmt.Println("quantity:", *numbPtr)
    fmt.Println("cream:", *boolPtr)
    fmt.Println("order:", order)
    fmt.Println("tail:", flag.Args())
}
```

```
$ ./coffee -flavor=chocolate -cream -order=incomplete
flavor: chocolate
quantity: 2
cream: true
order: incomplete
tail: []
$ ./coffee -flavor=chocolate -cream -order=incomplete -flag1 -flag2=true
flavor: chocolate
quantity: 2
cream: true
order: incomplete
tail: [flag1 flag2=true]
```

### 子命令

考虑实现以下命令：

```
$ siri send -recipient=john@example.com -message="Call me?" 
$ siri ask -question="What is the whether in London?"
```

`send` 和 `ask` 都是 `siri` 的子命令，通过 `flag.NewFlagSet` 实现子命令

siri.go：

```go
package main

import (
	"flag"
	"fmt"
	"os"
)

func main() {
	askCommand := flag.NewFlagSet("ask", flag.ExitOnError)
	questionFlag := askCommand.String("question", "", "Question that you are asking for.")

	sendCommand := flag.NewFlagSet("send", flag.ExitOnError)
	recipientFlag := sendCommand.String("recipient", "", "Recipient of your message.")
	messageFlag := sendCommand.String("message", "", "Text message.")

	if len(os.Args) == 1 {
		fmt.Println("usage: siri <command> [<args>]")
		fmt.Println("The most commonly used git commands are: ")
		fmt.Println(" ask   Ask questions")
		fmt.Println(" send  Send messages to your contacts")
		return
	}

	switch os.Args[1] {
	case "ask":
		askCommand.Parse(os.Args[2:])
	case "send":
		sendCommand.Parse(os.Args[2:])
	default:
		fmt.Printf("%q is not valid command.\n", os.Args[1])
		os.Exit(2)
	}

	if askCommand.Parsed() {
		if *questionFlag == "" {
			fmt.Println("Please supply the question using -question option.")
			return
		}
		fmt.Printf("You asked: %q\n", *questionFlag)
	}

	if sendCommand.Parsed() {
		if *recipientFlag == "" {
			fmt.Println("Please supply the recipient using -recipient option.")
			return
		}

		if *messageFlag == "" {
			fmt.Println("Please supply the message using -message option.")
			return
		}

		fmt.Printf("Your message is sent to %q.\n", *recipientFlag)
		fmt.Printf("Message: %q.\n", *messageFlag)
	}
}
```

### 自定义参数类型

kubernetes proxy 命令行参数中有个 `etcd_servers` 的选项，因为 `etcd` 以集群方式部署，使用的时候可能会如下：

```
$ proxy -etcd_servers http://192.168.100.10:4001,http://192.168.100.11:4001,http://192.168.100.12:4001
```

很显然 `etcd_servers` 是 `net/url` 包里面 `url.URL` 类型组成的数组，不在 `flag` 包默认支持的类型范围内。

cmd/proxy/proxy.go#L32

```go
var (
	configFile     = flag.String("configfile", "/tmp/proxy_config", "Configuration file for the proxy")
	master         = flag.String("master", "", "The address of the Kubernetes API server (optional)")
	etcdServerList util.StringList
)

func init() {
	flag.Var(&etcdServerList, "etcd_servers", "List of etcd servers to watch (http://ip:port), comma separated (optional)")
}

func main() {
	flag.Parse()
    //...   
}
```

`flag.Var` 函数的第一个参数是 `flag.Value` 接口类型，需要实现 `String() string`，`Set(string) error` 两个方法。

go/src/flag/flag.go

```go
// Value is the interface to the dynamic value stored in a flag.
// (The default value is represented as a string.)
//
// If a Value has an IsBoolFlag() bool method returning true,
// the command-line parser makes -name equivalent to -name=true
// rather than using the next command-line argument.
//
// Set is called once, in command line order, for each flag present.
// The flag package may call the String method with a zero-valued receiver,
// such as a nil pointer.
type Value interface {
	String() string
	Set(string) error
}

// Var defines a flag with the specified name and usage string. The type and
// value of the flag are represented by the first argument, of type Value, which
// typically holds a user-defined implementation of Value. For instance, the
// caller could create a flag that turns a comma-separated string into a slice
// of strings by giving the slice the methods of Value; in particular, Set would
// decompose the comma-separated string into the slice.
func Var(value Value, name string, usage string) {
	CommandLine.Var(value, name, usage)
}
```

pkg/util/list.go#L24

```go
type StringList []string

func (sl *StringList) String() string {
	return fmt.Sprint(*sl)
}

func (sl *StringList) Set(value string) error {
	for _, s := range strings.Split(value, ",") {
		if len(s) == 0 {
			return fmt.Errorf("value should not be an empty string")
		}
		*sl = append(*sl, s)
	}
	return nil
}
```

`String()` 将 struct 转化为 string，`Set(string)` 将会在 `flag.Parse()` 方法执行时被调用，`StringList` 实现了接口 `flag.Value`，将参数根据 `,` 分割后，赋值给 `etcdServerList`。

结合起来的例子：custom_flag.go

```go
package main

import (
	"flag"
	"fmt"
	"strings"
)

type StringList []string

func (sl *StringList) String() string {
	return fmt.Sprint(*sl)
}

func (sl *StringList) Set(value string) error {
	for _, s := range strings.Split(value, ",") {
		if len(s) == 0 {
			return fmt.Errorf("value should not be an empty string")
		}
		*sl = append(*sl, s)
	}
	return nil
}

func main() {
	var etcdServerList StringList
	flag.Var(&etcdServerList, "etcd_servers", "List of etcd servers to watch (http://ip:port), comma separated (optional)")
	flag.Parse()

	for i, item := range etcdServerList {
		fmt.Printf("etcd server %d: %s\n", i, item)
	}
}
```

```
$ go run custom_flag.go -h
Usage of /tmp/go-build1467972498/b001/exe/custom_flag:
  -etcd_servers value
    	List of etcd servers to watch (http://ip:port), comma separated (optional)
$ go run custom_flag.go -etcd_servers http://192.168.100.10:4001,http://192.168.100.11:4001,http://192.168.100.12:4001
etcd server 0: http://192.168.100.10:4001
etcd server 1: http://192.168.100.11:4001
etcd server 2: http://192.168.100.12:4001
```

## pflag

[pflag](https://github.com/spf13/pflag) 在 flag 的基础上补充了符合 posix 标准的命令行解析规范，下面简单就 kubernetes 用到的功能来举例说明。


### 标记参数废弃

cmd/kubelet/app/options/options.go

```go
// AddFlags adds flags for a specific KubeletFlags to the specified FlagSet
func (f *KubeletFlags) AddFlags(mainfs *pflag.FlagSet) {
	fs := pflag.NewFlagSet("", pflag.ExitOnError) // 跟flag一样创建子命令
	defer func() {
		// Unhide deprecated flags. We want deprecated flags to show in Kubelet help.
		// We have some hidden flags, but we might as well unhide these when they are deprecated,
		// as silently deprecating and removing (even hidden) things is unkind to people who use them.
		fs.VisitAll(func(f *pflag.Flag) { // 遍历visit调用
			if len(f.Deprecated) > 0 {
				f.Hidden = false
			}
		})
		mainfs.AddFlagSet(fs) // 主flagset增加子flagset
	}()

	f.ContainerRuntimeOptions.AddFlags(fs)
	f.addOSFlags(fs)

	fs.StringVar(&f.KubeletConfigFile, "config", f.KubeletConfigFile, "The Kubelet will load its initial configuration from this file. The path may be absolute or relative; relative paths start at the Kubelet's current working directory. Omit this flag to use the built-in default configuration values. Command-line flags override configuration from this file.")
	fs.StringVar(&f.KubeConfig, "kubeconfig", f.KubeConfig, "Path to a kubeconfig file, specifying how to connect to the API server. Providing --kubeconfig enables API server mode, omitting --kubeconfig enables standalone mode.")
	// ...
	// DEPRECATED FLAGS
	fs.DurationVar(&f.MinimumGCAge.Duration, "minimum-container-ttl-duration", f.MinimumGCAge.Duration, "Minimum age for a finished container before it is garbage collected.  Examples: '300ms', '10s' or '2h45m'")
	fs.MarkDeprecated("minimum-container-ttl-duration", "Use --eviction-hard or --eviction-soft instead. Will be removed in a future version.")
	// ...
}
```

### 参数重写

例如希望参数使用「_」和「-」分隔符表现一致，像 `--my_flag == --my-flag`：

```go
// WordSepNormalizeFunc changes all flags that contain "_" separators
func WordSepNormalizeFunc(f *pflag.FlagSet, name string) pflag.NormalizedName {
	if strings.Contains(name, "_") {
		return pflag.NormalizedName(strings.Replace(name, "_", "-", -1))
	}
	return pflag.NormalizedName(name)
}

cleanFlagSet.SetNormalizeFunc(WordSepNormalizeFunc)
```

## cobra

[cobra](https://github.com/spf13/cobra) 能够快速的创建 CLI 接口的应用程序，kubernetes 的组件都已经迁移到使用该库作为程序的启动入口。

cmd/kubelet/kubelet.go

```go
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

cmd/kubelet/app/server.go

```go
const (
	// Kubelet component name
	componentKubelet = "kubelet"
)

// NewKubeletCommand creates a *cobra.Command object with default parameters
func NewKubeletCommand() *cobra.Command {
	cleanFlagSet := pflag.NewFlagSet(componentKubelet, pflag.ContinueOnError)
	cleanFlagSet.SetNormalizeFunc(cliflag.WordSepNormalizeFunc)
	kubeletFlags := options.NewKubeletFlags()
	kubeletConfig, err := options.NewKubeletConfiguration()
	// programmer error
	if err != nil {
		klog.ErrorS(err, "Failed to create a new kubelet configuration")
		os.Exit(1)
	}
	
	cmd := &cobra.Command{
		Use: componentKubelet,
		Long: `The kubelet is the primary "node agent" that runs on each
node. It can register the node with the apiserver using one of: the hostname; a flag to
override the hostname; or specific logic for a cloud provider.`,
		// The Kubelet has special flag parsing requirements to enforce flag precedence rules,
		// so we do all our parsing manually in Run, below.
		// DisableFlagParsing=true provides the full set of flags passed to the kubelet in the
		// `args` arg to Run, without Cobra's interference.
		DisableFlagParsing: true,
		Run: func(cmd *cobra.Command, args []string) {
			// initial flag parse, since we disable cobra's flag parsing
			if err := cleanFlagSet.Parse(args); err != nil {
				klog.ErrorS(err, "Failed to parse kubelet flag")
				cmd.Usage()
				os.Exit(1)
			}
			// ...
			// run the kubelet
			if err := Run(ctx, kubeletServer, kubeletDeps, utilfeature.DefaultFeatureGate); err != nil {
				klog.ErrorS(err, "Failed to run kubelet")
				os.Exit(1)
			}
		},
	}
	cleanFlagSet.BoolP("help", "h", false, fmt.Sprintf("help for %s", cmd.Name()))

	// ugly, but necessary, because Cobra's default UsageFunc and HelpFunc pollute the flagset with global flags
	const usageFmt = "Usage:\n  %s\n\nFlags:\n%s"
	cmd.SetUsageFunc(func(cmd *cobra.Command) error {
		fmt.Fprintf(cmd.OutOrStderr(), usageFmt, cmd.UseLine(), cleanFlagSet.FlagUsagesWrapped(2))
		return nil
	})
	cmd.SetHelpFunc(func(cmd *cobra.Command, args []string) {
		fmt.Fprintf(cmd.OutOrStdout(), "%s\n\n"+usageFmt, cmd.Long, cmd.UseLine(), cleanFlagSet.FlagUsagesWrapped(2))
	})

	return cmd
}
```

`cmd.Run` 定义了一个函数用来做命令行注册解析，参数校验等一系列操作，最后调用 `Run(ctx, kubeletServer, kubeletDeps, utilfeature.DefaultFeatureGate)` 来真正启动 kubelet。`cmd.Run` 定义的函数在启动过程中被 `command.Execute()` 调用。

## 继续 canoe 项目

上文已经增加了日志，这次接着增加服务启动方法

cmd/server/server.go

```go
package main

import (
	"math/rand"
	"os"
	"time"

	"github.com/wgnc/canoe/cmd/server/app"
	"github.com/wgnc/canoe/pkg/component-base/logs"
)

func main() {
	rand.Seed(time.Now().UnixNano())

	command := app.NewServerCommand()
	logs.InitLogs()
	defer logs.FlushLogs()

	if err := command.Execute(); err != nil {
		os.Exit(1)
	}
}
```

cmd/server/app/server.go

```go
package app

import (
	"context"
	"fmt"
	"os"

	"github.com/spf13/cobra"
	"github.com/spf13/pflag"
	"github.com/wgnc/canoe/cmd/server/app/options"
	cliflag "github.com/wgnc/canoe/pkg/component-base/cli/flag"
	"github.com/wgnc/canoe/pkg/component-base/version/verflag"
	"k8s.io/klog/v2"
)

const (
	componentName = "canoe"
)

func NewServerCommand() *cobra.Command {
	cleanFlagSet := pflag.NewFlagSet(componentName, pflag.ContinueOnError)
	cleanFlagSet.SetNormalizeFunc(cliflag.WordSepNormalizeFunc)

	canoeFlags, err := options.NewCanoeFlags()
	// programmer error
	if err != nil {
		klog.ErrorS(err, "Failed to create a new kubelet configuration")
		os.Exit(1)
	}

	cmd := &cobra.Command{
		Use: componentName,
		Long: `The Canoe is a simple http Server.
		
HTTP endpoint: HTTP endpoint passed as a parameter on the command line. This endpoint
is checked every 20 seconds (also configurable with a flag).`,
		DisableFlagParsing: true,
		Run: func(cmd *cobra.Command, args []string) {
			// initial flag parse, since we disable cobra's flag parsing
			if err := cleanFlagSet.Parse(args); err != nil {
				klog.ErrorS(err, "Failed to parse kubelet flag")
				cmd.Usage()
				os.Exit(1)
			}

			// check if there are non-flag arguments in the command line
			cmds := cleanFlagSet.Args()
			if len(cmds) > 0 {
				klog.ErrorS(nil, "Unknown command", "command", cmds[0])
				cmd.Usage()
				os.Exit(1)
			}

			// short-circuit on help
			help, err := cleanFlagSet.GetBool("help")
			if err != nil {
				klog.InfoS(`"help" flag is non-bool, programmer error, please correct`)
				os.Exit(1)
			}
			if help {
				cmd.Help()
				return
			}

			// short-circuit on verflag
			verflag.PrintAndExitIfRequested()
			cliflag.PrintFlags(cleanFlagSet)

			// validate the initial KubeletFlags
			if err := options.ValidateCanoeFlags(canoeFlags); err != nil {
				klog.ErrorS(err, "Failed to validate canoe flags")
				os.Exit(1)
			}

			// run the server
			if err := Run(canoeFlags); err != nil {
				klog.ErrorS(err, "Failed to run canoe")
				os.Exit(1)
			}
		},
	}

	// keep cleanFlagSet separate, so Cobra doesn't pollute it with the global flags
	canoeFlags.AddFlags(cleanFlagSet)
	options.AddGlobalFlags(cleanFlagSet)
	cleanFlagSet.BoolP("help", "h", false, fmt.Sprintf("help for %s", cmd.Name()))

	// ugly, but necessary, because Cobra's default UsageFunc and HelpFunc pollute the flagset with global flags
	const usageFmt = "Usage:\n  %s\n\nFlags:\n%s"
	cmd.SetUsageFunc(func(cmd *cobra.Command) error {
		fmt.Fprintf(cmd.OutOrStderr(), usageFmt, cmd.UseLine(), cleanFlagSet.FlagUsagesWrapped(2))
		return nil
	})
	cmd.SetHelpFunc(func(cmd *cobra.Command, args []string) {
		fmt.Fprintf(cmd.OutOrStdout(), "%s\n\n"+usageFmt, cmd.Long, cmd.UseLine(), cleanFlagSet.FlagUsagesWrapped(2))
	})

	return cmd
}

func Run(s *options.CanoeFlags) error {
	fmt.Printf("hello %s\n", s.HostnameOverride)
	return nil
}

```

cmd/server/app/options/option.go

```go
package options

import (
	"fmt"
	_ "net/http/pprof" // Enable pprof HTTP handlers.

	"github.com/spf13/pflag"
)

type CanoeFlags struct {
	// HostnameOverride is the hostname used to identify the kubelet instead
	// of the actual hostname.
	HostnameOverride string

	// cloudProvider is the provider for cloud services.
	CloudProvider string
}

func NewCanoeFlags() (*CanoeFlags, error) {
	return &CanoeFlags{}, nil
}

// AddFlags adds flags for a specific KubeletFlags to the specified FlagSet
func (f *CanoeFlags) AddFlags(mainfs *pflag.FlagSet) {
	fs := pflag.NewFlagSet("", pflag.ExitOnError)
	defer func() {
		// Unhide deprecated flags. We want deprecated flags to show in Kubelet help.
		// We have some hidden flags, but we might as well unhide these when they are deprecated,
		// as silently deprecating and removing (even hidden) things is unkind to people who use them.
		fs.VisitAll(func(f *pflag.Flag) {
			if len(f.Deprecated) > 0 {
				f.Hidden = false
			}
		})
		mainfs.AddFlagSet(fs)
	}()

	fs.StringVar(&f.HostnameOverride, "hostname-override", f.HostnameOverride, "If non-empty, will use this string as identification instead of the actual hostname. If --cloud-provider is set, the cloud provider determines the name of the node (consult cloud provider documentation to determine if and how the hostname is used).")

	// DEPRECATED FLAGS
	fs.StringVar(&f.CloudProvider, "cloud-provider", f.CloudProvider, "The provider for cloud services. Set to empty string for running with no cloud provider. If set, the cloud provider determines the name of the node (consult cloud provider documentation to determine if and how the hostname is used).")
	fs.MarkDeprecated("cloud-provider", "will be removed in 1.23, in favor of removing cloud provider code from Kubelet.")

}

// ValidateKubeletFlags validates Canoe's configuration flags and returns an error if they are invalid.
func ValidateCanoeFlags(f *CanoeFlags) error {
	if f.HostnameOverride == "localhost" {
		return fmt.Errorf("the hostnameOverride must not be localhost")
	}

	return nil
}
```

好了，尝试启动执行下看看：

```
➜  canoe go run cmd/server/server.go -h       
The Canoe is a simple http Server.
		
HTTP endpoint: HTTP endpoint passed as a parameter on the command line. This endpoint
is checked every 20 seconds (also configurable with a flag).

Usage:
  canoe [flags]

Flags:
      --add-dir-header                   If true, adds the file directory to the header of the log messages
      --alsologtostderr                  log to standard error as well as files
      --cloud-provider string            The provider for cloud services. Set to empty string for running with no cloud provider. If set, the cloud provider determines the name of the node (consult cloud provider documentation to determine if and how the hostname is used). (DEPRECATED: will be removed in 1.23, in favor of removing cloud provider code from Kubelet.)
  -h, --help                             help for canoe
      --hostname-override string         If non-empty, will use this string as identification instead of the actual hostname. If --cloud-provider is set, the cloud provider determines the name of the node (consult cloud provider documentation to determine if and how the hostname is used).
      --log-backtrace-at traceLocation   when logging hits line file:N, emit a stack trace (default :0)
      --log-dir string                   If non-empty, write log files in this directory
      --log-file string                  If non-empty, use this log file
      --log-file-max-size uint           Defines the maximum size a log file can grow to. Unit is megabytes. If the value is 0, the maximum file size is unlimited. (default 1800)
      --log-flush-frequency duration     Maximum number of seconds between log flushes (default 5s)
      --logtostderr                      log to standard error instead of files (default true)
      --one-output                       If true, only write logs to their native severity level (vs also writing to each lower severity level)
      --skip-headers                     If true, avoid header prefixes in the log messages
      --skip-log-headers                 If true, avoid headers when opening log files
      --stderrthreshold severity         logs at or above this threshold go to stderr (default 2)
  -v, --v Level                          number for the log level verbosity
      --version version[=true]           Print version information and quit
      --vmodule moduleSpec               comma-separated list of pattern=N settings for file-filtered logging
➜  canoe go run cmd/server/server.go --hostname-override localhost
E1019 14:04:39.606410  297750 server.go:72] "Failed to validate canoe flags" err="the hostnameOverride must not be localhost"
exit status 1
➜  canoe go run cmd/server/server.go --hostname-override node1    
hello node1
➜  canoe go run cmd/server/server.go --hostname-override node2 -v3
I1019 14:12:05.992992  298465 flags.go:32] FLAG: --add-dir-header="false"
I1019 14:12:05.993045  298465 flags.go:32] FLAG: --alsologtostderr="false"
I1019 14:12:05.993086  298465 flags.go:32] FLAG: --cloud-provider=""
I1019 14:12:05.993099  298465 flags.go:32] FLAG: --help="false"
I1019 14:12:05.993110  298465 flags.go:32] FLAG: --hostname-override="node2"
I1019 14:12:05.993119  298465 flags.go:32] FLAG: --log-backtrace-at=":0"
I1019 14:12:05.993134  298465 flags.go:32] FLAG: --log-dir=""
I1019 14:12:05.993160  298465 flags.go:32] FLAG: --log-file=""
I1019 14:12:05.993170  298465 flags.go:32] FLAG: --log-file-max-size="1800"
I1019 14:12:05.993179  298465 flags.go:32] FLAG: --log-flush-frequency="5s"
I1019 14:12:05.993198  298465 flags.go:32] FLAG: --logtostderr="true"
I1019 14:12:05.993208  298465 flags.go:32] FLAG: --one-output="false"
I1019 14:12:05.993217  298465 flags.go:32] FLAG: --skip-headers="false"
I1019 14:12:05.993227  298465 flags.go:32] FLAG: --skip-log-headers="false"
I1019 14:12:05.993236  298465 flags.go:32] FLAG: --stderrthreshold="2"
I1019 14:12:05.993244  298465 flags.go:32] FLAG: --v="3"
I1019 14:12:05.993253  298465 flags.go:32] FLAG: --version="false"
I1019 14:12:05.993265  298465 flags.go:32] FLAG: --vmodule=""
hello node2

```

接下来将为 canoe 项目增加优雅退出处理。
