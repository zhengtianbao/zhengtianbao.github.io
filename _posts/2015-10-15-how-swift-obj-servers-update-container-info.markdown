---
title: "how swift object servers update container info"
date: 2015-10-15 15:46:53
categories: ["2015"]
tags: [swift]
---

继上篇文章所改的 Swift 使用 Ceph 作为后端存储后，3 obj，3 container，3 account，2 proxy 工作正常。

因为我们需要是保证 Swift 的高可用性，理论上要求挂了 1 obj，1 container，1，account，1 proxy 后依然正常。

然而实际情况却有点出乎意料:

1. upload 对象成功（ceph pool 中能找到该对象）
2. list container 有一定几率无法列出上传上去的对象

根据第 1 点，可以推测出的是 object server 已经接收到 proxy 上传的对象了，而且也成功保存到 ceph 中了。

根据第 2 点，猜测是 container server 的 db 中没有同步到最新数据导致 list 的时候获取不到，因此就研究下了 container 是如何更新自己 db 的数据的。

首先，Swift 的设计是分 account，container，obj 三层，account db 中保存 container 的元信息，container db 中保存 obj 的元信息。

而更新元信息是按照 obj --> container --> account 自底而上的顺序调的，也就是说 obj server 处理完 PUT 请求（上传一个对象）通过 HTTP 调 container 的 API 更新其 db，可能是类似添加一条记录来保存这个对象信息之类的。这样当 container 收到GET 请求（列出某个 container 下的对象）时就能通过 db 查询出对象信息。

其中令人疑惑的就是 obj 是发给哪个 container server 更新信息（obj 和 container 的对应信息怎么获取的），还是说发给所有的 container server 都更新（每个 obj 都发一遍所有 container 产生信息冗余）？

以 upload 一个对象为例从 Swift 代码本身上分析：

1. proxy server 通过 ring 得到需要保存的 obj server，这里因为 object-ring 的 replicas 为 3，因此就假设目标 obj 为（o1 and o2 and o3）
2. proxy server PUT -> obj server（o1，o2，o3）
3. obj server 保存对象到后端存储（ceph）
4. obj server 通过 request header 中的 `X-Container-Host` 判断需要更新的container server PUT -> container server（c1 or c2 or c3）
5. container server 更新 db

关键是第 4 步，obj server 是通过请求头中的信息判断要更新的 container 地址，而这个请求头是 proxy server 发过来的，也就是说 proxy server 已经确定了哪些 obj server 更新对应的 container server。

```python
    def _backend_requests(self, req, n_outgoing,
                          container_partition, containers,
                          delete_at_container=None, delete_at_partition=None,
                          delete_at_nodes=None):
        headers = [self.generate_request_headers(req, additional=req.headers)
                   for _junk in range(n_outgoing)]

        for header in headers:
            header['Connection'] = 'close'

        for i, container in enumerate(containers):
            i = i % len(headers)

            headers[i]['X-Container-Partition'] = container_partition
            headers[i]['X-Container-Host'] = csv_append(
                headers[i].get('X-Container-Host'),
                '%(ip)s:%(port)s' % container)
            headers[i]['X-Container-Device'] = csv_append(
                headers[i].get('X-Container-Device'),
                container['device'])

        for i, node in enumerate(delete_at_nodes or []):
            i = i % len(headers)

            headers[i]['X-Delete-At-Container'] = delete_at_container
            headers[i]['X-Delete-At-Partition'] = delete_at_partition
            headers[i]['X-Delete-At-Host'] = csv_append(
                headers[i].get('X-Delete-At-Host'),
                '%(ip)s:%(port)s' % node)
            headers[i]['X-Delete-At-Device'] = csv_append(
                headers[i].get('X-Delete-At-Device'),
                node['device'])

        return headers
```

这段代码意思是假设有 N 个 obj，M 个 container，proxy 只要求前 M 个 obj 发送更新 container 请求。

这就很好的解释了测试结果，假设 o3 和 c3 挂了，o1 虽然能上传对象到 ceph 但是有 1/3 几率更新目标 container 为 c3 时失败，导致 list container 的时候获取不到。

## 参考链接

<http://www.allthingsdistributed.com/2008/12/eventually_consistent.html>
