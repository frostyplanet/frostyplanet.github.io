---
layout: post
categories: fs storage
date: 2013/05/18 18:30:00
title: 文件系统知识入门 (reiserfs3.6)
---

昨晚睡觉之前看了一下[a short history of brtfs](http://lwn.net/Articles/342892/)写得非常好的一篇文章，有种眼前一亮认知被刷新了的感觉。

如果说ext是来自过去，brtfs/ceph是未来，reiserfs的设计还是足够现代的 (忽略政治因素和个人因素)，关于文件系统树，关于packing small objects into a block这些理念都能在brtfs里面找到。所以我还是觉得应该回头重新了解一下reiserfs3.6，虽然namesys的官网都已经不存在了，and reiser4 will probably never make it to the mainline。

说到底，数据库设计有CAP的矛盾，文件系统里面也有各种矛盾：locality，碎片和元素据等空间利用率，查找效率，小文件效率，大文件吞吐...所以应该带着这些问题来看。

------- 不太自然的分割线----------

层次
--------
在ext文件系统中，有block group，inode这些比较固定的层次。inode都只能是block aligned的，所以存在利用率不足、inode用光了这些尴尬的问题。目录索引、日志这些都是后来才加上去的，也有挺别扭的各种特性开关。
reiserfs里面能看到除了超级块、用于标记空闲的bitmap块、日志块之外，其他所有的东西都作为对象放在b树里头。(其实reiserfs称之为s+tree, 在内核代码里面有stree.c)
而brtfs更进一步，所有东西都在cow的b树里头。

文件系统树
--------------------
block按照在b树中的层次分internal node和作为叶子的formated node，都是以一个块为长度。其实要较真的话的确比普通b+树有些变化。

<b>internal node的块</b>是一个标准的b树节点的样子, 其中的pointer指向下层的块。
<table>
<tr bgcolor="#FFCCCC">
<td bgcolor="#99FFCC">&nbsp;Block_Head&nbsp;</td>
<td bgcolor="#FFCCFF">&nbsp;Key 0</td>
<td bgcolor="#FFCCFF">&nbsp;Key 1</td>
<td bgcolor="#FFCCFF">---</td>
<td bgcolor="#FFCCFF">&nbsp;Key N</td>
<td bgcolor="#FFCCCC">&nbsp;Pointer 0</td>
<td>&nbsp;Pointer 1</td>
<td>---</td>
<td>&nbsp;Pointer N</td>
<td>&nbsp; Pointer N+1</td>
<td bgcolor="#FFFFCC">....Free Space....</td>
</tr>
</table>

<b>formated node是树的叶子节点。</b>
<table>
<tr>
<td bgcolor="#99FFCC">&nbsp;Block_Head&nbsp;</td>
<td bgcolor="#FFCC99">&nbsp;IHead 0</td>
<td bgcolor="#FFCC99">&nbsp;IHead 1&nbsp;</td>
<td bgcolor="#FFCC99">&nbsp;IHead 2</td>
<td>---</td>
<td>&nbsp;IHead N</td>
<td bgcolor="#FFFFCC">....Free Space....</td>
<td>&nbsp;Item N&nbsp;</td>
<td>---</td>
<td>&nbsp;Item 2</td>
<td>&nbsp;Item 1</td>
<td>&nbsp;Item 0</td>
</tr></table>

在这样一个formated node里面可以pack很多个item，item是排序的，可以看到head和body是分开放在block的两头，顺序相反向中间增长。head里面有指针到后面的body。
struct block_head里面的blk_right_delim_key就是作为叶子节点的key，应该就是这个block里面最后一个item的key (我猜的)，不过从其他文档说
已经不用了(only kept for compatibility), 其实也可以根据节点里面最后一个item得知。

所有formated node的路径(path)高度都是相同的。而unformated node(用来放大文件的数据)是挂在formated node中的indirect item下面，所以path的长度+1。

item分类
--------

* direct tems  :  tails of files
* indirect items  N * (point to unformated node) (file blob)
* directory items :  deHead * N + filename * N 
 N = item_head.ih_entry_count
* stat item: 文件元数据

Files consists of:

* StatData item + [Direct item]                      (for small files : size from 0 bytes to MAX_DIRECT_ITEM_LEN=blocksize-112 bytes)

or

* StatData item + multiple InDirect item  + [Direct item]      (for big files     : size    > MAX_DIRECT_ITEM_LEN bytes)

Directory consists of:

* StatData item + multiple Directory item 


Key
-------
这里贴的是version2的格式
<table border="1">
<thead>
<tr>
	<th> Name </th>
	<th> Size </th>
	<th> Description </th>
</tr>
</thead>
<tbody>
<tr>
	<td> Directory ID </td>
	<td align="center">  4 </td>
	<td>  the identifier of the directory where the object is located </td>
</tr>
<tr>
	<td> Object ID </td>
	<td align="center">  4 </td>
	<td>  the actual identifier of the object ("inode number") </td>
</tr>
<tr>
	<td> Offset </td>
	<td align="center">  60 bits </td>
	<td>  the offset in bytes that this key references 一个文件或目录用多个item来表示，offset会递增 </td>
</tr>
<tr>
	<td> Type </td>
	<td align="center">  4 bits </td>
	<td>  the type of item. Possible values are:<br>Stat: 0<br>Indirect: 1<br>Direct: 2<br>Directory: 3<br>Any: 15 </td>
</tr>
</tbody>
</table>

 Only stat items have an offset of 0. Files (direct and indirect items) and directories always start with an offset of 1 so that they are sorted behind the stat item in the leaf nodes. For directory items the "offset" field contains the hash value and generation number of the leftmost directory header of the directory item (see below), not the offset in bytes.

可以看到，根据b树的特性，同一个目录下面的文件和目录会在索引中邻近，并且会按照在父目录中的位置排序。这里面object id指的就是unix中的inode编号。

Node Layout和tree ordering是值得好好看的一部分
[有时间再待续]

