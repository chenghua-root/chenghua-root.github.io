---  
layout: post  
title:  "Linux内存管理"  
date:   2019-09-08 16:48:00 +0530   
---  
  
<style>  
.tablelines table, .tablelines td, .tablelines th {  
  border: 1px solid black;  
  }  
</style>  
  
## 背景  
美团云图片服务(与OpenStack::Swift混合部署)在20180529遇到个别节点延迟增加的情况，查看现象为CPU系统使用率偏高。  
perf采样如下：  
![图片服务异常火焰图](https://chenghua-root.github.io/images/memory-image-perf.png)  
基于火焰图中的compact_zone()，结合网上资料分析是由于没有连续的大块内存来满足内存大页分配导致。  
关闭透明内存大页后现象消失，系统恢复正常。  
- echo never > /sys/kernel/mm/redhat_transparent_hugepage/enabled  
- echo never > /sys/kernel/mm/redhat_transparent_hugepage/defrag  
- echo no > /sys/kernel/mm/redhat_transparent_hugepage/khugepaged/defrag  
  
事后作者做了关于内存大页的分享。  
理解为什么需要内存大页和内存大页的实现原理，需要了解虚拟内存机制，在准备分享过程中收集了如下资料供自己参考。  
后面图片均为网图。  
文末附有透明内存大页的测试。
  
## 概述  
进程管理、虚拟内存和文件系统是单机系统最重要的几个底层原理。本文主要讲解虚拟内存机制。  
  
虚拟内存由底层硬件和操作系统两者软硬件结合来实现，是物理内存、内存地址、地址翻译、磁盘文件、系统内核和进程空间的交互。主要提供3个能力：  
  
1. 给所有的进程提供一致的地址空间，每个进程都认为自己是在独占使用单机系统的存储资源。  
2. 保护每个进程的地址空间不被其他进程破坏，隔离了进程的地址空间。  
3. 内存或内存结合磁盘为进程提供高速的存储访问。  
  
虚拟内存机制主要包括如下几块内容：  
1. 虚拟内存和物理内存  
2. 地址映射  
3. 物理内存分配  
4. 虚拟内存分配  
5. 内存大页  
6. page cache  
  
  | 名词 | 含义|  
  | -----  | ----  |  
  | x86 | x86系统，32位x86系统 |  
  | x86-64 | 64位x86系统 |  
{: .tablelines}  
  
## 虚拟内存描述  
  
### 虚拟地址空间  
虚拟地址：  
- 每个进程使用的都是虚拟地址，所有进程看到一样的虚拟地址空间。  
- 32位系统的虚拟地址空间是0~2^32；  
- 64位系统的虚拟地址空间为0~2^48(64位系统：48位地址总线，64位数据总线)。  
  
Linux内核把虚拟地址空间划分为两部分：用户地址空间和内核地址空间。  
  
32位系统(3G/1G划分)：  
- 用户空间3G：0x0000,0000 - 0xBFFF,FFFF  
- 内核空间1G：0xC000,0000 - 0xFFFF,FFFF  
  
64位系统：  
- 用户空间128T：0x0000,0000,0000,0000 - 0x0000,7FFF,FFFF,FFFF(高16位与第48位都为0)  
- 内核空间128T：0xFFFF,8000,0000,0000 - 0xFFFF,FFFF,FFFF,FFFF(高16位与第48位都为1)  
  
32位与64位系统具体地址分布如下：  
![32位与64位系统地址分布](https://chenghua-root.github.io/images/memory-virtual-space02.png)  
  
32位系统详细内存空间划分如下：  
![32位地址分布](https://chenghua-root.github.io/images/memory-32-addr.png)  
  
地址空间与数据结构关系：  
![地址空间与数据结构关系](https://chenghua-root.github.io/images/memory-mm-struct.png)  
- 每个进程包含一个task_struct结构，task_struct包含一个mm_struct。mm_struct管理进程内的所有内存。mm_struct包含多个vm_area_struct。每个vm_area_struct管理一段地址空间，分别有代码段、数据段、BSS段、堆、一个或多个MMap段、栈。  
  
### 页面和页表  
- 系统把虚拟地址空间划分为虚拟页（后面简称页或页面），以页为单位进行划分，页面大小为4KB或2MB。若页面大小为4KB，32位系统包含2^32/4K = 2^20 = 1M个页面；64位系统包含2^48/4K = 2^36个页面。  
- 系统通过页表对页面进行管理，页表是存放在主存中的，每个进程包含一个单独页表。  
- 每个页面对应一个页表项(PTE)，32位系统的每个页表项占用4个字节。那么页表需要2^20个\*4字节= 4MB的空间。  
- 64位系统页表项假设也为4个字节，那么页表需要2^36 * 4字节 = 2^38 = 256GB的空间。如果保存所有的页表项，空间开销太大。  
- 由于页表项加载是按需加载的，没有分配的虚拟页不需要建立页表项。所以我们可以采用多级页表的形式。即没有分配的页面不需要建立页表项。  
- 32位系统采用两级页表结构：32 = 10(页目录) + 10(页表) + 12(页内偏移量OFFSET)  
- 64位系统(inter Core i7)采用4级页表结构：48 = 9(1级页目录PGD) + 9(2级页目录PUD) + 9(3级页目录PMD) + 9(4级页表PT) + 12(页内偏移量OFFSET)  
  
## 物理内存描述  
### 物理内存的划分  
- Linux将物理内存按固定大小的页面(参考虚拟页，如4K)划分内存，在内核初始化时，会建立一个全局的struct page结构的数组mem_map[]。如系统有76G物理内存，则物理内存页面数为76\*1024^3/4K =  19\*1024\*1024个页面，mem_map[]的长度为19922944，即数组中的每个元素和物理内存页面一一对应，整个数组就代表着系统中全部物理页面。  
- 在服务器中，存在NUMA架构和UMA架构，Linux将NUMA中内存访问速度一致的部分称为一个节点（Node），用sturct pglist_data(typedef pg_data_t)表示。每个节点通过pg_data_t->node_next连接起来。  
- UMA指所有CPU访问的内存都是一致的，无论是访问速度还是范围。NUMA指每个CPU除了包含一致的内存外，还有不一致的访问内存。如某些或全部CPU还有自己独立的访问内存。  
- 每个节点又进一步划分为许多块，称为区域（Zones）。区域表示内存中的一块范围，区域用struct zone_struct(typedef zone_t)数据结构表示。  
  
每个区域（Zone）中有多个页面（Pages）组成。节点、区域、页面三种关系如下图：  
![节点区域页面](https://chenghua-root.github.io/images/memory-node-zone-page)  
  
### Zone分类：  
![](https://chenghua-root.github.io/images/memory-zone.png)  
- 上图中包含两个node，分别为node0和node1，每个node包含不同的zone。  
  
ZONE_DMA：是低内存的一块区域，由标准工业架构（Industry Standard Architecture）设备使用，适合DMA内存。x86架构中，该部分区域大小限制为16MB(24位DMA地址总线)。  
  
ZONE_DMA32：该部分区域为支持32位地址总线的DMA内存空间。仅在64位系统有效，32位系统中，这部分区域为空。在x86-64架构，这部分区域范围为16MB~4GB。  
  
ZONE_NORMAL：属于ZONE_NORMAL的内存被内核直接映射到线性地址。这部分区域仅表示可能存在这部分区域，如在64位系统中，若内存只有4GB物理内存，则所有的物理内存都属于ZONE_DMA32，而ZONE_NORMAL区域为空。  
  
ZONE_HIGHMEM：是系统中剩下的可用内存，但因为32位内核地址空间有限，这部分内存不直接映射到内核。  
  
在x86架构中32位系统中内存有三种区域：ZONE_DMA，ZONE_NORMAL，ZONE_HIGHMEM。  
不同类型的区域适合不同需求。32位3G/1G地址空间划分时，三种类型区域如下:  
  - ZONE_DMA   0 ~ 16MB  
  - ZONE_NORMA  16MB ~ 896MB  
  - ZONE_HIGHMEM  896MB ~ 结束  
  
64位系统中，内存三个区域为DMA、DMA32和NORMAL：问题：为什么64位系统中没有ZONE_HIGHMEM  
  - ZONE_DMA  0 ~ 16MB  
  - ZONE_DMA32  16MB ~ 4GB  
  - ZONE_NORMAL   4GB ~ 结束  
  
### Zone的类型：  
zone的迁移/整理类型是指对这个zone下面的页面的操作限制：  
  - Unmovable:  页框内容不可移动，在内存中必须固定，核心内核分配的大部分页面  
  - Reclaimable: 页框内容可回收，不能直接移动。如可以swap到磁盘。  
  - Movable: 页框内容可移动，属于用户空间应用程序的页面，通过页表映射，因此只需要更新页表项，并把数据复制到新位置就可以了。如物理页碎片整理。无论是在DMA区域或者Normal区域，movable类型的内存最多。  
  
## 地址映射  
前面分析了虚拟内存和物理内存，虚拟内存相当于是一种抽象，现在需要把抽象的虚拟地址映射到真实的物理地址上。本节主要分析虚拟地址到物理地址的映射。  
  
相关概念：  
  - 段页式内存管理：Linux只采用了页式内存管理，参考上面的虚拟内存描述。  
  - 线性地址：也称为虚拟地址，在32位系统中，采用无符号整形，最大可以达到4GB。  
  - 物理地址：真正物理内存上的地址。  
  - 控制寄存器：CR0、CR2、CR3和CR4。其中页面目录基地址存放在CR3中。  
  
### 线性地址到物理地址的映射  
映射步骤：  
1. 从CR3寄存器中获取页面目录(一级页表、页全局目录)的基地址；  
2. 以线性地址的第一级为下标，在目录中取得相应页目录的基地址；  
3. 以线性地址中的第二级、第三级....为下标，取得对应页目录的基地址；  
4. 在最后一级（32位系统为第二级，64位系统为第四级）为下标，在所得到的页表中获得相应的页面描述项；  
5. 将页面描述项中给出的页面基地址与线性地址中的offset位相加得到物理地址。。  
  
32位系统线性地址到物理地址的转换：  
![](https://chenghua-root.github.io/images/memory-mapping-32bit.png)  
  
64位系统线性地址到物理地址的转换：  
![](https://chenghua-root.github.io/images/memory-mapping-64bit-v2.png)  
  
CR3寄存器的值是从哪里设置的？  
- 内核在创建进程时，会分配页面目录，页面目录的地址就保存在task_struct结构中，task_struct结构中有个mm_struct结构类型的指针mm，mm_struct结构中有个字段pgd就是用来保存该进程的CR3寄存器的值。  
  
问题：  
1. 基地址和物理地址的关系，基地址如何映射物理地址，PAGE_OFFSET  
2. 每个用户进行地址转换的时候，查询的页目录表和页表，是存在内核空间还是用户空间？这些表是所有用户进程共享，还是独立的？(**每个进程都有一个页面目录（多级页表），存放在内核地址空间。**)  
3. 页面表项是在什么时候创建的  
  
### TLB  
TLB（Translation lookaside buffer，页表寄存器缓冲）  
由上一节可知，页表是被存储在内存中的。我们知道CPU通过总线访问内存，肯定慢于直接访问寄存器的。而做一次地址转换需要访问四次内存(64位机器)，开销很大。  
为了进一步优化性能，现代CPU架构引入了TLB，用来缓存一部分经常访问的页表内容。  
![](https://chenghua-root.github.io/images/memory-tlb.png)  
  
问题：  
1. 64位系统的地址转换速度是否比32位系统慢?  
  
## 物理内存的分配  
前面分析了虚拟内存、物理内存以及它们之间的映射。现在我们开始使用内存，即申请内存。当我们调用malloc或mmap的时候，会在进程地址空间申请一段内存空间，在映射过程中就需要为这段申请的内存空间分配对应的物理内存，本节主要分析物理内存的申请。  
  
物理内存的分配由内核进行管理。在用户态C语言中，我们对内存分配使用malloc()或calloc()函数。在内核态则没法使用它们，内核态有专门的内存申请和释放函数。如alloc_pages()，free_pages()。  
  
### 空闲页面的管理  
  
在讲物理内存分配之前，我们先了解一下空闲物理内存的管理。  
  
在Linux内核中，空闲内存管理的基本单位是页面/页帧/物理页，即以页面为单位来管理物理内存。  
  
Linux内核管理的每个内存空闲块都是2的幂次方个物理地址连续的页面，幂次方的大小为order。把1个空闲页面的放在一起，2个空闲页面（物理地址连续）放在一起，4个空闲页面（物理地址连续）放在一起……2^MAX_ORDER-1个页面（物理地址连续）放在一起。  
  
空闲页面组织，如下图所示：  
![](https://chenghua-root.github.io/images/memory-free-page-manage.png)  
- 在版本2.6.32中，MAX_ORDER定义为11，内核管理的最大连续空闲物理内存大小为2^10个页面，即为4MB。  
  
**区域（Zone）与空闲页面：**  
在区域的数据结构中，有个数组free_area[MAX_ORDER]来保存每个空闲内存块链表：  
```  
struct zone {  
  struct free_area free_area[MAX_ORDER];  
  unsigned long padding[16];  
};  
```  
这样free_area[MAX_ORDER]数组中的第1个元素，指向内存块大小为2^0即1个页面的空闲页面链表；数组中的第2个元素指向内存块大小为2^1即2个页面的空闲页面链表。  
  
数据类型free_area结构体定义如下：  
```  
struct free_area {  
  struct list_head free_list[MIGRATE_TYPES];  
  unsigned long nr_free;  
};  
  
```  
- free_list：空闲页面块的双链表；  
- nr_free：该区域中的空闲页面块数量；  
- MIGRATE_TYPES：常量：不同类型的内存区域个数。参考上面截图cat /proc/pagetypeinfo  
  
每个空闲页面链表上各个元素（大小相同的连续物理页面），通过struct page中的双链表成员变量来连接，如下图所示：  
![](https://chenghua-root.github.io/images/memory-free-page-link.png)  
  
上面提到Linux内核描述物理内存有三个节点：节点、区域和页面。空闲页面的管理只是在区域（Zone）这一层，节点（Node）下的每个区域都管理着自己的空闲物理页面。空闲页面管理与节点、区域之间的关系如下图：  
![](https://chenghua-root.github.io/images/memory-free-page-manage-in-zone.png)  
  
### 伙伴算法  
伙伴系统（Buddy System）在理论上是非常简单的内存分配算法。它的用途主要是尽可能减少外部碎片（external fragmentation），同时允许快速分配和回收物理页面。为了减少外部碎片，连续的空闲页面，根据空闲块大小，组织成不同的链表。前面介绍的空闲物理页面管理就是伙伴系统的一部分。  
  
当一个需求为申请4个连续页面时，检查order=2下面是否有空闲块。若该链表上有空闲块，则分配给用户，否则向下一个级别（order）的链表中查找。  
  
回收：标记页面为空闲块，若相邻物理页面为空闲，则尝试合并生成更大的连续物理页面块；若有合并，则要更新free_area[]中链表元素。更新相关统计信息。  
  
Buddy系统信息查看：  
![](https://chenghua-root.github.io/images/memory-buddy-info)  
  
### 内存分类  
内存可以根据使用类型来进行分类：  
 - 匿名内存：Anonymous。普通malloc申请的内存。  
 - 基于文件的内存：File-Back。有对应文件的内存，如代码段，swap内存，page cache内存。  
  
## 虚拟内存分配  
应用程序分配内存使用malloc()/free()或者mmap()/unmmap()。  
malloc()使用sbrk()和brk()系统调用或者mmap()/unmmap()系统调用。申请内存小于128KB（可设置）时使用sbrk()/brk()，大于等于128KB时使用mmap()。  
sbrk()和brk()既可以申请内存也可以释放内存。  
用户层也可以直接调用sbrk()、brk()和mmap()。  
内核层使用vmalloc()、kmalloc()、get_free_pages()。  
  
**mmap & mm_struct & vm_area_struct**  
- malloc()或mmap()操作都是在用户虚拟地址空间中分配内存块，但这些内存在物理上往往都是离散的。  
- 这些进程地址空间在内核中使用struct vm_area_struct数据结构来描述，简称VMA，也被称为进程地址空间或进程线性区。  
- task_struct中的一个条目指向mm_struct，它描述了虚拟存储器中的当前状态。其中pgd指向第一级页表(页全局目录)的基址，而mmap指向一个vm_area_struct(区域结构)的链表，其中每个vm_area_struct都描述了当前虚拟地址空间的一个区域(area)。当内核运行这个进程时，它就将pgd存放在CR3控制寄存器中。  
- 线程栈也由mmap()分配。主线程调用pthread_create创建线程时由mmap()分配创建线程栈空间，这些线程栈在mmap的区域内。
![](https://chenghua-root.github.io/images/memory-mmap.png)  
  
问题：  
1. 段错误是如何产生的：访问不存在于所有vm_area_struct中的虚拟地址?  
2. protection exception产生：如修改代码段(只读段)所在的vm_area_struct?  
  
**malloc & brk**  
- 用户调用malloc库函数或mmap系统调用申请内存。
- malloc依赖brk系统调用或mmap系统调用，当申请内存小于等于M_MMAP_THRESHOLD(128KB)时使用brk，大于M_MMAP_THRESHOLD时使用mmap.
  
示例：  
1. 分配A=30KB，分配B=40KB  
![](https://chenghua-root.github.io/images/memory-malloc-01.png)  
2. 分配C=200KB，分配D=100KB，释放C=200KB  
![](https://chenghua-root.github.io/images/memory-malloc-02.png)  
3. 释放B=40KB，释放D=100KB，内存紧缩（空闲内存超过128KB）  
![](https://chenghua-root.github.io/images/memory-malloc-03.png)  
  
**Slab & kmalloc**  
- slab和kmalloc为内核的内存分配。  
- 伙伴系统以page为单位进行操作。但内核在很多场景并不需要如此大的内存分配，slab就是用在这种场景。  
- slab分配器最终还是由伙伴系统来分配出实际的物理页面，只不过slab分配器在这些连续的物理页面上实现了自己的算法，以此来对小内存块进行管理。  
- kmalloc函数基于slab机制，分配的内存大小也是对齐到2^order个字节。  
  
**vmalloc**  
- vmalloc用于分配虚拟地址连续的内核内存空间，物理上不要求连续。  
- 同时vmalloc分配的虚拟地址范围在VMALLOC_START/VMALLOC_END之间。  
  
**总结**  
![](https://chenghua-root.github.io/images/memory-page-alloc.png)  
  
## 内存大页  
在 Linux 操作系统上运行内存需求量较大的应用程序时，由于其采用的默认页面大小为 4KB，因而将会产生较多 TLB Miss 和缺页中断，从而大大影响应用程序的性能。当操作系统以 2MB 甚至更大作为分页的单位时，将会大大减少 TLB Miss 和缺页中断的数量，显著提高应用程序的性能。  
  
使用 hugetlbfs(内存大页) 之前，首先需要在编译内核 (make menuconfig) 时配置CONFIG_HUGETLB_PAGE和CONFIG_HUGETLBFS选项，这两个选项均可在 File systems 内核配置菜单中找到。  
内核编译完成并成功启动内核之后，将 hugetlbfs 特殊文件系统挂载到根文件系统的某个目录上去，以使得 hugetlbfs 可以访问。命令如下：  
- mount none /mnt/huge -t hugetlbfs  
  
此后，只要是在 /mnt/huge/ 目录下创建的文件，将其映射到内存中时都会使用 2MB 作为分页的基本单位。值得一提的是，hugetlbfs 中的文件是不支持读 / 写系统调用 ( 如read()或write()等 ) 的，一般对它的访问都是以内存映射的形式进行的。  
  
大页在进程空间和物理地址上都是连续的。  
  
接下来我们查看一下普通页面和大页映射方式的差别。普通四级页表映射和三级页表内存大页的映射：  
![](https://chenghua-root.github.io/images/memory-huge-page-mapping-64bit.png)  
-  从上图可以看到，内存大页由于页内地址为2M，2^21。在第三级页目录就需要进行映射了，及线性地址映射到物理地址时，第三级pmd就相当于普通映射的第四级page table.  
  
### 透明内存大页（transparent huge page: THP）：  
由于使用大页需要做额外的操作，包括配置和写代码方面。所以后面又提供了透明大页的功能，即底层默认开启大页，且开发者不需要要关心。  
RHEL6中默认开启。  
什么时候会分配透明内存大页：  
- 内核会在任何可能的时候分配内存大页。  
- Linux process will receive 2MB pages if the mmap region is 2MB naturally aligned.？  
- 内核地址空间使用透明内存大页进行映射。  
- 透明内存大页在物理地址上连续。  
  
内核总是会尝试分配大页，当获取不到的时候才回退到4KB pages.  
THP是由多个4KB组成的，因此可以被swap；整个大页一起swap，swap时以页为单位，跟普通页的swap方法一致。  
THP目前只能在anonymous memory使用。  
  
**内存大页设计：**  
- graceful fallback：当分配不了大页的时候，可以优雅的回退  
- 由于内存碎片导致分配大页失败时，应该在同一个vma里面进行分配内存，没有失败和较大的延迟  
- 当一些进程退出，更多的大页可用时，会自动通过khugepaged迁移普通的页面到大页上  
  
### 透明内存大页参数讨论:  
重点关注前三个参数。  
1. /sys/kernel/mm/transparent_hugepage/enabled  
  - always: 总是使用大页  
  - madvise:只在MADV_HUGEPAGE区域内使用  
  - never：完全不使用THP  
2. /sys/kernel/mm/transparent_hugepage/defrag  
  - always：应用程序分配大页失败时，会阻塞等待并去compact memory以使能立即分配THP。  
  - defer：后台唤醒kswapd进程来回收页面，并唤醒kcompactd来合并内存，THP在此后将可以使用。  
3. /sys/kernel/mm/transparent_hugepage/khugepaged/defrag  
  - 内核线程khugepaged周期性自动扫描内存，自动将地址连续可以合并的4KB的普通Page并成2MB的Huge Page。  
  - 当transparent_hugepage/enabled=always时，khugepaged会自动启动；=never时，khugepaged会自动关闭。  
  - yes: 合并时整理碎片  
  - no: 合并时不整理碎片  
4. /sys/kernel/mm/transparent_hugepage/khugepaged/pages_to_scan  
  - 每轮khugepaged浏览的页数（4096）  
5. /sys/kernel/mm/transparent_hugepage/khugepaged/scan_sleep_millisecs  
  - khugepaged运行间隔时间（10000）  
6. /sys/kernel/mm/transparent_hugepage/khugepaged/alloc_sleep_millisecs  
  
## Cache管理机制  
在Linux操作系统中，当应用程序需要读取文件中的数据时，操作系统先分配一些内存，将数据从存储设备读入到内存中，然后再将数据分发给应用程序；当需要往文件中写数据时，操作系统先分配内存接收用户数据，然后再讲数据从内存写到磁盘。文件cache管理指的就是对这些由操作系统分配，并用来存储文件数据的内存的管理。  
  
Cache管理的优劣通过两个指标衡量：一是Cache命中率；二是有效Cache的比率，有效Cache是指真正会被访问到的Cache项。  
  
### Cache的位置和作用  
内存、文件系统、磁盘的层次模型：  
![](https://chenghua-root.github.io/images/memory-page-cache.png)  
* 在 Linux 中，具体的文件系统，如 ext2/ext3/ext4 等，负责在文件 Cache和存储设备之间交换数据  
* 位于具体文件系统之上的虚拟文件系统VFS负责在应用程序和文件 Cache 之间通过 read/write 等接口交换数据  
* 内存管理系统负责文件 Cache 的分配和回收  
* 虚拟内存管理系统(VMM)则允许应用程序和文件 Cache 之间通过 memory map的方式交换数据  
  
可以看出Cache是在VFS下面，读取文件的过程为read()->VFS->Cache()->磁盘:  
![](https://chenghua-root.github.io/images/memory-vfs-page-cache.png)  
  
如何定位读取文件的offset在哪个cache page中：  
- 通过基数树来实现。即为cache与物理文件的映射。  
![](https://chenghua-root.github.io/images/memory-page-cache-trie.jpg)  
  - 上图为快速定位8位(bit)长度的文件长度。  
  
## swap分区、内存回收、kswapd  
swap，指的是一个交换分区或文件。从功能上讲，交换分区主要是在内存不够用的时候，讲部分内存上的数据交换到swap空间上，以便让系统不会因为内存不够用而导致OOM或者其他情况。  
所以，当内存使用存在压力，触发内存回收的行为时，就可能会使用swap空间。  
swap分区的实际使用是跟内存回收行为紧密结合的。内存回收和swap的关系，需要思考一下几个问题：  
  
1. 为什么要进行内存回收  
  - 内核需要为任何时刻突发到来的内存申请提供足够的内存。所以一般情况下需要保证有足够的空闲空间。  
  - 由于page cache的使用，内存需要周期性的回收page cache  
  - 由于启用了swap分区，内存需要周期性的把不活跃的内存swap到swap分区  
  - 当真的大于空闲空间的内存申请到来的时候，会触发强制内存回收  
  所以当内存回收时，内核设计两种不同的策略：  
  - 一个是使用kswapd进程对内存进行间接回收（包括page cache和swap cache）  
  - 另一个是直接内存回收（直接回收是回收哪些内存）  
  内存回收时主要扫描四个队列：  
  - anon的inactive  
  - anon的active  
  - file-back的inactive  
  - file-back的active  
  内存回收操作主要针对的就是内存中的文件页(file cache)和匿名页。  
  - anon的匿名内存回收，主要使用swap  
  - file-back的文件映射页的回收，主要的释放手段是回写磁盘和清空  
  
2. swappiness参数是用来调节什么的？  
  /proc/sys/vm/swappiness的取值范围为[0, 100]，是一个百分比。  
  - 值越高，内核会越积极的使用swap  
  - 值越低，就会降低对swap分区的使用积极性。  
  - 如果值为0，内存在free和page cache使用的页面重量小于高水位标记(high watermark)之前，不会发生交换。  
  swappiness的作用即为调节在回收内存的时候，更倾向于清空page cache，还是进行swap交换。  
  
3. kswapd什么时候会进行swap操作？  
  kswapd进程要周期性的对内存进行检查，达到一定阈值的时候开始进行内存回收  
  
4. 什么是内存水位标记(watermark)？  
  Linux为内存使用设置了三种内存水位标记：high, low, min。含义分别为：  
  - 剩余内存在high以上，表示剩余内存较多  
  - 剩余内存在low-high的范围内，内存使用存在一定压力  
  - 剩余内存在min-low的范围内，内存使用存在比较大的压力  
  - 剩余内存小于min，内核是保留给特定情况下的，内存已无法分配  
  当系统可用内存（free + page cache）低于low的时候，内核的kswapd开始起作用，进行内存回收。直到剩余内存达到high的时候停止。  
  如果内存低于min时，申请内存时就会触发直接回收。  
  watermark的值：  
  - min：cat /proc/sys/vm/min_free_kbytes，单位为1KB  
   67584  
  - low、high: cat /proc/zoneinfo，单位为page(4KB)  
  
 Node 0, zone |  Normal  
  -----  | ----  
  pages free  |   40342  
  min   |   16477  
  low   |   20596  
  high  |   24715  
{: .tablelines}  
  
问题：  
- 直接内存回收，是回收的哪些内存?  
  
## 问题：  
1. 哪些情况下需要分配连续的物理内存  
- 内核空间kmalloc分配内存需要物理上连续，通过偏移映射。VMALLOC_START/VMALLOC_END之间，在物理上不需要连续，不能通过偏移映射。  
- 内存大页/透明内存大页需要物理连续的内存分配  
2. 为什么进程是资源分配单位，线程是调度单位  
- 一个进程下的所有线程共享同一个地址空间  
- 在内核空间中为每个用户线程分配了一个内核栈，栈底部有个thread_info描述整个线程  

## 透明内存大页测试
**测试目的：**
- 测试透明内存大页分配和缺页中断次数

**测试方法：**
1. 先获取分配1个字节时的page-faults次数=105 // mmap()的length参数必须大于0  
![](https://chenghua-root.github.io/images/memory-thp-test-one-byte.png)  
2. 关闭内存大页，测试申请16MB  
  缺页次数4201:  
![](https://chenghua-root.github.io/images/memory-thp-test-close.png)  
  缺页次数增加4096=4201-105，缺页内存大小：(4201 - 105) * 4 / 1024 = 16MB，符合预期  
3. 开启内存大页，测试申请16MB  
  测试之前透明内存大页使用量：108544 KB
![](https://chenghua-root.github.io/images/memory-thp-test-used-before.png)  
  测试之后透明内存大页使用量：122880 KB，较测试前增加(122880 - 108544) / 1024 = 14MB  
![](https://chenghua-root.github.io/images/memory-thp-test-used-after.png)  
  缺页次数624：  
![](https://chenghua-root.github.io/images/memory-thp-test-open.png)  
分析：  
 - 缺页次数增加624 - 105 = 519  
 - 透明内存大页增加14MB，缺页次数7  
 - 有2MB内存没有通过透明内存大页申请，缺页次数为2MB / 4KB = 512  
 - 透明内存大页缺页次数(7) + 2MB普通缺页测试(512) = 519符合预期  

**测试代码:**  
```  
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/mman.h>

//#define MALLOC_SIZE 1
#define MALLOC_SIZE (16 * 1024 *1024)
int main() {
  void *buf = NULL;
  int ret = 0;
  int size = MALLOC_SIZE;

  buf = mmap(NULL, size, PROT_READ | PROT_WRITE,
    MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
  if (buf == (void *)-1) {
    fprintf(stderr, "mmap fail!\n");
    ret = -1;
    goto exit;
  }

  memset(buf, 'f', size);
  sleep(3);

  ret = munmap((void*)buf, size);
  if (ret != 0) {
    fprintf(stderr, "munmmap fail!\n");
    goto exit;
  }

exit:
  return ret;
}
```  

## 参考  
  linux系统总览: https://www.jianshu.com/p/82b4454697ce  
  内存分配的原理--molloc/brk/mmap: http://abcdxyzk.github.io/blog/2015/08/05/kernel-mm-malloc/
