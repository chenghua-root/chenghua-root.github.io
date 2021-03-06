---  
layout: post  
title:  "纠删码存储"  
date:   2019-12-29 00:00:00 +0530   
---  
  
<style>  
.tablelines table, .tablelines td, .tablelines th {  
  border: 1px solid black;  
  }  
</style>  
  
## 概述  
  三副本是存储领域常见的数据冗余方法，把三个副本存放在不同的节点或机房中，以提高数据可靠性。但三副本存在效率偏低的问题，即每个数据都存储三份，存储成本较高。  
  如何在不降低数据可靠性的同时提高存储效率，这里引入纠删码（Erause Code）存储。  
  本文讲解**纠删码的基本原理**，并介绍**美团对象存储在纠删码使用的实践。**  
  文章逻辑结构为：由上一个章节的讨论引出下一个章节的内容。
  - 什么是纠删码  
  - 常用的纠删码：里德-所罗门码  
  - 里德所罗门码用到的矩阵：范德蒙矩阵  
  - 矩阵运算溢出：有限域  
  - 利用矩阵运算恢复损坏数据时恢复成本高：LRC(Locally Repairable Codes)  
  - 如何加快矩阵运算：ISA-L(Inter®Intelligent Storage Acceleration Library)  
  - 生产环境中我们应该选用哪种编码组合，以及**纠删码存储如何进行机房容灾**  
  - 如何解决恢复数据延迟高的问题  
  
## 纠删码  
  纠删码是一种数据保护方法，它将数据块扩展、编码，生成编码块（也称校验块、冗余块），通过编码块来恢复损坏的数据。  
  
**朴素解释：**  
  　数据块x1=a，数据块x2=b，对其扩展编码生成编码块x3=x1+x2=a+b。  
  　x1 = a  
  　x2 = b  
  　x1 + x2 = a + b  
  　当x1损坏时，可以通过{x2=b, x1+x2=a+b}计算恢复。  
  
## 里德-所罗门码（Reed-solomon codes）  
  纠删码的实现方式有多种，工程中主要采用Reed-Solomon（RS）编码。  
  RS是存储系统较为常用的一种纠删码，它有两个参数k和m，记为RS(k,m)或EC(k, m)。k代表数据块个数，m代表编码块个数，**损坏任意小于等于m块数据都可恢复。**  
  RS是通过矩阵运算来实现编码、解码。基本原理如下：  
  　1 0 0 0 　D0　 D0  
  　0 1 0 0 　D1　 D1  
  　0 0 1 0 　D2　 D2  
  　0 0 0 1 * D3 = D3  
  　1 1 1 1　　　　C0 = D0 + D1 + D2 + D3  
  　1 2 4 8　　　　C1 = D0 + 2 * D1 + 4 * D2 + 8 * D3  
  如上根据矩阵运算生成两个编码块C0和C1，丢失任意一个或两个数据块，都可以根据编码块恢复。  
  
  编码基本方法：  
  - 数据块：如上例包含4块数据，D0~D3  
  - 编码矩阵：矩阵包含上下两部分了，上部分为单位矩阵，其大小等于数据块的长度=k * k；下半部为为范德蒙矩阵=k * m。上面例子中k=4, m=2.  
  - 编码：根据编码矩阵和数据D0~D3计算得到两个编码块C0和C1  

  解码基本方法：  
  - 解码矩阵：假设此时丢失D0和D1，**抽取可用数据的行生成解码矩阵**  
  　　0 0 1 0 　D0 　D2  
  　　0 0 0 1 * D1　= D3  
  　　1 1 1 1 　 D2 　C0（D0+D1+D2+D3）  
  　　1 2 4 8 　D3 　C1（D0 + 2 * D1 + 4 * D2 + 8 * D3）  
  - 矩阵运算：上面左边的矩阵生成逆矩阵，转换成下面的计算：  
  　　D0 　 2　6　2 -1 　 D2 　　 2 * D2 + 6 * D3 + 2 * C0  + -1 * C1  
  　　D1 = -3 -7 -1　1　 * D3　= -3 * D2 + -7 * D3 + -1 * C0 + 1 * C1  
  　　D2 　 1　0　0　0 　 C0　　　　D2  
  　　D3 　 0　1　0　0 　 C1　　　　 D3  
  - 恢复数据:
    - D0 = 2 * D2 + 6 * D3 + 2 * C0  + -1 * C1 = 0
    - D1 =-3 * D2 + -7 * D3 + -1 * C0 + 1 * C1 = 1 
  
## 范德蒙矩阵  
  上面提到了范德蒙矩阵。为什么要使用范德蒙矩阵？  
  范德蒙矩阵保证了抽取的行生成的解码矩阵是满秩矩阵，只有满秩矩阵才能生成可逆矩阵（或者只有满秩矩阵才能保证n个等式与n个变量的情况下可解），即方程组有解。  
  只要保证组成的解码矩阵是满秩矩阵（可逆矩阵）则可以被使用，因此除了使用范德蒙矩阵，还可以使用其他矩阵，如柯西矩阵。  
  
## 有限域（伽罗华域）  
  当D0、D1、D2、D3不是0、1、2、3等较小的数，而是非常大的数，在做矩阵运算时计算复杂性高（效率很低），且中间值或结果溢出的情况。  
  有限域就是用来解决计算结果溢出的。  
  如有限域{0, 1, 2, 3, 4, 5, 6}，为了保证其计算结果也在域中，我们可以自定义一套运算规则。最常见的方式是对结果取余。  
  　如{0, 1, 2, 3, 4, 5, 6}：定义加法x ++ y = (x+y) % 7，乘法x ** y = (x * y) % 7。++和 ** 为重新定义的加法符号和乘法符号。  
  如两个int32数据做乘法后其结果长度也为int32.  
  有限域的实际规则较复杂，可参考[伽罗华域（Galois Field）上的四则运算](https://blog.csdn.net/shelldon/article/details/54729687)  
  
## LRC（Locally Repairable Codes）  
  LRC编码是为了解决恢复损坏数据成本高的问题。  
  实际使用中，k值较大，如Azure使用的是(k=12, m=4)，每次恢复1个数据块需要读取k=12块数据（11个数据块+1个编码块），恢复成本设为k=12。假设每个块大小为1GB，则磁盘和网络开销都需要12GB。  
  因为损坏1个块的概率远大于损坏2个块，下面讨论的恢复成本均针对损坏1个块的情况。  
  为了减小恢复成本，这里引入分组和局部编码块的概念。如数据块分为两组，每组1个局部编码块，1个或多个全局校验块。  
  
### LRC原理  
  以LRC(6, 2, 1)为例，6个数据块(分为两组)+2个局部编码块+1个全局编码块，k=6, m=3：  
  
![LRC-6-2-1](https://chenghua-root.github.io/images/ec-lrc-6-2-1.jpg)  
  
  - C0为第一组的局部编码块，跟D0, D1, D2有关，对应矩阵行1  1  1  0  0  0  
  - C1为第二组的局部编码块，跟D3, D4, D5有关，对应矩阵行0  0  0  1  1  1  
  - C2为全局编码块，跟全部数据块都有关, 对应矩阵行1  2  4  8 16 32  
  思考：为什么全局块没有选择范德蒙矩阵第一行(1 1 1 1 1 1)？  

  当损坏D0时，根据D1, D2, C0即可恢复D0，恢复成本为k/2=3，减少了恢复成本。读者可以参考上面里德-所罗门码章节的恢复方法尝试计算D0。  
  
### LRC的缺点  
  LRC减少了恢复成本，但存在可靠性的牺牲。如LRC(6, 2, 1)包含3个编码块，却不能保证在损坏任意的3块数据时都可恢复。  
  在LRC(6, 2, 1)中，损坏任意的1块或2块数据都可恢复，损坏3块时部分情况不可恢复。  
  不可恢复情况如下：  
  - 0个全局块+3个数据块在同一组  
  　如损坏D0, D1, D2后无法恢复。因为C1跟D0, D1, D2无关，根据C0和C2只能生成两个方程组，无法求得D0, D1, D2三个未知数。  
  - 1个全局块+2个数据块在同一组  
  　如损坏C1, D0, D1无法恢复  
  
  Azone实际采用的是LRC(12, 2, 2)方案，分为两组，恢复成本为k/2=6。  
  LRC(12, 2, 2)共4个编码块，损坏任意的1到3块均可恢复，损坏4块时不可恢复的情况如下：  
  - 0个全局块+4个同一组  
  - 1个全局块+3个同一组  
  - 2个全局块+2个同一组  
  损坏4块时，不可恢复率为13.846%  
  
## ISA-L（Inter® Intelligent Storage Acceleration Library）  
  编码和解码本质都是做计算，需要消耗CPU资源并产生延迟。  
  ISA-L是Inter提供了一套用来提高计算速度的加速库，用于降低CPU消耗&计算延时。  
  
## 编码组合  
  上面例子中，举例了EC(4, 2)，LRC(6, 2, 1), LRC(12, 2, 2)三种情况。实际使用时我们应该选择哪种组合方式呢？  
  编码组合主要考虑的问题：  
  - 存储效率  
  - 存储可靠性  
  - 恢复成本  
  - 恢复延迟  
  - 机房容灾  
  
  我们先列出几种典型编码组合，对其进行分析。分析完后读者可根据需求自己组合。  
![EC组合](https://chenghua-root.github.io/images/ec-combination.jpg)  
  可靠性计算参考[磁盘故障与存储系统的年失效率估算](http://oceanbase.org.cn/?p=151)  
```  
 3R:            probability_3R(1, 1000, 1, 3)  
 EC(6, 2):      probability_EC(1, 1000, 24, 3, 8)  
 EC(10, 4):     probability_EC(1, 1000, 24, 5, 14)  
 EC(12, 4):     probability_EC(1, 1000, 24, 5, 16)  
 LRC(12, 2, 2): probability_EC(1, 1000, 24, 5, 16) + probability_EC(1, 1000, 24, 4, 16)*decimal.Decimal(str(0.13846))  
 EC(6, 6):      probability_EC(1, 1000, 24, 7, 12)  
 LRC(6, 2, 4):  probability_EC(1, 1000, 24, 7, 12) + probability_EC(1, 1000, 24, 6, 12)*decimal.Decimal(str(0.06926))  
```  
  - 存储效率：上表所示的编码组合中，纠删码存储效率都高于3副本  
  - 存储可靠性：EC(6, 2)低于3副本，LRC(12, 2, 2)与3副本在一个数量级，其它组合都高于3副本  
  - 恢复成本：恢复成本只考虑损坏1块的情况（损坏1块的概率 >> 损坏2块的概率）。3副本恢复成本为1，EC恢复成本为k，LRC恢复成本为k/2  
  - **机房容灾**：上面的存储可靠性是基于磁盘/节点为粒度来计算的，但没有考虑机房容灾的情况  
    - 3副本把副本存放在不同机房即可实现机房容灾  
    - EC(6, 2)、EC(10, 4), EC(12, 4), LRC(12, 2, 2)都不支持容灾，原因留给读者  
    - EC(6, 6)可以把6个数据块和6个编码块分别放在两个机房来实现机房容灾。但其机房容灾较弱，原因为当个某机房受灾时，另外一个机房损坏任意一块数据时就不可恢复了  
    - LRC(6, 2, 4)能很好的支持机房容灾，但需要3个机房。具体方法为两个组（包含组内数据块+对应局部编码块）分别放在两个机房，4个全局块放在第3个机房  
  　 当某个机房受灾，且剩下两个机房损坏任一一块时均可恢复，已达到机房容灾目的  
  - 除了在内部实现机房容灾外，还可以通过其它方法实现机房容灾  
  　如Facebook采用的是EC(10, 4)，两个不同机房的EC1(10, 4)和EC2(10, 4)的数据部分做XOR运算生成第3份数据，把第3份数据存放在第三个机房来实现机房容灾  
  　这里也存在某个EC组所在机房受灾后，第3份数据损坏1块则不可恢复的情况。因此推测第3份数据也会生成一个或两个编码块，即EC(10, 1)或EC(10, 2)。总的存储效率为20/((10+4) * 2+(10+1))=0.513  
  
### EC组合可靠性和恢复成本不可兼得  
  EC(6, 2)和EC(12, 4)的存储效率相等，虽然EC(6, 2)的恢复成本低，但由于其可靠性不符合要求。因此，在这两组对比中（同样存储效率）应该选择EC(12, 4)。  
  **生产环境中：**  
  - **在不考虑机房容灾的情况下，推荐LRC(12, 2, 2)：存储效率较高，可靠性满足要求，恢复成本较低=6。 Azure和美团对象存储采用了此编码组合。**  
  - **在考虑机房容灾的情况下，推荐LRC(6, 2, 4)：存储效率比3副本提升50%，可靠性高于3副本，恢复成本低=4。美团对象存储（跨机房容灾）采用了此编码组合。**  
  
## 恢复延迟  
  恢复延迟是使用纠删码存储的一大挑战之一，因为恢复延迟直接影响数据可用性。3副本损坏1块时可直接读取其它两块。纠删码存储读取损坏的数据块时需要等待其被恢复后才能读取，设恢复带宽50MB/s（受磁盘影响），如数据块大小1GB时恢复时长为1GB/50MB/s=20s，延迟较高，用户敏感。  
  美团对象存储采取了三个措施来解决这个问题，供读者参考：  
  1. 在线恢复和离线恢复分开：  
  　在线恢复指恢复立即读取的数据，即数据块中的某一小块数据。美团对象存储每次读取大小<=2MB，恢复tp99延时在100ms。  
  　然后再离线恢复整个数据块。  
  2. 对在线恢复的数据进行缓存。  
  　按一定规则，确保读取同一块数据时，恢复选在同一个节点，则可通过缓存避免重复恢复。  
  3. 在协议层对小文件进行缓存  
  　美团对象存储在协议层对小文件（<=2MB）缓存，缓存命中率达到80%。进一步减少数据块损坏的影响。  
  
## 其它  
  ISA-L中使用范德蒙矩阵时，部分情况下编码矩阵不可逆。  
  “Be aware that sometimes the generated Vandermonde matrix is not always invertible and not fully MDS.” [ceph-14.2.4/src/erasure-code/isa/README](https://fossies.org/linux/ceph/src/erasure-code/isa/README)  
  经过测试，上面列出的组合方式中，LRC(6, 2, 4)存在部分理论上可逆但生成的编码矩阵不可逆的情况。  
  LRC(6, 2, 4)不可逆5种情况为：  
  - 损坏5块（排列顺序为6个数据块 + 2个局部校验块 + 4个全局校验块）：  
　　1 1 0 0 1 1 0 0 1 0 0 0  
  - 损坏6块：  
　　1 1 0 0 1 1 0 0 1 0 0 1  
　　1 1 0 0 1 0 0 1 1 0 1 0  
　　0 1 0 1 0 1 1 0 1 1 0 0  
　　0 1 0 0 1 1 1 0 0 1 1 0  
  即使包含以上5中情况，LRC(6, 2, 4)的可靠性也高于3副本，且不影响机房容灾。
  使用范德蒙矩阵时才存在如上不可逆的情形，读者可选择使用柯西矩阵。
  
## 参考  
  矩阵求逆 http://www.yunsuan.info/matrixcomputations/solvematrixinverse.html  
  磁盘故障与存储系统的年失效率估算 http://oceanbase.org.cn/?p=151  
  有限域 https://zh.wikipedia.org/wiki/%E6%9C%89%E9%99%90%E5%9F%9F  
  Microsoft、Google、Facebook的erasure code技术进展及系统分析 http://www.voidcn.com/article/p-ekqxcpoy-ss.html  
