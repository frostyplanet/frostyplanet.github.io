---
layout: post
title: 进程无端挂掉了之后可以调查什么
categories: linux
tags: linux gentoo 运维
date: 2013/02/08 17:00:00
---
近来redis偶尔会在某个时刻挂掉，找了一通。虽然说c程序突然挂掉，原因可能就两个：要么就是被OOM(out of memory kill)，要么就是代码有问题(segment fault之如此类的)。由于我以前并不是个正经的运维，所以调查起来还是花了一点时间。

1. syslog
-----------
因为OOM的消息是由内核发出的，不同的系统的syslog/syslog-ng/rsyslog的配置不一样，有可能在/var/log/syslog，有可能在/var/log/message，
也有可能在/var/log/kern.log，只需要跟据syslog的配置kenrel消息被定向到哪个文件，然后找"Out of memory"就行了。

2. core file limit
---------------------
如果真的是代码问题，因为core文件是默认不产生的，那么你需要在/etc/security/limits.conf解除大小限制 (详见manpage LIMITS.CONF(5))，然后重启服务

	*               soft    core            -1
	*               hard    core            -1

3. core文件的位置
----------------------
core文件会产生在程序工作目录下面（如果程序代码中chdir或chroot过的话），或者是和启动服务的脚本有关，因为可能的位置实在太多了，所以也挺难找的。不同的系统有不同的bug report工具可以收集core的信息，redhat有abrt，ubuntu有apport。不过原理就是在/proc/sys/kernel/core_pattern，
指定了core文件是直接产生(默认值是core)，还是调用某个程序把core的内容写到它的stdin中。

server fault有一个帖子讨论到这个[问题](http://stackoverflow.com/questions/2065912/core-dumped-but-core-file-is-not-in-current-directory)。

4. debuging symbol的问题
------------------------
光有corefile，没有调试符号也没用。因为我们现在用gentoo，其他系统没有了解过。

首先需要在CFLAGS中加上"-g"来产生调试符号，因为porage在编译完之后还会把符号strip掉，所以还需要给make.conf的FEATURES变量加上splitdebug 或者nostrip。详见[gentoo doc](http://www.gentoo.org/proj/en/qa/backtraces.xml)

5. 进程资源监控
---------------
要持续监控一个进程的资源的话，serverfault有一个[帖子](http://stackoverflow.com/questions/1663766/programmatic-resource-monitoring-per-process-in-linux) 讨论到这个问题。凡是ps能看的数据，在/proc/[pid]/stat都能获取到。
