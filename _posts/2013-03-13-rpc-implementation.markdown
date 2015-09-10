---
layout: post
categories: coding
date: 2013/03/13 22:30:00
title: 写了个python的rpc实现来替代thrift
---

最近线上用apache Thrift实现的服务端变得越来越不靠普，现象就是用TServer.TThreadedServer做的服务端accept之后，ssl handshake的时候一直出错，让所有客户端都无法链接。重启也不一定能消除。线上用了大半年，原来每个月挂一两次，现在变成几个小时就挂。
抛的异常像这样，但是一直找不到答案：

	[Errno 8] _ssl.c:504: EOF occurred in violation of protocol
	Traceback (most recent call last):
	  File "**********/server.py", line 48, in accept
		server_side=True, ssl_version=ssl.PROTOCOL_TLSv1)
	  File "/usr/lib64/python2.7/ssl.py", line 381, in wrap_socket
		ciphers=ciphers)
	  File "/usr/lib64/python2.7/ssl.py", line 143, in __init__
		self.do_handshake()
	  File "/usr/lib64/python2.7/ssl.py", line 305, in do_handshake
		self._sslobj.do_handshake()
	SSLError: [Errno 8] _ssl.c:504: EOF occurred in violation of protocol

python的ssl模块底层是用的openssl库，_sslobj的方法都在c binding里头, 所以上面说的那个异常要调试实在比较困难。
其实我们这个服务每分钟的请求数只有500个左右，也不算多...

因为实在忍受不了随时要重启服务。另外由于对thrift不爽其实由来已久，协议很复杂，文档不完善，server端也写得不咋地 (TSimpleServer性能不能指望，TThreadPoolServer除了用SIGKILL以外的信号都不能kill掉，TNonblockingServer没能跑起来好像也不支持SSL)。
看上去很新颖的东西未必符合自己需要，不得不自己写一个替代品。

首先花了一日来研究ssl握手怎么在异步模式下工作。关键就是ssl.wrap_socket之后有一个do_handshake的步骤，
可以在wrap的时候通过参数让他不自动调用，然后在你的异步代码中把handshake显式整合进去。另外就是异步读写的时候，由于ssl socket是包装过的，i/o事件处理的时候需要捕捉SSL_ERROR_WANT_READ和SSL_ERROR_WANT_WRITE , 相当于普通socket的EAGAIN。

在我之前的写框架里面搞定异步ssl之后，做rpc的server其实就比较简单，可以用内省把函数加进服务端handle。客户端的封装还是费了一点心思。相比thrift需要预定义结构和接口生成代码, 自己做这些当然没什么必要，改成用pickle来序列化request和response。然而由于对象不方便序列化，调用的参数和返回值只能用普通的内置类型来做结构体，如果结构体层次复杂的话其实挺麻烦的，因为我不想写太多代码检查结构和捕捉异常，于是想到参考BeautilfulSoup的方式，写了一个结构体的wrapper，重载一些操作符，可以用比较懒人的方式访问：

    d = {'a': 1, 'b':2}
    e = AttrWrapper (d)
    print "e.name.name:", e.name.name 	# empty string
    print "e.a:", e.a, "e.b:", e.b   	# 1 2
    print "e:", e  						# {'a':1, 'b':2}  #但e是封装过后的对象
	print "e:", e.unwrap ().items()     # [('a', 1), ('b', 2)]  #unwrap之后获得原本的dict
    l = [1, 2, d]
    e2 = AttrWrapper (l)
    print "e2:", e2  					# [1, 2, {'a': 1, 'b': 2}]
    print "e2[0]:", e2[0]  				# 1
    #print e2[3] 						# IndexError
	print "e2[2]:", e2[2]				# 获得上面和d一样的封装过后的对象
    print "b:", e2[2].b 				# 2
    e_empty = AttrWrapper ({})
    print "e_empty:", e_empty  			# empty string
    assert not e_empty
    assert not e_empty.a
    #print e2.a   						# KeyError
    #print e[0]   						# IndexError
    #print e.a.b  						# KeyError


然后搞定之后花了一天来移植业务逻辑，没日没夜地搞了两天，上线之后看着各种日志在刷屏，有种淡淡的空虚。

这个框架的代码在[github python-socket-engine](https://github.com/frostyplanet/python-socket-engine), 见test/test_rpc.py

ps:  回头想了想，发现上面 print "e.name.name" 不抛异常是个错误的feature，因为这里的情景并不等同于访问层次很深的dom树，事实上e.name就应该返回None

