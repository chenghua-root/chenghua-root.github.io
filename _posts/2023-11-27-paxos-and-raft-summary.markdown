---  
layout: post  
title:  "Paxos和Raft学习总结"  
date:   2023-11-27 00:00:00 +0530  
---  
  
# 概述  
本文整理了作者学习共识算法paxos和raft的各类资料，并对部分资料的内容做了概括。  
  
# Paxos  
## Paxos算法历史  
1988年Liskov发表 <<Viewstamped Replication: A New Primary Copy Method to Support Highly-Available Distributed Systems>>  
1989年Lamport提出了Paxos算法，以希腊小岛Paxos命名此算法。论文题目为"The Part-Time Parliament"，以"兼职议会"为主题阐述兼职的议员们如何达成一致协议。这两篇论文是两位作者独立完成的，二者本质上是一样的，Liskov那篇还要早些，但是因为上述种种轶事，大家最终还是称之为Paxos。  
1990年论文"The Part-Time Parliament"提交给TOCS, 由于描述的比较晦涩，三位审稿人没有通过论文的发表。  
1996年Butler W.Lampson发表 [How to Build a Highly Availability System using Consensus](https://www.microsoft.com/en-us/research/publication/how-to-build-a-highly-available-system-using-consensus/)，里面用到了paxos算法的思想，经过Lampson的大力宣传，Paxos才逐渐为理论研究界的人们所重视。  
1998年Paxos论文得以"重新"发表。  
2001年Lamport发表了更容易理解的"Paxos Made Simple"，原因是"The Part-Time Parliament"对大多数人来说仍然难懂。  
2006年google的分布式锁chubby使用paxos算法解决了分布式共识问题，paxos算法开始火起来。  
  
## Lamport  
[Paxos Made Simple](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf)  
Basic paxos: 使用两阶段确定一个值。  
Multi paxos: 如何确定多个值。引入状态机和leader的概念。  
  
[The Part-Time Parliament](https://lamport.azurewebsites.net/pubs/lamport-paxos.pdf)  
未读。  
对paxos算法进行描述并进行证明。  
  
[Fast Paxos](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/tr-2005-112.pdf)  
未读。  
对理解一致性本质有帮助。  
  
[generalized consensus and paxos](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/tr-2005-33.pdf)  
未读  
包罗分布式里所有问题的数学解决方案  
  
## Ongaro  
[Paxos summary](https://ongardie.net/static/raft/userstudy/paxossummary.pdf)  
由于Multi-Paxos算法在论文里面描述的不够具体，Raft作者Ongaro对Basic-Paxos和Multi-Paxos做了具体的描述。  
Basic-Paxos参考后面的"可靠分布式系统-paxos的直观解释", 这里描述下Multi-Paxos.  
  
proposal number (n) = (round number, server ID)  
  
每个Acceptor持久化的状态:  
- lastLogIndex: the largest entry for which this server has accepted a proposal  
- minProposal: the number of the smallest proposal this server will accept for any log entry, or 0 if it has never received a Prepare request. This applies globally to all entries.  
  
Each acceptor also stores a log, where each log entry i ∈ [1, lastLogIndex] has the following fields:  
- acceptedProposal[i]: the number of the last proposal the server has accepted for this entry, or 0 if it never accepted any, or ∞ if acceptedValue[i] is known to be chosen  
 - 0: never accepted any  
 - n: last proposal the server has accepted for this entry  
 - ∞: acceptedValue[i] is known to be chosen. 即已经被提交  
- acceptedValue[i]: the value in the last proposal the server accepted for this entry, or null if it never accepted any  
  
Define firstUnchosenIndex as the smallest log index i > 0 for which acceptedProposal[i] < ∞  
  
运行：  
- 收到请求后如果没有leader，prepare时(proposal number, index), index为本机acceptor的firstUnchosenIndex, nextIndex = index + 1  
  - acceptor返回noMoreAccepted的条件为没有大于此prepare.index的index接受过value  
    - 收到多数派的noMoreAccepted才能当选leader(实现时可以设定为返回的多数派中的noMoreAccepted都等于true)  
    - 因此proposal所在节点的acceptor的index至少要追赶到最大的被chosen的index(完成多数派写)  
      - 更大的index如果没有在majority返回，则不再追  
    - 因此当选leader之前会把日志至少追平到被chosen的index  
    - 使用basic paxos追赶日志  
- 若已是leader，则index = nextIndex++直接运行accept  
  
总结:  
- **任意节点无论日志情况怎么样都可以当选为leader，成功当选leader之前会通过basic paxos把日志至少追平所有被chosen的日志**  
- **由于leader本任期当选之前需要把日志追上并且设置为chosen, 因此不存在本任期之前的index未被chosen**  
- 当选leader后针对连续不同的index并发发起accept，因此在本任期内存在某些index先达成多数派并且被设置为chosen  
  - 因此整体的视角看，未被chosen的空洞(有比此index更大的index被设置为chosen了)只会存在于一个任期  
  - 但是从单个acceptor看，其日志状态有:  
    - 不同的未被chosen的proposal number(非空洞): 存在，不同的leader导致  
      - | ∞ | 3.4 | 4.2 |  
    - 不同的未被chosen的proposal number(非空洞), 且大的proposal number在前面: 存在，不同的leader导致，前面的大的proposal number覆盖了前任leader的proposal number  
      - | ∞ | 4.2 | 3.4 |  
    - 未被chosen空洞，即最后一个被chosen之前的index有未被chosen: 当leader时留下的，因为不是leader则会"被"chosen最小未被chosen(firstUnchosenIndex)的index  
      - | ∞ | 3.4 | ∞ |  
    - 未被chosen空洞是否包含不同的proposal number: 不存在，只有leader才会留下空洞，当选leader之前的已经被chosen  
      - | ∞ | 3.4 | 4.2 | ∞ |  
    - 日志空洞: 存在  
      - | ∞ |   | 3.4 |      leader或者follower都可能  
      - | ∞ |   | 3.4 | ∞ |  leader时  
      - | ∞ |   | ∞ |        leader时  
  - 总结: **未被chosen的空洞只会是当leader时留下，且只会有一个任期的空洞**  
- 因此，**multi paxos是支持乱序提交的(被chosen)**, 是否能乱序应用则依赖业务逻辑  
- 副本状态机(replicated state machine)定义是要求顺序apply  
  
[Implementing Replicated Logs with Paxos](https://ongardie.net/static/raft/userstudy/paxos.pdf)  
PPT分享: 引入Paxos，达成一个值会遭遇哪些场景，如何生成Proposal Number, Basic Paxos举例，Multi Paxos实现。  
  
[Paxos Exam](https://ongardie.net/static/raft/userstudy/quizzes.html)  
  
[Paxos Exam Rubric](https://ongardie.net/static/raft/userstudy/rubric.pdf)  
  
## 郁白Paxos三部曲  
[Paxos三部曲之一 : 使用Basic-Paxos协议的日志同步与恢复](http://oceanbase.org.cn/archives/90)  
每个成员的身份都是平等的情况下，多个Basic Paxos如何实现，包括获取LogID，产生ProposalID, Prepare阶段，Accept请求阶段，Accept处理阶段，一致性的读。  
  
[Paxos三部曲之二 : 使用Multi-Paxos协议的日志同步与恢复](http://oceanbase.org.cn/archives/111)  
Basic Paxos效率底下，引入Multi-Paxos，包括leader选举，confirm日志的优化，新任leader对日志的重确认，"幽灵复现"日志的处理。  
  
[Paxos三部曲之一 : Paxos成员组变更](http://oceanbase.org.cn/archives/160)  
  
## drdr xp  
[如何浅显易懂地解说 Paxos 的算法？ - drdr xp的回答 - 知乎](https://www.zhihu.com/question/19787937/answer/1258191694)  
评论里面的讨论。Basic Paxos P1b选择最大的accepted proposal的value作为本次的提交值:  
1. 为什么要这么做  
 - 以免已经accepted到value被推翻/覆盖。  
2. 为什么挑选最大的就能保证已被集群accepted的值不会被推翻  
 - 假设同时有v1和v2存在；因为成功的v2的phase-1跟成功的v1的phase-2一定有交集，而v2被运行到phase-2表示v2没有在phase-1看到v1, 所以v1没有成功的完成phase-2且再也不会成功。所以一定不用选v1. 依次类推v2和v3等等，最终就只能选最大的。  
  
[可靠分布式系统-paxos的直观解释](https://blog.openacid.com/algo/paxos/)  
1. 结合场景推出basic-paxos的方法  
2. basic paxos是什么  
 - 一个可靠的存储系统：基于多数派读写  
 - 每个paxos实例用来存储一个值  
 - 用2轮rpc来确定一个值  
 - 一个值被确定后不能被修改  
 - 确定是指被多数派确定写入  
 - 强一致性  
3. "一个值被确定后不能修改"与现实中同一个key被多次赋不同的值怎么理解  
  basic paxos是对key达成一个值，达成后不能修改。一个key被多次赋不同的值不属于basic paxos的范畴，可以理解为运行了多个独立的basic paxos实例。对于multi paxos和raft，则相当于是一个key的不同版本赋予不同的值，使用paxos/raft index来区分。  
4. 写走两阶段多数派，读也要走一轮多数派才能保证读到确定的值  
  
总结:  
- basic paxos是确定一个值，且确定的值(多数派写成功)不能被修改。如何避免被修改(覆盖写导致丢失更新)的问题:  
 - 对某个议案达成一致，延伸到数据的可靠性，多副本达成一致  
 - 直接写，存在覆盖的问题，因此需要在写之前执行一遍多数派读，询问值是否已经被确定，只要有一个返回有写入值，则应该使用此值，因为此值可能已经被确定（完成多数派写），如三个节点，节点一和二完成了多数派写，预读返回节点二和三  
 - 并发问题，两个客户端并发执行多数派读，依然存在覆盖的问题，即两个客户端询问的时候都返回没有值，然后并发更新，先更新的被覆盖  
 - 记录最后一个做过写前读取的操作，并且只允许最后一个完成写前读取的进程可以后续写入。存在下列情况:  
  - 写多数派成功，值被确定  
  - 写少数派后客户端宕掉，写入少数派的值被后面的多数派读发现，被推广  
  - 写少数派后客户端宕掉，没有被多数派发现，被覆盖  
  - 活锁，accept请求时已经有新的prepare覆盖  
- 读取操作至少要进行一次多数派读  
 - 如果读取的多数派返回为空，或者返回的vrnd一样, 则返回空或者获取的值  
 - 如果返回的vrnd没有相等的多数派(有更大的vrnd，且val一致；实践中可以判断最大的vrnd没有形成多数派)，则需要走accept阶段，且多数派返回后才能返回。  
  
- 需要持久化的状态: acceptor  
 - rnd: acceptor看到的最大的rnd  
  - minProposal: the number of the smallest proposal this server will accept, or 0 if it has never received a Prepare request  
 - vrnd: acceptor接受v的rnd  
  - acceptedProposal:  the number of the last proposal the server has accepted, or 0 if it never accepted any  
 - v: acceptor接受的值  
  - acceptedValue: the value from the most recent proposal the server has accepted, or null if it has never accepted a proposal  
  
- 需要持久化的状态: proposer  
 - maxRound: the largest round number the server has seen  
  - 新的proposer number为maxRound + 1, 并持久化maxRound += 1  
  
[200行代码实现基于paxos的kv存储](https://blog.openacid.com/algo/paxoskv/)  
  
## 欢哥  
[Paxos的正确性：觉得Paxos是对的却不知为啥？探讨下Paxos背后的数学](https://zhuanlan.zhihu.com/p/104463008)  
未读。  
  
[">" 还是 ">=" ？ 探讨下微信团队开源的 Phxpaxos 与标准Paxos的差异](https://zhuanlan.zhihu.com/p/111665673)  
paxos论文描述的是大于:  
"Phase 1. (a) A proposer selects a proposal number n and sends a prepare request with number n to a majority of acceptors. (b) If an acceptor receives a prepare request with number n **greater than** that of any prepare request to which it has already responded, then it responds to the request with a promise not to accept any more proposals numbered less than n and with the highest-numbered proposal (if any) that it has accepted."  
  
TLA+ spec对Phase1b的公式描述也是大于, m.bal > maxBal[a]. [Paxos.tla](https://github.com/tlaplus/Examples/blob/master/specifications/Paxos/Paxos.tla)  
  
>=存在的问题: 达成一致的值被覆盖(使用通用的ballot才会出现)  
1. Proposer P1发送Ballot为b1的Phase1a消息；  
2. P1收到了某个Quorum里所有Acceptor回复的"1b"消息，但是不包括自己节点的Acceptor(假设自己的磁盘太忙，phase1b的持久化没有完成);  
3. P1发送"2a"消息<b1, v1>给各个Acceptor, 然后自己宕机(<b1, v1>被Acceptor们选定了);  
4. P1重启，自己没有关于b1的任何记录  
5. P1再次用Ballot b1执行Phase1，又成功(因为符合条件)。  
6. P1提议<b1, v2>，又被accept.  
  
## PhxPaxos  
[Paxos理论介绍(1): 朴素Paxos算法理论推导与证明](https://zhuanlan.zhihu.com/p/21438357)  
[Paxos理论介绍(2): Multi-Paxos与Leader](https://zhuanlan.zhihu.com/p/21466932)  
[Paxos理论介绍(3): Master选举](https://zhuanlan.zhihu.com/p/21540239)  
[Paxos理论介绍(4): 动态成员变更](https://zhuanlan.zhihu.com/p/22148265)  
  
## google  
[Paxos Made Live - An Engineering Perspective](https://static.googleusercontent.com/media/research.google.com/en//archive/paxos_made_live.pdf)  
未读。  
Google在paxos上的实践。  
  
## 参考  
- [分布式系统] Paxos算法的发展历史: https://www.crazybytex.com/thread-241-1-1.html  
- PaxosWiki: https://en.wikipedia.org/wiki/Paxos_(computer_science)  
- Paxos Made Simple译文: https://www.iteblog.com/pdf/2337.pdf  
- 如何浅显易懂地解说 Paxos 的算法？ - 朱一聪的回答 - 知乎: https://www.zhihu.com/question/19787937/answer/82340987  
- [何登成 Paxos/Raft 分布式一致性算法原理剖析及其在实战中的应用](https://github.com/hedengcheng/tech/blob/master/distributed/PaxosRaft%20%E5%88%86%E5%B8%83%E5%BC%8F%E4%B8%80%E8%87%B4%E6%80%A7%E7%AE%97%E6%B3%95%E5%8E%9F%E7%90%86%E5%89%96%E6%9E%90%E5%8F%8A%E5%85%B6%E5%9C%A8%E5%AE%9E%E6%88%98%E4%B8%AD%E7%9A%84%E5%BA%94%E7%94%A8.pdf)  
  
# Raft  
## raft.github.io  
[The Raft Consensus Algorithm](https://raft.github.io/)  
  
## Ongaro  
[CONSENSUS: BRIDGING THEORY AND PRACTICE](https://web.stanford.edu/~ouster/cgi-bin/papers/OngaroPhD.pdf)  
未读。  
Raft作者Diego Ongaro的博士论文。  
  
[In Search of an Understandable Consensus Algorithm](https://raft.github.io/raft.pdf)  
Raft论文。  
Paxos难以理解; 状态机; Raft算法实现: leader选举，日志复制。  
安全性:  
- 选举限制: 只有包含全部提交日志的节点才能被选举为leader  
- 提交前任日志存在哪些问题，如何提交  
  
## TiKV  
[raft read](https://cn.pingcap.com/blog/lease-read/)  
Raft Log Read: 走一遍raft log。follower也能响应请求。  
ReadIndex Read: 通过心跳确认在某个commit index之前自己是leader  
- 参考[Raft Read Index 实现轻量级线性一致读](https://juejin.cn/post/7045607498308878367)  
- 如果 Leader 在当前任期还没有提交过日志，先提交一条空日志  
- Leader 保存记录当前 commit index 作为 readIndex  
- 通过心跳，询问成员自己还是不是 Leader，如果收到过半的确认，则可确信自己仍是 Leader  
- 等待 Apply Index 超过 readIndex  
- 读取数据，响应 Client  
Read Lease: 通过leader lease的方式来保证leader还处于有效期，避免心跳  
  
[线性一致性和 Raft](https://cn.pingcap.com/blog/linearizability-and-raft/)  
  
[TiKV 功能介绍 – Raft 的优化](https://cn.pingcap.com/blog/optimizing-raft-in-tikv/)  
- Batch  
- Pipeline: 不用等上一轮的日志返回就可以发送下一轮  
- Append Log Parallelly: 写本地和写follower并发  
  - **日志提交的多数派不必须包含leader自己**
- Asynchronous Apply: 不用等commit就返回客户端。至少follower可以不用等commit  
  
- SST Snapshot: 通过rocksdb injest接口直接将SST file load 进入rocksdb.  
  
# Paxos VS Raft  
## 日志  
### 幽灵日志  
幽灵日志: 查询没有的日志后面又出现了  
  
[[Paxos三部曲之二] 使用Multi-Paxos协议的日志同步与恢复](http://oceanbase.org.cn/archives/111)  
解决方法:  
- 我们在每条日志的内容中保存一个generateID，leader在生成这条日志时以当前的leader ProposalID作为generateID  
- 按logID顺序回放日志时，因为leader在开始服务之前一定会写一条StartWorking日志，所以如果出现generateID相对前一条日志变小的情况，说明这是一条“幽灵复现”日志（它的generateID会小于StartWorking日志），要忽略掉这条日志。  
  
[关于Paxos "幽灵复现"问题看法](https://zhuanlan.zhihu.com/p/40175038)  
- 幽灵日志本质是超时未知的日志  
 - paxos对于未知的日志处理方式是推进
 - raft对于未知的日志处理方式是覆盖/舍弃
- raft如何解决幽灵复现  
  - I: 1, 2, 3  // 表示index  
  - A: 1, 2, 2  // 表示term  
  - B: 1, 2  
  - C: 1, 2  
  - B当选leader, 不包含未形成多数派的日志(index=3)，当选leader后如果没有产生一条no-op日志则提供读服务，则对于客户端访问未包含多数派的日志(index=3)会返回不存在  
  - A重新当选leader，推广未提交的日志(index=3)形成多数派并提交，客户端访问时返回存在，则发生幽灵复现  
  - raft解决方法是：  
    - 新当选leader提交一条no-op日志(term=3)后才提供读服务  
    - 则未被提交的日志(index=3)会被覆盖，或者未包含no-op日志的A不能重新当选leader, 则避免了幽灵复现  
  
  
### leader日志  
- 达成多数派的日志都能被提交，本质上leader节点不一定需要包含在多数派里面  
- 如果日志在本地应用(RSM)之前不需要在本地提交，则leader节点可以不包含已经在本地应用的日志  
- 问题：  
  - paxos/raft算法中，leader提交日志之前是否要保证已经写入本地  
  - 不需要。参考etcd / tikv实现  
  
## OceanBase  
[OceanBase的一致性协议为什么选择 paxos而不是raft?](https://www.zhihu.com/question/52337912)  
paxos最大的优势是支持乱序提交。但乱序提交不表示能乱序应用。  
按Paxos论文中复制状态机(RSM)定义，状态机必须严格按照顺序执行(apply)命令。业务根据自己的场景可以选择乱序执行，则此业务一定不能被定义为复制状态机.  
因此paxos乱序提交的优势，需要业务支持乱序提交把优势体现出来。  
  
## PolarStore Parallel Raft  
  
如何评价 PolarFS 以及其使用的 Parallel Raft 算法？ - 暗淡了乌云的回答 - 知乎  
https://www.zhihu.com/question/278984902/answer/562927275  
  
[干货长文：从业界实现剖析共识协议本质 - PolarFS，Oceanbase，PhxPaxos](https://zhuanlan.zhihu.com/p/662811049)  
  
[面向云数据库，超低延迟文件系统PolarFS诞生了](https://mp.weixin.qq.com/s/4s7lDKlQjV1mUoVv558Y7Q)  
  
Parallel Raft乱序应用:  
- Raft日志只允许顺序确认，顺序提交，顺序应用。如果leader同步日志给follower是通过单连接顺序接收，follower接收后也是顺序处理，顺序同步写日志，则raft没有乱序提交的优化空间。  
- 如果leader与follower并发发送数据，leader & follower并发异步写日志(后提交的写请求先返回)等情况，即后面的日志比其前面的日志先达成多数派，则可以尝试优先提交后面的日志。  
- 乱序提交不代表可以乱序应用，乱序应用会导致状态机的状态顺序变化与日志的最终顺序不一致，因此乱序应用依赖具体的应用场景。  
- PolarFS支持乱序应用:  
 - "由于PolarFS之上运行的是Database事务处理系统，它们在数据库逻辑层面的并行控制算法使得事务可以交错或乱序执行的同时还能生成可串行化的结果。这些应用天然就需要容忍标准存储语义可能出现的I/O乱序完成情况，并由应用自身进一步保证数据一致性。因此我们可以利用这一特点，在PolarFS中依照存储语义放开Raft一致性协议的某些约束，从而获得一种更适合高I/O并发能力发挥的一致性协议。"  
 - 因此当乱序应用的日志与前面的日志无冲突时，可以先应用后面的日志  
  
## 其他  
raft算法与paxos算法相比有什么优势，使用场景有什么差异？ - 朱一聪的回答 - 知乎  
https://www.zhihu.com/question/36648084/answer/82332860  
  
[比较raft ，basic paxos以及multi-paxos](https://zhuanlan.zhihu.com/p/25664121)  
