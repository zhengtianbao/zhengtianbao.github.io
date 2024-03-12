---
layout: post
title: "wsgiref threading usage"
date: 2014-10-30 20:46:53
categories: Python
---

<div width="1024" height="512" align="center">
![wsgi_threading_usage](/images/wsgi_threading_usage.jpeg)</div>

本文记录了在使用 wsgiref 的 WSGIServer 时碰到的并发请求阻塞问题，事情的起因是在一个忙碌的周四接到个任务，需求是要模拟编写一个支付宝接口用来测试现有系统的支付流程完整性。

现有的系统架构如下：

* 后端系统A：提供 WSGI 的 API 接口，通过消息队列对外发送消息或者接受其他组件的消息，执行周期任务等
* 前端系统B：调用后端系统A的 API 接口，接收后端系统A发送的消息进行处理，UI 展示层

支付宝作为一个第三方应用，设计上只能与前端B产生交互，支付宝的支付流程如下：

1. 前端系统B通过某个订单的 id 向后端系统A找到对应的支付信息：订单号，金额，partner_id，支付成功返回通知地址，加密后的 key 等 
2. 前端系统B组装这些信息发送 GET 请求，带上相关参数发送到支付宝的指定的 API 地址
3. 支付宝支付完成后会发送一个通知给 notify_url，当然这个通知是通过 POST 请求形式发送给前端系统B的
4. 前端系统B收到通知后需要调用后端系统A的 API 接口告诉它支付完成，更改这个订单的状态以及其他操作

好了，我要实现的 fakepay 需要满足的就是 2，3 两步：

* 提供一个 API 地址，在收到前端系统B第 2 步提交支付请求的时候模拟已经付款成功，然后发送通知（POST）给 notify_url（这个 URL 肯定是前端B的一个接口）

功能很简单，看样子也用不了几行代码，于是我在后端系统A现有的 API 接口上新增了一个 */fakepay* 的 API 接口，大概的实现如下:

```python
def fakepay():
    log('fakepay successful')
    do_something()
    # POST notify_url
    try:
        do_request(notify_url，'POST'，body)
    except Timeout:
        return False
    return True
```

注意 `do_request`，它会阻塞直到整个 HTTP 请求正确结束，一个请求总是会阻塞直到收到 *response*。

考虑下整个流程:

1. 前端系统B请求后端系统A的 fakepay 接口
2. fakepay 接口请求前端系统B的 notify_url，线程阻塞等待前端系统B返回
3. 前端系统B接受到 fakepay 发送的支付成功 POST 通知，发送请求给后端系统A，等待后端系统A处理订单结束，返回 response 给 fakepay
4. fakepay 收到 response，继续向下执行返回 response 给前端系统B
5. 支付流程结束

事实上，`do_request` 应该放在另外一个线程里做才比较合理:

```python
import threading

def notify():
    do_request(notify_url，'POST'，body)

def fakepay():
    do_something()
    t = threading.Thread(target=notify)
    t.daemon = False
    t.start()
    return True
```

就像上面这样采用异步的方式，避免了 fakepay 的等待。但是，这与我概念中的 server 为每一条 HTTP 连接都会分配一个线程的概念不符，因为即使在 fakepay 的线程中阻塞住了，server 应该还是能够处理其他 API 请求的. 然而坑爹的就是我们的 *application* 跑在 wsgiref 的 WSGIServer 上。

研究了下 wsgiref 的源码实现：

```
wsgiref.WSGIServer --> BaseHTTPServer.HTTPServer --> SocketServer.TCPServer
```

TCPServer 里有这样的注释：

```
    # The distinction between handling，getting，processing and
    # finishing a request is fairly arbitrary.  Remember:
    #
    # - handle_request() is the top-level call.  It calls
    #   select，get_request()，verify_request() and process_request()
    # - get_request() is different for stream or datagram sockets
    # - process_request() is the place that may fork a new process
    #   or create a new thread to finish the request
    # - finish_request() instantiates the request handler class;
    #   this constructor will handle the request all by itself
```

假设 `socket` 收到第一个请求（*select socketfd*），进行请求一的 `process_request` 需要 10 秒，由于它是与 *server* 处在同一线程，如果当在第 2 秒收到第二个请求时，*server* 会阻塞在处理第一个请求处，因此无法及时处理第二个请求（这个时候第一个请求与第二个请求的 *client* 端都在等待 *server* 的响应），10 秒后第一个请求处理完毕，*server* 线程继续执行到 *select socketfd*，获取到第二个请求，再进行 `process_request`，处理第二个请求，完毕后继续等待新连接请求（*select socketfd*）。

也就是说通过 `select` `accept` 到一个http连接后会交由 `process_request` 处理，例如上面的 `fakepay`，如果在 `process_request` 过程中阻塞了就不会
到下一个 `select` 了，所以 TCPServer 的 `process_request` 必须是 **non-blocking** 的。

由此 `select()->get_request()->process_request()` 所谓的非阻塞，是指在 `select` 阶段的 I/O 多路复用，但是在 `process_request` 阶段要是 block 住了，就回不到 `select` 了。所以，如果 `process_request` 是需要花费一定时间的 block 任务，就要放到子进程或者线程里去处理，不要影响主进程的执行。

因为 wsgiref.WSGIServer 继承自 SocketServer.TCPServer，因此它是单线程执行的，*wsgi application* 必须是非阻塞的，只有处理完一条请求才能执行下一条。

所幸 SocketServer 中提供了 ThreadingTCPServer，在 `process_request` 时采用 threading，这样每个请求都开了个独立的线程就不会阻塞了。

只需要继承它就可以作为改良版的WSGIServer了：

```python
import socket
import SocketServer

class WSGIServer(SocketServer.ThreadingTCPServer):

    """BaseHTTPServer that implements the Python WSGI protocol"""

    application = None

    def server_bind(self):
        """Override server_bind to store the server name."""
        SocketServer.TCPServer.server_bind(self)
        host，port = self.socket.getsockname()[:2]
        self.server_name = socket.getfqdn(host)
        self.server_port = port
        self.setup_environ()

    def setup_environ(self):
        # Set up base environment
        env = self.base_environ = {}
        env['SERVER_NAME'] = self.server_name
        env['GATEWAY_INTERFACE'] = 'CGI/1.1'
        env['SERVER_PORT'] = str(self.server_port)
        env['REMOTE_HOST']=''
        env['CONTENT_LENGTH']=''
        env['SCRIPT_NAME'] = ''

    def get_app(self):
        return self.application

    def set_app(self,application):
        self.application = application
```

当然，可以也选择 eventlet 库，nova-api 就使用它作为 wsgi server。

PS：用之前 `monkey_patch` 一下，green the world! (ゝ∀･)
