---
categories: linux io
date: 2013-01-28 19:00:00
title: 限制虚拟机的块设备IO
layout: post
---

由于虚拟环境上面各种类型的应用都有，当某一个用户占用太多io资源，那么同一个物理设备上面的其他虚拟设备必然受到影响。
虽说CFQ会尽量公平调度，但是当iostat看到utility接近100%的时候，某个blkback进程进入了D的状态，其他进程抢不到CPU资源，系统（包括dom0和其他domU）都不可避免的感到卡。
因为有些用户不太会理会管理员的提醒去改进应用实现，所以必然需要从外部来进行限制。

一个临时的方法就是用ionice。比如查出某个domU的id是21，然后ps auxf | grep blkback.21看到这几个进程

	root     27179  0.0  0.0      0     0 ?        S    Jan11   0:04  \_ [blkback.21.xvda]
	root     27180  0.0  0.0      0     0 ?        S    Jan11   0:00  \_ [blkback.21.xvda]
	root     27181  0.5  0.0      0     0 ?        S    Jan11  77:09  \_ [blkback.21.xvdc]

然后对io占用最重的那个设备执行 "ionice -c idle -p 27181"，把他变成最低的优先级class，虽然物理设备的io吞吐不会有什么减少，但是其他domU卡的情况应该能缓解很多。如果你用kvm而不用xen，那么对qemu的进程操作有一样的效果。

另一个方法就是用cgroup的blkio模块，提供了两类功能：一类是创建不同的group，按照比例分配io资源；
另一类是你根据系统整体的io极限，来适当限制某个设备的读写吞吐量和读写次数。
使用方法在内核文档cgroups/blkio-controller.txt都有，我就不详细写了。

比如把某个设备号是252/23的lvm设备的写操作次数限到800，可以

	echo "252:23 800" > /sys/fs/cgroup/blkio/blkio.throttle.write_iops_device

需要补充一下的是，因为这并不是一个普通文件，这里面&gt;或&gt;&gt;都是没关系的，不会覆盖其他设备的设定。如果要去除刚才的限制把命令中的那个800换成0就行了。
不过注意的是，重启机器会失效，所以需要生成一些脚本来每次启动的时候初始化参数。

