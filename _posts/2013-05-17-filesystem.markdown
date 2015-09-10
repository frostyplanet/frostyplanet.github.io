---
layout: post
categories: fs storage
date: 2013/05/17 23:20:00
title: 文件系统知识入门
---

不知道为什么会转头看文件系统，大概因为以前曾经试图看过reiserfs看不出个所以然来，完成一下心愿。因为一天之内看得太多，需要记一下，组织得有些乱。

Directory Index
--------------
[link1](http://ext2.sourceforge.net/2005-ols/paper-html/node3.html)
[link2](http://static.usenix.org/publications/library/proceedings/als01/full_papers/phillips/phillips_html/index.html)
早期的时候ext2为了实现简单，一个目录结构里面（文件/子目录）是挨个放在一起只能线性搜索的，当一个目录里面文件很多，缺点显而易见。所以开发者给打了一个patch，实现了一个类似B+树的Htree索引结构，
一直被ext2/3/4沿用下来。
key是文件名的hash，合并/分裂都和B+tree或者B*tree类似，叶子节点是正常的目录项，只不过存储的时候不排序，当分裂/合并的时候才需要重新排序。

相比之下，Reiserfs是B*树，也是使用固定长度的hash做键值。而JFS，是使用变长健的prefix B-tree,叶子节点是full-length。定长的hash key的好处也很显然，一是内部节点可以放很多个key，可以做到高扇出（因此树的高度可以更小）;
二是方便key对齐，键值拷贝的时候和查找都比较方便。

觉得要支持变长健的b树实现起来其实挺麻烦，除了prefix B-tree之外，参考论文"Pagination of B*-trees with variable length records", "efficient optimal pagination of scrolls"，还有[S(b)tree](http://people.apache.org/~shv/STrees.html)，一种目的是提高满度(存储更紧凑)的generialized B-tree。

The problem of inode limit on ext2/3/4
--------------------------------------
以前会遇到一个ext2文件系统inode用光了的情况，这是因为在mkfs的时候inode就预先分配了，可以看到[ext4 disk layout](https://ext4.wiki.kernel.org/index.php/Ext4_Disk_Layout#Layout)里头，有专门的inode bitmap和inode table，superblock里头s_blocks_per_group, s_inodes_per_group都是静态的。不像reiserfs、btrfs是可以动态分配的。

File-Block Mapping: Direct block, Indirect block
------------------------------------------------
传统的单层/二层/三层块映射表，在教科书里已经说得很多了，每个表项只能映射一个block。缺点就是如果有很大的连续空间，在映射表里头还是要老老实实地添加每个映射项，个数多的时候读取起来也慢。
所以为什么新的设计ext4、brtfs改成了extent tree的映射方式。

File-Block Mapping: Extent
---------------------------
有一系列图文并茂的文章讲到ext4上面的extent。

[digital forensics understanding ext4: part1](http://computer-forensics.sans.org/blog/2010/12/20/digital-forensics-understanding-ext4-part-1-extents)

[digital forensics understanding ext4: part3](http://computer-forensics.sans.org/blog/2011/03/28/digital-forensics-understanding-ext4-part-3-extent-trees)

[digital forensics understanding ext4: part4](http://computer-forensics.sans.org/blog/2011/04/08/understanding-ext4-part-4-demolition-derby)

也可以参考[具体的数据结构](https://ext4.wiki.kernel.org/index.php/Ext4_Disk_Layout#The_Contents_of_inode.i_block)

struct ext4_inode.i_flags  如果有标志位EXT4_INODE_EXTENTS就表示用extent来映射，否则就是传统的direct/indirect mapping。

ext4_inode.i_block(60字节)，由一个ext4_extent_header打头，根据其中的depth是否为0，决定后面的是最多四个的ext4_extent (extent树的叶子节点)，还是作为根的ext4_extent_idx(中间节点)。

其中叶子节点的ee_block表示数据的逻辑块号，ee_start_hi&lo表示映射的物理块号，由于ee_len是16位的，一个extent就可以映射2^16个块，比direct/Indirect mapping要节省很多用于元数据的空间。
。而中间节点的ei_leaf_hi/lo表示下面子节点的物理块号。

ext4的block allocate policy
------------------------------
1. allocate 8k to the file first, 

2. if initial 8k used up,  delay commit until timeout/ sync() / kernel out of mem

2. try to keep a file in the same block group as the directory

3. optimise locality of directory near root 

可惜没有文章讲的更详细，ext4为了保持向前兼容性，还是被一些过时的设计所限制，回头还是应该去看看brtfs是怎么做的。

