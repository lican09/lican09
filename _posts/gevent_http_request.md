---
title: 如何用python异步并发http请求之gevent版
date: 2017-11-26 19:51:25
tags: python
---

### 业务场景

受领导任务安排，测试一个第三方接口。需要发送10万条样本数据的请求，并对返回的数据作分析。

粗略预算了一下，每个请求平均响应时间200ms，10万个请求的时间总和在5.5小时。

当然我不会傻傻的用单线的同步脚本去跑任务。同时也很对写一个同步脚本，手动开多个进程去跑的这种方法感觉很low。

下面会分享一个优雅的方法：使用gevent的队列并发请求测试数据接口。

### 解决方案

#### gevent简介:

gevent在python后台开发中是必不可少的工具库，它的强大在于它能使同步的python代码在IO等待时间挂起，并执行其它任务，达到异步的运行效果，从而提高程序的运行效率，达到高并发的功能。

PS: 期待我后面的文章讲解如何在生产环境正确使用gevent大幅度提高服务器性能吧:>)

[廖雪峰的python教程](https://www.liaoxuefeng.com/wiki/001374738125095c955c1e6d8bb493182103fac9270762a000/001407503089986d175822da68d4d6685fbe849a0e0ca35000)是这样说的:

> gevent是第三方库，通过greenlet实现协程，其基本思想是：
> 当一个greenlet遇到IO操作时，比如访问网络，就自动切换到其他的greenlet，
> 等到IO操作完成，再在适当的时候切换回来继续执行。由于IO操作非常耗时，
> 经常使程序处于等待状态，有了gevent为我们自动切换协程，就保证总有greenlet在运行，
> 而不是等待IO。

[官方文档](http://sdiehl.github.io/gevent-tutorial/)介绍如下:

> gevent是一个基于libev的并发库。它为各种并发和网络相关的任务提供了整洁的API。

#### 代码编写:

代码如下:

```
from gevent import monkey; monkey.patch_socket()
import gevent
from gevent.queue import Queue, Empty
from pymongo import MongoClient
import requests


n = 10  # 并发数量
tasks = Queue(maxsize=2 * n)
collection = MongoClient().db.test


def worker():
    """执行任务"""
    try:
        while True:
            url = tasks.get(timeout=1) # decrements queue size by 1
            print('Worker got task %s' % url)
            resp = requests.get(url)
            if resp.response_code == 200:
                # 将请求返回数据存入数据库，方便分析
                collection.update({"url": url}, {'$set': {"response": resp.json()}})
    except Empty:
        print('Quitting time!')


def boss():
    """发布任务"""

    # 从mongodb中获取测试数据
    for data in collection.find({"response": {'$exists': False}}):

        tasks.put(data['url'])


def run():
    """开始运行"""

    # 两个协程发布任务即可
    bosses = [gevent.spawn(boss) for i in range(2)]

    workers = [gevent.spawn(worker) for i in range(n)]

    gevent.joinall(workers + bosses)

    print('Done!')


if __name__ == "__main__":
    run()
```

* 以上代码未真实运行测试，仅供参考。


#### 代码分析:

gevent.spawn方法会添加n个协程，以达到并发请求量为n的效果。

同时gevent.spawn也会添加2个boss协程以分配任务，并将该任务丢入queue队列中，以保证线程安全。

可根据请求的服务器性能来调节并发请求量n的值。笔者在测试某大厂的接口时，并发量达到100毫无压力，简直不要太爽!

#### 总结

gevent使用协程达到并发请求的效果，相较于多进程更加轻量级，因此更加节省运行资源。


** *申明：该网站所有文章均为原创，转载请著名出处:`http://blog.lican.site`，谢谢！* **

<div id="SOHUCS" sid="gevent_http_request"></div>

