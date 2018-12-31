---
layout: post
title: Windows 下 GoLand 中运行 Golang 程序出现 runnerw.exe 问题的解决
author: DuckSoft
categories: [踩坑]
tags: [Windows, GoLand, Golang, Gogland, runnerw.exe, IDE, LionNatsu]
image: mountains.jpg
---

刚刚接触Golang的时候对Golang的各种强制性措施非常不爽，同时那个时候Golang除了我不熟悉的多线程、并发之外也没有多少亮点，所以学了一段时间之后就把它冷落了。

近期在朋友的启发下准备重拾Golang语言，于是在JetBrains Toolbox里下载了Golang的IDE——GoLand。

还记得很久之前这个IDE叫Gogland，被 [@LionNatsu](https://github.com/LionNatsu)（狮子）笑称为“苟腺”（其实也是狮子带我入的Golang坑）。

轻车熟路建立了Golang的工程，写了一个Hello World程序：

```go
package helloworld

import "fmt"

func main() {
	fmt.Println("Hello golang!")
}
```

写好之后在IDE里按下`Ctrl+Shift+F10`运行编译运行程序，结果竟然出现了下面的错误：

```
runnerw.exe: CreateProcess failed with error 216 (no message available)
```

通过网上的搜索得知，这是因为**没有给程序指定包名main**造成的程序找不到入口点的问题。修改程序，将包名改为`main`后，问题圆满解决。

```go
package main

import "fmt"

func main() {
	fmt.Print("Hello golang!")
}
```

----

（搬运自旧 Hexo 博客）

