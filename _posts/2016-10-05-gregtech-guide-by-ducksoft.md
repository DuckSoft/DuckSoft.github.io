---
layout: post
title: DuckSoft 的格雷电力系统教程
author: DuckSoft
categories: [游戏]
tags: [游戏, Minecraft, GT5, GT5U, 格雷科技, 教程, 电力]
image: gregtech-machines.jpg
---

## 序
> 最后更新：`2019/10/8 04:51 GMT+08`

这是一篇充满了**欢乐**气息的文章。这篇文章作于与百度化学吧的吧友们在 Minecraft 服务器上快乐玩耍格雷科技的期间。由于原版 Minecraft 的耐玩度不高，我们引入了工业2等科技类模组来延长游戏时间；然后，一位叫 @EAirPeter 的大魔王出现了：他向我们隆重推荐了 GregTech 格雷科技这个变态 mod。之后，我们一行人就陷入了被这个 mod 支配的恐惧。原版的工业2加上了格雷科技这样一个 mod 之后，原本很简单就能做出来的机器的合成配方莫名其妙地就变得特别复杂；同时，科技树也变成了“科技网”，各种复杂的前置条件、前置机器让一群一心求发展的人丈二和尚摸不着头脑；合成一个机器所需要的各种复杂的工具、以及工具本身复杂的制作方式，让一群抖M在各类科技的世界中极尽欢乐，这里就是我们的“极乐净土”。

这是一篇用**血与泪的教训**凝成的文章。这篇文章所创作的目的，在当时并不是出于理性的科学研究的态度的。当时游戏的一行人对格雷科技所知甚少，一切游戏进度靠摸索，一切高危操作（操作不当可能引起电网上连接的所有机器全部爆炸，一朝回到解放前）都要靠单机模拟甚至手动备份服务器存档来避免悲剧。出于预防的目的，我和 @LionNatsu 等人细心研究了 GregTech 5 的电力系统及其内在机制，写出这样一篇文章来让服务器上的玩家对于格雷科技的一些设定有更清楚的认识，同时也要好好地为自己的操作负起责任（？）。

这是一篇充满了**历史**气息的文章。这篇文章大部分作于 DuckSoft 的 Blackberry 9000 设备上，全键盘黑莓手机为文章的创作平添不少动力。当 DuckSoft 空闲的时候，他就会掏出黑莓手机，细心斟酌，然后对文章内容进行增增减减。百度化学吧 Minecraft 群自 2011 年 12 月 24 日建立以来，自成文之日已有近 8 年时间。期间大家频繁地在 sumdream 的麦田服务器、凑钱从淘宝上租用的“速特互联” MultiMC 服务器甚至其他的科技服务器里出没，甚至在某些时间曾制霸服务器。种地、撸树、挖矿、造房……乃至于上线时一个个 `hi~`、`233` 的问候，都是那么地令人怀念。

虽然现在，大家已经很少有时间能聚在一起好好地玩 Minecraft 了，但是**那些珍贵的记忆片段，即使时隔八年之久，现在也依然清楚于心。**今日偶然在翻阅自己的 GitHub 的时候发现了这篇雪藏多年的文章，为自己当时的钻研精神所打动，也让我回想起了不少温馨的过往。

**谨以此文致百度化学吧 Minecraft 官方群永远的大家。**

## Chapter 1：格雷电力系统初步
由于工业2实验版作者对原有工业2的电力系统进行了大刀阔斧的删减，格雷作者对新版工业2的电力系统很不满意。在格雷作者的努力下，格雷有了一套几乎全新的电力系统。这个系统引入了与现实世界极为接近的电压、电流、电阻率等诸多物理量，使得格雷的电力系统变得非常具有科学性。同时这些物理量的引入，使得因过压过流而导致导线、机器爆炸起火的惨案重返工业2 mod。因此，格雷的玩家们至少应当对格雷的电力系统有一个粗略的了解，以避免出现家破档毁人崩溃的惨案。

尽管格雷的电力系统相对于工业2实验版中的电力系统来说是一次推倒重来，但格雷仍然继承了工业2中电力系统的一些特征，比如电能的单位仍为 EU ，电功率的单位仍为 EU/t 等等。但由于工业2实验版中淡化了电压、电流等概念，造成了“锡线比钻石导线还屌”等可笑情况，甚至在其源代码中，竟然用有与平台相关的误差的浮点数类型去表示电能，这让包括格雷作者在内的程序猿们很不爽。因此，格雷决定抛弃原有的工业电力系统，转而建立一套自己的电力系统，以满足各种抖M的需要（炸机、导线起火、格雷式集束炸弹等等）。

在格雷中，除了格雷变压器以外，没有任何一台格雷机器能直接接受工业电网的供电，这保证了格雷电网的稳定性。

格雷的用电器分为若干个电压等级（*Tier*），每一个电压等级都有一个别名和一个指定的电压值与其对应，例如 Tier 0 的别名是ULV（*Ultra Low Voltage*, 超低压），电压值是 8 EU/p。不难发现，在格雷中，电压的概念变得明确了起来。在工业中，电压的概念一直与电功率的概念相互混淆不清。而在格雷中，电压（单位为EU/p, 每电能包所含电能数目）和电功率（单位为EU/t, 每tick所产生或消耗的电能总数）这两个概念都有明确而严谨的定义。在这里，要引入一个新的概念：电能包（*Energy Packet*）。众所周知，电能能在导线上传输，当然也可以直接让电源贴着用电器来传输。电能在传输过程中，是以若干个相互独立的电能包的形式存在的。这些电能包从电源发出，沿着导线移动到需要能量的用电器当中，然后为用电器所吸收。电能包内所含电能的多少即为该电能包的电压（*Voltage*），而某一电路节点上单位时间（一般取 tick）内同时通过的电能包的数目即为该节点处的电流（*Current*）。注意到电压的单位是 EU/p，在电流的单位取 p/t 时，由于电功率等于电压乘以电流，因此电功率的单位便成了熟悉的 EU/t ，这无疑更加体现了格雷电力系统的科学性。

我们知道，在格雷的电力系统中，损耗（*Loss*）无处不在。格雷的损耗种类大体上可分为效损（*Efficiency loss*，即由发电机（如燃气轮机）或用电器自身效率所决定的损耗）、线损（*Cable loss*，即在电能的传输过程中由于导线具有电阻造成的能量损耗）和缓损（*Buffer loss*，即在从缓冲区中抽取电能时所造成的电能的损耗）。一般来说，效损和缓损与设备本身的属性有关，无法减少；而线损则可通过合理选择导线类型、布局导线网络来尽量减少，因此研究线损对格雷能量布局有重要意义。

缓损是格雷中损耗最常见、最基本的形式。几乎所有的格雷设备，无论是发电机（*Power Generators*）还是用电器（*Power Consumers*），在设备内部都有一个电力缓冲区（*EU Buffer*）来暂时存储所得的电能，这与工业2是基本一致的（工业2有缓冲区但没缓损）。根据设备类型及设备等级的不同，缓冲区的大小也不尽相同，例如存电箱、发电机的电能缓冲区大小普遍要比用电器的电能缓冲区的要大，又如高电压等级的用电器内的缓冲区大小普遍比低电压等级的电能缓冲区的要大。在机器需要能量的时候，机器从外界接受的电能包内的电量便被倾倒进电能缓冲区内储存起来，而这一过程并无损耗；正如缓损的定义所述，真正的损耗出现在从缓冲区内抽取能量的操作中。

根据机器电压等级的不同，每次从缓冲区抽取的电能包的电压等级也不同，这造成了缓损的大小也不同。给定一个电压等级t，其对应的电压值（单位 EU/p）为 8*4^t，而对应的缓损（单位 EU/p）为 2^t。例如，一台 LV 级别（电压等级 t = 1, 32 EU/p）的机器，每当机器从缓冲区抽取一个电能包（32 EU）的电能时，缓冲区实际减少了 32 + 2缓损 = `34EU`的电能。通过合适的数学分析，不难得出，机器的电压等级越高，缓损造成的能量利用效率损失便越小，具体数值计算将在之后的章节提到。

线损是格雷电力网络中能量的头号杀手，也是发展的限制条件之一。我们知道，电能在导线（*Cable*）上传输时是以电能包的形式存在的，而线损正是发生在电能包进入某格导线的过程中。当电能包进入某格导线时，格雷系统会从电能包中减去与该格导线的电阻率等值的电能数，直至减为 0 而电能包被抛弃。例如，有一个 32 EU 的电能包进入一格电阻率为2的导线，那么导线在将电能包传出时，电能包内的电能将只剩下 30 EU。值得注意的是，由于线损是针对每一个流经的电能包的而不是针对整个能量流的，因此减少需要流经导线的电能包的数量（即增大每电能包所含能量数，即升压）可以在一定程度上有效减小线损。后面的章节会有详细的讨论。

### 章末自检

1. 格雷为什么要建立自己的电力系统？格雷的电力系统与工业 mod 实验版中的有何主要区别？
2. 在格雷中，什么是电能包？什么是电压、电流、电功率？这些物理量在格雷中各自的标准单位是什么？
3. 在格雷中，效损、缓损和线损各是由什么原因造成的？试举几个实际例子。

### 章末习题
有一条长为 120 m 的高压导线（耐压 512 V），其电阻率为 1，在导线的一个末端输入 128V 4A 的电能，另一末端接一个足够大的储电箱。回答下列问题：

1. 直接输入导线的电功率是多大?
2. 如果以此方式输电，传到储电箱时，每个电能包中还剩多少电能？也就是还剩多大电压？电流变了吗？
3. 这种输电方式在导线上损耗了多大的电功率？若存电箱的容量为 8 000 000EU，设 1s = 20 tick，至少要多长时间才能充满电箱（保留到 1 s）？
4. 如果保持输入导线的电功率不变，改用 32 V 来输电，导线至少要能耐受多大的电流？试分析使用 32V 输电方案的不合理性。
5. 试计算题干所述的输电方案的输电效率（保留到 0.01%）。（提示：输电效率＝实际获得的电功率÷输入的电功率）
6. 若保持输入导线的电功率不变，改用 512 V 高压输电，试通过计算该方案的输电效率来说明该方案比题干的方案更好。

## Chapter 2：玩转格雷高压输电
> 未完待续中……

## 附录
下面是一些有用的数据，供读者参考使用。

### 附录1：电压名称、电压等级及电压值的换算
> 本表中 Tier 即为电压等级，与电压的换算关系为 8*4^tier。 

| 简写 | 英文全称          | 对应电压值 | Tier |
| ---- | ----------------- | ---------- | ---- |
| ULV  | Ultra Low Voltage | 8          | 0    |
| LV   | Low Voltage       | 32         | 1    |
| MV   | Medium Voltage    | 128        | 2    |
| HV   | High Voltage      | 512        | 3    |
| EV   | Extreme Voltage   | 2048       | 4    |
| IV   | Insane Voltage    | 8192       | 5    |
| LuV  | Ludicrous Voltage | 32768      | 6    |
| ZPMV | ZPM Voltage       | 131072     | 7    |
| UV   | Ultimate Voltage  | 524288     | 8    |
| MaxV | Maximum Voltage   | 2147483647 | 14   |