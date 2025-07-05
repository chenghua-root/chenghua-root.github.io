---  
layout: post  
title:  "AWS Aurora"  
date:   2025-06-21 00:00:00 +0530  
---  
  
本文内容和结构遵照Aurora论文[1]  
  
*斜体为笔者补充的内容*  
  
# 前言  
  
日志下推，减少网络流量，快速恢复，容灾。  
  
# 1. INTRODUCTION  
当前(2017)情况：  
- 计算存储分离  
- 单磁盘不再热点  
- 瓶颈在节点之间的网络  
- 2PC同步协议需要面对硬件和软件故障，跨数据中心的高延迟  
  
Aurora:  
- 使用redo log解决上面的问题  
- 多租户横向扩展存储服务  
- redo log与实例集群解耦  
- 实例继续包含传统数据库内核的大部分组件：查询处理器，事务，锁，缓冲池，访问方法和撤销管理  
- redo log的保持，持久化存储，崩溃恢复，备份/恢复都卸载到存储服务  
  
三大特点：  
- 大规模场景下的持久性，可靠性和恢复  
- 传统数据库的底层卸载到新的智能存储  
- 化解分布式存储中的多阶段同步，崩溃恢复，检查点  
  
# 2. DURABILITY AT SCALE  
  
## 2.1 Replication and Correlated Failures  
实例的生命周期与存储的生命周期并不高度相关。实例可能会出现故障，客户可能会将其关闭，也可能会根据负载对其进行扩缩容。基于这些原因，将存储层与计算层解耦是有必要的。  
  
容灾：多副本。  
  
quorum-based voting protocol.  
- 读取最新的数据: Vr + Vw > V  
- 防止写冲突: Vw > V / 2  
  
## 2.2 Segmented Storage  
  
MTTF: Mean Time to Failure  
MTTR: Mean Time to Repair  
PGs: Protection Groups. 6 10GB segment, 2 in each AZ(3az)  
  
PGs随着volume增长而分配。目前支持最大到64TB(2^32(page no) * 16KB(page size))。  
  
Segment:  
- 最基本的管理单位：损坏、修复  
- 10GB(segment size) 在10s修复, 基于10Gbps带宽(*最新的multi version page*)。  
- 无法恢复的场景：一个AZ故障 + 2个副本故障, 不太可能发生。  
  
## 2.3 弹性的优势(Operational Advantages of Resilience)  
  
故障容忍度高：  
- 应对长期和短期故障  
- 能处理长期AZ损失同时，也能处理电力事件和回滚不良软件部署导致的停机  
- 能处理多秒级的仲裁成员不可用的系统，也能处理存储节点上的网络拥塞或节点负载  
  
支持：  
- 热点管理。磁盘或节点被标记为热点后，迁移到其它冷节点  
- 操作系统和安全补丁。节点短暂的不可用。  
- 软件系统升级。  
- 保证：每次只操作一个AZ，并且同一PG只有一个segment受影响。就能够在存储服务中使用敏捷的方法和快速部署。  
  
# 3. 日志即数据库(THE LOG IS THE DATABASE)  
  
阐述：第二节描述的分段复制存储系统用在传统的数据库上会在网络I/O和同步阻塞方面带来无法承受的负担。  
方法：日志处理卸载到存储服务，通过实验验证如何大幅度减少网络IO。其它各种技术：最小化同步阻塞和不必要的写入。  
  
## 3.1写放大的负担  
  
高负担:  
- 6副本, 写放大  
- MySQL产生多种IO。产生需要同步等待的场景，扩大延迟。  
- 链式负载能够减少计算节点的写放大，但还是避免不了同步和延迟问题。  
  
- 数据页，redo log: 记录修改前后的页面差异。  
  
![Figure2](https://chenghua-root.github.io/images/aurora-figure2.jpg)  
  
传统MySQL架构：  
- log, binlog, data(page), double-write, FRM files(元数据)  
- EBS mirror  
- 备份实例  
- S3: redolog, binlog归档到S3, 支持时间点恢复(PITR)  
  
写入方式和写入的数据都不受欢迎：  
- 延迟累加：步骤1,3,5是顺序且同步的  
- 抖动放大：异步写入，需等待最慢的操作完成  
- 易受异常操作影响：从分布式系统角度看，此模型具有4/4写入仲裁。*4：双写 + EBS mirror？*  
  
## 3.2 redolog处理卸载到存储  
  
- 唯一跨网络的写：redolog  
- 基于redolog, 后台按需生成数据库页  
- 避免了不必要的写入：为了防止存储中的页(16KB)撕裂而双写。先写入“双写缓冲区”, 成功后再写入数据文件(.ibd)  
  
![Figure3](https://chenghua-root.github.io/images/aurora-figure3.jpg)  
  
Aurora中的网络IO:  
- 主实例把日志写入存储服务  
- 主实例把日志和元数据流式传输到副本实例  
- 基于逻辑段(PG)，数据库引擎对日志进行拆分并有序批处理的发送给6个副本，等4个副本确认写成功，则认为日志记录是持久的或已硬化的  
- 副本使用redolog将其更改应用到他们的缓冲区缓存中  
  
![Table1](https://chenghua-root.github.io/images/aurora-table1.jpg)  
  
MySQL vs Aurora  
- 跨多个可用区的同步镜像MySQL配置 vs RDS Aurora跨多个可用区的副本  
- 半个小时事务处理量: 780,000 vs 27,378,000. 35倍  
- 每个事务的IO数量: MySQL在忽略EBS内部和副本实例的IO情况下(相当于使用本地盘的IO数量)，7.4 vs 0.95(6副本)  
 - Aurora单节点处理的IO相当于MySQL的46分支一(0.95/7.4/6)  
 - 节省的数据写入量, 可以积极的被复制数据使用，以提高耐用性和可用性。  
  
崩溃恢复  
- 传统数据库崩溃后，系统必须从最近的检查点开始，并重播日志以确保所有持久化的redolog都已应用  
- Aurora: redolog由存储层连续，异步的应用，并在整个集群中分布式处理。  
 - 当读取任何page需要应用一些redolog时，那么崩溃和恢复过程就会隐式的包含在读取page过程中了。  
 - 实例(数据库引擎)重启不需要额外的工作。  
  
## 3.3 存储服务要点  
  
要点：  
- 尽量减少前台写入请求延迟。  
- Aurora中，后台处理会影响前台请求的性能，需要做好控制。  
  
![Figure4](https://chenghua-root.github.io/images/aurora-figure4.jpg)  
  
存储节点活动（图4）:  
1. 接收日志记录并将其添加到内存中；  
2. 将记录持久化到磁盘并ack；  
3. 组织记录并识别日志中缺失部分；（SDK基于segment拆分日志时记录前一条日志的LSN）  
4. 与对等节点进行通信以填补缺失部分；  
5. 应用日志记录生成新的数据页；(不会每一条日志都生成新的数据页)  
6. 定期将日志和新页面上传到S3;  
7. 定期对旧版本进行垃圾回收；  
8. 最后定期验证页面上的CRC；  
  
上面每个步骤都是异步的，只有1和2处于前台IO路径中，会影响延迟。  
  
# 4. 日志不断前进  
  
## 4.1 解决方案概述: 异步处理  
  
基于redolog构建数据库，redolog是有序单调递增的，每条日志记录都有一个与之关联的日志序列号(LSN), 由数据库引擎生成。  
MySQL中的LSN根据日志有序记录的字节增长。如从启动开始，第一条日志32字节，第二条日志15字节，则第三条日志的(start)LSN为47.  
  
结合redolog的特点:  
- 我们简化共识协议: 异步方式处理, 而不是使用2PC  
- 收到ack后不断的推进一致性点和持久化点  
- PG内部互相沟通对齐日志  
- 结合运行时状态，我们只需要读一个副本的数据，而不用quorum读。除非状态丢失或需要重建的时候。  
  
处理并发事务时崩溃, 对每个事务来说是否回滚怎么回滚是独立的。**数据库引擎记录了这些事务的跟踪信息：事务完成，undolog**。  
重启后，在允许引擎访问前，**存储层需要解决分布式存储视角的数据一致性, 而不是事务层面的一致性**。  
  
各类LSN:  
- **VCL**(Volume Complete LSN): 此LSN之前的所有前向日志都已记录。  
 - recovery时，大于VCL的日志都会被truncate.  
 - *一个volume对应一个数据库，已经持久化的LSN, 计算层计算。根据每个PGCL，计算出整个volume的VCL, 小于等于VCL的日志都完成持久化*  
- **CPLs**(Consistency Point LSNs). 数据库会基于CPLs进一步限制Points来截断日志。  
- **VDL**(Volume Durable LSN): 小于等于VCL的最大的CPL作为VDL，其后的日志都会被截断。  
 - VCL=1007, CPLs=(900, 1000, 1100), 则VDL=1000，即只保留到1000的日志  
  
 - *故障恢复时truncate掉无用的数据*  
 - *发送VDL给读实例*  
  
因此：Completeness和Durability在这里是两个不同的概念。而CPL可被视为存储系统事务的有限形式划分，这类事务必须按序接受。  
如果客户端无需进行此类划分，可以简单的将每条日志记录标记为CPL.  
  
实践中，数据库和存储交互如下：  
1. 每条事务被拆分为多个mini-transactions(MTRs)，需要保证有序且原子的处理这些MTRs。  
2. 每个MTR包含多条连续的日志(as many as needed)  
3. **MTR中的最后一条log就是一个CPL**. 由计算层生成.  
  
恢复时，存储告诉数据库durable point of each PG, 数据库计算出VDL，并以此来截断日志。  
  
## 4.2 常规操作  
读，写，提交和复制。  
  
### 4.2.1 Writes  
数据库与存储交互，维护状态，包含：quorum, advance volume durability, 已提交事务。  
对于实例：数据库接收each batch of log records的write quorum ack, 来推进VDL。  
  
限流  
- 任意时刻：都有大量的并发事务，产生事务各自的redolog.  
- 数据库对每条log分配有序唯一的LSN，此LSN会被限制在VDL + LAL(LSN Allocation Limit: 10 million).  
- 目的：在存储和网络跟不上的时候, 避免计算层相较于存储层走的太远。  
  
每个PG的segments只包含属于此pages的日志。这些日志串起来(backlinks)来跟踪日志记录的完整度，以计算(establish) **Segment Complete LSN(SCL)**.由存储层计算.  
SCL标明属于此PG的所有小于等于SCL的日志都已经收到。  
SCL被存储节点用来发现和交互缺失的日志。  
  
*计算层通过quorum协议，询问SCL可以计算出Protection Group Complete LSN(PGCL)*. 计算层计算。  
  
### 4.2.2 Commits  
  
在Aurora, 事务提交是异步的。提交事务时也会产生LOG，设为commit LSN, 当VDL >= commit LSN，则表示事务提交完成。  
然后使用专门的线程回复客户事务提交情况。  
  
### 4.2.3 Reads  
  
在Aurora, 从buffer cache中读取pages, 不存在则发起存储IO。  
  
针对新读取的page需要缓存, 如果buffer cache满，则剔除一个page.  
  
传统数据库中，如果victim page是dirty page, 则在替换前需要刷新到磁盘上。以保证后续读取到最新的page.  
  
然而Aurora不需要刷回，但同样会保证读取到buffer cache的page是最新版本的。  
  
该保证通过以下方式实现：对于页的 “页 LSN”（标识与该页最新更改相关的日志记录）大于或等于 VDL（卷数据日志）的page，会将该页从缓存中逐出。  
此协议确保：(a) 页中的所有更改已在日志中持久化；(b) 当缓存未命中时，只需请求与当前 VDL 对应的页版本，即可获取其最新持久化版本。  
- *为什么是大于等于VDL的page会被逐出，因为这些更新的page的log没有完成quorum Writes, 即没有被VDL包含?*  
  
正常情况下，不需要quorum读来保证一致性。读取的时候会设定一个read-point, 等于此刻请求时的VDL。  
数据库可以在含有此page的六个节点中选择已经包含read-point版本的节点。返回的page需要保证满足MTR的一致性。  
  
数据库(引擎层，计算层)直接跟踪每个存储节点的log情况，以及每个segment的SCL，自然知道哪个segment满足read-point.  
  
PGMRPL(Protection Group Min Read Point LSN):  
- 计算层生成  
- 数据库知道当前有哪些读，即知道PG最小的read-point, 读取时告诉PG segment.  
- 数据库保证不会产生低于PGMRPL的read point, 则小于PGMRPL的日志都可以被应用回收。  
  
实际的并发控制协议在数据库引擎中执行，就像数据库页面和撤销段在本地存储中组织一样，与传统MySQL相同。  
  
### 4.2.4 Replicas  
  
在Aurora, 1主15只读实例挂载同一个Storage volume. 只读实例不产生写开销。  
最小化lag, 主产生的日志发送给存储集群时也发送给只读实例。  
只读实例按序消费日志。**如果对应的page在cache中，则应用，如果不在则直接忽略**。  
从主节点视角看，只读实例异步消费日志, 主节点提交事务与只读实例消费独立。  
只读实例消费日志时遵循如下两条规则：  
- 应用小于等于VDL的日志  
- 属于同一个的MTR的日志需要原子的应用，以保证只读节点提供一个一致性的视图。但不保证是最新的。  
- 通常情况下，只读节点落后主节点20ms或更少  
  
## 4.3 Recovery  
  
传统数据库采用如 ARIES 这类恢复协议，该协议依赖预写日志（WAL）的存在，而 WAL 能够精准记录所有已提交事务的内容。  
  
定期将脏页刷新到磁盘并向日志写入检查点记录，以粗粒度的方式建立持久性点。崩溃时，磁盘上的page还是存在缺失已提交的数据或者包含多余的未提交的数据。  
恢复时结合checkpoint, 使用redolog和undolog来恢复到一个一致状态。 崩溃时in-flight的事务将会被回滚。  
崩溃恢复开销较大。减少检查点间隔会降低开销，但会干扰前台事务。Aurora则不存在这样的权衡。  
  
传统数据库的一个简化点：事务执行和恢复使用同一redolog applicator. 应用是同步且在前台执行。  
Aurora也使用这一特点（事务执行和恢复使用同一redolog applicator），只是把其从计算层解耦，放置在存储节点并发的在后台执行。  
数据库启动时与存储服务配合，Aurora恢复很快(通常小于10s), 即使崩溃时同时处理100, 000事务。  
  
结合VDL截断日志来达到一个运行状态。  
  
上面的运行状态还没处理in-flight事务(无法继续提交)，**对于那些in-flight事务还需要做undo操作**。  
不过这些undo操作可以在数据库online后，待系统从undo segments收集完in-flight事务后再做。  
  
# 5. PUTTING IT ALL TOGETHER  
  
![Figure5](https://chenghua-root.github.io/images/aurora-figure5.jpg)  
  
鸟瞰图5：  
- 存储节点使用本地盘  
- 备份到S3  
- 数据库引擎是社区版MySQL/InnoDB的一个分支  
 - 区别主要为读写数据的方式  
  
社区版InnoDB: 按照LSN顺序写redolog到WAL, 修改buffer page，产生写操作。  
事务提交：只要求redolog持久化。对应的buffer pages使用双写来避免partial page writes.  
page writes: 发生在后台，或者从cache中淘汰时，或者创建checkpoint.  
  
除了IO子系统，InnoDB还包括事务子系统，lock manager, B+树的实现，以及相关的MTR。  
MTR只是InnoDB中的概念，包含一组操作(分裂合并B+树)需要被原子的执行。  
  
*总结：VDL只保证了MTR的原子性（B+树的完整一致），但不满足数据库的ACID的一致性, 数据库恢复时需要结合undo来满足。剔除无法完成提交的事务的相关数据。*  
  
Aurora InnoDB作为社区版的变种，MTR中需要被原子执行的log, 按照log属于不同的segments，被组织成batchs.  
每个MTR的最后一条日志被标记为一个一致性点。  
Aurora满足社区MySQL完全一致的隔离性(标准的ANSI级别: 快照隔离或者一致性读).  
  
Aurora的只读实例不断从主实例中获取事务开始和提交的信息，并使用这些信息为本地只读事务提供快照隔离。  
注意，并发控制完全在数据库引擎中实现，不会影响存储服务。  
存储服务呈现一个统一的底层数据视图，从逻辑上讲，这与将数据写入社区版InnoDB的本地存储所得到的结果相同。  
  
Aurora使用RDS作为其管控平台。每个实例包含一个Host Manager(HM), 监控集群的监控以及决定是否执行故障处理，或者替换实例。  
每个实例作为cluster的一部分，包含一个writer或者0个或多个reader.  
一个cluster的所有实例部署在同一地理region(如，us-east-1, us-west-1), 不同的AZs中。  
  
为了安全，对用户层，数据库层，存储层的网络进行隔离  
VPC(Virtual Private Cloud):  
- Customer VPC  
- RDS VPC  
- Storage VPC  
  
使用DynamoDB存储元数据。  
  
# 6. PERFORMANCE RESULTS  
  
## 6.1 标准的Benchmarks  
  
随着实例大小write/read线性增长。  
  
# *总结*  
SCL(存储层)-\>PGCL(计算层)  
PGCL(计算层)-\>VCL(计算层)  
MTR-\>CPLs  
min(VCL, max(CPLs))-\>VDL(计算层计算)  
write/read inst-\>PGMRPL-\>发送给Storage, GC log  
VDL + LAL(LSN Allocation Limit)-\>运行事务能分配的最大的LSN  
