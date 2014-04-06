---
layout: post
title:	Practical, transparent operating system support for superpages
category:	大内存页，内存管理
---
## Practical, transparent operating system support for superpage
&emsp; 在读上一篇VSwapper时发现作者引用了一篇文章，讲大内存页的，刚好这阵子观察到LKML里面对于THP的讨论也很热，于是决定读一下。下载之后发现作者中居然有Alan Cox，吓尿了：原来他还发文章的啊，而且居然是OSDI上的！ 那这篇文章更应该好好读一下了。

&emsp; 内存技术不断发展，内存容量越来越大，但是，TLB并没有得到相应的发展速度，于是产生了一个问题：必然对于同一个TLB entry，所需要映射的内存变的更多了。这样，miss率也相应的上升，引起性能的下降。于是现代处理从硬件上开始支持大页，而且支持多种尺寸的大页，然而页太大了也不好：太大了之后，进程缺页时，有可能实际只需要4KB，但一下子映射了64KB给它，浪费掉了。再者，如果是从磁盘往内存读文件的话，一次64KB显然大大增加了IO开销，而且，大页很容易导致内存碎片的问题。这样看来，不用大页不行，只用大页也不行，一个显然的trade-off。那么肯定是只用小页不行，只用大页也不行，就看怎么去平衡了。

&emsp; 上面便是这篇文章的主要的motivation，另外还提到了一点：现在处理器的Cache做的比以前好多了，也大多了。TLB miss的时候，如果所需页实际上在内存或者磁盘里还好，因为本来读写这一页就需要多耗些时，可是万一这一页的内容根本就在CPU Cache里，那就真悲剧了：本来在Cache里的啊，结果被TLB拖慢了。

&emsp; 当然，这个部位的问题，也不一定完全是操作系统层能够解决的，于是有人就开始做了：干嘛要求一个superpage中必需所有base page都是可用的？像filesystem或者内存里那样，允许有memory hole的情况出现不就好了嘛。还有人说了，虚拟地址连续物理地址就一定连续吗？我们在MMU转换地址的时候再加一层不就好了嘛。有两篇ISCA是做这个的，后面读一下，但是文章里说了，这两种做法并没有得到商用。

&emsp; 于是这篇文章的诸位大牛就开始想办法了，而且想的非常全面，不得不佩服的确是对真正的OS了解的人，绝对不是闭门造车搞出来的忽悠人的想法：

- Reservation-based allocation   在产生一个page fault时，先给后面有可能被再次被page fault到的虚拟地址预留一定的物理页，注意只是预留，并没有真正的map。那究竟应该预留多少？下面会讲。
- Preferred superpage size policy  应该预留多少？如果预留小了，不好，因为后面的物理页可能接下来被别人占用，如果到时候再想扩大地盘的时候，已经来不及了。 1）如果所分配的页实际上是给一个fixed in size的目标去用的话，只要大的程度包含所需要的物理页，不妨碍到别的地方，而且不超过object的大小，那越大越好。2）像堆或者栈内存，刚才那个取最大可能的superpage进行预留就不好了：因为这本来发起的申请就是一页内存，而且“不能超过object的大小“也不合适了：本身就是在扩地盘。作者的做法是”the size of this superpage is limited to the current size of the object."
- Preempting reservations  既然有所保留，那么就有可能保留错。保留错了就需要允许别人在需要的时候把其它人保留的东西抢走。当物理内存紧张，或者内存碎片过多时，这种抢占就会发生。文中采用的便是当发现新的superpage allocation不能满足时，将已有的保留区域抢占掉，对于被抢占区域的选择，使用LRU策略。当然，还有一种做法是，后面的如果发现不能保留，那么便直接退回到小内存页。
-  Fragmentation control 随着机器的运行，内存中必然产生越来越多的碎片。 FreeBSD中存在三个链表：active list, inactive list, cache list，其中cache list中的内存页是没有经过任何映射的，因此可以很容易的被回收。为了达到最多的可用大页，文章中将cache list中的页也聚焦到了空闲链表中，这样，如果分配时的大页包括cache中的页，那便可以直接分配，注意这里只是预留下来，如果随后这个page cache页被用到了，那么只需将保留的大页的规模缩小，如果没被用到，那么可以在其它进程用到时直接映射，因为它里面的内容是clean的。另外，在FreeBSD下面，如果一个进程结束执行了，那么属于这个进程的那些page cache并不会被立即清理掉，而是等有memory pressure时再进行清理，文章对于这里做了改进，一旦进程退出执行，并且它所打开的 file object引用数为0，那么这个文件所对应的address space里的page cache页会被全部deactive掉。
- 虽然从一开始就按superpage来保留了内存，可是有可能用着用着发现还不够用，那么这时候就需要在原有的保留的基础上，再往外扩一部分。那么在什么时机开始扩？文中给出了两种方法：在现有保留完全被用完的情况下，开始往外扩，或者当现有保留被使用了一定比例的时候往外扩。实际做测试时是用的前者。
-  Paging out dirty superpages  前面提到过，page out superpage有一个很大的问题就是，由于一个大页只有一个脏比特位，那么当这个大页是部分脏的时候，在写出时就会导致很多不必要的IO， 文中的解决方法是：当一个干净的大页被写时，立即将其分解成若干base pages。还提到了另一个方法，就是在建立每个页映射时，都计算一个SHA-1值，由于SHA-1算法的冲突概率极小，那么在写出时再计算一下该页的SHA-1值，如果发现两个值相等，那么就直接停止对这个页的IO。但是作者说他们试了下，overhead太大，都15%了。  其实如果这个功能有硬件支持的话，我感觉倒挺好的。这让我想起了live migration中的XOR压缩算法，异曲同工之妙，那篇文章发到VEE上去了。

&emsp; 实验是在FreeBSD上面做的。。。大牛就是厉害，原来FreeBSD也在玩。。。得出来最好的效果是一个做矩阵转置的benchmark，speed up 居然有7.54x之多！


