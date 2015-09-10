---
layout: post
categories: coding
date: 2013/03/28 13:30:00
title: transwarp project
---
最近很多vps的ssh和pptp都被封掉，包括linode日本和美国的一些地方，不知道是因为某软件的协议和ssh很像，还是因为他们把ssh代理的方式都要x掉。
在日本的一台vps和另一台美国的vps研究了一下，发觉被针对的不单单是针对特定协议。任何端口的tcp三次握手都很难达成，icmp包也周期性的不可达(正常的时候延时并不大)，估计是已经进化到路由表污染的方式。
虽然三层路由封锁我们没有办法，不过还是应该减少特定代理方式的使用以免触发什么规则。

所以这几天写了个简单的东西，设想很简单，做一个sock5代理(可以运行在本地没人知道)，用私有协议把请求加密运送到远端出口，协议特征尽量少，异步方式支持大量链接。由于异步调用比较难写，观察到现在好像没有明显的bug。
repo在[这里](https://github.com/frostyplanet/transwarp)。换了一个vps之后部署起来，看youtube还算流畅。

另外要让ssh登陆用上sock5代理的方式，需要装net-analyzer/openbsd-netcat，然后在 ~/.ssh/config中配置 (其中ip和端口就是你的代理地址):

	ProxyCommand /usr/bin/nc.openbsd -X 5 -x 127.0.0.1:9000 %h %p
