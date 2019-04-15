---
title: Tornado异步接口正确编写方法
date: 2017-12-04 20:52:17
tags: python
---

### Tornado简介:

[官方介绍](http://tornado-zh.readthedocs.io/zh/latest/guide/intro.html)如下:
> Tornado 是一个Python web框架和异步网络库，起初由 FriendFeed 开发。
> 通过使用非阻塞网络I/O, Tornado 可以支持上万级的连接，处理 长连接, WebSockets,
> 和其他需要与每个用户保持长久连接的应用。

原理:

* Tornado使用了linux系统底层的`epoll`来实现它的高并发（在BSD和Mac OS X系统调用的`kqueue`）;

* 一个tornado的实例(instance)其实是一个`IOLoop`类的实例；

* `IOLoop`实例会不断轮询并处理当前活跃的连接。


### 一个简单的“Hello world"示例(同步阻塞):

示例一:
```
import tornado.ioloop
import tornado.web

class MainHandler(tornado.web.RequestHandler):
    def get(self):
        self.write("Hello, world")

if __name__ == "__main__":
    application = tornado.web.Application([
        (r"/", MainHandler),
    ])
    application.listen(8888)
    tornado.ioloop.IOLoop.current().start()
```

该示例为同步阻塞的写法，项目开发中考虑到服务器的性能，一般不会这样写代码。
想了解异步非阻塞的写法，请继续往下看。

### 一个更高级的"Hello world"示例(异步非阻塞):

示例二:
```
import tornado.ioloop
import tornado.web
from tornado import gen

class MainHandler(tornado.web.RequestHandler):

    @gen.corountine
    def get(self):
        res = yield foo()
        raise gen.Return(self.finish(res))

    @gen.corountine
    def foo():
        gen.sleep(10)
        raise gen.Return("Hello, world")

if __name__ == "__main__":
    application = tornado.web.Application([
        (r"/", MainHandler),
    ])
    application.listen(8888)
    tornado.ioloop.IOLoop.current().start()
```
示例中有一个异步非阻塞的`GET`方法，其中方法`foo`是一个有io等待的方法被`get`所调用。
每一个异步方法都需要加上装饰器`@gen.corountine`，并使用`yield`来异步调用;
这是Tornado官方推荐的写法，不过还有其他异步写法，请自行查阅[官方文档](http://www.tornadoweb.org)。

### 一种异步并行处理的写法：

示例三:
```
@gen.coroutine
def get(self):
    http_client = AsyncHTTPClient()

    response1, response2 = yield [
        http_client.fetch(url1),
        http_client.fetch(url2)
    ]

    response_dict = yield dict(
        response3=http_client.fetch(url3),
        response4=http_client.fetch(url4)
    )
    response3 = response_dict['response3']
    response4 = response_dict['response4']

```
上面代码中，`AsyncHTTPClient`是Tornado内置的一个异步http client方法，
`response1` 和 `response2` 是同时并发的执行，这两个方法完成后，
执行`response3` 和`response4` 的并发请求。在适当情况下，异步并发运行可以节省大量io等待时间，
是一种提高服务器性能的方法。

### 总结

*笔者工作中对Tornado, Django, Flask这三个web框架均有所涉猎, 以下是个人经验总结:*

* Tornado的优势是长连接。在长连接高并发的场景下优先选择该web框架。

* 对于个人学习是有好处的。完成一个Tornado的项目开发会使你对于python的异步处理有更深入的理解(笔者亲身感受)。

* 在只有短连接请求的业务场景下，tornado相对于gevent的性能并没有太大优势。
此时，推荐选择Flask 或 Django等同步网络框架 + `gevent`的方案。

* 相较于python的另外两个流行的web框架：Flask和Django，Tornado所支持的异步第三方库较少；
而且各种原生支持异步连接数据库(比如redis,mysql,mongodb)的第三方库很多功能也不够完善，开发门槛和难度相对较高。

* 对于不会使用的新手，很容易写成同步的接口。笔者就曾见过有同事用tornado框架写同步接口的项目。


** *申明：该网站所有文章均为原创，转载请著名出处:`http://blog.lican.site`，谢谢！* **

<div id="SOHUCS" sid="tornado_ioloop"></div>
