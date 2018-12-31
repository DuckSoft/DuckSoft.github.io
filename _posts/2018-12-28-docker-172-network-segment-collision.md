---
layout: post
title: 记一次 Docker 网桥 172.17 网段与宿主机网络环境的冲突
author: DuckSoft
categories: [踩坑]
tags: [Docker, 运维, 网段, keepalived, 高可用, nginx, MySQL, Vert.x, Tomcat, Ubuntu, CentOS, PostgreSQL, rsync]
image: angry-docker.png
---

## 开端：从迁移到折腾

最近所维护的两台服务器搬迁到了新的网络环境（HP ProLiant DL580 Gen9 × 2，下文称 200 服务器、202 服务器），需要手动对服务器上运行的生产环境作出一定的调整。同时由于原 200 服务器主机上的操作系统和软件版本均偏低，查杀过的木马病毒疯狂重生，正好趁这次机会对原有环境做一次大洗牌。

在重做系统之前，我们需要对原有生产程序做备份。由于原 200 服务器上存在 DDoS 木马，一旦木马重生时服务器处于网络连接状态，整个机房的网络全部都会处于拥塞状态，这决定了我们不能通过网络备份的方法备份 200 服务器的数据，不过 202 服务器并没有这个问题。

于是 202 服务器由小伙伴使用机房内一台空闲的服务器做存储机器，直接使用 `scp` 命令进行万兆内网传输，光速备份完毕；而我则将随身携带的 Seagate Backup Plus Slim 1TB USB3.0 移动硬盘手动分出一个 ext4 格式的分区并物理插接到 200 服务器的前置 USB 口上，挂载之后将生产环境数据尽数拷出。这样，数据备份就基本上做完了。

再三确认没有其他需要备份的数据后，忍受着机房风扇巨大的噪音，我们将两台服务器的操作系统从 CentOS 6.5 更换到了 Ubuntu 18.04 LTS。漫长的等待过后，接下来就是重建生产环境了。之前 200 服务器上部署的服务有 nginx，202 服务器上部署的服务有 Vert.x 和 PostgreSQL，另外两个服务器上部署的服务分别是 MySQL 和 Tomcat。原有的布局无疑既浪费资源又性能不高（活活浪费了 4 台双路 E7 高配服务器），经过考虑之后，我们打算把 200 和服务器和 202 服务器通过 keepalived 做成 nginx + Tomcat + Vert.x 的主从热备，而原有的两台 256 GB 内存的较新的服务器上则运行 MySQL 和 PostgreSQL 以充分利用内存资源；同时将 Vert.x 服务装入 Docker 以方便自动化构建与部署，在服务器之间建立单向 rsync 以提供文件的自动同步。（此处略去几小时的折腾）

## 疑惑：Service Unavailable?

经过一堆琐碎的配置后，此次运维工作终于告一段落。两个主要客户中的一个客户测试表示测试通过，另一个客户则表示所在的机房无法连接到服务器上两个服务中的任何一个。实地检测以及抓报表明，客户机在与服务器进行 TCP 握手过程中的 SYN 包可能被远端直接丢弃，很可能是客户机受到了硬件防火墙的阻挡。但小伙伴在远端的 `tcpdump` 表明确实收到了客户机机房的 SYN 包，只是服务器并未给予任何回应，这证明问题出在服务器上。

经过一系列的排查之后，在查看 `ip addr` 时我们突然发现：Docker 的网桥 `docker0` 所处的网段 `172.17.x.x` 正好与报告问题的客户机抓包的远端 IP 一致，这样看来很有可能是 Docker 的网桥与宿主机实际的网络环境发生了冲突。如果是这样，一定还有其他人与我们的情况非常类似。于是经过搜索引擎的一番搜索后，果不其然发现了所搜寻的信息。测试表明：我们停止 `docker.service` 服务，客户报告 nginx 网页服务恢复正常；我们启动服务，客户随即报告 nginx 和 Vert.x 服务均无法访问，问题已经定位。

## 解决：继续折腾

我们尝试了搜索引擎给出的各种方案，包括但不限于停止 `docker.service` 服务、删除旧网桥、添加新网桥、重启 `docker.service` 服务，修改 Docker 的配置文件指定网桥 IP，修改 Docker 的启动参数指定网桥 IP 等等。然而这些方法并没有一个能用的！

无奈翻看 Docker 的帮助文档，发现了两个有用的指令：

* `docker network create` —— 创建 Docker 网络
* `docker network prune` —— 删除无用的 Docker 网络

我们使用以下步骤，成功地排除了问题：

1. 启动 Docker 服务：

   `systemctl start docker.service`

2. 使用不冲突的网段，创建新 Docker 网桥：

   `docker network create \`
   `--driver=bridge \`
   `--subnet=172.254.0.0/16 \`
   `--ip-range=172.254.6.0/24 \`
   `--gateway=172.254.6.254 br0`

3. 删除旧容器：

   `docker rm -f <container_name>`

4. 以新网桥重新启动容器：

   `docker run -d --network=br0 \`
   `--name <container_name> \`
   `<__other_arguments__> <image_name>`

5. 将旧网桥 down 掉（已不再占用）：

   `ip link set docker0 down`

6. （可选）清理 Docker 不再使用的网桥：

   `docker network prune`

## 结语

有句话说得好：不作死就不会死。

然而我想说，如果你不去作死，你可能会错过一些别人看不到的风景。

