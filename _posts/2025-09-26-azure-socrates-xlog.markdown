---  
layout: post  
title:  "Azure Socrates Log Service"  
date:   2025-09-26 00:00:00 +0530  
---  
  
本文内容和结构遵照Socrates XLOG论文[1]  
  
本论文为 Socrates: The New SQL Server in the Cloud 的后续论文，理解本论文需要先看Socrates  
  
早期版本上线后存在一些挑战，**本论文详细举例了如何进行工程实践来解决这些挑战，如：任务状态机复杂度管理，以及对所有客户端的log请求统一管理(XLOG Pool)**  
  
*斜体为笔者补充的内容*  
  
# ABSTRACT  
XLOG是Azure Hyperscale分布式数据库(对应论文socrates)的集中式日志服务模块。  
负责把redo log分发给所有客户端，对应的消费者为page servers和secondary compute nodes(只读计算节点).  
  
挑战：  
- 规模增长，page server + Secondary数量变多, 客户端数量增加  
- 需要处理更多的请求 + 对相应的IO子系统也带来了更多挑战  
  
方案:  
- 线程耗尽：基于协程的异步日志处理框架  
- 引入XLOG Pool: 整合客户端的日志读取请求，减少冗余I/O并提升冷启动性能  
- RateLimiter(令牌桶): 避免Azure存储的限流问题  
- 数据完整性: 出口校验，端到端校验  
  
生产环境可支持128TB数据库。  
阐述了一个托管关键任务应用并管理数百 PB 数据的真实世界云数据库服务，如何通过创新其日志服务来提升可扩展性、可靠性和成本效益。  
  
  
# INTRODUCTION  
  
一个集中式的云数据库日志服务面临的工程挑战。  
  
# BACKGROUND  
  
## A. The Log Service  
  
XLOG:  
- C++写的服务  
- 响应日志请求  
  
![Fig. 1. Hyperscale Architecture](https://chenghua-root.github.io/images/socrates-xlog-figure1.png)  
  
- 持久化日志：日志被发送给Landing Zone(LZ)和XLOG, 当日志在LZ完成持久化，XLOG就把日志移到Broker, XLOG中的内存cache.  
- 读日志: XLOG响应clients的日志请求顺序，Broker(内存), local cache(LC, 本地磁盘), LZ, long-term storage(LT).  
  
- 服务整个database(对应mysql实例?), 支持128TB  
- 4 CPU, 40GB RAM的XLOG最大支持10TB的database  
- 每个database一个XLOG  
- XLOG本地磁盘不保存任何关键状态，以方便在VM之间迁移。所有必须的数据都存储在Azure Storage  
 - 目的: upport a large-scale database with a minimally sized XLOG service  
  
## B. XLOG Request Processing  
  
RBIO:  
- 基于TCP, 类似Apache Thrift或者Protocol Buffers.  
- 提供RPC接口，如GetLogBlocks(bsn, size, timeout)  
 - timeout: 当日志不够size的时候，可以等待的时间. long polling模式，避免暂无日志或者日志不够时立即返回。  
  
XLOG分层读取：  
- 依次尝试从Broker, LC, LZ, 直到LT去读取日志  
- 此设计在概念上看似合理，实际部署中暴露了若干复杂性问题和运维调整。  
- 后文详细探讨这些问题，重点分析对性能的影响，并提出可能的解决方案。  
  
## C. Long-Polling and Thread Starvation  
  
Long-Polling  
- 客户端从服务端(XLOG)拉取日志，若有日志则立即返回  
- 若没有日志，为了减少询问开销，XLOG不立即返回，而是等有数据或者等服务端超过API指定的超时时间后再返回  
  
Thread  
- original thread and request handling model  
- 线程数量与分配的CPU资源存线性比例  
- 为了减少context switch和控制内存开销，使用了线程绑定  
  
问题  
- 假设每个CPU分配10个RPC服务线程  
- 如果无新日志时，超时（通常1秒）后才返回，大多数情况下没问题，当客户端(Page Server)数量一多，则存在部分请求不能立即响应的问题  
- For a 100TB database, 800 Page Servers are needed (each covering 128GB of data) for a total of 1600 clients(PS热备).  
  
![Fig. 2. Original request processing model using a simple thread pool to handle incoming requests.](https://chenghua-root.github.io/images/socrates-xlog-figure2.png)  
  
## D. I/O Scalability Problems  
  
XLOG分层寻找.  
  
在系统升级期间，部分客户端副本会被下线并进行补丁更新；当这些副本重新上线时，它们会请求那些已从内存缓存中被截断的日志。在某些情况下，这些副本的滞后量会超过 128GB——这是本地 SSD 缓存的容量上限。这种情况下，就必须从远程存储在 Azure 高级存储中的着陆区（LZ）读取数据，而该存储介质的速度要慢得多。  
  
客户端的滞后程度越严重，针对着陆区（LZ）blob 的读取操作就越多。这会加剧存储限流问题 ——XLOG 实际上会代表所有客户端向 LZ blob 发起读取 I/O 请求。  
  
每个客户端分配一个迭代器对象，LZ迭代器使用了预读机制（因为读取操作是顺序的）。随着客户端越来越多，内存需求增加。  
  
# Improvements to thread handing  
  
方案与效果：  
- 使用协程来解决线程耗尽的问题。  
- 使用非阻塞来代替long-polling的阻塞等待行为。  
- 改进后：请求的当前执行状态会被封装到一个对象中，并被推迟到延迟队列中，并记录下次恢复执行时的状态(*状态机模型*)。线程可以被释放出来处理下一项任务。  
- 这使得只要有工作可做，线程能够始终保持忙碌状态（而不是像之前被阻塞）. 此模式可以应用于任何阻塞情况，包括“等待日志的long polling”，以及异步I/O读取数据时。  
  
实现挑战：  
- 由于各种技术原因，很难将任何外部库集成到我们的生态系统中，因此这个异步日志处理框架是从零开始编写的。自主开发库能够让我们根据自身需求对其进行调优。  
- 该框架并非自动化的：开发者需要负责将线性执行（且可能存在阻塞）的子例程拆分为有限状态机，定义状态及其转换规则，并按照特定接口将业务逻辑重构到这些状态模块中。而框架会负责这些状态机的调度与执行，包括停滞检测（功能）。  
  
![Fig.3 Asynchronous log processing pipeline](https://chenghua-root.github.io/images/socrates-xlog-figure3.png)  
  
图3展示了一条请求之旅:  
(1) 新请求被RBIO处理，并被封装为一个item. (之前是请求被分配给线程池处理，线程会被此请求“全部”占用).  
(2) item放入work queue, item包含了执行状态等以及其它属性。这样请求就可以被阻塞和跨线程执行。  
(3) 放入work queue的item会被某个线程取出并执行  
  
(4) 如果执行时需要阻塞，则被丢入deferral queue  
(5) 系统会定期检查延迟队列（例如，当工作线程的任务队列为空时，或者在某些工作项可能符合恢复条件的适当时机），任何符合恢复条件的工作项都会被移回工作队列以进行进一步处理。有些工作项可以无需延迟直接处理（例如，那些无阻塞的工作项），并且在实际操作中会按顺序运行。  
(6)一旦工作项处理完成，系统会创建响应并将其发送回客户端。  
  
**状态机复杂度控制**:  
- 针对包含复杂的内部处理的工作项，引入了“子工作项”的概念，主要是为了简化状态管理，并让工作项的生命周期更易于理解和分析。  
- 子工作项会与其父工作项建立关联，并且执行方式与其他普通工作项一致，但有一个例外：当子工作项被加入队列后，父工作项会进入延迟状态；而当子工作项完成时，它会释放父工作项的一个引用计数，使父工作项能够恢复执行。  
- 大多数工作项只包含一个子工作项，不过从实际功能来看，该框架也支持多个子工作项。这些子工作项可以并行执行。只是目前尚未有使用该功能的需求。  
  
In Fig. 3, a and b are pending work items, waiting to run. cand d are deferred work items that are either waiting for their children to complete, or another condition that would make them eligible to resume. Work items e,  f and g are currently being executed by threads in the pool.  
  
工作项:  
- 实心矩形代表顶级请求。虚线框的工作项代表子项。目前，嵌套层级没有限制，目前为止只需要单一层级的嵌套。  
- 每个工作项都知晓自身的唤醒条件，这一条件通过虚方法IsReadyToResume() 实现：框架在处理延迟队列时会调用该方法，以此判断对应的工作项是否可以被放回工作队列。  
  
- 例如：一个顶级请求会被表示为一个工作项（AsyncLogReadRequest）。如果该请求发现所需的日志存在于 Broker 中，它会创建一个子工作项（ReadBrokerWorkItem）并将其与父工作项关联，以此处理这个子状态机。当 Broker 完成处理后，它会将结果传递给父工作项，通知父工作项继续执行，然后自行销毁。一旦有工作线程可用，处理流程就会恢复执行父工作项。  
  
这些改进使得 XLOG 能够支持更多的客户端，并且彻底消除了线程饥饿问题。具体而言，通过这些优化，XLOG 在仅使用 4 核 CPU 的情况下，就能轻松处理 100TB 的数据库规模，而无需按照数据库大小成比例地增加线程数量。  
# IV. IMRPOVEMENTS TO I/O HANDLING  
  
客户端读日志的行为模式：  
1. 顺序读  
2. 所有客户端读取的日志一样  
3. 客户端日志流位置接近  
  
XLOG Pool: 管理所有指向各种存储卷(本地SSD, LZ, LT)的IO操作。维护这自己独立的内存缓存。  
  
请求先尝试从原来的缓存(Broker)读取，如果没有命中：  
- 之前的实现方式：查找各自（客户端）的私有迭代器，并从请求位置开始扫描；  
- 现在：会在XLOG Pool缓存中检查读取的块  
  
![Fig. 4. XLOG Pool high level architecture](https://chenghua-root.github.io/images/socrates-xlog-figure4.png)  
  
队列与线程:  
- 如果没找到(XLOG Pool缓存)，会把请求范围加入队列，会与其它范围(其它客户端请求)进行聚合，去重和合并  
- 这些范围由一定数量的后台线程处理。每个存储层级只需要一个线程  
- 工作是扫描各自排序的 I/O 范围列表，并通过从所管理的存储介质中按顺序读取数据来处理这些范围，然后将数据缓存到 XLOG Pool 缓存中。  
- 异步处理  
- 如果请求的范围无法在第一个存储层级得到满足，就会被移至下一个存储层级的填充任务队列（类似于瀑布流），由相应的填充任务异步处理（范围的聚合和合并如前所述）  
  
![Fig. 5. Request ranges in each storage tier](https://chenghua-root.github.io/images/socrates-xlog-figure5.png)  
  
图5-请求range的处理:  
- 带阴影的框表示 BSN（块序列号, *类似MySQL LSN*）已不在该层级中。  
- （a）LC 填充层级中的一个范围被转移到 LZ 填充层级，因为发现它低于 LC 所覆盖的最小边界。  
- （b）出于同样原因，LZ 层级的一个范围被转移到 LT 层级。  
- （c）LZ 层级的一个范围被转移，并与 LT 层级中一个重叠的范围合并。  
  
每层range队列管理:  
- 每个填充任务都执行预读和顺序（不一定是连续的）读取操作，以最大限度地提高其存储设备的 I/O 吞吐量。  
- **请求可能在任意时刻进入（排序的）列表，而且迭代器在接收这些请求时可能处于任意位置。为避免打乱迭代器的位置并保留预读缓冲区，读取操作会继续按顺序进行，只有在完成一个完整周期后才会从起始位置重新开始。可以将其想象成单向电梯，或者借用计算机图形学中的术语 “扫描线顺序”。**  
- 每个存储层级的未处理范围列表相对较短，保持其排序的成本不高，而且时间无论如何都受限于 I/O 操作。如果列表长度过长，需要进行 O (n) 的内存扫描，那么很可能会出现其他问题（即 I/O 设备无法跟上输入负载）  
- 对 I/O 操作进行去重，每个存储设备的负载大幅降低，从而在这些设备作为存储服务时减少了节流情况。许多相同的请求也能够从内存中得到处理，无需进行冗余的 I/O 操作。  
- **无论客户端数量多少，迭代器的数量（以及因此产生的内存占用）都是固定的，这使得系统能够支持更多的客户端进行扩展。**  
  
XLOG Pool最能发挥作用的场景是冷启动性能。由于 Broker 内存缓存和本地 SSD 缓存丢失，大量涌向 LZ 和 LT 的读取操作会将日志请求吞吐量降至极低水平。而有了 XLOG Pool，即使内存缓存尚未完全填充，也能高效地为更多客户端提供服务。  
  
限流：  
- 使用令牌桶，三个参数：桶填充速率(r), 桶大小(s), 填充间隔(t)  
- r = 190MB/s  
- s = 19MB  
- t = 0.1s  
- *桶容量为一个token(每个间隔周期产生19MB), 相当于限速周期为0.1s*  
  
# V. IMPROVEMENTS TO DATA INTEGRITY(数据完整性)  
  
多层级验证：  
- 出口验证  
- 端到端校验和  
- xxx  
  
## A. 出口验证  
  
目的：内存空间是否可信  
- 维护一个内存数据结构，包含日志块标记(BSN), 字节长度，32位校验和. 每条记录占8+2+4 = 14字节  
- 使得我们能够在较小的内存占用下存储大量的日志历史。例如，对于 64MB 的内存，假设每个日志块的平均大小为 32KB，那么可以覆盖平均 100GB 的日志。  
- 提供每个日志块之前，会在哈希表中查找其标识，看它是否之前被提供过。  
- 如果有，则将此次的校验和和长度与之前的进行比较。32 位校验和与字节长度的组合足以保证块的内容没有发生变化。  
- 如果检测到不一致，该块将不会被提供，并且进程会自行终止，因为此时它的内存空间已不可信。之后，集群管理器会根据情况决定在同一节点或不同节点上重启该进程。  
  
## B. 端到端校验和验证  
  
- 计算节点生成日志时，在其头部标记一个校验和  
- 在XLOG, LZ, LT, 客户端使用的时候校验  
  
## C. 关键代码路径中的安全整数运算  
  
- header + records + tail(offsets, lengths). 当数据出错时，offset或length会异常增大等  
  
# VI. PERFORMANCE EVALUATION  
  
## A. 异步日志处理的能力  
  
![Fig. 6. Async Pipeline Performance](https://chenghua-root.github.io/images/socrates-xlog-figure6.png)  
  
- 32核320个线程  
- 异步关闭：客户端数量超过320个后，出现长尾(线程饥饿)，日志消费速率降至个位  
- 异步开启: 消费速率不受客户端数量影响  
  
## B.异步日志处理对TPCC基准测试的影响  
  
![Fig. 7. TPCC Performance vs. DB Size with a baseline run at 10 Page Servers.](https://chenghua-root.github.io/images/socrates-xlog-figure7.png)  
  
- 2核, XLOG线程数20, 10000个实例  
- 10个pageserver服务器配置作为性能基准(每分钟事务)  
- 异步关闭: pageserver数量超过20个后数量越多性能下降越厉害  
- 异步开启: 几乎无影响  
  
## C. 速率限制器和XLOG池的性能  
  
LZ本身有限速，当多个pageserver读取大量日志, 并发的IO请求带宽和超过LZ限速, 每个pageserver的带宽会急剧降低。  
共享XLOG池消除了重复IO.  
  
## D. XLOG池对XLOG冷启动的影响  
  
XLOG池提高了并行客户端从远程层读取的整体吞吐量，因此它也改善了XLOG的冷启动性能。  
  
客户的工作负载在XLOG重启时正在生成高流量日志，那么XLOG重启对客户的影响是巨大的。消费追不上生产。  
  
使用XLOG池后，XLOG重启的影响被限制在不到五秒钟。  
  
## E. 对生产工作负载的影响  
  
- 节约配置(请求管理+XLOG池), 并且一个XLOG可服务多个数据库实例  
- 增强系统弹性: 自动检测数据损坏, 并启动快速自恢复  
- DevOps减少: 大量的页服务器可能同时离线然后同时恢复在线，用读取请求淹没XLOG。经过上面的改进后，已经能够应付这样的场景。  
  
# VII. RELATED WORK  
  
# VIII. 经验教训与未来工作  
  
经验:  
- 线程池并非解决可扩展性挑战的通用方案  
- 云存储服务（特别是 Azure Blob 存储）的交互，在集群处于大规模下的性能优化至关重要。  
- 在处理管道中的多个阶段验证校验和有助于早期检测并防止损坏数据的传播，从而提高整体系统可靠性。  
  
未来工作:  
- 统一各种内存缓存  
- 正在考虑的途径是将 XLOG 拆分为多个更小的服务（代理/缓存服务、数据下刷服务、日志租约服务等）。这将带来多重好处，例如在推出错误负载时减少爆炸半径，因为在滚动升级后它们将位于独立服务中从而保持内存缓存，更好地测试 XLOG 执行的独立功能  
- 考虑对 LZ 使用替代存储设备，例如使用三路同步复制的 SSD 日志来代替 XIO，以降低写入延迟。  
  
# IX. 结论  
- 通过精心的软件工程，在处理长轮询风格的请求时，少量线程可以比大量线程更高效。  
- 通过整合 I/O，减轻了对设备的压力，并将吞吐量显著提升至比以往更高的水平，从而改善了大规模下数据库的性能。  
- 通过为关键缓冲区添加详细的内存验证，在客户数据被破坏之前避免了严重的错误。  
  
[1]. Scaling and Hardening XLOG: the SQL Azure Hyperscale Log Service  
