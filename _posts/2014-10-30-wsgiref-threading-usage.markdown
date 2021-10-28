---
layout: post
title:  "wsgiref threading usage"
date:   2014-10-30 20:46:53
categories: Python
---

事情的起因是在一个忙碌的周四, 接到个任务, 内容是模拟一个支付宝测试下现有系统的支付流程完整性. 
现有的系统架构如下:

* 后端系统A: 提供WSGI的API接口, 通过消息队列对外发送消息或者接受其他组件的消息, 执行周期任务等
* 前端系统B: 调用A的API接口, 接收A发送的消息进行处理, UI展示层

支付宝作为一个第三方应用, 设计上只能与前端B产生交互, 支付宝的支付流程如下:

1. 前端B通过某个订单的id向后端A找到对应的支付信息: 订单号, 金额, partner_id, 支付成功返回通知地址, 加密后的key等 
2. 前端B组装这些信息以GET请求参数的方式发送到支付宝的指定的API地址
3. * 支付宝支付完成后会发送一个通知给notify_url, 当然这个通知是POST形式发送给前端B的
4. 前端B收到通知后需要调用后端A的API接口告诉它支付完成, 更改这个订单的状态以及其他操作

好了, 我要实现的fakepay需要满足的就是2, 3两步:

* 提供一个API地址, 在收到前端B第2步提交支付请求的时候模拟已经付款成功, 然后发送通知(POST)给notify_url(这个URL肯定是前端B的一个接口)

功能很简单, 看样子也用不了几行代码, 于是我在后端系统A现有的API接口上新增了一个*/fakepay*的API接口, 大概的实现如下:

{% highlight python %}
def fakepay():
    log('fakepay successful')
    do_something()
    # POST notify_url
    try:
        do_request(notify_url, 'POST', body)
    except Timeout:
        return False
    return True
{% endhighlight %}

注意`do_request`, 它会阻塞直到整个HTTP请求正确结束, 一个请求总是会阻塞直到收到*response*.
考虑下整个流程:

1. 前端B请求后端A的fakepay接口
2. fakepay接口请求前端B的notify_url, 线程阻塞等待前端B返回
3. 前端B接受到fakepay发送的支付成功POST通知, 发送请求给后端A, 等待后端A处理订单结束, 返回response给fakepay
4. fakepay收到response, 继续向下执行返回response给前端B
5. 支付流程结束

事实上, `do_request`应该放在另外一个线程里做才比较合理:

{% highlight python %}
import threading

def notify():
    do_request(notify_url, 'POST', body)

def fakepay():
    do_something()
    t = threading.Thread(target=notify)
    t.daemon = False
    t.start()
    return True
{% endhighlight %}

就像上面这样采用异步的方式, 避免了fakepay的等待. 但是, 这与我概念中的server为每一条http连接都会分配一个线程的概念不符,
因为即使在fakepay的线程中阻塞住了, server应该还是能够处理其他API请求的. 然而坑爹的就是我们的*application*跑在wsgiref的WSGIServer上.

研究了下wsgiref的源码实现:

{% highlight text %}
wsgiref.WSGIServer --> BaseHTTPServer.HTTPServer --> SocketServer.TCPServer
{% endhighlight %}

TCPServer里有这样的注释:

{% highlight text %}
    # The distinction between handling, getting, processing and
    # finishing a request is fairly arbitrary.  Remember:
    #
    # - handle_request() is the top-level call.  It calls
    #   select, get_request(), verify_request() and process_request()
    # - get_request() is different for stream or datagram sockets
    # - process_request() is the place that may fork a new process
    #   or create a new thread to finish the request
    # - finish_request() instantiates the request handler class;
    #   this constructor will handle the request all by itself

{% endhighlight %}

假设`socket`收到第一个请求(*select socketfd*), 进行请求一的`process_request`需要10秒, 由于它是与*server*处在同一线程, 如果当在第2秒收到第二个请求时, *server*会阻塞在处理第一个请求处,
因此无法及时处理第二个请求(这个时候第一个请求与第二个请求的*client*端都在等待*server*的响应), 10秒后第一个请求处理完毕, *server*线程继续执行到*select socketfd*, 
获取到第二个请求, 再进行`process_request`, 处理第二个请求, 完毕后继续等待新连接请求(*select socketfd*).

也就是说通过`select` `accept`到一个http连接后会交由`process_request`处理, 例如上面的`fakepay`, 如果在`process_request`过程中阻塞了就不会
到下一个`select`了, 所以TCPServer的`process_request`必须是**non-blocking**的.

由此`select()->get_request()->process_request()`所谓的非阻塞, 是指在`select`阶段的I/O多路复用, 但是在`process_request`阶段要是block住了, 就回不到`select`了.
所以, 如果`process_request`是需要花费一定时间的block任务, 就要放到子进程或者线程里去处理, 不要影响主进程的执行.

因为wsgiref.WSGIServer继承自SocketServer.TCPServer, 因此它是单线程执行的, *wsgi application*必须是非阻塞的, 只有处理完一条请求才能执行下一条.

所幸SocketServer中提供了ThreadingTCPServer, 在`process_request`时采用threading, 这样每个请求都开了个独立的线程就不会阻塞了.
只需要继承它就可以作为改良版的WSGIServer了:

{% highlight python %}
import socket
import SocketServer

class WSGIServer(SocketServer.ThreadingTCPServer):

    """BaseHTTPServer that implements the Python WSGI protocol"""

    application = None

    def server_bind(self):
        """Override server_bind to store the server name."""
        SocketServer.TCPServer.server_bind(self)
        host, port = self.socket.getsockname()[:2]
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

{% endhighlight %}

当然, 可以也选择eventlet库, nova-api就使用它作为wsgi server.

ps: 用之前`monkey_patch`一下, green the world!(ゝ∀･)

