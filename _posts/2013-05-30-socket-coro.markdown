---
layout: post
title: python-socket-engine v1.1  with generator coroutine support
categories: coding
date: 2013/05/30 22:33:32
---

[项目页面](https://github.com/frostyplanet/python-socket-engine)

先说一下过往的历史。

这其实是一个自娱自乐的项目。刚开始的时候只是想做一个纯python的兼容多线程的异步通信框架。虽然现在python下面选择很多，写web的有各种wscgi的实现，通用的也有很多。
像标准库里面的SocketServer(最多只能加上ThreadingMixIn/ForkingMixIn)；
asyncore没有超时，伪异步模型，没有定长的读写函数，在一个handler里面实现复杂的状态机也是个很头痛的事情；
twisted，thrift等好像都是要把逻辑拆成transport/dispatch/server, 从设计模式上觉得很复杂不灵活，另外也局限于同步模式的效率问题。
而gevent打monkey patch是很方便，替换了threading api和读写操作，不过想起来以前在perl下面用Coro/EV/AnyEvent的时候有诡异的各种疑难杂证，这也不例外。

所以总体上没有一个合心水的。所以参照像POE那样的基于回调的样子，自己写一个，每个异步(connect/read/write/readline)的操作都有两个回调(ok/error)，自动对读/写/空闲连接做超时检查，出错的时候触发error回调并且自动close connection。
如果需要性能，那就把业务逻辑组织成串联起来的回调，如果需要短平快的实现，那就做线程池的同步实现，都十分灵活。
不过这也是写了很多服务端和客户端逐渐改进过来的，历时两年多了。

Change Log
---------------------

v0.3 用在配管项目，那个时候每个connection都有一个状态，底层的poll实现只能注册一种事件，读写还不能是同时的，开始的时候异步方式要比同步方式慢一倍。

v0.4 用在克隆项目，做了一个简单的异步http方式接口

v0.7 加了ssl支持和rpc client/server

v0.9 写了transwarp私有协议代理，改成每个connection读写可以不互相影响，网速快的时候可以跑出十多兆的流量吞吐，本地回环上的顺序读写测试可以达到千兆速率。异步回调方式已经可以和同步方式一样快了

v1.0 加了readahead的特性，最大限度减少了异步方式额外开销。

v1.1 加了coroutine支持。异步回调可以转变为在generator中yield的写法。

API约定
-----------------------

基本上每个server其实都有一个(或分阶段有多个) readable的handler，用来放主要的逻辑入口。新链接被listener accept之后，先经过new_conn_cb() 回调来根据自定义的逻辑过滤或握手，然后被put_sock（）放进poll里头等待i/o事件。

在可读时候就触发回调readable_cb()。然后经过一定的read和write操作，当要暂停处理的时候调用remove_conn（）从事件引擎中移除 (比如转交给线程池的worker做阻塞方式读写什么的) 。或者处理完毕再watch_conn() 把链接放进poll中等待readable事件。在watch_conn之后connection会变成idle状态，如果链接是长链接的话，也可以通过idle timeout超时来移除长期不活动链接。

如果是客户端方式的connection的话，里面相应的readable回调为空，读写操作和服务端一样。

性能优化思路
------------------------

对于python来说，每一次函数调用的开销都是可见的，系统调用的开销更大（因为异步模式需要register/unregister 事件和回调）。
所以为什么一开始的时候，异步回调的server会比同步模式的server要用额外的2/3到一倍的时间。越是内层的事件循环，每个语句的开销更加明显，有时候频繁访问的对象属性多一层引用，或者多几个额外的赋值，都会有影响。

往poll中register fd的开销是最明显的。所以每次read_unblock()/write_unblock()都要先尝试读/写，遇到EAGAIN之后才往poll中注册。还有注册fd的时候先看看自己维护的字典，如果注册的事件一样，那就只改回调函数，省去了重新注册fd。

对于已经完成的读操作，由于框架不知道后面是不是还要读，所以并不会自作主张的把已经注册的读事件注销，最初要使用者显式调用remove_conn()，但如果不注意的漏写了的话，在一些度操作之后可能由于写操作的时间比较长，先前注册的读回调被意外触发，这当然会引发诡异的程序行为。所以在v1.0版本中，通过在读操作完成之后，把先前注册的读回调替换成readahead的回调，readahead缓冲如果满了再注销。这样下次读的时候就直接从缓冲中取就行了。一举两得，上层既不用去管下层的事件register/unregister，同时也节省了一些系统调用的开销。而写操作，如果往poll中注册过事件的话，操作完了当然是要立刻注销的。

对于EPoll的edge-trigger特性，也会有问题需要避免。比如说system buffer中有50k数据可用，一次事件触发的时候由于业务逻辑的需要只读了20k，如果没有unregister fd的话，下一次watch_conn()之后可读事件并不会触发，所以在实现read_unblock的时候，总是比要求的长度去多读一点，多出来的放在readahead buffer中，下次watch的时候如果readahead buffer有数据的话，可以跳过register fd的步骤，立刻调用readable_cb() ，如果没有数据的话，证明上一次的已经读完了，这时候register fd是安全的。

关于回调的效率，由于有时候读写可以跳过poll的注册直接完成，直接执行的比间接回调要快，但是直接执行回调会容易出现递归过多超过了栈大小的问题，所以给每个connection对象保存了递归调用的计数，超过的时候就要把回调放进
一个pending的队列中等下一个poll循环触发。

暂时就想到这么多了。

关于EV、event等事件引擎
----------------------------

由于之前都是用select.poll和select.epoll来实现事件循环，试了一下pyev，不像通常认为的C实现的要比python快，反而比起先前的我用select.epoll写的模块没有性能优势，可能由于里面由于复杂的各种逻辑(timer/process watcher等)额外的开销, 另外EV为了安全起见使用的EPoll后端是level-trigger的，好像没有改成edge-trigger的选项。

写了这么久终于轮到协程了
-------------------------------

回调方式的异步的问题就是程序写起来不直观，如果有多个读/写操作的话，每一步都要拆成一个回调。有时候函数的参数也容易传错。更重要的是，抛异常的时候，如果刚好是被多个地方用到的共同的回调，
看堆栈信息很难找到出问题来源。

比如说看下面这样一个测试代码片段，是不是眼花了? 写过一次的代码都不想再写第二遍。而且回调定义的顺序需要和执行顺序倒过来。

    def test_connect (self):
        event = threading.Event ()
        err = None
        def __on_err (conn):
            print conn.error
            self.fail (conn.error)
        def __on_conn_err (e):
            err = "conn" + str(conn.error)
            event.set ()
        def __on_read (conn):
            print "read"
            self.client.engine.close_conn (conn)
            buf = conn.get_readbuf ()
            if buf == self.data:
                print "ok"
                pass
            else:
                err = "test connect error, invalid data received: %s" % (buf)
            event.set ()
        def __on_write (conn):
            print "write"
            self.client.engine.read_unblock (conn, len (self.data), __on_read, __on_err)
            return
        def __on_conn (sock):
            print "on connect"
            self.client.engine.write_unblock (Connection (sock), self.data, __on_write, __on_err)
            return
        self.client.engine.connect_unblock (self.server_addr, __on_conn, __on_conn_err)
        print "connect unblock"
        event.wait ()
        if not err:
            print "* test connect ok"
        else:
            self.fail (err)

而coroutine就是把同步过程异步化的最好的方式。但是我并不想用gevent或者evenlet这些侵入式的实现，如果有纯python的话就最好了。前天找了一下终于看到了[bluelet](git://github.com/sampsyo/bluelet.git)这个东西。众所周知，由于python generator只能保存一层堆栈，所以bluelet里面做了一个专门的事件循环来跟踪不同coro的关系。不过现有版本额外开销有点大，所以拿过来重写了一下，顺便和我的框架整合。
纯python实现的好处就是，多开若干真实的线程，把接受到的链接round robin到不同的线程中poll，或者用异步方式接收请求，把会阻塞的工作放到线程池中处理，可以最大限度的利用cpu。

对照上面的代码片段，coro方式的如下，应该看起来直观很多了。唯一要注意的是不能忘了yield。

    def test_connect (self):
        event = threading.Event ()
        err = None
        def _client ():
            _data = None
            try:
                engine = self.client.engine
                print "connecting"
                conn = yield engine.connect_coro (self.server_addr)
                print "connected"
                yield conn.write (self.data)
                _data = yield conn.read (len (self.data))
                conn.close ()
            except Exception, e:
                getLogger ("client").exception (e)
                self.fail (e)
                return
            if _data == self.data:
                print ("conn & read ok")
                event.set ()
            else:
                err = "test connect error, invalid data received: %s" % (buf)
                event.set ()
                self.fail (err)
            return
        self.client.engine.run_coro (_client)
        event.wait ()


coroutine模式性能测试
----------------------

环境intel i3-2310M (关了超线程)

阻塞方式4线程客户端，每个线程顺序读写100k数据并且循环50000次，测试从开始到结束的时间，多次测试取最快的一次，在本地回环接口测试。

服务端规定要每次接收到100k数据后才能完整写回。

异步回调方式的服务端 : 18.549s (v0.9), 17.954s (v1)

单线程阻塞方式的伪并发服务端: 19.98s (v0.9), 18.68s (v1)

纯python协程的服务端:  优化前（基于bluelet的原有代码） 31.154s,   优化后(v1.1) 21.735s 

所以还是有一些额外开销，暂时比较满意了。


