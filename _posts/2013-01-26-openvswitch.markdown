---
layout: post
categories: linux network
date: 2013-01-26 16:05:00
title: OpenVSwitch
---

在生产环境用了OpenVSwitch（简称OVS）几个月，觉得应该总结一下。

OpenVSwitch提供一个二层交换机所应该有的功能，替代了linux中的bridge模块，支持vlan,
DSCP方式的QOS，流控，镜像，还支持openflow标准。
在之前我们用bridge来做交换，做流控要研究十分难懂的tc，做bridge中的流量处理要用到ebtables，而QOS标志位的修改在ebtables的特性列表里面还是TODO状态，（不知道有了OVS之后还会不会有人做）。不过OVS还是比linux的native bridge复杂很多，如果你不需要做这么多事情，bridge足够你用了。

至于OpenFlow，按照我的理解，只是一个标准，有别于传统二三层流量处理方式，你可以在一个openflow控制器的table中加入你定义的对特定特征的流量做特定的动作。
如果有兴趣研究openflow的话可以去看他们的白皮书，[openflow.org](www.openflow.org)上面还有一个可以运行在各种系统上的demo。目前已经有一些厂商生产支持openflow的硬件。

说到OVS和OpenFlow的关系，OVS(native)本身是一个OpenFlow的软控制器，可以通过ovs-ofctl命令配置规则，比如匹配某特征的流量，可以是默认动作(normal), 广播(flood), 发到特定端口，对包的某部分做修改，丢弃，等等。你也可以配置OVS把流量发给一个非本地的OpenFlow控制器（如NOX）来处理，不过这超出我们的需求范围了，并没有研究过。


性能据测试，如果不用bridge兼容模式的话，ovs的性能会比native bridge要好。

管理方面，OVS本身有一个json文件做的db，只要这个db存在那么用ovs-vsctl做的操作都会一直有效。除此之外也可以用过python-openvswitch模块来做操作，不过话说这个模块还有点不完善，我觉得特别是在数据类型的转换有不恰当的地方。

顺便吐曹几句，据我所知，目前linux厂商中只有Ubuntu正式维护了OpenVSwitch的包，RedHat作为老牌商业发行版已经落后了。ubuntu在云业务上面的努力值得称赞，除了OVS的支持之外还看到12.04正式支持XCP。而RedHat在6.x中却已经抛弃了Xen的支持，耐人寻味。Xen已经合并进3.0内核主干。保守的RedHat还在还在花很大力度维护自己的2.6.32分支。
OVS在ubtuntu中安装还不算太复杂，但因为大家并没有把他看作linux标准部件，还是要改一些配置。

安装
-----

	apt-get remove ebtables

我这里假定你不再需要传统的兼容bridge模块
	
	apt-get remove bridge-utils

	apt-get install openvswitch-switch
	
假如你用ubuntu正式维护的内核，可以直接装他们编译好的内核模块

	apt-get install openvswitch-datapath-dkms

否则你需要和你的内核源码一起编译

	apt-get install openvswitch-datapath-source
	module-assistant auto-install openvswitch-datapath

修改网络接口配置 /etc/network/interfaces
	
	auto eth0
	iface eth0 inet manual
	up ifconfig $IFACE up
	down ifconfig $IFACE down

	auto xenbr0
	iface xenbr0 inet manual
	up /sbin/ifconfig $IFACE IP地址 netmask 掩码
	post-up ip route add default via 网关
	down ifconfig $IFACE down

由于init脚本和OVS整合度不高，所以在启动的时候会等待网络接口而卡很久，所以要修改 /etc/init/failsafe.conf, 把下面sleep的很多秒的时间改成sleep 1

	$PLYMOUTH message --text="Waiting for network configuration..." || :
	sleep 1

	$PLYMOUTH message --text="Waiting up to 60 more seconds for network configuration..." || :
	sleep 1

配置网桥

	ovs-vsctl add-br xenbr0 && ovs-vsctl add-port xenbr0 eth0 

然后重启就好了。


资料
----------------------------------

关于OVS和OpenFlow的文章其实已经很多了，所以我就不花太多文字来说怎么用OVS。

官方的Document已经十分详细，但是命令很多，csdn上有个人写了一些笔记。
http://blog.csdn.net/yahohi/article/details/6631934

这里有文章介绍怎么给kvm环境安装的
http://networkstatic.net/tag/openvswitch

其他的OpenFlow控制器
[noxrepo.org](http://www.noxrepo.org/)

关于OpenFlow的NORMAL和LOCAL两种动作的区别
http://mailman.stanford.edu/pipermail/openflow-discuss/2011-October/002768.html

Xen hypervisor的整合
-------------------

如果你用XCP，XCP本身已经和OVS高度整合了，但是我们还在用Xen hypervisor，还是需要自己在/etc/xen/scripts中写一个给ovs专门的脚本。
我放了一个基本功能的在github [GIST](https://gist.github.com/4641579) ，如果你要实现mac过滤，防火墙，流控，vlan什么的，因为和特定管理机制、外部数据有关，你可以在其他地方实现再从vif脚本调用。

关于流量限速
-----------------

在虚拟环境中，对vm的限速需求有两种，一种是从vm流向网桥，一种是网桥流向vm。

Xen的netback驱动支持限速(通过vif的rate参数)，但是只能限vm流向网桥的，也就是往外发的流量，对于在vm里面从外面拖东西的流量并没有办法。他的实现很简单，只是在netback里面有一个周期性的计数器，要超过了就在这个周期不再发送，并不是队列方式的流量整形。

而且netback的限速也有一个缺点，每次修改只能重启vm生效。我曾经实验过，修改xenstore中的参数并没有效果，除非你用xm命令把netback拆除再安上，但这会引起vm的不稳定而且可能需要尝试不确定的次数才会把网络重新连上。也曾经在csdn看过一篇文章，修改netback的代码让他去watch xenstore中的变化，但改内核是成本最大的方式。所以通过netback来限速并不是一个好的方式。

用了OVS之后，上面的问题都可以解决。假如接口叫做vif1.0，限制1M bps。

*  限制vm外发的流量，这里单位是kbps

    ovs-vsctl set Interface vif1.0 ingress_policing_rate=1000

	ovs-vsctl set Interface vif1.0 ingress_policing_burst=100

要删除限速，把上面的rate的参数改成0就行了。
	
*  限制vm接收的流量，这里单位是bps

	ovs-vsctl -- set port vif1.0 qos=@newqos -- --id=@newqos create qos type=linux-htb other-config:max-rate=1000000 queues=0=@q0 \
	-- --id=@q0 create queue other-config:max-rate=1000000

但是删除这个限速有一个比较麻烦的地方，就是他会在OVS的数据库中分别在Qos表和Queue表各有一条记录，
Qos记录会引用Queue的id，所以你要先查到Queue和Qos的id，再删除，还没有一条简单的命令能完成这件事情。可以ovs-vsctl list和find命令来查询他的数据库，或者也可以用过pyhton-openvswitch模块来和他的数据库做交互。这里会有另外一个坑，但暂时没时间说了。

所以总的来说OVS很先进，但是距离我们平常使用的交换机，易用性还有待提高。

