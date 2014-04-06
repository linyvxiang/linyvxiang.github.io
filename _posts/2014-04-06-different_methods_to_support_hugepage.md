---
layout: post
title:	Using 4KB Page Size for Virtual Memory is Obsolete
category:	大内存页，内存管理
---
## Using 4KB Page Size for Virtual Memory is Obsolete
&emsp;一个水会上的post，没什么创新点，而且结论也比较扯，读它的目的主要是为了review一下对于Hugepage支持的几种基本的方法。

- Allocation 

&emsp;  在产生pagefault时即将一整个大页的各个subpages全都load进来。然后将这个大页看成一个整体填到页表中去。Solaris MPSS采用的就是这种方法。在IRIX和HP-UX中也都采用了在缺页时即映射整个大页的方法。但是在IRIX中，所期望的页大小是可以通过用户进行系统调用的方式，在编译的时候就确定。在HP-UX中，页的大小可以被用户指定，或者通过系统根据进程所需的内存大小来进行动态的调整。

-  Reservation
 
&emsp;在基于reservation的大页分配中，当缺页时，首先先"referencing an address in a superpage"，但是这个大页其实并没有真正的被分配，即相当于先占了个坑。当属于这个大页的所有base page都（或者达到了一个 比例)fault in时，这些base page才真正的被promote成一个大页。这种方法在Alan Cox的那篇OSDI‘02用到过。显然，相对于Allocation日的方式，这种方式的优点是既可以使用大页，又可以减小没有完全使用大页时进程的memory footprint。当然，由于不是一次性载入，产生了更多的page fault，这会对速度有一定影响。

- Relocation

&emsp;在基于relocation的模式中，base page还是像往常一样fault in，但是内存的使用情况会被跟踪，一旦发现如果某些页被频繁访问，那么便认为这时有创建一个大页的必要。这时，将会去寻找一片连续的空闲物理内存页，然后将这些页迁移过去。
  