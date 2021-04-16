---
layout: post
title: 排查 Nintendo Switch 使用 HTTP 代理时提示“网络需要注册”的诡异事件
author: DuckSoft
categories: [踩坑]
tags: [Shadowsocks, V2Ray, Rust, HTTP, RFC, Nintendo, Switch, 代理, Portal]
image: switch-registration-required.png
---

几天前的一个夜晚，正在 Telegram 某群里畅聊的我突然收到了来自 Gmail Bot 的一则消息提醒。点开一看，原来是有人在 GitHub 上 [Shadowsocks-Rust](https://github.com/shadowsocks/shadowsocks-rust) 的[讨论区](https://github.com/shadowsocks/shadowsocks-rust/discussions/)（Discussions）里问了一个问题，作为 Shadowsocks Rust 团队中的一员，GitHub 自然是应该通知我的。

点开[消息](https://github.com/shadowsocks/shadowsocks-rust/discussions/491)，大致的意思是说，消息的作者用 Shadowsocks-Rust 的 HTTP 入站功能给自己的手机和笔记本电脑开代理用都没有问题，但是在 Nintendo Switch 上用就会弹个框说“网络需要注册”。这个时候，如果点确定，Switch 就会进入一个纯白的页面，上面就俩字——“ok”；页面一关闭，Nintendo Switch 就自动断开网络连接了。此外，作者补充，以上的情况在使用 Shadowsocks-Windows 开代理的时候是没有的。

## 思考

我简单看完消息，思考了一会儿，有了几个突破口：

首先，Nintendo Switch 这个“网络需要注册”指的应该是 WiFi 网络中的 [Captive Portal](https://en.wikipedia.org/wiki/Captive_portal)，也就是像国内的宾馆、机场、星巴克之类的虽然没有 WiFi 密码，但是还是需要连上之后，在弹出的特定的网页里进行登陆才能上的那种 WiFi 的页面，而 Nintendo Switch 不知道抽了什么风，竟然认为这个网络环境下需要登陆 Portal 才能上网。

其次，HTTP 代理主要分两种类型：一种是 `CONNECT` 方法的代理，另一种是直接使用原生的 `GET`/`POST` 等 HTTP 方法的代理；前者是直接将连接升级成 TCP 连接，而后者的处理过程要繁琐许多，想来必然是后者出了岔子，有可能是各种 Header 的处理和保留方式让 Nintendo Switch 不太满意，于是索性直接拒绝了。

最后，Shadowsocks-Windows 的 HTTP 代理的后端用的是 Privoxy 的，而 Shadowsocks-Rust 应该是自己写的，后者有可能在一些机制的处理上和前者不太一样，从而导致了一些问题。

## 复现

想了这么多东西，可能性是无穷的，只有实际试试才知道是什么情况。我想起了自己前几天还在上面玩 Xenoblade: Definitive Version 的 Nintendo Switch，于是将其从充电座上一把薅下来。

从我的 Qv2ray 节点库里随手找了一个自用的节点，然后用下面的命令在笔记本上启动 Shadowsocks-Rust：

```bash
sslocal-rust --server-url 'ss://xxxxxxxxx' --local-addr 0.0.0.0:8001 --protocol http
```

这样，Shadowsocks-Rust 就在本地的 `8001` 端口开放了一个 HTTP 代理服务器。

之后，我在那无法破解的日版非续航升级摇杆疯狂飘逸上扬的破 Switch 上一通操作，连接到笔记本所连接的路由器上，然后将 HTTP 代理打开，地址指向笔记本所开启的 HTTP 服务器。设置完成，果断按下“连接”按钮。

一阵任天堂特有的“噔～噔！噔～噔！……”循环过后，果然复现了上面网友所提到的问题：“网络需要注册”。之后的现象则与上面网友所陈述的一模一样，着实透着诡异。

我在 Switch 上一通操作，把 HTTP 代理的端口换到了我本机自用的 V2Ray 监听的 `8000` 端口上。这次按下“连接”之后，没有“噔噔”几下就正常连接上了，提示“连接成功”。于是，我在本机上开了一个 Privoxy，把 V2Ray 监听的 `1080` 端口在 `8002` 端口转换成 HTTP 代理，再次如法炮制，又是“连接成功”。

从目前的现象看来，问题**可能**确实出在 Shadowsocks-Rust 的实现上了。

## 猜想

为了彻底弄清楚这几声“噔噔”到“连接成功”、以及从“噔噔……噔噔……”到“网络需要注册”的过程中 Nintendo Switch 究竟干了什么，我在本地使用 `create_ap` 开了一个桥接到我的 WiFi 接口 `wlan0` 的热点，并使用 Wireshark 监听接口 `ap0`，以获取在不使用 HTTP 代理下，Nintendo Switch 所发送的数据包的情况；收集到我想要的数据之后，我又在开启 Wireshark 的状态下重新使用 V2Ray 和 Shadowsocks-Rust 开设 HTTP 代理，并分别采集了对应的数据包。

实验的最终结果汇总如下：

|                           原始数据                           |                            V2Ray                             |                       Shadowsocks-Rust                       |
| :----------------------------------------------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
| ![raw](https://user-images.githubusercontent.com/7822648/114284591-7b99c380-9a83-11eb-850e-f125383c31d6.png) | ![v2ray](https://user-images.githubusercontent.com/7822648/114284569-658c0300-9a83-11eb-8fb4-28dee121b119.png) | ![ss-rust](https://user-images.githubusercontent.com/7822648/114284508-04fcc600-9a83-11eb-8a63-ec2f880b1e15.png) |
|                             成功                             |                             成功                             |                           **失败**                           |

从图中可以看到，V2Ray 的 HTTP 代理对服务器返回的原始 HTTP 数据包的 Header 的键值名的大小写是原样保留的，经过代理的数据包除了多了 `Proxy-Connection`、 `Keep-Alive` 之外，没有其他的变化；而观察 Shadowsocks-Rust 的代理回包，我们可以发现，服务器返回的 HTTP Header 的键值名**全部被转换成了小写**。

我又复习了一下 RFC 2616: *Hypertext Transfer Protocol -- HTTP/1.1* 的内容。在原文的 4.2 小节，有这样的一段话：

> **Message Headers**: Each header field consists of a name followed by a colon (":") and the field value. ***Field names are case-insensitive.***

翻译成人话就是：**HTTP/1.1 Header 字段的键值名称是大小写不敏感的！**

也就是说，Shadowsocks-Rust 这样的做法，其实是符合 RFC 规范的。那么这个问题究竟出在哪里呢？

## 验证

仔细地观察了上面的 Header，我一眼发现了一个神奇的 Header：`X-Organization`。曾经依稀记得似乎有这样一条 RFC 标准规定，以 `X-` 开头的 Header 是应用自己定义的 Header，其他应用程序不鸟它就好了。那么，作为一个程序员的直觉在隐隐地告诉我，会不会是任天堂的程序员在写代码的时候把 `X-Organization` 这个字串直接写死了一个 `strcmp`，而忘记了 HTTP/1.1 的 Header 其实是不区分大小写的呢？抑或是他们认为这个 Header 只会由任天堂自己的服务器产生，不会有其他的大小写形式呢？

想到这里，结合具体的抓包内容，我有了一个邪恶的想法：既然任天堂那么想看正常大小写的 `X-Organization`，而且这个返回的内容甚至是固定的，那么我们就写一个 MITM 的程序，把首次访问的这个内容直接替换成正常的大小写，之后的连接再正常转发给代理，这样就能最终确定是这个问题。于是，我用 Golang 写了下面的一段程序：

```go
package main

import (
	"flag"
	"io"
	"log"
	"net"
)

var (
	ListenAddr = flag.String("listen",  "0.0.0.0:8002", "listen address")
	RemoteAddr = flag.String("remote", "127.0.0.1:8001", "remote address")
)

func main() {
	listener, err := net.Listen("tcp4", *ListenAddr)
	if err != nil {
		panic(err)
	}
	conn, err := listener.Accept()
	if err != nil {
		panic(err)
	}
	log.Printf("conn accepted: %v", conn.RemoteAddr().String())
	b := make([]byte, 4096)
	n, err := io.ReadAtLeast(conn, b, 20)
	if err != nil {
		panic(err)
	}
	log.Printf("received:\n%s", string(b[:n]))
	_, err = conn.Write([]byte("HTTP/1.1 200 OK\r\nContent-Length: 2\r\nExpires: Sat, 10 Apr 2021 21:00:12 GMT\r\nCache-Control: max-age=0, no-cache, no-store\r\nPragma: no-cache\r\nDate: Sat, 10 Apr 2021 21:00:12 GMT\r\nConnection: keep-alive\r\nX-Organization: Nintendo\r\nContent-Type: text/plain\r\n\r\nok"))
	if err != nil {
		panic(err)
	}
	log.Printf("fake response sent")
	conn.Close()
	for {
		conn, err := listener.Accept()
		if err != nil {
			continue
		}
		go func() {
			defer conn.Close()
			remoteConn, err := net.Dial("tcp4", *RemoteAddr)
			if err != nil {
				return
			}
			defer conn.Close()
			go io.Copy(remoteConn, conn)
			io.Copy(conn, remoteConn)
		}()
	}
}
```

通过不断调整上面硬编码的回包内容，我最终确定了：当且仅当 `X-Organization` 的 Header 的大小写不同的时候，Nintendo Switch 才会认为这个 HTTP 代理后面的网络还需要登陆，从而把回包的实际内容当成了 Captive Portal 网页的内容，然后显示了一个白屏的“ok”。这也解释了为啥如果用户手动关掉，Nintendo Switch 还会断开网络连接——需要登陆不登陆，当然不能用啦！

我把相应的情况立刻更新到了 [GitHub Discussions](https://github.com/shadowsocks/shadowsocks-rust/discussions/491#discussioncomment-594981)，发现并报告问题的老哥也点起了大拇指、露出了笑脸。同时，我也把这个问题报告给了 Nintendo Support，希望他们能修复这个兼容性问题。

## 修复

问题来了：既然已经确定是 Nintendo Switch 那边的问题了，我们是不是就可以收手了呢？

答案是否定的。Shadowsocks-Windows（Privoxy）、V2Ray 方面的 HTTP 代理能正常工作，而 Shadowsocks-Rust 的不能，这充分说明了我们的实现本身还是有那么一点点问题的。况且要等任天堂修复这个东西，估计得等到猴年马月去，不如发动我们开源社区响应快的优势，直接把问题干掉。

想着想着，我掏出了 CLion，克隆了 Shadowsocks-Rust 的库，熟练地定位到了 HTTP 代理所对应的代码。通过调出 TRACE 等级的日志，我发现原来 Shadowsocks-Rust 底层使用的是 [`hyperium/hyper`](https://github.com/hyperium/hyper) 这个库，而这个库在封装 HTTP/1.1 请求的时候，直接把 Header 的大小写全部抹平了！

同时，在搜索的过程中，我非常尴尬地又发现了一个 [Issue](https://github.com/hyperium/hyper/issues/2313)。这个 Issue 正是 Shadowsocks-Rust 的主要作者 [@zonyitoo](https://github.com/zonyitoo) 在 `hyper` 上提出的问题，希望 `hyper` 的作者支持一下保留原有的 Header 大小写，以方便代理软件的开发，Issue 的最后修改日期显示为 2020 年 10 月 28 日——好家伙，原来早就发现了！

在重新发现 Issue 的同时，我又发现了一个叫 [@nox](https://github.com/nox) 的作者最近发起了一个 [PR](https://github.com/hyperium/hyper/pull/2480)，在写这个功能的支持，并且已经可以使用，只是还在大量重构之中。于是，我将 `Cargo.toml` 的对应的库替换成作者所在的分支的版本，将作者提供的 `.http1_preserve_header_case(true)` 方法修饰到 HTTP 客户端和服务器的相应 Builder 代码上，再次编译，问题已经解决，于是随手发了一个 [PR](https://github.com/shadowsocks/shadowsocks-rust/pull/492) 到 Shadowsocks-Rust 项目，等主要维护者上线处理。

## 后文

主要维护者 [@zonyitoo](https://github.com/zonyitoo) 之后看到了 PR，表示等上游 [@nox](https://github.com/nox) 的功能稳定之后再合并，同时也对我如何联系上 Nintendo 的支持部门表示惊奇——

> **Where did you send ticket to Nintendo?** I found it long time ago and couldn't find a way to tell them about their **stupid code**.

我于 2021 年 4 月 11 日上午通过 Nintendo Support 向日本方面发出了一个 Support Ticket（お問い合わせ番号：`210411-000346`），在 4 月 13 日上午收到了来自任天堂方面的回复，原文如下：

> このたびはお手数をおかけしまして、誠に申し訳ございません。
>
>  お客様より頂戴いたしましたご意見・ご要望は、関連部門に報告させていただきます。
>
>  今後とも弊社製品をご愛顧賜りますようお願い申しあげます。

人话：已向有关部门报告，谢谢。

还是那句话，**要等任天堂修复这个东西，估计得等到猴年马月去啊**！你们任天堂，还得学习一个……咳……RFC……
