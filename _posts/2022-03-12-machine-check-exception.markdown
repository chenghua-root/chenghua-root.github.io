---  
layout: post  
title:  "Machine Check Exception"  
date:   2022-03-12 00:00:00 +0530   
---  
  
<style>  
.tablelines table, .tablelines td, .tablelines th {  
  border: 1px solid black;  
  }  
</style>  
  
## 背景  
集群达到一定数量后，一些较小概率发生的硬件错误会出现，如典型的硬件出错：内存错误。  
![mcelog:内存错误](https://chenghua-root.github.io/images/mce-00-mcelog.png)  
这些硬件错误在Linux系统中有一套发现和处理机制，被称为Machine Check Exception(MCE)。  
  
## 概述  
本文主要讲解MCE是什么，如何查看和注入MCE错误，以及如何处理MCE错误，并以ECC内存错误为例详细分析。  
  
## 什么是MCE  
Machine Check Exception (MCE) 是CPU发现硬件错误时触发的异常(exception)，中断号是18，异常的类型是abort。  
![Exceptions and Interrupts](https://chenghua-root.github.io/images/mce-01-exception-code.png)  
  
查看系统启动后出现MCE中断的触发次数：  
less /proc/interrupts  
![less /proc/interrupts](https://chenghua-root.github.io/images/mce-02-exception-count.png)  
  
## 导致MCE的原因  
导致MCE的原因主要有：  
- 总线故障、内存ECC校验错、cache错误、TLB错误、内部时钟错误等。  
- 除了上面的硬件故障引发MCE，不恰当的BIOS配置、firmware bug、软件bug也有可能引起MCE。  
  
## MCE故障恢复分类  
MCE故障分为可自动纠正类型和无法恢复类型。  
**自动纠正类型：**  
在 Linux 系统上，如果发生的MCE错误属于可以自动纠正的类型，那么系统保持继续运行，MCE错误日志会记录在一个ring buffer中（这个ring buffer通过设备文件/dev/mcelog来访问），系统通常会通过cron任务(Centos6)或者mcelog demon(Centos7)把ring buffer中的MCE日志写入/var/log/mcelog文件中。  
  
**无法恢复类型：**  
如果发生的MCE错误属于无法恢复的类型，那么系统会panic，错误信息会输出在终端上和message buffer里。  
  
## MCE记录与查看  
### 系统如何记录MCE  
每个CPU有一组寄存器称为 Machine-Check MSR (Model-Specific Register)，用于Machine-Check的控制与记录，分为全局寄存器和若干Bank寄存器（CPU的硬件单元分成若干组，每一组称为一个Bank）。当发生MCE时，错误信息记录在全局状态寄存器 MCG_STATUS MSR 和Bank寄存器 MCi_STATUS MSR 中。如下图所示：  
![MSRs](https://chenghua-root.github.io/images/mce-03-machine-check-register.png)  
  
内核通过读写 Machine-check MSRs 来记录 MCE 信息，并通过一个内核数据结构 struct mce 将MCE 信息保存到一个字符设备 /dev/mcelog 中，此设备最多可以保存 32 个结构体，即 32 条记录。  
```  
linux-3.10/arch/x86/include/uapi/asm/mce.h  
struct mce {  
    __u64 status;    /* 对应 IA32_MCi_STATUS MSR */  
    __u64 misc;  
    __u64 addr;  
    __u64 mcgstatus; /* 对应 IA32_MCG_STATUS MSR */  
    __u64 ip;  
    __u64 tsc;  /* cpu time stamp counter */  
    __u64 time; /* wall time_t when error was detected */  
    __u8  cpuvendor;    /* cpu vendor as encoded in system.h */  
    __u8  inject_flags; /* software inject flags */  
    __u16  pad;  
    __u32 cpuid;    /* CPUID 1 EAX */  
    __u8  cs;       /* code segment */  
    __u8  bank; /* machine check bank */  
    __u8  cpu;  /* cpu number; obsolete; use extcpu now */  
    __u8  finished;   /* entry is valid */  
    __u32 extcpu;   /* linux cpu number that detected the error */  
    __u32 socketid; /* CPU socket ID */  
    __u32 apicid;   /* CPU initial apic ID */  
    __u64 mcgcap;   /* MCGCAP MSR: machine check capabilities of CPU */  
};  
```  
MCE错误详细解读可参考Inter手册第3卷，15章MACHINE-CHECK ARCHITECTURE，16章INTERPRETING MACHINE-CHECK ERROR CODES。  
  
## 用户如何获取MCE  
用户空间有一个后台进程mcelog, 可以读取 /dev/mcelog 中的数据将其解析为用户可读的 mcelog 信息。mcelog 信息默认保存在 /var/log/mcelog 中。内核在记录了一次 MCE 时,会在 dmesg 中打印一条消息: Machine check events logged.  
![获取mcelog](https://chenghua-root.github.io/images/mce-04-mcelog.png)  
  
## MCE故障注入  
MCE错误不常发生，想观察MCE错误可以通过注入的方式产生软件层面的MCE错误。  
如下为通过mce-inject向运行的Linux内核中注入MCE错误。  
  
### 注入MCE故障准备工作：  
内核需要加载mce-inject后才能进行注入  
- Requires a Linux 2.6.31+ kernel with CONFIG_X86_MCE_INJECT   
- enabled and the mce-inject module loaded (if not built in)  
![inject-model-load](https://chenghua-root.github.io/images/mce-05-inject-model-load.png)  
  
### 注入可恢复MCE故障：  
sudo ./mce-inject test/corrected  
mcelog显示有MCE错误：  
![inject-test-corrected](https://chenghua-root.github.io/images/mce-06-inject-test-corrected.png)  
  
产生dmesg信息：  
![corrected-dmsg](https://chenghua-root.github.io/images/mce-07-inject-test-corrected-dmsg.png)  
  
通过注入mce的方式显示未产生mce中断：  
![corrected-no-interrupt](https://chenghua-root.github.io/images/mce-08-inject-test-corrected-no-interrupt.png)  
  
### 注入不可恢复故障：  
sudo ./mce-inject test/uncorrected  // **注意**：会导致节点重启  
![inject-test-uncorrected](https://chenghua-root.github.io/images/mce-09-inject-test-uncorrected.png)  
  
## 如何处理MCE错误：以ECC内存为例  
本节讨论系统和业务如何处理MCE错误。  
  
### ECC  
ECC(Error Checking and Correcting)是一种错误检查和自动纠正的技术，它能够容许错误并可以将错误更正，使系统得以持续的正常工作，不因错误而中断。ECC内存正是带有ECC功能的内存。  
```  
sudo dmidecode --type memory  
Handle 0x005A, DMI type 17, 40 bytes  
Memory Device  
	Array Handle: 0x0052  
	Error Information Handle: Not Provided  
	Total Width: 72 bits  
	Data Width: 64 bits  //ECC Memory DataWidth TotalWidth 64 72  
```  
  
ECC技术实际上是通过编解码的方式来修正错误。ECC技术只能纠正1个bit错误和检查2个bit错误，对1bit以上的错误无法纠正，对2bit以上的错误不保证能检测出。  
  
### Bad Page Offline  
当发生内存错误时，mcelog会跟踪记录这一情况。如果出现1bit内存错，错误可以被修正。如果附近“同一组内存”中的其它bit出现错误，将无法修复。因此当出现内存出错时，系统需要采取措施以防止或降低无法修复的情况。  
操作系统以页(通常4KB)管理内存，并能够“下线”掉出错的页并停止使用。Linux提供了soft_page_offline来对页面内容进行迁移。  
mcelog以Page(4KB)为粒度记录追踪内存错误（corrected memory errors），记录每个page出错的次数。当一个页面在24小时出现超过设定的阈值次数（默认配置10次），则会被“下线”掉，不再被分配使用。  
如图1第4行 "Corrected memory errors on page xxx exceed threshold 10 in 24h"  
```  
/etc/mcelog/mcelog.conf  
  
[page]  
# Try to soft-offline a 4K page if it exceeds the threshold  
memory-ce-threshold = 10 / 24h  
memory-ce-trigger = page-error-trigger  
memory-ce-log = yes  
memory-ce-action = soft  
```  
  
Linux内核从2.6.33v开始支持page soft-offlining的功能。页面数据被拷贝到其它空闲页面，当前页面从内存管理系统中删除。  
此功能被称为soft-offlining是因为对应用程序来说是透明的。系统自动把错误页的内容拷贝到一个新的页并重新映射即可。  
Bad page offlining的状态只会记录在内存中。当系统重启后，被offline的page讲会被重新使用，直到被触发新的offline。  
  
### 业务应该怎么处理  
对于ECC内存错误，系统会自动纠正，并迁移下线“损坏”的page。所以“一般业务”不需要关注。  
对于PolarStore，采用DPDK内存库实现内存的分配和管理，同时使用hugepage来保证大块连续的物理内存。  
但**DPDK为了内存安全会pin住所使用的hugepage，导致系统不能透明的offline page。**  
参考图1中倒数第3行 "Offlining page xxx failed Device or resource busy..."  
  
**dpdk hugepage pin住说明：**  
- 参考“My current understanding is that there is no way to pin memory and hugepages can now be moved around, so uio would be unsafe.” [引文链接](http://mails.dpdk.org/archives/dev/2017-January/053965.html)  
  
**如何处理被pin住的出现故障的内存页：**  
- 修改mcelog daemon，在其监听到EC类型的MCE异常通知kernel进行migrate内存之前，业务可以选择先停止服务，即“释放”掉pin住的内存页。等系统offline成功后再重启服务。  
- 扫描/var/log/message的日志信息并进行告警处理。  
  
## 相关链接  
Linux kernel下载地址：https://mirrors.edge.kernel.org/pub/linux/kernel/  
Intel® 64 and IA-32 Architectures开发手册第三卷下载地址： https://software.intel.com/content/www/us/en/develop/download/intel-64-and-ia-32-architectures-sdm-combined-volumes-3a-3b-3c-and-3d-system-programming-guide.html  
ECC内存：https://www.crucial.cn/learn-with-crucial/memory/what-is-ecc-memory  
mcelog: memory error handling in user space: https://halobates.de/lk10-mcelog.pdf  
mce-inject: https://github.com/andikleen/mce-inject  
