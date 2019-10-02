---
layout: post
title: 破解 91porn 的视频网址加密机制
author: DuckSoft
categories: [安全]
tags: [JavaScript, 混淆, 91porn]
image: cutting.jpg
---

## 0x0: 写在前面
自从之前写的一个油猴脚本 [`91porn-utility`](https://github.com/DuckSoft/91porn-utility) 所利用的白嫖 VIP 会员的方法被官方封禁之后，这个脚本的卖点也就从之前的“破解 VIP 高清视频”变成了“清爽去广告”。

随着脚本在 GitHub 上的点击量不断激增，截至发文之日，脚本的下载量已经突破 13k 人次，各式各样的人也在 Issue 列表里留下了各种言论。有的 Issue 发了一个 Bug Report，内容是“脚本太牛逼的bug”；有的 Issue 劝我不要公开这个脚本，以免利用方式被官方封禁（事实上他一口奶中了）。不过，其中一个人的 Issue 格外扎眼——他想付费让我做一个 91porn 的爬虫。

说实话，刚看到这个 Issue 的时候，我的眼里满是轻蔑——91porn 这种网站的爬虫难道还值得掏钱去请人做吗？尽管如此，我还是出于对技术的好奇，掏出 GoLand 对着窗口就是一顿乱敲，搭建了一个简单的并发爬虫框架。

Boilerplate 代码写完了，接下来应该是去网站上实地考察了。爬取视频页码、爬取视频页面到单个视频的链接列表、批量获取所有的视频链接，这些都轻车熟路地完成了。终于只需要解决一个问题，就能让这个爬虫变得有价值了——把单个视频网页的网址转换成实际播放的视频的网址。

当我再次抱着轻蔑的心态点开某个 NSFW 的视频页面的源代码，定位到一段特别扎眼的代码时，我愣住了：

```javascript
<script>
<!--
document.write(strencode("Ynl6Fy4tKR1pC3MtIQMpWBt+CntRZyN1IhBwIX8tCgcmAGVgJV0JQX8oAAsFFF0fEXMfFgEzBRJ9Pg90HytYHxISAhpVXwFYFWFfdjR2IGdmJGktYg9jczRqASIEJ2UhDwFyJQsfGSIYPhwFGDcMGn4sI04PFCBcKiscdCk+BUAGWxMWLhEBNQYKY1x8cldK","214aJucw3X1WBndaQLbK5/bCniHY0ymrkj0Qi7n83BksImdkr7NlMIHh3DBFPhmkqVS56lPaGP0Dy3vU+H+X/Z1Ff/mkfc3yVTCpZLCNHjY4VMMb3xv6BPG2ccNAJyPyLhIftVWCJ8R+","Ynl6Fy4tKR1pC3MtIQMpWBt+CntRZyN1IhBwIX8tCgcmAGVgJV0JQX8oAAsFFF0fEXMfFgEzBRJ9Pg90HytYHxISAhpVXwFYFWFfdjR2IGdmJGktYg9jczRqASIEJ2UhDwFyJQsfGSIYPhwFGDcMGn4sI04PFCBcKiscdCk+BUAGWxMWLhEBNQYKY1x8cldK"));
//-->
</script>
```

刷新一下页面，发现这段代码又变成了另一个：

```javascript
<script>
<!--
document.write(strencode("NC19FwABclwtGQ5VEEYOVBMFPFEVD3ZYdTh7TSIhBSEYXhZ+DlpRGCAEOEsOFGE9AS4eGQZMAAIrAmYBFjN8FlAAHngOBCkBdwpKRTFXEzsqNRtTHycccD06BEAHPwwmHTgYEDkLbSFoe3UIHhF5LTQGKT0iGWU3MwEAU1AQGUASejBJG3U4PRtEVScqJlBK","de3adY86wJL/s+CmY7TaqG7n9AC5mubTU4COB06alnS3BmXIbjOcJ6Mxex+3YpIb3DOWm7x88Ls/UdEaxXJ+PpVAnoI4HhBSSBpGlX7M8/09Pk8UyRpJls0YzIRf3WLyXIj9A2nKWvdP","NC19FwABclwtGQ5VEEYOVBMFPFEVD3ZYdTh7TSIhBSEYXhZ+DlpRGCAEOEsOFGE9AS4eGQZMAAIrAmYBFjN8FlAAHngOBCkBdwpKRTFXEzsqNRtTHycccD06BEAHPwwmHTgYEDkLbSFoe3UIHhF5LTQGKT0iGWU3MwEAU1AQGUASejBJG3U4PRtEVScqJlBK"));
//-->
</script>
```

按理来讲，一个视频的地址是固定的，这从网页加载完毕之后的开发者工具上能明显地看出来。但是这个给予网页视频实际地址的代码，每次刷新完毕之后都会发生变化，显然这背后隐藏着什么不为人知的秘密。

于是乎，我们的探索从这里开始。

## 0x1: 寻根溯源

WIP。
