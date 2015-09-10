---
layout: post
categories: distributed db storage
date: 2013/05/07 19:52:00
title: vector clock and eventual consistency
---

学习了一下Amazon Dynamo的文章。因为分布式系统CAP原理的限制，他们选择了损失一致性的前提来把可用性做到最大，强调的是always writable，还有根据业务动态调整Quorum NRW参数，可以想象为什么当时会引起很大的轰动。

既然是最终一致性，也就只要并发存在，数据肯定存在不一致的时候。出现多个版本的数据的情况至少有两种：
1. 不同的客户端并发读写, 2. 在最新的数据还没有传递到相应副本节点的时候，这个节点被选择成为协调写操作的节点。

跟踪多个版本，用到了vector clock。(Vector Clock的研究有很多，可以看[A Practical Tour of Vector Clock Systems](http://net.pku.edu.cn/~course/cs501/2008/reading/a_tour_vc.html)这篇survey)。不过需要注意的是和[wikipedia](http://en.wikipedia.org/wiki/Vector_clock)上面说的并不太一样。数据库只在写入的时候增加vector clock, 一个value的数据副本发送到其他节点的时候，附带的vector clock是不会增加的。一个节点上收到多个版本的数据的时候，如果任意两个版本中存在偏序，小的那个可以被丢弃；如果判断不了，就交给客户端在读取的时候去由业务逻辑做合并。

比如说，下面的偏序关系是成立的： 

	A1 < A1B1,  B1 < A1B1,  A1B1 < A1B2

但是当有A1B1和A1C1的时候，就意味着存在冲突了，由客户端逻辑合并之后的版本就是A1B1C1，覆盖掉之前的A1B1和A1C1。

如果把vector clock中的版本方式换成一定精度的时间戳，也是可以的，而且让解决冲突的时候比较方便一些，当vector clock变得过长需要截断的时候也方便一些。

另一个基于dynamo思想的k-v存储Riak，他们有两篇vector clock的文章也说的比较显浅[why-vector-clocks-are-easy](http://basho.com/why-vector-clocks-are-easy/)，[why-vector-clocks-are-hard](http://basho.com/why-vector-clocks-are-hard/)。只不过他们的不同之处是将vector clock暴露给客户端，用client-id去作向量的维度，区别于dynamo论文说的用coordinator的node-id作维度。不过我觉得这种方法在把复杂性暴露给客户端，还有在client很多的情况下让vector clock变得很大这些缺点，优点其实可以忽略掉。
我觉得既然放弃了事务特性而选择了最终一致，那就意味着业务逻辑必须要接受一致性损失。如果非要纠结的话，那还不如读写串行化或者改用其他支持事务的实现来得更实际。


