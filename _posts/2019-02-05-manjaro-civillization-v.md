---
layout: post
title: 解决 Sid Meier's Civilization V 在 Manjaro Linux 上无法运行的问题
author: DuckSoft
categories: [踩坑]
tags: [Steam, 游戏, Sid Meier's Civilization, 文明5, Linux, Arch Linux, Manjaro]
image: civ5.png
---

## 症状
在 Steam 里直接启动 Sid Meier's Civilization V，一个黑窗口一闪而过。

## 溯源
定位到游戏目录，尝试直接运行游戏：
```
$ ./Civ5XP 
GUID Assets\dlc\dlc_01\mongol.civ5pkg 7a036b7fb9a80e8dea7b73fb58c5a288
GUID Assets\dlc\shared\upgrade1.civ5pkg e818fa28902977b42ee5e3426f5112e6
[S_API FAIL] SteamAPI_Init() failed; no appID found.
Either launch the game from Steam, or put the file steam_appid.txt containing the correct appID in your game folder.
段错误 (核心已转储)
```

显然出现了蜜汁问题。使用 `gdb ./Civ5XP` 加载，并在 `gdb` 提示符下键入 `start` 并回车以开始调试，两个警告映入眼帘：

```
GNU gdb (GDB) 8.2.1
Copyright (C) 2018 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-pc-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from ./Civ5XP...(no debugging symbols found)...done.
(gdb) start
Temporary breakpoint 1 at 0x865ac09
Starting program: /home/ducksoft/.local/share/Steam/steamapps/common/Sid Meier's Civilization V/Civ5XP 
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/usr/lib/libthread_db.so.1".
warning: the debug information found in "./libopenal.so.1.13.0" does not match "./libopenal.so.1" (CRC mismatch).

warning: the debug information found in "./libuuid.so.1.3.0" does not match "./libuuid.so.1" (CRC mismatch).


Temporary breakpoint 1, 0x0865ac09 in main ()
```

显然是 `libopenal` 和 `libuuid` 这两个库可能出现了版本不兼容的问题。

## 意外解决

注意到游戏文件夹下有与这两个库同名的文件，于是尝试性的**把所有含有这两个名称的库全部删掉**（如下图中**被选中的文件**），再运行，发现竟然**就能够正常启动了**！

![removed-libs](https://user-images.githubusercontent.com/7822648/52285214-ec735b80-29a0-11e9-856b-a59166eae083.png)

于是开心地打开游戏，摸了<ruby>一会<rt>半天</rt></ruby>鱼。

顺便附送一键禁用/<ruby>恢复<rt>何弃疗？</rt></ruby>这些库文件的懒人代码：
```bash
# 禁用
rename .so .s-disabled-o $HOME/.steam/steam/steamapps/common/Sid\ Meier\'s\ Civilization\ V/{*openal*,*uuid*}
```

```bash
# 恢复
rename .s-disabled-o .so $HOME/.steam/steam/steamapps/common/Sid\ Meier\'s\ Civilization\ V/{*openal*,*uuid*}
```

## 后续

<ruby>摸鱼后<rt>事后</rt></ruby>去 STFW，找到了正统的解决方案：

首先为了满足游戏的 `lib32-openal` 需求，我们可以直接安装位于 `multilib` 仓库里的 `steam-runtime` 包（**笔者已经安装好了，所以直接删掉那些冲突文件即可正常进入游戏**）。

完成安装之后，在启动游戏前使用 `LD_PRELOAD` 环境变量优先加载位于系统的 `libopenal.so.1` 库，阻止游戏加载同文件夹下的错误版本，然后再运行游戏就不会出现问题了：

```bash
export LD_PRELOAD=/usr/lib32/libopenal.so.1
./Civ5XP
```

把上面的代码保存为 Bash 脚本文件然后存到同一目录备用即可。

## 不过
考虑到游戏的启动选项里面并不能更改什么环境变量之类的东西，只要不手贱检查文件完整性，还是痛快删文件吧，一劳永逸。
