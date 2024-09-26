---  
layout: post  
title:  "Azure Socrates"  
date:   2024-09-20 00:00:00 +0530  
---  
  
本文内容和结构遵照Socrates论文[1]  

*斜体为笔者补充的内容*  
  
# 前言  
Microsoft Azure & Microsoft Research 在SIGMOD 2019发表了介绍Azure新一代数据库架构的论文socrates，产品命名为Azure SQL Database Hyperscale，一款类似AWS Aurora的云原生数据库。  
在此之前，Azure已有类似传统的提供单机及主备服务的Azure SQL Database，以及类似Google Spanner的主打Global Distribution/Multi-Master特性的Azure Cosmos DB.  
由此，SQL Database/SQL Database HyperScale/CosmosDB构成了Azure的低中高三个档位的数据库产品，让微软具有了门类最全的DBaas(DataBase as a service)服务。  
  
# 介绍  
业务上云的需求  
- 安全，快速，消费弹性  
- 高可用，大实例，高性能，弹性  
传统的数据库(*DB2, Oracle*)不能满足上面的需求。因此，过去十年，云厂商探索了新的OLTP数据库系统。主要为计算存储分离，AWS Aurora最先使用。  
  
本论文socrates(苏格拉底)，讲解一种新的OLTP架构。不止计算和存储分离，并且，**日志和存储分层实现Durability和Availability:**  
- Durability(持久性): implemented by the log. 对于防止数据丢失非常关键  
- Availability(可用性): implemented by the storage tier. 用于提供高质量的服务(可用且高性能)  
把这两个特性分开，有非常大的潜能:  
- 与可用性相比，持久性不需要做拷贝(非副本，传统数据库主备实例的拷贝)  
- 与持久性相比，可用性不需要固定数量的副本(后文中PageServer中的副本)  
![Table 1: Socrates Goals: Scalability, Availability, Cost](https://chenghua-root.github.io/images/socrates-goals.png)  
Table1对比了SQL Server和苏格拉底.   
- 更大的实例：本地磁盘->远端存储  
- 可用性提升：  
- 扩缩容：不再需要全量拷贝数据  
- 存储副本需求：  
  - SQL Server 4x copies(+backup)：四个主备实例分别独立存储；  
  - 苏格拉底 2x copies(+backup): PageServer的热备，XStore算backup? PageServer算1x copy, XStore通过EC等算1x copy?  
- Recovery: SQL Server备实例已经同步了全量的数据，苏格拉底存储分离，把备实例切换为主耗时O(1)  
- Commit Latency: SQL Server需要把日志同步到备实例以保证可靠性，苏格拉底把日志的可靠性交给XLog  
  - 3ms vs. 0.5ms  
- Log Throughput:   
论文讲解了苏格拉底如何实现这些提升的。  
  
# 数据库架构的发展情况  
## SQL DB  
基于HADR架构(high-availability and disaster recovery)，经典的日志状态机。Primary Node(PN)处理所有的更新，并ship log到Secondary Node(SN).   
另外，还定期备份到Azure's Standard Storage(XStore)  
- **每5分钟**备份一次log  
- **每天**备份a delta of the whole database(发生变化的数据：更新过的和新增的)  
- **每周**全量备份一次  
  
## HADR介绍[2]  
- PN+SN同一个region: 同步  
- PN+SN不同的region: 异步  
- PN+部分SN同一个region: 同步；与部分SN不同的region: 异步  
  
Secondary负责只读，如果Primary失败，则某个SN变成PN。一个PN + 三个SN. 如果四个node都出问题，则存在数据丢失，因为日志每5分钟备份一次。  
  
HADR架构成功服务了百万实例。  
- 高性能: 每个compute node都有全量的本地拷贝，读取时不需要访问远端存储。  
- 不好的地方：实例大小受限于本地存储容量。与实例大小相关的operation也是问题。比如，扩充一个node，耗时与实例大小成比例。包括，备份/恢复，scale-up/down等操作。这也是为什么实例大小限制为4TB.  
  
## Spanner: log-replicated state machines  
- 为了解决O(size-of-data) issures, 逻辑划分数据到多个partition  
- 每个partition的拷贝使用paxos来保证一致性  
- Spanner支持地理复制，并在TrueTime设施的帮助下保持所有副本的一致性，TrueTime设施是一种基于数据中心的时间源，可以限制不同副本之间的时间漂移。为实现负载均衡和容量管理，拆分和合并是动态的。  
  
## shared disk: AWS Aurora  
计算存储分离。  
  
## Oracle  
基于Exadata和Oracle RAC  
  
# SQL SERVER的重要特性  
苏格拉底基于SQL Server构建，复用了SQL Server的功能和相关的服务，因此介绍SQL Server几个的特性：  
- record支持多版本。提供快照读（实现快照隔离）  
- Buffer Pool Extension(BPE): spills in-memory database buffer pool to a local SSD file  
  - 苏格拉底进一步升级为**Resilient Buffer Pool Extension(RBPEF)**  
    - Resilient: 弹性可恢复  
    - 减少MTTR(mean-time-to-recovery)，不需要从远端拉取  
- 基于快照的备份和恢复  
  - 利用Azure Storage(XStore)提供的快照技术来支持实时备份  
  - 备份和恢复的耗时固定，与数据库大小解耦，不消耗计算层的CPU和IO。使用XStore的快照机制，百TB的实例在分钟级别完成文件的加载。当然，恢复到指定时间，需要更长的时间来apply log, 启动Server并刷新缓存，但时间开销都不依赖于实例大小。  
  - 备份、恢复是一个重要的例子，即苏格拉底的一些关键操作流程耗时不受数据库大小的影响。  
- 使用抽象层FCB(File Control Block). 苏格拉底探索新的FCB, 对计算节点隐藏了存储节点的层次细节。因此，不用改变SQL Server的核心模块。【抽象】  
  
# SOCRATES架构  
## 设计目标和原则  
HADR服务了大量实例，总结了很多经验，基于这些经验设定苏格拉底的设计目标和原则  
- Local Fast Storage vs. Cheap, Scalable, Durable Storage // 如何选择和使用  
- Bounded-time Operations   
  - **Operations耗时与实例大小解耦**  
- From Shared-nothing to Shared-disk   
  - HADR架构:shared-nothing，数据库存储多个全量副本在本地。无法达到100TB的目标，单机没有那么大的存储。  
- Low Log Latency, Sepration of Log  
  - **日志是OLTP数据库的瓶颈**。事务需要在日志完成持久化之后才能完成提交。  
  - 苏格拉底使用一个**独立的日志服务**。因此，可以单独设置和调试日志系统。  
  - 相比HADR复制状态机，日志系统的**quorum机制执行速度更快**。因此，表现更好的提交性能。  
- Pushdown Storage Functions  
  - 把一些function从计算层卸载到存储层，以得到性能的显著提升。包括backup, checkpoint(把修改过的page存储到XStore), IO filtering, 解放Primary Compute和日志系统, 这两个关键瓶颈. 【负载转移】  
- Reuse Components, Tuning, Optimization  
  - 复用已有的生态和工具，组件。  
  - 已有的实例兼容迁移到苏格拉底。  
  - 支持T-SQL和数据库管理API。  
  - SQL-Server是一款成熟的企业级应用。因此，查询优化，query runtime, 安全，事务管理，恢复等直接复用。苏格拉底也继续拥抱scale-up满足大多数OLTP负载要求。  
  
## 苏格拉底架构  
苏格拉底架构遵循前面提到的原则和目标。  
- 计算存储分离  
- **分层扩展的存储**  
- bound Operations. 即操作耗时不与数据库大小相关  
- Log从计算和存储中分离【日志是系统瓶颈】  
- 尝试把Functions转移到存储层  
- 复用已有的组件和系统  
![Figure 2: Socrates Architecture](https://chenghua-root.github.io/images/socrates-architecture.png)  
苏格拉底四层架构  
- 计算节点: 缓存数据在memory和ssd, RBPEX  
- XLOG service: low commit latency and good salability. PN 单写. 其它节点(如Secondary, PageServer)异步消费日志，以保持数据最新。  
- storage tier: Page Server. 每个PageServer存储一个partition的一份拷贝。  
  - serve pages to compute nodes. bulk Operations: loading, index creation, DB reorgs, deep page repair, table scans  
  - checkpoint data pages and create backups in XStore  
  - 缓存其**所属的所有数据**在内存或者local SSD以加速访问(RBPEX)  
- XStore: scalable, durable, cheap storage service based on hard disks. 存在吞吐和延迟的限制  
**计算节点和PageServer are stateless**. 数据存放在XLog和XStore  
  
## XLOG Service  
![Figure 3: XLOG Service](https://chenghua-root.github.io/images/socrates-xlog-service.png)  
Primary写日志到Langding Zone(LZ), 基于Azure Premium Storage service(XIO)实现。  
- XIO: 三副本(跨机房?) 性能，成本，可用性，可靠性的tradeoff  
**Landing Zone**  
- Fast, Small  
- 被当做circular buffer使用，日志格式与传统的SQL Server格式兼容  
- 关键属性：支持多个读者可以读取一致性的日志，而不需要与生产者同步。  
  - 分层系统中最小化层之间的同步，可以使系统更具扩展性和弹性  
**XLOG Process**  
Primary除了写LZ外，还并行写日志到XLOG Process.  
为了避免写LZ失败，而日志却被消费，导致的不一致，需要做一定的同步。  
**同步:**  
- 日志写到XLOG Process时，先放入Pending Blocks  
- 待Primary告知哪些日志持久化LZ成功后(状态变为"hardened")，才移入LogBroker  
- 在LogBroker中排序和补齐  
**Destaging(离台)**  
- 为了分发和归档日志，process destaging把日志写入local SSD cache for fast access  
- 以及写入XStore用于长期保存(long-term archive: LT)，默认保存30天，用于PITR和灾备  // 直接写入XStore, 没有再读一遍  
LZ(XIO)和LT(XStore)分别满足了快速提交和成本取舍。  
  
**日志的消费**  
只读节点和Page Server从XLOG service“**拉取**”日志。  
拉取消费日志的方式，使得架构更加具有扩展性，因为**LogBroker不需要跟踪日志的消费者**，如可能存在数百个Page Server. (PageServer自己计算和维护消费情况)  
  
**LogBroker**  
LogBroker在内存中维护了一个存储log blocks的hash map: 被叫做Sequence Map.  
理想情况下，所有的日志都能从Sequence Map中获取。  
如果已从Sequence Map中淘汰，则依次尝试从local SSD, LZ, LT中读取。  
  
XLOG进程实现了分布式DBaaS系统所需的其他一些通用功能：日志生命管理、LT blob清理、备份/恢复情况的记录、日志使用者的进度报告、块过滤等。所有这些功能都经过仔细选择，以保持XLOG进程的无状态特性，允许轻松的水平扩展，并避免影响XLOG的主功能: serving and destaging log.  
  
## Primary Compute Node and GetPage@LSN  
苏格拉底PN节点行为跟普通SQL Server的一致。**不感知只读实例，也不感知数据在远端**。  
但也有如下不同点:  
- 存储层行为，如checkpoint, backup/restore, page repair等交给PageServer以及更低层的存储  
- 苏格拉底PN使用VFS机制写日志到LZ  
- PN也使用RBPEX  
- 苏格拉底本地没有全量数据，只是缓存热点数据在内存和SSD中(计算存储分离)  
  
getPage(pageId, LSN): **读取>=LSN最小版本的page数据**  
- "LSN identifies a page log sequence number with a value at least as high as the last PageLSN of the page."  
- "According to the GetPage@LSN protocol, the Page Server may return a page from the future. "  
  
考虑如下的事件顺序  
1. 在local buffers更新page X  
2. 写更新日志。由于压力，本地缓存中淘汰了page X  
3. Primary读取page X  
  
为了保证读取到page的最新版本，需要满足如下条件:  
1. 等待page应用到X-LSN  
2. 返回Page X  
  
Primary如何读取到page的最新版本。PN并不记录所有page的LSN。PN构建一个hash map(on pageId), 尝试记录被淘汰page的last modifyed LSN。  
记录方式：  
- 构建一个固定大小的map, 包含固定数目的bucket, 被淘汰的pageId映射到某个bucket  
- 每个bucket会被映射多个page, 此bucket只记录映射page的最大的LSN  
- 如果读取的某个page没有被修改过，也可以直接hash获取对应bucket的最大的LSN  
- bucket个数: slots = BufferPoolSize / PageSize / srv_n_page_hash_locks(16) / 2  
- 读取page时，使用page对应的bucket记录的LSN (>= page's last modifyed LSN)  
NDB计算节点也采用了类似的方法来登记。  
  
*读取时不直接全部使用最大LSN的目的是，某些page还没有shipping成功，PageServer无法保证返回最新的数据，因此需要通过某种方式判断请求的page是否已经完成shipping, 如果没完成则阻塞。*  
- *NDB阻塞方法: 在LogSDK基于LSN来判断, 请求的LSN大于还未完成shipping的LSN，则阻塞（返回错误）.*  
- *Socrates: XLOG Service不维护消费状态，计算节点直接从PageServer读取，则阻塞判断放在PageServer*  
*因此，读取page时LSN使用大于等于其最新的LSN，但尽量小，以避免被阻塞。*  
*PageServer需要向前推进自己的shipping LSN, 以尽量避免阻塞read page请求。*  
*某个PageServer即使一直没有所属page的日志，也会通过心跳来向前推进自己的shipping LSN点。*  
  
## Secondary Compute Node  
SN的实现采用复用的设计原则。Secondary按照HADR同一的方法来apply log. 不同点是，secondary不需要存储log block, 这是XLog service的工作。  
  
处理只读事务: 满足Snapshot Isolation  
  
与HADR架构不同的是，苏格拉底的Secondary没有独立的全量数据拷贝。这样才能做到计算节点无状态。  
Secondary处理日志时对应的page可能不在buffer中，有多种处理策略，一种是去拉取后更新。secondary的cache几乎与Primary的缓存状态一致. 当前的实现是，log对应的page不存在，则忽略. NDB也采用忽略的策略。  
  
## Page Servers  
PS功能:  
1. 维护数据库的一个分区，并应用日志  
2. 响应计算节点的GetPage@LSN  
3. 执行checkpoint 和 发起backup  
  
只处理本partition的log，响应本partition的page  
与计算节点一样，使用RBPEX，recoverable cache. 缓存此partition所有的page.  
XStore不可服务时，PS可以继续运行，只写入了本地RBPEX未写入XStore的page会被标记。当XStore恢复时，checkpoint继续推进过程中会拉取这些page.  
  
*PS需计算或记录本node已经ship到XStore的LSN*  
*checkpoint异步批量操作，并记录本partition的page已经完成shipping的LSN到XStore.*  
*Logs in XLog和Pages in XStore保证了全部的数据*  
  
启动一个新的PS时，可以异步lazy加载其partition的所有page, 在加载过程中即可以提供服务，以做到快速启动。  
PS与XStore交互来创建快照和备份。  
XStore for Durability and Backup/Restore  
XStore提供高效的backup和restore, 基于log-structured设计(备份只需要在日志中保留一个指针)  
  
XStore(HDD)很慢，相当于传统数据库的本地硬盘，而计算节点和PS则扮演了内存的角色。  
一旦这样理解了，就能更直观的理解苏格拉底的checkpointing, recovery, backup, restore  
  
checkpointing是指PageServer周期性的ship被修改过的page.  
  
Backup, 利用XStore的快照功能，能在一个固定的时间完成备份，并对应一个时间戳。  
- 当用户请求一个Point-In-Time-Restore operation(PITR), restore workflow 找到快照, 其完成时间点小于PITR; 再拉取快照到PITR的log range  
- 基于此快照拷贝数据到新的blobs(需要真正的全量拷贝数据吗)，启动一个新的PS instance，新的XLog process, 并应用日志到PITR请求的时间点  
- PITR恢复完成后，则可以提供服务了  
PITR高效：打快照，恢复快照，元数据操作，耗时较短。  
基于此，快照实例很快可以使用，不过在page全部加载到PS之前，性能会打折。  
  
# SOCRATES AT WORK  
**设计原则**，这些mini-services包括Primary, Secondary, XLOG, PS都按照如下设计  
"The Socrates mini-services like Primary, Secondaries, XLOG, and Page Servers are autonomous and decoupled and communication is asynchronous whenever possible."  
- 自治  
- 解耦  
- 异步  
mini-services不感知其它mini-services. 如Primary不知道有多少台PS.  
  
这些设计也遵照上面的原则:  
- distributed checkpointing(across all Page Servers), 每个Page Server 管理自己范围内的page  
- fail-over of Primary, scaling up and down(serverless)  
- 创建一个新的Secondary  
- management of leases for log consumption  
- creating new Page Servers  
  
# 讨论 & 苏格拉底的部署  
Socrates架构颇具优点。  
- 计算存储分离，存储可以超越单机进行扩展  
- 有助于在云服务中建立更细粒度的随用随付原则，客户为他们的存储使用量付费，独立为他们消耗的计算付费  
  
计算存储分离让socrates得到了这些优点，但也有缺点，读取的时候跨节点访问导致延时更大  
Socrates通过计算节点的缓存(内存+磁盘)来弱化此问题  
  
socrates的创新在于可用性和持久性分开。  
- 分层的XLOG和XStore负责持久性，计算节点和PageServer负责可用性，如果他们不可用，数据不会丢失，但变的不可用。  
- 可用性和持久性分开，最大的优势，能够更灵活更加细粒度的控制availability / performance / cost之间的trade-off. 可以根据应用需求来定制部署。  
  
最小部署：  
XLOG和XStore是必须的，可以只启一个Primary Compute Node，不启动Secondary，启动一个PageServer:负责整个实例。  
*问题：*  
1. *PS这一层做了实例/业务隔离*  
2. *根据最小部署，推测XLog Service是嵌在DBEngine里面的, 或者作为独立进程跟DBEngine部署在同一个pod*  
  
PS的可用性提升:  
1. 更多的page Server, 更细粒度的partition，单个PS负责的partition更小，恢复更快。更多的partition，性能也更好。  
     基于网络和硬件参数，一台PS负责的partition大小128GB为最宜。 // pod + 本地SSD  
2. PageServer做热备，replica is ready and hot.   
  
*其它方式：*  
- *升级时，启动一个新版本的PS，变热后，把服务从旧的PS切到新的PS*  
  
跨地域部署对提高可用性和性能有帮助。  
- 允许把Secondary和PS部署在不同的可用域。业务可以访问所在可用域的Secondary，但是shipping log需跨数据中心会导致开销增加。  
  
苏格拉底架构另外一个重要的优势是可以利用已有的云服务。  
XStore高效的备份（PITR），更加规模和更便宜的方式支持持久性。  
XLOG tier依赖Azure Premium Storage(使用SSD的高性能存储)  
计算节点和PS只做数据库功能。不用管其它服务的多副本，更趋于云服务化。  
  
# 性能测试和结果  
CDB: Cloud Database Benchmark, 用于测试Azure上的微软数据库  
 Experiment 1: CDB Default Mix, Throughput, Production Cluster  
![Table 2: CDB Throughput: HADR vs. Socrates (1TB)](https://chenghua-root.github.io/images/socrates-cdb-throughput.png)  
Experiment 2: Caching Behavior  
![Table 3: Socrates Cache Hit Rate (CDB)](https://chenghua-root.github.io/images/socrates-cache-hit-cdb.png)  
![](https://chenghua-root.github.io/images/socrates-.png)  
![Table 4: Socrates Cache Hit Rate (TPC-E)](https://chenghua-root.github.io/images/socrates-cache-hit-tpc-e.png)  
- 320GB/30TB ~= 1%,  32%的缓存命中率  
Experiment 3: Update-heavy CDB, Log Throughput  
![Table 5: CDB Log Throughput: HADR vs. Socrates](https://chenghua-root.github.io/images/socrates-cdb-log-throughput.png)  
为什么Socrates表现更优:  
- HADR计算节点除了响应客户端请求，还需要并行的处理日志和数据库的备份  
- Socrates则可以利用XStore的快照进行备份，从而在Socrates Primary达到更高的日志生产率  
  
# 结论  
1. 存储计算分离，有效的利用已有的云服务，带来更高的可用性和弹性  
2. 持久性和可用性分离，是一个创新，还没有在其它地方看见  
3. 更灵活的满足用户对成本，性能，可用性的需求  
  
Socrates是一个创新性的架构，依然理解和探索其的全部潜能。  
- 多主  
- HTAP  
- 审计和安全  
  
# *学习&总结*  

- 持久性(可靠性)与可用性解耦  
  - 持久性: 依赖XLog Service: LT(LZ(Local SSD(Memory: Sequence Map)))和XStore  
  - 可用性: 依赖Primary/Secondary node, PageServer  
- 模块之间自治，异步，解耦，模块自身无状态  
  - mini-services包括Primary, Secondary, XLOG, PS都按照自治，异步，解耦的原则进行设计  
  - XLog Service不维护消费者的状态  
  - 计算节点和PageServer都可以做到无状态  
- 简化链路  
  - 计算节点read page不经过XLog Service  
- 操作耗时与实例大小解耦  
  - 所有的page存储在XStore上，依赖XStore的快照能力能够快速（分钟级别）进行PITR，满足本地快照构建实例的速度  
- Page存储全部构建在XStore上  
  - 直接利用XStore的快照来支持本地快照和PITR的功能  
- PageServer是计算+缓存的角色  
  - 全量缓存本partition的page  
  - PageServer上启动一个DBEngine(改动后)来管理缓存和  
  
[1] Socrates: The New SQL Server in the Cloud  
[2] https://learn.microsoft.com/en-us/azure/azure-sql/virtual-machines/windows/business-continuity-high-availability-disaster-recovery-hadr-overview?view=azuresql#availability-groups-hadr  
