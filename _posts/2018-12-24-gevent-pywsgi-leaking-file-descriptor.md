---
layout: post
title: 记 gevent 的文件描述符泄漏问题
author: DuckSoft
categories: [踩坑]
tags: [Python, Web, 后端, Flask, gevent, WSGI, 泄漏, 文件描述符, Docker, Tornado]
image: landmine.jpg
---

## 开端

最近用 Docker 向服务器上部署了一批使用 Flask + gevent 的 Web 应用程序，经过不同的时间、不同的访问量后，这些应用程序总会变得对网络请求无应答，显然这些程序因为某种原因已经 down 掉了。

使用 `docker logs` 命令查看容器日志，一排排的正常请求记录映入眼帘，不断滚动；随后则是满屏幕滚动的 Traceback。没有动态视力技能，加之日志实在太长，懒癌发作的笔者直接敲了 `Ctrl+C` 停掉了日志回显，改用 `docker logs --tail=50` 命令查看最后 50 行日志，自然是看到了之前滚动的 Traceback。这些 Traceback 的最后一行都是相同的内容：

```
OSError: [Errno 24] Too many open files
```

而出错的区域正是落在 `gevent.pywsgi` 模块。猜想应该是因为某种原因，`pywsgi` 模块打开了过多的文件而没有关闭，导致文件描述符数目超出了限制。

还是那句话，有问题，找谷歌。果不其然，在 `gevent` 的 GitHub 的 [Issues #649](https://github.com/gevent/issues/649) 找到了我想要的答案。

## 溯源

Issue 主有一个日访问 250 万次的 API 地址（约 20 次/s），使用最原生的 `gevent pywsgi` （甚至没有用上 Flask 等框架）作出最简单的回应。启动服务器，开始接受流量，仅仅 1 小时之后就发现 Python 宿主进程使用了 200 多个文件描述符，并且文件描述符的数量随时间还在增长，当超过系统限制之后程序不断报错。

gevent 的主要贡献者 @jamadden 回应称，这是因为 `gevent` 在处理 Keep-Alive 的长连接时**没有引入有效的超时机制**，因此每当有一个连接超时就会积累一个文件描述符，最终积累导致服务器崩溃。同时他也给出了几种可行的解决方案：

* 设置 `Connection: close` 标头以使连接主动关闭
* 设置上游的反向代理（如 nginx、HAProxy 等）处理这些连接
* [手写超时处理代码](https://github.com/gevent/gevent/issues/649#issuecomment-141439481)

然而我并没有选择这些方案中的任何一个。我选择更为直接的方案——**换框架**。

## 补牢

其实本来我就对 `gevent` 这个 Python 轮子非常不爽，因为每次构建基于 Alpine 的 Docker 镜像的时候，`gevent` 总是那个只用 `pip` 永远装不上的包——你得用 `apk add py3-gevent`跑一遍才行。这不仅破坏了基于 `requirements.txt` 单一依赖清单的美好，也给 Docker 构建过程带来了很多冗余。嗯，终于有机会把他从我的项目中甩掉了。

然而问题又来了，甩掉 `gevent` 之后又该投向什么呢？再次询问万能的谷歌爸爸，最终在 Flask 的[一份文档](http://docs.jinkan.org/docs/flask/deploying/wsgi-standalone.html)中找到了想要的答案。文中提到了使用 Tornado 作为容器部署 Flask 应用的方法。于是脑子一热，就用 Tornado 把工程改写了一下，竟然可以正常运行（我的 App 名叫 `dog`）：

```python
if __name__ == '__main__':
    from tornado.wsgi import WSGIContainer
    from tornado.httpserver import HTTPServer
    from tornado.ioloop import IOLoop
    from dog import app

    http_server = HTTPServer(WSGIContainer(app))
    http_server.listen(3333)
IOLoop.instance().start()
```

于是改写 `Dockerfile`，甩掉该死的 `gevent` 依赖，添加了 `tornado` 到 `requirements.txt`，重新构建并部署了我的 Web 应用程序。

## 小插曲

顺手查了一下 Tornado 的相关资料，这是一个异步的 Web 框架，然而非常滑稽的是 Flask 是一个同步的 Web 框架。用异步的 Web 框架做同步的 Web 框架的 WSGI 容器，看起来似乎并没有什么好处（因为还是阻塞的）。

再次翻阅了 Flask 的官方文档，在[相同标题的页面](http://flask.pocoo.org/docs/1.0/deploying/wsgi-standalone/)内，赫然发现 Tornado 已经被删掉了（！！！），顿时有了一种看了假文档的感觉。

不过，令人心安的是，Tornado 称自己完全可以处理好长连接的问题，至少可能已经解决了我们的问题，所以……再懒一回吧，233。



