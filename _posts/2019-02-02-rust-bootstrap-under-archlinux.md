---
layout: post
title: 在 Arch Linux 下配置 Rust 开发环境
author: DuckSoft
categories: [编程]
tags: [Rust, 编程语言, Arch Linux, Linux, JetBrains, IntelliJ IDEA, rustup]
image: forest.jpg
---

## 0x00: 写在前面
作者有点懒，先放在这里占个位置。

## 0x01: Rust 编译器的安装器
受益于 Arch Linux 伟大的包管理系统（当然我并没有在赞美安全性掉渣的 `pacman` 以及 `yaourt`，我们这里使用 `yay`），我们仅需一条命令就能搞定 Rust 的安装。

在 Arch Linux 的官方仓库中，Rust 以两种形式存在：
* 一为 Rust 的“安装器”——`rustup`；
* 一为现成的 Rust 编译器套件 `rust`。

并且这两个包**互相冲突**。

惨痛的教训告诉我们，使用 `rustup` 的安装方式要远远好于后者的安装方式。最重要的是，接下来我们要配置的宇宙最强无敌 IDE —— JetBrains CLion 的 Rust 插件对 `rustup` 有强依赖，而后者的安装方式将使得 Rust 插件完全不能正常工作（包括语法高亮和正常的编译执行等）。

所以如果你已经通过 `rust` 包安装了 Rust 编译器套件的话，不要犹豫赶紧使用 `pacman -Rss rust --noconfirm` 干掉这个包（然而删包用 `pacman` 好象并没有什么安全性问题）。之后再用 `yay -Syy rustup` 安装 `rustup`，这样我们就完成了 Rust **安装器**的安装。

## 0x02: 正式安装 Rust 编译器
什么，你说我们折腾了这么久才装上一个安装器？我的 Rust 编译器呢？别急，我们使用 `rustup toolchain` 功能来安装 Rust 的全套工具链。在这里我们有三个版本的 Rust 套装可供选择：
 - `stable`：稳定版
 - `beta`：测试版
 - `nightly`：（更不稳定的）测试版

由于成文之时 Rust 的很多常用高级功能都已经 stablize，所以在这里建议大家安装 `stable` 版本（当然如果你需要 `nightly` 中的一些功能也可以选择安装 `nightly` 等版本）。使用下面的命令来安装选定版本的 Rust 套装：

```bash
# 下面的 stable 可以换成 beta 或者 nightly，取决于你的具体需求。
# 更多用法可以参照 rustup toolchain --help 中的文档。
rustup toolchain install stable
```
等待他下载安装完成就大功告成啦！


## 0x03: 配置 `rustup` 镜像源
什么？你跟我说下载速度只有 `20 kb/s`（没错是 bit）？下载到 95% 就不动了？你需要的只是一个镜像源！

> 中国科学技术大学 `rustup` 镜像源
> 
> http://mirrors.ustc.edu.cn/help/rust-static.html

由于 Arch Linux 伟大的特性，我们就不需要设置 `RUSTUP_UPDATE_ROOT` 环境变量了，按文中所述设置用于更新 `toolchain` 的 `RUSTUP_DIST_SERVER` 环境变量即可：

```bash
# 设置 toolchain 更新环境变量
export RUSTUP_DIST_SERVER=https://mirrors.ustc.edu.cn/rust-static
# 安装最新的 stable 工具链
rustup toolchain install stable
```

截至 `2018-02-02 14:00:00 (GMT+8)`，镜像源仍然可用。

## 0x04: 配置 CLion Rust 插件
首先你得安装 CLion。这次我们依然有两个选择：
* 我们可以选择使用 Arch Linux 自带的包管理系统，安装 [Arch Linux User Repository](https://aur.archlinux.org/) (**AUR**) 上的 `clion` 相关包。你所需的只是 `yay -S clion`。
* 我们也可以选择 JetBrains 官方提供的 [Toolbox App](https://www.jetbrains.com/toolbox/) 来安装并管理 JetBrains 的全系产品，不经过系统本身的包管理器。

作者更加推荐使用后者的方法安装。（JetBrains 全家桶真香）

下面是安装方法：在 CLion 任意界面使用组合键 `Ctrl+Shift+A`，键入 `Plugins` 并敲击回车键进入插件管理界面。在弹出的对话框中点选 `Install JetBrains Plugins` 按钮，在新弹出的对话框的搜索框中键入 `Rust`，在列表中选中 Rust 插件并在右侧栏中点击绿色的安装按钮即可完成安装。

如果出现下载速度过慢的问题，请自行在 IDE 设置中配置代理以规避（不是解决）这个问题。

## 0xFE: 选择 CLion 的原因
可能有一些读者有这样的疑问：CLion 本身是基于 IntelliJ IDEA 开发的，为什么我们要跑去用 CLion 来开发 Rust，而不是使用 IntelliJ IDEA 呢？

当然，选择 IntelliJ IDEA 也能奏效。但是作者出于下面几点考虑，将 Rust 插件放在了 CLion 上而非 IDEA 上：

1. IntelliJ IDEA 主要是用于 JVM 系语言的开发，而 CLion 主要是用于 C/C++ 系语言的开发。从语言的本质上来说，用于 C/C++ 语言的 CLion **更贴近 Rust 的本质**。
2. IntelliJ IDEA 本身很**重**，启动时要加载很多我们在 Rust 开发时用不到的插件。而 CLion 相对来说加载的插件会少一些，也会在使用的时候更流畅一些。
3. CLion 的 Debugger 里面加入了对 Rust 的支持，而如果使用 IntelliJ IDEA 则无法享受到便利的 Debug 功能。（此处灰常感谢 [@PoiScript](https://github.com/PoiScript)）

不过最后，其实还是更希望 JetBrains 出一个 RustLion 之类的 Rust 专用的 IDE 呢。