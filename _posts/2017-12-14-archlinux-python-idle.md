---
layout: post
title: 解决 Arch Linux 下 Python IDLE 无法运行的问题
author: DuckSoft
categories: [踩坑]
tags: [Arch Linux, Linux, Python, IDLE, pacman]
image: landmine.jpg
---

解决方案如下，先放为敬：

```bash
pacman -S python-pmw
```

---

深夜在 Oracle VM VirtualBox 里折腾 Arch Linux，偶然看到 `pacman` 输出中闪过的 `python-idle` ，手贱在终端里打了 `idle`，试图启动熟悉的 Python IDLE，然而却得到了这样的输出：

```
IDLE can't import Tkinter.
Your Python may not be configured for Tk. ***
```

看起来是没装 `tkinter`，于是下意识 `pip install tkinter`，然而系统竟然提示我没装 `pip`！
于是 `pacman -S python-pip`，再重新执行 `pip install tkinter`，得到以下输出：

```
Could not find a version that satisfies the reqirement tkinter (from versions: )
No matching distribution found for tkinter
```

一脸懵逼，难道是 Python 版本不对？不存在的。

于是放弃瞎搞，STFW，遂得文首解药。

---

（搬运自旧 Hexo 博客）