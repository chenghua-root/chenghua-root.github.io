---  
layout: post  
title:  "Time, Clocks, and the Ordering of Events in a Distributed System"  
date:   2021-11-05 00:00:00 +0530   
---  
  
<style>  
.tablelines table, .tablelines td, .tablelines th {  
  border: 1px solid black;  
  }  
</style>  
  
论文链接: [Time, Clocks, and the Ordering of Events in a Distributed System](http://lamport.azurewebsites.net/pubs/time-clocks.pdf)  
推荐指数: 五颗星  
易读性：前半部分五颗星，后半部分三颗星  
  
## 概述  
如何定义分布式系统中事件的顺序，Lamport介绍了如下内容：  
- 分布式系统中事件发生的先后概念：happen before  
- 基于happen before的逻辑时钟（偏序|局部)  
- 逻辑时钟在给定条件后的全局化(全局有序)，并给出了一个基于全局有序的解决资源争夺问题的方法  
- 物理时钟及其他问题  
  
## 偏序，Happen Before  
a -> b: 不依赖物理时钟，a发生在b之前，即a happen before b.  
满足如下三个条件之一，即为happen before关系:  
(1) If a and b are events in the same process, and a comes before b, then a -> b.  
(2) If a is the sending of a message by one process and b is the receipt of the same message by another process, then a -> b.  
(3) If a -> b and b -> c then a -> c. Two distinct events a and b are said to be **concurrent** if a -/-> b and b -/-> a.  
此偏序为具有反自反，禁对称，传递性的[严格偏序(反自反偏序)](https://zh.wikipedia.org/wiki/%E5%81%8F%E5%BA%8F%E5%85%B3%E7%B3%BB)  
  
## 逻辑时钟  
此逻辑时钟可理解为序列号，如ID, LSN.  
定义Ci(a)为事件a在process i上发生的时钟(clock).  
**Clock Condition**定义如下, 对任意的事件a, b:  
- if a -> b then C(a) < C(b). 即如果a happen before b, 则a发生的时钟小于b发生的时钟. 反之则不成立.  
  
根据happen before的定义，如果下面两个条件满足，则Clock Condition成立。  
C1. If a and b are events in process Pi, and a comes before b, then Ci(a) < Ci(b).  
C2. If a is the sending of a message by process Pi and b is the receipt of that message by process Pj, then Ci(a) < Cj(b).  
  
**IR1**:  
为了满足C1, 遵循如下IR1规则(implementation rule):  
- Each process Pi increments Ci between any two successive events.  
  
**IR2**:  
为了满足C2，遵循IR2:  
(a) If event a is the sending of a message m by process Pi, then the message m contains a timestamp Tm = Ci(a).  
(b) Upon receiving a message m, process Pj sets Cj greater than or equal to its present value and greater than Tm.  
  
遵循了IR1和IR2, 则C1和C2得到了保证，然后Clock Condition也被满足了。至此，则保证了一个正确的逻辑时钟系统.  
  
## 事件全局有序(Ordering the Events Totally)  
偏序只对部分事件进行了排序。但在分布式系统中，全局有序也是非常有必要的，因此需对偏序进行扩展以得到全局有序。  
偏序举例:  
- process1: a -> b.  
- process2: c -> d.  
- p1上的事件a向p2上的事件c发送message: a -> c.  
此时，b和d是concurrent的.  
  
定义 => 如下：  
if a is an event in process Pi and b is an event in process Pj, then a => b if and only if either  
(i) Ci(a) < Cj(b) or  
(ii)Ci(a) = Cj(b) and Pi < Pj  
Pi < Pj是为了全局有序下的一种规定或约束，也可以规定Pj < Pi, 此时若Ci(a) = Ci(b), 则 b => a.  
  
根据时钟系统，会有不同的全局序。**但任何时候偏序是唯一的，且全局序必须满足偏序。**  
  
全局有序对解决一些问题是很有必要的。  
  
资源分配条件：  
Condition(I) 资源是互斥的  
Condition(II) 按序分配资源，需满足偏序关系  
Condition(III) 资源使用后释放，则所有请求都能获得资源  
  
Condition(II)并没有说明两个并发产生的请求如何分配.  
即使按照下面描述，使用一个中心控制系统按照请求到达的顺序分配也是有问题的。  
  
P0:调度系统.  
P1: sends a request to P0, sends message to P2.  
P2: upon receiving message from P1, then sends a request to P0.  
Result:  
- It is possible that P2's request to reach P0 before P1's request.  
- 若P2's request先分配，则违法了偏序P1 -> P2  
  
如果有了全局序就解决了上面的问题。  
  
为了解决此问题，我们设计一套满足IR1和IR2的时钟系统，并以此对全部事件定义全局序。有了全局序就好解决问题了，如按全局序分配资源。  
  
为了避免细节的讨论，做出如下假设：  
1. 任意两个Process之间，消息接收顺序与发送顺序相同  
2. 发出的任意一条消息都会被接收到  
3. 任意两个Process之间都可以直接通信  
  
每个Process都维护一个只有自己可见的request queue.  
每个request queue都初始一个message T0:P0 request resource. P0表示初始被分配资源的process，T0是初始的最小clock.  
现定义5个规则:  
1. 为了申请资源，Pi向所有其它的process发送message Tm:Pi request resource，并放入自己队列. Tm就是此消息的时间戳。  
2. Pj收到message Tm:Pi request resource后，放入自己的request queue，并回复一个Timestamped ACK给Pi.  
3. 释放资源：Pi从自己队列移除Tm:Pi request resource, 发送一条带时间戳的Pi release resource message给所有其它process.  
4. 当Pj收到Pi的release resource message, 从其request queue移除所有的Tm:Pi request resource message.  
5. 当如下两个条件满足时，则Pi获得此resource.  
  (i) 有一条Tm:Pi request resource message在其队列里，并且为队列里的ordered before(满足全局序=>的)其它所有request.  
  (ii) Pi has received a message from every other process timestamped later than Tm.  
  
很好证明如上5个规则满足conditions I, II, III.  
- with condition(ii) of rule 5, and assumption that messages are received in order, 保证了Pi学习到了later than Tm之前的所有的消息，按照确定Tm:Pi是当前最早的。按照全局序可以保证Condition II.  
- 规则3和规则4保证了资源只有被释放后才能被分配，则保证了Condition I.  
- 规则2保证了Pi发出资源请求后，规则5(ii)最后会被保证  
- 规则3和规则4保证了资源最终会释放，则规则5(i)会被保证，则Condition III会被满足.  
  
## Anomalous Behavior  
逻辑时钟与现实情况的矛盾。如：  
- 我们在computer A上执行requestA  
- 然后通过电话通知computer B在其上执行requestB.  
- 实际上requestB也可能获得一个比requestA更小的时间戳（代表逻辑时钟）导致requestB happen before requestA  
但此逻辑时钟给出的全局序与实际情况不符。其原因是若电话通知是一个外部行为，则系统无法感知实际上的requestA先于requestB。  
为避免这样的矛盾，有两种解法:  
1. 将外部的happen before关系(此处的requestA实际上先于requestB)引入系统内  
2. 引入实际物理时钟  
  
## Physical Clocks  
上面的Anomalous Behavior还可以通过物理时钟来解决，如requestA和requestB获取的时间戳贴近真实物理时间，保证requestA happen before requestB.  
机器的物理时钟与正确物理时钟存在偏差，只有机器的物理时钟满足一定精度要求才能保证，在不同的两台机器上根据物理时钟获取的时间戳保证requestA happen before requestB.  
- PC1. 机器上的物理时钟与正确物理时钟之间的误差很小. 存在k << 1, 对任意的i: |dCi(t) / dt - 1 | < k. 原子时钟能保证k <= 10^-6  
- PC2. 对所有的i, j: |Ci(t) - Cj(t) | < e  
使得e(两台机器之间的物理时钟误差) / (1 - k) <= u(两台机器之间的最短传输时间)，则能够使用机器的物理时钟作为时间戳来定义顺序。  
  
[Leslie Lamport本人对此论文的讨论。](https://www.microsoft.com/en-us/research/publication/time-clocks-ordering-events-distributed-system/)  
