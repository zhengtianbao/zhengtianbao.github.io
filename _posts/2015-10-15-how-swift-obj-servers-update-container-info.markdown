---
layout: post
title:  "how swift object servers update container info"
date:   2015-10-15 15:46:53
categories: Swift
---


继上篇文章所改的Swift使用Ceph作为后端存储后, 3 obj, 3 container, 3 account, 2 proxy工作正常.

因为我们需要是保证Swift的高可用性, 理论上要求挂了1 obj, 1 container, 1, account, 1 proxy后依然正常.

然而实际情况却有点出乎意料:

1. upload 对象成功(ceph pool中能找到该对象)
2. list container有一定几率无法列出上传上去的对象

根据第一点, 可以推测出的是object server已经接受到proxy上传的对象了, 而且也成功保存到ceph中了.

根据第二点, 猜测是container server的db中没有同步到最新数据导致list的时候获取不到, 因此就研究下了container是如何更新自己db的数据的.

首先, Swift的设计是分account, container, obj三层, account db中保存container的元信息, container db中保存obj的元信息;

而更新元信息是按照obj --> container --> account自底而上的顺序调的, 也就是说obj server处理完PUT请求(上传一个对象)通过HTTP调container的API更新其db,
可能是类似添加一条记录来保存这个对象信息之类的. 这样当container收到GET请求(列出某个container下的对象)时就能通过db查询出对象信息.

其中令人疑惑的就是obj是发给哪个container server更新信息(obj和container的对应信息怎么获取的), 还是说发给所有的container server都更新(每个obj都发一遍所有container产生信息冗余)?

以upload一个对象为例从Swift代码本身上分析:

1. proxy server通过ring得到需要保存的obj server, 这里因为object-ring的replicas为3, 因此就假设目标obj为(o1 and o2 and o3)
2. proxy server PUT -> obj server(o1, o2, o3)
3. obj server 保存对象到后端存储(ceph)
4. obj server通过request header中的`X-Container-Host`判断需要更新的container server PUT -> container server(c1 or c2 or c3)
5. container server更新db

关键是第4步, obj server是通过请求头中的信息判断要更新的container地址, 而这个请求头是proxy server发过来的, 也就是说proxy server已经确定了哪些obj server更新对应的container server.

{% highlight python %}
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
{% endhighlight %}

这段代码意思是假设有N个obj, M个container, proxy只要求前M个obj发送更新container请求.

这就很好的解释了测试结果, 假设o3和c3挂了, o1虽然能上传对象到ceph但是有1/3几率更新目标container为c3时失败, 导致list container的时候获取不到.


### 参考链接:

<http://www.allthingsdistributed.com/2008/12/eventually_consistent.html>

