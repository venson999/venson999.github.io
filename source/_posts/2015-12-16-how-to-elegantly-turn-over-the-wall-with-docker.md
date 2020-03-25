---
title: Docker之如何优雅的翻墙
date: 2015.12.16 16:51:31
categories:
    - 技术
tags: [Docker, GFW]
---

![GFW](http://venson-blog-images.oss-cn-beijing.aliyuncs.com/2015-12-16-how-to-elegantly-turn-over-the-wall-with-docker/270064-c3ca9f726b6376fe.jpg)

翻墙的感觉你懂的，freedom，但翻墙的姿势多种多样，如何选择最优雅的一种，也是一门学问。可以选择VPN或者Shadowsocks，也可以选择购买或者自建。购买现成服务的话，优点是即买即用，方便便宜，但缺点是不稳定，要么有今天没明天的，要么网络像抽风一样的。自建服务的话，优点是自由度高，风险低，缺点是租用国外VPS比较贵，自建服务比较麻烦，更换VPS运营商更麻烦。

下面我来介绍一种自以为优雅的姿势，建好VPS之后，全程只需几条命令，耗时不到5分钟（视网络情况而定），即可实现各种翻墙。

**VPS**
首先购买VPS，选谁家的根据自己喜好，我选的[DigitalOcean](https://www.digitalocean.com/?refcode=33bfc6209259)。付款用的Paypal，现在有中文版了，绑一张双币信用卡，很方便。然后建好虚拟机，登录进去，最好用**SSH Key**登录。

**Docker**
安装最新版本的Docker，只需一条命令。
```
% curl -sSL https://get.docker.com/ | sh
```

**VPN**
利用Docker安装VPN服务，只需一条命令，在执行命令之前需要先建立VPN账户密码文件，然后将以下命令中的{local_path_to_chap_secrets}部分替换为该文件路径，具体信息参照DockerHub [mobtitude](https://hub.docker.com/u/mobtitude/)/[vpn-pptp](https://hub.docker.com/r/mobtitude/vpn-pptp/)。
```
% docker run -d --privileged --net=host -v {local_path_to_chap_secrets}:/etc/ppp/chap-secrets mobtitude/vpn-pptp
```

**Shadowsocks**
利用Docker安装Shadowsocks服务，同样只需一条命令，将以下命令中$SSPASSWORD部分替换为自己设定的密码，具体信息参照DockerHub [oddrationale](https://hub.docker.com/u/oddrationale/)/[docker-shadowsocks](https://hub.docker.com/r/oddrationale/docker-shadowsocks/)。
```
% docker run -d -p 1984:1984 oddrationale/docker-shadowsocks -s 0.0.0.0 -p 1984 -k $SSPASSWORD -m aes-256-cfb
```

然后就没有然后了，尽情的翻滚吧~

本文对于小白来说可能不太好理解，多了解一下黑体字相关的概念会大有帮助，但是我觉得在Docker的淫威之下，不理解硬做也问题不大，没错，Docker就是这么强，Docker大法好！
