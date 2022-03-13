---  
layout: post  
title:  "A Critique of ANSI SQL Isolation Levels"  
date:   2021-11-05 00:00:00 +0530   
---  
  
<style>  
.tablelines table, .tablelines td, .tablelines th {  
  border: 1px solid black;  
  }  
</style>  
  
论文链接: [A Critique of ANSI SQL Isolation Levels](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/tr-95-51.pdf)  
推荐指数: 五颗星  
易读性：三颗星  
  
## 概述
基于ANSI SQL-92对隔离级别讨论  
基于悲观锁的隔离级别实现  
基于多版本的快照隔离级别与基于锁的隔离级别的区别，SI与RR的区别  
  
## ANSI SQL-92隔离级别讨论
ANSI SQL-92通过现象来定义隔离级别。  
包含严格解释(broad interpretation)和宽松解释(strict interpretation)。  
严格解释不能描述所有隔离级别。  
ANSI SQL-92使用宽松解释。但基于宽松解释的隔离级别不能完全对应基于single-version和锁实现的隔离级别现象。  
调整后的宽松解释=扩展的P0+调整后的P1、P2、P3 与 基于single-version和锁实现的隔离级别现象一致对应。  
  
**现象：**  
现象1：Dirty Read(脏读)  
现象2: Fuzzy or Non-re-peatable Read(不可重复读)  
现象3: Phantom(幻读)  
  
**严格解释：**  
A1: w1[x]...r2[x]...(a1 and c2 in any order)  
A2: r1[x]...w2[x]...c2...r1[x]...c1  
A3: r1[P]...w2[y in P]...c2...r1[P]...c1   
  
**宽松解释：**  
P1: w1[x]...r2[x]...((c1 or a1) and (c2 or a2) in any order)  
P2: r1[x]...w2[x]...((c1 or a1) and (c2 or a2) in any order)  
P3: r1[P]...w2[y in P]...((c1 or a1) and (c2 or a2) any order)  
  
**调整后的宽松解释：**  
P0: w1[x]...w2[x]...(c1 or a1)       (Dirty Write)  
P1: w1[x]...r2[x]...(c1 or a1)       (Dirty Read)  
P2: r1[x]...w2[x]...(c1 or a1)       (Fuzzy or Non-Repeatable Read)  
P3: r1[P]...w2[y in P]...(c1 or a1)  (Phantom)  
  
**现象与宽松解释之间的关系：**  
论文中举例解释了现象与宽松解释解释之间的关系。  
  
x=50, y=50.  
  
Dirty Read:  
T1从x向y转账40  
H1: r1[x=50] w1[x=10] r2[x=10] r2[y=50] c2 r1[y=50] w1[y=90] c1  
没有违背A1,A2,A3, 违背了P1，T2脏读x=10导致x+y=60.  
  
Non-Repeatable Read:  
T2从x向y转账40  
H2: r1[x=50] r2[x=50] w2[x=10] r2[y=50] w2[y=90] c2 r1[y=90] c1  
没有违背P1，没有违背A2，违背了P2，T1读取的x=50已过时(或T1重新读取x=10).  
  
Phantom:  
T2插入新行，并更新总行数z  
H3: r1[P] w2[insert y to P] r2[z] w2[z] c2 r1[z] c1  
没有违背A3, 但违背了P3.  
  
## 基于single-version和锁实现的隔离级别
well-formed: 对读写或预期读写的item加锁  
long duration: 加长锁，commit后释放  
short duration: 加短锁，读写完毕后释放  
two-phase locking: 两阶段锁  
  
well-formed + two-phase locking是串行化的基础。证明参考[事务原理及实现机制](https://chenghua-root.github.io/posts/database-transaction)  
  
![](https://chenghua-root.github.io/images/isolation_level_locking.jpg)
## Snapshot Isolation(SI)
基于snapshot的事务隔离。是一种多版本并发控制系统。存在写偏序的问题。  
  
Read Skew：  
A5A: r1[x]...w2[x]...w2[y]...c2...r1[y]...(c1 or a1)        (Read Skew)  
事务读取两个数据项，两个数据项读取的版本不一致，如r1读取的x和y.  
SI中不存在读偏序，r1[y]会读取到c2之前的版本。  
  
Write Skew:  
A5B: r1[x]...r2[y]...w1[y]...w2[x]...(c1 and c2 occur)      (Write Skew)  
也是涉及两个数据项，一个事务读取数据1，修改数据2。另外一个事务读取数据2，修改数据1.  
SI存在写偏序。  
举例：  
初始值x=0, y=0.  
T1: r1[x], w1[y=x+2]  
T2: r2[y], w2[x=y+3]  
按照SERIALIZABLE顺序执行T1、T2或T2、T1, 得到的结果为(x=5, y=2)或(x=3, y=5).  
按照SI隔离多版本并发执行T1和T2: r1[x=0] r2[y=0] w1[y=x+2] w2[x=y+3]，得到结果为(x=2, y=3).  
  
可能存在Phantom:  
A3: r1[P]...w2[y in P]...c2...r1[P]...c1  
P3: r1[P]...w2[y in P]...(c1 or a1)  (Phantom)  
由于多版本的实现，A3中两个r1[P]的结果是一致的。SI没有违背A3。SI违背了P3，参考Phantom说明中的举例。  
  
### SI vs Repeatable Read
![](https://chenghua-root.github.io/images/isolation_level_types.jpg)
SI相比RR有write skew，但幻读的“可能性”比RR底。  


## 参考  
偏序关系: https://zh.wikipedia.org/wiki/%E5%81%8F%E5%BA%8F%E5%85%B3%E7%B3%BB  
Time,Clocks,and the Ordering of Events in a Distributed System 读书笔记及个人见解: https://ld246.com/article/1463068950849  
译《Time, Clocks, and the Ordering of Events in a Distributed System》: https://www.cnblogs.com/hzmark/p/Time_Clocks_Ordering.html  
