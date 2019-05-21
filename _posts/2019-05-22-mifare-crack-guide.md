---
layout: post
title: 在 Arch Linux 下攻击 Mifare NFC 卡片的简明指南
author: DuckSoft
categories: [安全]
tags: [Arch Linux, Linux, 信息安全, NFC, Mifare, mfcuk, mfoc, nfc-mfclassic, ACR122U, Proxmark3]
image: mifare.png
---
> 本文仅用作原理验证以及教育目的，请自觉遵守相关法律法规规定。因使用本文所述技术造成的一切非法后果，本文作者恕不承担任何责任！

## 0x0：设备
### 0x00：必要设备
* 一台 ACR122U-A9，价格约 130 RMB（或者使用价格更亲民的 PN532，搭配 TTL 转 USB/串口模块使用）
* 一台运行 Arch Linux 的电脑

### 0x01：可选设备
* UID 卡扣/白卡若干 —— 用于复制卡
* Proxmark3 —— 比 ACR122U 更好的设备


## 0x1：软件
### 0x10：配置环境
```bash
# 安装 libnfc 驱动及 ACR122U-A9 驱动
yay -S libnfc acsccid --noconfirm
# 启动智能卡服务以支持 PN532
sudo systemctl start pcscd
```

### 0x11：安装工具
```bash
# 安装 mfoc 软件，位于 community 仓库
yay -S mfoc --noconfirm

# 安装修改版 mfcuk 软件，支持对国产加强卡的暴力破解
git clone https://github.com/DrSchottky/mfcuk
cd mfcuk
automake && autoconf && ./configure && make install
```


## 0x2：破解
> 注意：以下的所有操作都应当在 root shell 下运行，否则软件很有可能因权限不足而运行异常！

### 0x20：尝试默认密钥
执行以下命令：

```bash
mfoc -O dump_trial.mfd
```

可能出现的情况分以下三种：
1. 卡片没有任何一个扇区使用默认密钥。此时应考虑使用 `mfcuk` 工具至少找出一个密钥，再使用 `mfoc` 工具进行破解；
2. 卡片有至少一个扇区使用了默认密钥。此时 `mfoc` 可以使用嵌套认证漏洞，花费大概 10-20 分钟左右的时间将所有密钥推出，然后再将卡导出到指定文件。
3. 卡片全部扇区都使用了默认密钥。此时 `mfoc` 可以直接将卡导出到指定文件。

### 0x21：全扇区加密卡的处理
对于上一节的第一种情况，我们需要使用 `mfcuk` 工具对卡片进行 Darkside 攻击，破解出至少一个可用的密钥，再使用 `mfoc` 工具提供的嵌套认证漏洞利用来读出整卡。

执行以下命令：
```bash
mfcuk -C -R 0 -s 500 -S 500 -v 2 -w 5

# -C      --- 破解模式。
# -R 0    --- 破解 0 号扇区的密钥。
# -s 500
# -S 500  --- 关闭电磁场和开启电磁场的时间间隔。使用 PN532 破解的用户可以无视。
#             但对 ACR122U 来说，默认的 10ms 和 50ms 太快了，成功率很低。
# -v 2    --- 多显示一些信息。
# -w 5    --- treshold。防止 NACK 卡爆程序。
```

> 特别注意的是，虽然 AUR 上有现成的 `mfcuk-git` 仓库，但是官方代码对一个严重的问题并没有进行解决：对于一些较新的 Mifare 卡，这些卡在认证失败的时候会直接发送 NACK，导致原有的工具失效并频繁爆出 `mfcuk: ERROR: mfcuk_key_recovery_block() (error code=0x03)` 错误，详情可以原仓库 [Issue #28](https://github.com/nfc-tools/mfcuk/issues/28#issuecomment-319766380)。
>
> 而 DrSchhottky 的[Fork 版本](https://github.com/DrSchottky/mfcuk)通过引入 treshold 选项解决了这一问题。只需要在运行 `mfcuk` 时指定 `-w` 选项设置一个 treshold，例如 `-w 5`，类似的问题就不会再出现。这也是为什么我要推荐使用他的版本的原因。

等待大约 30-50 分钟，我们可以至少拿到 1 个可用的密钥。接下来，我们把拿到的密钥，例如 `ffeeccaabbee` 传递给 `mfoc`，把破解的任务转交给他来完成。在之前的 `mfoc` 参数列表中补充一个 `-k <密钥>`，即可指定使用某些密钥来进行嵌套认证漏洞攻击。成功之后，我们依然可以读出整卡到文件。


## 0x3：写卡

### 0x30：卡片类型与写卡
目前市面上能见到的 Mifare 卡一共有 5 种类型：
1. 普通 Mifare 卡。这种卡的 0 扇区是不可写的，所以对于要复制卡片的人来说，这种卡可能并不适用。大多见于被攻击的原卡，不过对于原卡来说，即使不能进行写入，也可以修改其他扇区的数据。
2. UID Mifare 卡。这种卡通过响应后门指令来实现所有扇区的可写，可以直接绕过密钥验证读取、擦出、写入任意扇区。对于需要复制卡片的人来说这种卡片简直是福音，但事实上恰恰有些系统会去验证卡片是否响应后门指令。这样的话，轻则拒绝访问，重则当场报警，慎之慎之。
3. CUID Mifare 卡。这种卡的所有扇区都是可写的，但是并不响应后门指令。这种卡片由于不响应后门指令，可以绕过一些防火墙系统的设计。但是有些变态的防火墙系统会尝试对 0 扇区进行写入以进行测试，这时轻则卡片写废、拒绝访问，重则当场完犊子，同样要小心。
4. FUID Mifare 卡。这种卡是一种经过改良的 UID 卡，不同的是卡片多出了一个烧死的特殊指令。一旦使用特殊的工具执行了烧死这一指令，卡片就从 UID 卡变回普通的 Mifare 卡。这种卡片连防火墙都拿他没辄，不过缺点是只能用一次，而且太贵了。
5. FCUID Mifare 卡。与 FUID Mifare 卡类似，基于 CUID 卡，不再赘述。

这些卡片的价格按照叙述顺序递增，写卡难度也按照叙述顺序递增。

### 0x31：普通 Mifare 卡的写卡方法
写入普通 Mifare 卡需要用到之前读出的整卡文件。其实文件里有用的部分仅仅是密钥，卡片本身只要密钥没变过，使用过期的文件同样可以对卡片进行读写。执行以下命令来将一个文件写进卡里：

```bash
nfc-mfclassic w a u newdata.mfd olddata.mfd

# newdata.mfd --- 新文件
# olddata.mfd --- 老文件（使用密钥）
```

这样将会更新 Mifare 卡除第 0 扇区外的其他扇区。

### 0x32：UID Mifare 卡的写卡方法
写入 UID 卡不需要使用之前读出的整卡文件（你甚至不用先破解），因为 `nfc-mfclassic` 工具支持 UID 卡的 `unlocked write` 模式，能够直接使用后门指令把数据导入卡内。执行以下命令来把一个文件写入 UID 卡内：
```bash
nfc-mfclassic W a u newdata.mfd

# W --- unlocked write 模式，UID 卡支持。
```

这样将会更新 UID Mifare 卡包括 0 扇区在内的所有扇区。

### 0x33：CUID Mifare 卡的写卡方法
很遗憾，`nfc-mfclassic` 不支持直接将整个文件写入 CUID 卡。因为 `nfc-mfclassic` 工具写入 0 扇区只能使用 `unlocked write` 模式，这恰恰避开了 CUID 卡的特性——不支持后门指令。我们可以使用 `nfc-mfclassic` 工具写卡之后，用第三方工具再将 0 扇区的数据进行写入。

在实际验证中，我使用 Proxmark3 成功地写入了 CUID 卡的 0 扇区。当然，在写入之前，你还需要通过 Mifare 卡的正常验证。我也使用 `libnfc` 配合 C++ 程序实现了一个简单的 CUID 工具，用于自动在写入之前将 CUID 卡的 0 扇区进行妥善处理。感兴趣的读者也可一试。

### 0x34：FUID/FCUID 卡的写卡方法
这些卡片需要特殊的指令才能进行。略。

## 0x4：未完待续
