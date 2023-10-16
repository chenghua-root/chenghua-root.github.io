---  
layout: post  
title:  "线性一致性及其验证"  
date:   2023-06-13 00:00:00 +0530  
---  
  
## 文章内容  
- 事务一致性和分布式系统一致性的区别  
- 顺序一致性和线性一致性  
- 线性一致性的验证  
  
## 一致性模型  
  
![linearizability-consistency-module](https://chenghua-root.github.io/images/linearizability-consistency-module.png)  
  
一致性模型如图所示，图左半部分为事务一致性模型，图右半部分是分布式系统一致性模型（CAP理论中的一致性）。  
  
从上到下，一致性模型级别越来越低，并发性越来越好。  
  
### 事务一致性模型
不同数据库对于事务隔离级别的定义和要求有所不同。  
- RC: 读已提交。OLTP数据库要求的最低隔离级别，如OceanBase和MySQL支持配置为RC  
- RR: 可重复读。经典数据库采用的隔离级别，如MySQL和Oracle  
- SI: 快照隔离。当前大多数分布式数据库采用的隔离级别，如TiDB  
- Serializable: 可串行化。最强的事务隔离级别，MySQL和Oracle可配置，FoundationDB提供SSI  
  
### 分布式系统一致性模型  
- Sequential: 顺序一致性。 如ZooKeeper提供顺序一致性 [ZooKeeper Consistency Guarantees](https://zookeeper.apache.org/doc/current/zookeeperInternals.html#sc_consistency)  
- Linearizable: 线性一致性, 线性一致也叫强一致性，严格一致性。  
  
分布式系统一致性其实主要是描述了在故障和延迟的情况下副本间的状态协调的问题。  
常见的实现线性一致性系统的最成熟的就是共识算法 (Consensus Algorithm)，例如 Paxos、Raft 等。  
  
本文主要讨论顺序一致性和线性一致性以及线性一致性的验证。  
  
### 事务一致性和分布式系统一致性的对比  
![linearizability-versus-serializability](https://chenghua-root.github.io/images/linearizability-versus-serializability.png)  
- DDIA Chapter9: Consistency and Consensus  
- 数据库事务一致性和CAP理论一致性对比。数据库事务一致性包含了并发隔离，如两个账户之间转账，不可能读取到中间状态。  
  
## 顺序一致性  
  
### 提出  
"... the result of any execution is the same as if the operations of all the processors were executed in some sequential order, and the operations of each individual processor appear in this sequence in the order specified by its program."   —Leslie Lamport, 1979  
  
"执行结果与这些处理器以某一串行顺序执行的结果相同，同时每个处理器内部操作的执行顺序又与程序描述的顺序一致。满足该条件的多处理器系统我们就默认为是sequential consistency的。"  
  
即满足如下两点:  
- **条件I**: 每一个并发内部的执行顺序在此串行顺序中得以保证 (在串行执行的顺序中，每个并发内部的happen-before关系要保证，并发之间不保证happen-before关系)  
- **条件II**: 多个并发的执行结果与某一串行顺序执行的结果相同 (读取的结果等于最新修改的值或初始值)  
  
### 举例  
![linearizability-sequential-example](https://chenghua-root.github.io/images/linearizability-sequential-example.png)  
图例:  
- 初始值: x = 0, y = 0;  
- P1写入了x=4, 读取了y=2;  
- P2写入了y=2, 读取了x=0;  
  
例子中在满足条件I的排列方式共有6种，即:  
- 顺序1: Write(x, 4), Read(y, 2)[不符合:y=0], Write(y, 2), Read(x, 0)  
- 顺序2: Write(x, 4), Write(y, 2), Read(y, 2), Read(x, 0)[不符合:x=4]  
- 顺序3: Write(x, 4), Write(y, 2), Read(x, 0)[不符合:x=4], Read(y, 2)  
- 顺序4: Write(y, 2), Write(x, 4), Read(x, 0)[不符合:x=4], Read(y, 2)  
- 顺序5: Write(y, 2), Write(x, 4), Read(y, 2), Read(x, 0)[不符合:x=4]  
- 顺序6: Write(y, 2), Read(x, 0), Write(x, 4), Read(y, 2). [**符合**]  
  
由于与某一串行顺序（顺序6）的执行结果相同，因此图中的执行历史满足顺序一致性。  
  
## 线性一致性  
  
### 描述  
对于线性一致性来说，它也是要对执行历史找到一个合法的顺序执行过程(只要能找到一个即可)。但是这个顺序执行过程，不仅要保留线程内部的先后顺序，还要保留线程之间的操作的先后顺序。  
如顺序一致性举例中，除了两个Read分别在同一个并发的Write之后发生（顺序一致性要求）, 还有两个Read在两个Write完成后发生，Read(x, 0)发生在Write(x, 4)完成后，结果不满足线性一致性。  
  
因此线性一致性除了满足顺序一致性的条件 I 和条件 II 之外，还要满足一个条件：  
- **条件III**: 不同进程的操作，如果在时间上不重叠，那么它们的执行先后顺序，在这个重排后的序列中必须保持一致。  
  
### 如何定义进程之间的先后顺序  
- 任何请求都有一个开始时间(invoke)，还有一个结束时间(responce)或者超时(timeout/无返回)  
- 请求的执行可能在开始时间到结束时间的任意时间点完成；如果超时，则完成时间点可能发生在[invoke, ~)  
  
图例:  
![linearizability-linearizable-example](https://chenghua-root.github.io/images/linearizability-linearizable-example.png)  
- A的第一个read(x=0)发生在所有操作之前，第三个read(x=1)发生在所有操作之后  
- A的第二个read(x=0/1)发生在B的两个read操作之间，这三个事件又与C.Write(x, 1)重叠  
 - A的第二个read, B的两个read，以及C的write存在如下四种排列:  
   - C.write(x, 1), B.read(x=1), A.read(x=1), B.read(x=1)  
   - B.read(x=0), C.write(x, 1), A.read(x=1), B.read(x=1)  
   - B.read(x=0), A.read(x=0), C.write(x, 1), B.read(x=1)  
   - B.read(x=0), A.read(x=0), B.read(x=0), C.write(x, 1)  
- 因此: 图中6个操作在满足条件 I 和条件 III 的情况下一共有四种排列，然后检查并发执行结果是否满足某一种串行排列的执行结果(条件 II)。  
  
## 顺序一致性vs线性一致性  
- 都是符合某个顺序的执行结果  
- 顺序一致性只需要保证每个并发内部在顺序执行中的happen-before关系  
- 线性一致性除了满足并发内部的happen-before关系，还需要满足并发之间的happen-before关系  
  
## 线性一致性的验证  
  
如何判断系统是否正确提供了线性一致性?  
  
首先在通过多次运行获得一系列不同的执行历史，接着验证每组历史是否满足线性一致性，**只要有一个不满足，便可以说系统不满足线性一致性。**  
  
**但如果没有发现不满足的历史，也不证明系统一定正确**。然而，在工程中通过对大量的执行历史的验证，使得我们对自己的系统更充满信心，这就足够了。那么现在的问题转变为：如何验证一组执行历史是否满足线性一致性。  
通过客户端可以看到一个读写请求的发起和结束时间，而其真正在服务端的执行可能发生在开始和结束中间的任意一点。因此，**验证一组执行历史是否满足线性一致性的关键就是对其找到一组满足三个条件的串行执行序列，如果这组执行序列存在，则可以说这组执行历史是满足线性一致的。**  
  
### 举例  
设x初始值为0，现在有四个进程并发执行，其中两个执行历史如下，一个满足线性一致性，一个不满足线性一致性。  
  
#### 满足线性一致性的执行历史1  
![linearizability-history-1](https://chenghua-root.github.io/images/linearizability-history-1.png)  
  
执行历史1可以找到一组满足三个条件的串行执行序列，C1.Put(x, 0), C4.Get(x)->0, C2.Put(x, 1), C3.Get(x)->1  
  
#### 不满足线性一致性的执行历史2  
![linearizability-history-2](https://chenghua-root.github.io/images/linearizability-history-2.png)  
  
我们尝试对执行历史2找到一组满足三个条件的串行执行序列。  
  
**我们先满足条件I和条件III:**  
条件I: 由于每个进程都只执行了一步，因此任何排列都满足条件I.  
条件III, C2必须在C1之后，C4必须在C1和C3之后  
  
因此满足条件I和条件III，有如下5种排列:  
 - C1, C2, C3, C4  
   - 按照串行执行，C4.Get(x)应该等于1, 不满足条件II  
 - C1, C3, C2, C4  
   - 按照串行执行，C3.Get(x)应该等于0, 不满足条件II  
 - C1, C3, C4, C2  
   - 按照串行执行，C3.Get(x)应该等于0, 不满足条件II  
 - C3, C1, C2, C4  
   - 按照串行执行，C3.Get(x)应该等于0, 不满足条件II  
 - C3, C1, C4, C2  
   - 按照串行执行，C3.Get(x)应该等于0, 不满足条件II  
  
**此执行历史没有找到一组执行序列满足3个条件，即此系统存在一组执行历史不满足3个条件**，因此此系统不满足线性一致性。  
  
## 线性一致性验证复杂度  
  
直观来看，线性一致性验证是一个排列问题，极端情况下的时间复杂度为O(N!), 即N个客户端共并发N个请求。事实上，Phillip B. Gibbons和Ephraim Korach在Testing Shared Memories中已经证明其是一个NP-Complete问题。虽然Gavin Lowe在Testing for Linearizability中给出了一些特殊限制下的多项式甚至是线性复杂度的算法，但在通用场景下，判定线性一致性并不是一个容易解决的问题，其搜索空间会随着执行历史的规模急速膨胀。  
  
### NP问题  
  
P问题  
- 如果一个问题可以找到一个多项式复杂度的算法，那么这个问题就是P(Polynomial)问题  
- x^n + x^n-1 + ... + 1  
- 冒泡排序: 能够找到一个x^2复杂度的算法，则冒泡排序是P问题  
  
NP问题  
- 不一定能在多项式复杂度内解决，但是能够在多项式时间内可以验证问题的一个解，这个问题就被称为NP(None-Deterministic Polynomial)问题（非决定性多项式问题）  
  
P = NP  
- 若问题的答案可以很快（多项式）验证，其答案是否也可以很快（多项式复杂度）被计算出来。  
  
NP-complete  
- NP问题的一个子集，即其它所有NP问题都能多项式时间内规约到NP-complete，同时也要保证能够在多项式时间内可以验证问题的一个解(NP问题的要求，NP-complete是NP的子集)  
  
NP-hard  
- those at least as hard as NP problems; i.e., all NP problems can be reduced (in polynomial time) to them.  
 - 至少是NP的难度  
- NP-hard problems need not be in NP; i.e., they need not have solutions verifiable in polynomial time.  
 - 不一定能在多项式复杂度内验证问题的解  
  
规约  
- 一个问题A可以约化为问题B的含义即是，可以用问题B的解法解决问题A。  
- 问题A可以规约化为问题B: A复杂度 <= B复杂度。  
 - 如果问题A能用问题B来解决，倘若B的时间复杂度比A的时间复杂度还低了，那么A的算法就可以改进为B的算法。两者的时间复杂度还是相同。  
- 规约性具有传递性：如果问题A可约化为问题B，问题B可规约化为问题C，则问题A可规约化为问题C。  
  
因此问题难度存在这样的关系: P <= NP <= NP-complete <= NP-hard  
  
![linearizability-np-complexity](https://chenghua-root.github.io/images/linearizability-np-complexity.png)  
  
### 线性化验证是NP-complete问题  
1. 证明线性化验证是一个NP问题：给出任何一个执行历史H的顺序化执行序列S，可以在多项式时间内验证这个顺序化历史S是否合法。  
2. 证明线性化验证是NP-hard：只需要从某一个已知的NP-complete问题规约到线性一致性验证问题即可。  
 - 找到一个NP-complete问题: 3SAT  
 - 3SAT问题(布尔可满足性问题: Boolean satisfiability problem)规约到线性一致性验证；论文TESTING SHARED MEMORIES中有推断过程  
  
因此: 线性化验证是个NP-complete问题  
  
## 验证算法  
给定一个执行history，怎么判断它是不是可线性化的呢？最简单的方法就是枚举，我们把给定history中包含的事件进行全排列，排除掉违反了happen before约束的那些排列方式，然后对剩余的进行验证，如果能找到一个合法的执行过程，我们就认为它这个history是可线性化的。但是这个算法的复杂度是n!，如果history包含了比较多的事件，基本上就没法接受了。  
  
虽然判定线性一致性的复杂度极高，但我们还是能够通过一些技巧，在大多数场景下，在工程可接受的时间内给出结果，这里介绍三个常见的，且一脉相承的通用算法。在此之前，先对算法面临的问题进行抽象，以下图执行历史为例，给出算法的输入和期待的输出：  
  
![linearizability-algorithm-example](https://chenghua-root.github.io/images/linearizability-algorithm-example.png)  
  
算法输入: 调用历史  
  
1. Client1: Invoke Put x=0  
2. Client2: Invoke Put x=1  
3. Client1: Return Put x=0  
4. Client3: Invoke Get x  
5. CLient4: Invoke Get x  
6. Client3: Return Get 1  
7. Client4: Return Get 0  
8. Client2: Return Put x=1  

算法输出: 执行序列  
1. Client1 Put x=0  
5. Client4 Get x=0  
2. Client2 Put x=1  
4. Client3 Get x=1  
  
即我们针对输入的调用历史找到了如上的一个满足一致性的执行序列。  
  
如果没有找到，即所有满足条件I和条件III的执行序列都不满足条件II，则系统不满足线性一致性。  
  
### WG(Wing & Gong)算法  
请求的调用历史中，存在着一种偏序关系：Prev，如果一个请求的Return发生时间早于另一请求的Invoke，我们便称其Prev另一个请求。显而易见，这种偏序关系是一致性验证算法必须要保留的。  
  
WG算法的思路非常简单：从调用历史中找出没有Prev的项，将其对应的请求执行并取出，之后对剩下的调用历史重复该算法，直到没有更多的调用历史或执行结果不满足。  
  
具体的算法伪代码如下：  
  
![linearizability-wg-algorithm](https://chenghua-root.github.io/images/linearizability-wg-algorithm.webp)  
- 这个算法是递归的，就是对于给定History，不断从里面找合法的minimal operation的过程。  
- 于minimal operation的定义如下：  
  - minimal op：no op happen before it.  
- 时间线上看，在最左侧的那个事件以及与它有交集的事件，都属于minimal operation。  
- 这个算法是在保证满足happen before的前提下进行搜索，排除掉那些不满足happen before关系的搜索路径。如果当前操作作用到对象后还是合法的，就继续递归往下走，看剩余的操作能否找到一个合法的顺序执行过程，如果剩余的能直接找到就返回成功。如果当前操作不合法，就把对象恢复到当前操作之前的状态，然后尝试下一个minimal op，继续搜索。  
- 这个算法比简单全排列要快，比如History中的事件之间很多都具有happen before关系，搜索会很快。但是如果有很多并行的事件，比如下图所示的极端情况下所有事件都是并行的，那么复杂度也会退化到n!。  
![linearizability-algorithm-complexity-n](https://chenghua-root.github.io/images/linearizability-algorithm-complexity-n.png)  
  
上面例子中，“Client1 Put x=0” 和 “Client2 Put x=1” 由于其Invoke前没有任何请求Return，可以首先被取出。假如选择“Client1 Put x=0”，将其对应的Invoke和Return从调用历史中取出，得到新的历史：  
  
2. Client2: Invoke Put x=1  
4. Client3: Invoke Get x  
5. CLient4: Invoke Get x  
6. Client3: Return Get 1  
7. Client4: Return Get 0  
8. Client2: Return Put x=1  
  
和一条已经序列化的请求：  
  
1. Client1 Put x=0  
  
此时可以看到剩余的历史中，每一个请求的Invoke前都没有其他请求的Return，因此都可以作为下一个取出的选择。假设这次选择Client3 Get(x)->1，然而，明显这个时候执行Get得到应该是0，与该请求的实际执行结果返回1不同，此时，需要回退并尝试其他取出策略。可以看出**WG算法其实是树的深度优先搜索**，其搜索树如下图，其中每个节点标识的是本次尝试序列化的请求对应的调用历史中的Invoke序号：  
  
![linearizability-wg-tree](https://chenghua-root.github.io/images/linearizability-wg-tree.png)  
  
由于找到一个线性一致序列便可以停止，图中满足三个条件的线性一致序列为BEGIN->1->5->2->4，因此其中虚线部分是不会被实际执行的。  
  
### WGL算法  
WGL算法由Gavin Lowe在WG算法的基础上进行改进，其改进的方式主要是对搜索树的剪枝：通过缓存已经见过的配置，来减少重复的搜索。缓存配置有两部分组成：  
- 当前已经序列化的请求  
- 当前x值  
  
由上面的搜索过程可知，如果当前序列化的请求和当前的x值完全相同，则后续的搜索过程一定一致，因此可以略过。  
  
### P-compositionality算法  
P-compositionality算法利用了线性一致性的Locality原理，即如果一个调用历史的所有子历史都满足线性一致性，那么这个历史本身也满足线性一致性。因此，可以将一些不相关的历史划分开来，形成多个规模更小的子历史，转而验证这些子历史的线性一致性，例如kv数据结构中对不同key的操作。上面提到了算法的计算时间随着历史规模的增加急速膨胀，P-compositionality相当于用分治的办法来降低历史规模，这种方法在可以划分子问题的场景下会非常有用。  
  
## 如何降低验证复杂度  
就算有如上的一些算法和优化，可线性化验证的复杂度在极端情况下仍可能是指数级的。实际进行测试验证的时候，我们还可以通过如下途径来尽量降低验证的复杂度。  
  
### 尽量减少timeout的Operation  
线性一致性模型的nonblocking属性，允许一个事件只有invoke，没有response。对应到实际系统中，就是操作执行结果不确定，比如rpc调用timeout，对于这种情况我们不知道这个操作是成功还是失败。对于这种情况，在线性化验证过程中，会认为这个事件是在整个History结束之后才结束。这样这个事件就与发生在它的invoke之后的所有事件都是并行的，那么进行线性化验证时需要搜索的空间也随之变大。在极端情况下，如果所有操作都是timeout，那么所有的操作就都是并行的，整个验证复杂度是指数级的。  
  
需要尽量减少这种操作timeout的情况。举例来说，我们可以把读请求的超时认为是失败，原因是读请求不影响系统的状态，而对于失败的操作来说，Jepsen在进行线性化验证时会直接忽略掉。  
  
### 通过采用多个对象来增加并发  
受限于线性化验证复杂度的影响，需要限制对于同一个对象的操作个数，这样意味着测试压力不能太大。而根据locality属性，对于针对多个对象的History我们可以按照对象划分，只需要分别验证每个对象的History是否是可线性化的。  
因此如果我们希望增加系统的压力，产生更多的操作，可以通过增加对象个数来实现。以寄存器模型来说，我们可以通过构造多个寄存器对象进行操作来实现。Jepsen测试框架中除了普通的register模型外，本身也提供了对multi-register模型的支持，用户可以直接使用。  
  
## 验证工具  
  
分布式一致性验证工具有Jepsen和Porcupine.  
  
Jepsen 是由 Kyle Kingsbury 采用函数式编程语言 Clojure 编写的验证分布式系统一致性的测试框架，其核心库knossos使用的是WGL算法。  
  
Porcupine 一个用 Go 实现的线性一致性验证工具。是基于 P-compositionality 算法。  
  
## 参考  
Principles of Eventual Consistency https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/final-printversion-10-5-14.pdf  
Consistency model https://en.wikipedia.org/wiki/Consistency_model  
条分缕析分布式：到底什么是一致性？ https://mp.weixin.qq.com/s/qnvl_msvw0XL7hFezo2F4w  
多角度理解一致性 https://qiankunli.github.io/2022/04/06/replica_consistency_level.html  
如何验证线性一致性 https://catkang.github.io/2018/07/30/test-linearizability.html  
