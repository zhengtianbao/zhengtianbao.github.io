---
layout: post
title: "Golang HTTP Client 请求 BIG-IP 负载均衡失效问题记录"
date: 2024-05-28 10:14:57 +0800
categories: ["2024"]
tags: [golang, nginx, BIG-IP]
---

## 问题发现

事情是这样的，我们的 prometheus 开启了 remote_write 功能，将指标数据推送到 kafka-adapter，然后转发给 kafka 交由后续服务消费处理。随着接入系统指标的增加，发现 kafka-adapter 负载过高，因此进行横向扩容，通过负载均衡将 prometheus 的请求进行转发。

在测试环境通过 nginx 配置功能测试正常后上生产环境，但是最后的验证阶段发现 promethues 的数据只会向一台 kafka-adapter 发送数据，唯一的区别是生产环境是通过 BIG-IP 做负载均衡配置。在我的认知里 BIG-IP 只会更加可靠，为什么会出现这样的现象呢？

## 排查步骤

### BIG-IP 配置出错？

我们首先怀疑是 BIG-IP 的配置问题，因此先用 go 编写一个服务，功能很简单，返回给客户端本机的 IP 地址，这样通过客户端不断发送请求拿到的响应信息就能判断负载均衡是否正常工作。

```go
package main

import (
    "fmt"
    "net"
    "net/http"
)

// GetLocalIP returns the non loopback local IP of the host
func GetLocalIP() string {
    addrs, err := net.InterfaceAddrs()
    if err != nil {
        return ""
    }
    for _, address := range addrs {
        // check the address type and if it is not a loopback the display it
        if ipnet, ok := address.(*net.IPNet); ok && !ipnet.IP.IsLoopback() {
            if ipnet.IP.To4() != nil {
                return ipnet.IP.String()
            }
        }
    }
    return ""
}

func Index(w http.ResponseWriter, r *http.Request) {
    fmt.Println("receive a request from:", r.RemoteAddr, r.Header)
    w.Write([]byte(GetLocalIP()))
}

func main() {
    var s = http.Server{
        Addr:    ":18088",
        Handler: http.HandlerFunc(Index),
    }
    s.ListenAndServe()
}
```

分别在 192.168.56.1， 192.168.56.20 两台服务器上启动 server 服务。

配置 BIG-IP，创建一个测试 Pool，将上面两个服务地址作为 Member 添加到 Pool 中，Load Balancing Method 配置为 Round Robin，在 Virtual Server 的 Load Balancing Pool 中选择该 Pool。

```
client(192.168.56.1) -> BIG-IP(192.168.56.20:8080) -> server(192.168.56.1:18088)
                                     \--------------> server(192.168.56.20:18088)
```

通过 curl 命令，请求 BIG-IP 的 VIP 地址 192.168.56.20:8080：

```
$ curl 192.168.56.20:8080
192.168.56.1
$ curl 192.168.56.20:8080
192.168.56.20
$ curl 192.168.56.20:8080
192.168.56.1
$ curl 192.168.56.20:8080
192.168.56.20
```

根据这个结果可以认为 BIG-IP 配置的负载均衡是正常的。因此保留同样的 BIG-IP 配置将 pool 中的 member 替换为 kafka-adapter，client 替换为 prometheus，理论上请求将会均衡分发到后端，但是事与愿违，出现了流量只在一台 kafka-adapter 的现象。

### nginx 为什么正常？

既然 BIG-IP 不正常，那为什么 nginx 却可以将流量均匀分发呢？

检查 nginx 负载均衡配置：

```
upstream server_list {
  server 192.168.56.1:18088 weight=1;
  server 192.168.56.20:18088 weight=1;
}

server {
  listen 9099;
  server_name localhost;

  location / {
    proxy_pass http://server_list;
  }
}
```

实在是很简单的配置，看不出啥问题。

### 从源码分析

现在只能从 prometheus 源码入手，看它是如何发送请求的，毕竟在「源码面前，了无秘密」。

prometheus remote_write 部分代码在：`storage/remote/client.go`

```go
// Store sends a batch of samples to the HTTP endpoint, the request is the proto marshalled
// and encoded bytes from codec.go.
func (c *Client) Store(ctx context.Context, req []byte) error {
	httpReq, err := http.NewRequest("POST", c.url.String(), bytes.NewReader(req))
	if err != nil {
		// Errors from NewRequest are from unparsable URLs, so are not
		// recoverable.
		return err
	}
	httpReq.Header.Add("Content-Encoding", "snappy")
	httpReq.Header.Set("Content-Type", "application/x-protobuf")
	httpReq.Header.Set("User-Agent", userAgent)
	httpReq.Header.Set("X-Prometheus-Remote-Write-Version", "0.1.0")

	ctx, cancel := context.WithTimeout(ctx, c.timeout)
	defer cancel()

	httpResp, err := c.client.Do(httpReq.WithContext(ctx))
	if err != nil {
		// Errors from client.Do are from (for example) network errors, so are
		// recoverable.
		return recoverableError{err}
	}
	defer func() {
		io.Copy(ioutil.Discard, httpResp.Body)
		httpResp.Body.Close()
	}()

	if httpResp.StatusCode/100 != 2 {
		scanner := bufio.NewScanner(io.LimitReader(httpResp.Body, maxErrMsgLen))
		line := ""
		if scanner.Scan() {
			line = scanner.Text()
		}
		err = errors.Errorf("server returned HTTP status %s: %s", httpResp.Status, line)
	}
	if httpResp.StatusCode/100 == 5 {
		return recoverableError{err}
	}
	return err
}
```

可以看到 `Store()` 方法只是设置了一些 Header，调用 `http.Client` 的 `do()` 方法发送 HTTP POST 请求。

抽取同样的逻辑，编写一个简单的 client 代码，间隔 1s 向 BIG-IP 的 VIP 发送一条请求：

```go
import (
    "fmt"
    "time"
    "io/ioutil"
    "net/http"
)

func main() {
    c := &http.Client{}
    req, err := http.NewRequest(http.MethodGet, "http://192.168.56.20:8080", nil)
    if err != nil {
        panic(err)
    }
    fmt.Printf("%#v\n", *req)

    for {
        resp, err := c.Do(req)
        if err != nil {
            fmt.Println("http get error:", err)
            return
        }
        defer resp.Body.Close()

        b, err := ioutil.ReadAll(resp.Body)
        if err != nil {
            fmt.Println("read body error:", err)
            return
        }
        fmt.Println("response body:", string(b))
        time.Sleep(1 * time.Second)
    }
}
```

请求结果：

```
$ ./client
http.Request{Method:"GET", URL:(*url.URL)(0xc0000c0000), Proto:"HTTP/1.1", ProtoMajor:1, ProtoMinor:1, Header:http.Header{}, Body:io.ReadCloser(nil), GetBody:(func() (io.ReadCloser, error))(nil), ContentLength:0, TransferEncoding:[]string(nil), Close:false, Host:"192.168.56.20:8080", Form:url.Values(nil), PostForm:url.Values(nil), MultipartForm:(*multipart.Form)(nil), Trailer:http.Header(nil), RemoteAddr:"", RequestURI:"", TLS:(*tls.ConnectionState)(nil), Cancel:(<-chan struct {})(nil), Response:(*http.Response)(nil), ctx:context.backgroundCtx{emptyCtx:context.emptyCtx{}}}
response body: 192.168.56.1
response body: 192.168.56.1
response body: 192.168.56.1
^Csignal: interrupt
```

居然复现了，所有的请求经过 BIG-IP 后都只落到一台后端服务上。

## 问题分析

### keep-alive

go 语言中 `http.Client` 底层的数据连接建立和维护是由 `http.Transport` 实现的，`http.Transport` 结构有一个 `DisableKeepAlives` 字段，其默认值为 `false`，即启动 `keep-alive`。这里我们将其置为 `true`，即关闭 `keep-alive`，然后将该 `Transport` 实例作为初值，赋值给 `http Client` 实例的 `Transport` 字段。

```go
func main() {
    tr := &http.Transport{
        DisableKeepAlives: true,
    }
    c := &http.Client{
        Transport: tr,
    }
    // ...
}
```

设置为 `true`，将会在 HTTP header 包含 `Connection: "close"`，可以通过 server 的日志看到请求头：

```
receive a request from: 192.168.56.20:60360 map[Accept-Encoding:[gzip] Connection:[close] User-Agent:[Go-http-client/1.1]]
receive a request from: 192.168.56.20:60376 map[Accept-Encoding:[gzip] Connection:[close] User-Agent:[Go-http-client/1.1]]
receive a request from: 192.168.56.20:53602 map[Accept-Encoding:[gzip] Connection:[close] User-Agent:[Go-http-client/1.1]]
receive a request from: 192.168.56.20:53604 map[Accept-Encoding:[gzip] Connection:[close] User-Agent:[Go-http-client/1.1]]
```

这里还有一个值得注意的点是接受到请求的源端口每次都是不同的，意味着每次请求都会建立一次连接。

这就解释了为什么 nginx 可以负载均衡，因为 nginx 配置的是 7 层负载均衡，会在转发过程中在 header 里自动添加 `Connection: "close"`，因此即使 go 中 `Transport` 没有配置 `DisableKeepAlives: true`，但是行为是每次都会断开重新建立连接。

那 BIG-IP 为什么不行，因为我们默认配置的 Virtual Server Type 选择的是 TCP，只在 4 层进行转发，因为这条 HTTP 连接一直没有断，可以理解为只转发了一条请求。这也解释了之前通过 curl 命令测试却能均匀分发的原因，因为执行 curl 命令每次都是从新的客户端端口建立的新请求，自然能否负载均衡到不同的后端。

### HTTP 协议规范

最初早期的 HTTP 1.0 协议只支持短连接，即客户端每发送一个请求，就要和服务器端建立一个新 TCP 连接，请求处理完毕后，该连接将被拆除。显然每次 TCP 连接握手和拆除都将带来较大损耗，为了能充分利用已建立的连接，后来的 HTTP 1.0 更新版和 HTTP 1.1 支持在 HTTP 请求头中加入 Connection: keep-alive 来告诉对方这个请求响应完成后不要关闭链接，下一次还要复用这个连接以继续传输后续请求和响应。后 HTTP 协议规范明确规定了 HTTP/1.0 版本如果想要保持长连接，需要在请求头中加上 Connection: keep-alive，而 HTTP/1.1 版本将支持 keep-alive 长连接作为默认选项，有没有这个请求头都可以。

go 语言 HTTP 包的 `http.Server` 和 `http.Client` 的实现默认将所有连接视为长连接，不会关闭连接。那 `http.Server` 如何关闭闲置超时的连接呢？

```go
    var s = http.Server{
        Addr:        ":8080",
        Handler:     http.HandlerFunc(Index),
        IdleTimeout: 5 * time.Second,
    }
```

可以通过 `IdleTimeout` 参数指定。

完。
