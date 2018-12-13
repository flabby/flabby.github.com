---
layout: post
title: shared nothing vs. shared disk vs. shared memory
category : distributed 
tagline : db
tags : [architecture, shared nothing, shared disk, shared memory]
---
{% include JB/setup %}


本文回顾一下数据库系统中的常见架构shared memroy,  Shared disk， shared nothing，以及一些观点；
秉承着“one size does not fit all”的原则，这些架构有着各自的优势，并且很多系统也在混合使用不同的架构和执行模式。

### Intro
3种常见的多处理器的数据库架构：
* shared memory (SM，share everything)， 共享集中式的内存和磁盘，主流的DBMS都支持这种并行机制，包括 IBM DB2,
  Oracle, 和Microsoft SQL Server；
* shared disk (SD)， 有自己独立的内存，共享磁盘，包括Oracle RAC
* shared nothing (SN)， 不共享内存和磁盘，通过高速网络交互通信，这种模型有IBM DB2, Informix, Tandem,  Teradata以及Greenplum；

从写路径分析， SD的多个写结点需要协调各自的锁(lock)，当写结点增加，问题会凸显出来，影响扩展性； SN因为每个节点之间没有共享，可以线性扩展，但是需要分布式的多阶段提交；
> SD当然也可以做partition，和SN的区别就是数据的物理存放。SD总是通过网络连接disk的，remote disk也可以提供很好的吞吐和IO性能；

从读路径分析，SD会存在资源竞争和饥饿，cache效率也会更低；SN不可避免的会出现多个节点的数据搬运，根据用户场景和分区策略不同，会有不同的数据搬运的情况。很多NoSQL系统会使用SN和shard，提供高并发；

各家也有不同的观点。

### Case for shared nothing

**Stonebraker在80年代，通过12个点的综合对比，认为SN是最高效的选择**；

首先，扩展能力是一个基础的能力，SM不符合这一标准，排除之；其次，SD各个方面都没有领先(没有获得1分的)；同时，在SN弱势的几个点里，SN同样可以使用一些自动化的设计策略来减低设计复杂度，而repartition也是针对load balancing的常见对策；对通信消息数量方面，对于大多数的应用场景，都不需要考虑这些问题。
**总结起来就是在扩展性、可调性和大多数delightful的场景下，SN和其他两种(SD、SM)相对，不落下风**。

具体的12个点对比是这样的。(表格中的1表示最好，2表示次好，3表示第三好；)
* SM可以简单的修改现有算法来实现，因此在宕机恢复(crash-recovery)方面最好，并发控制上锁表容易成为热点。SN以依赖分布式的死锁检测，以及多阶段的提交协议，实现更为复杂；SD需要协调锁表(lock table)的多个副本，同步对一个或者多个log的更新，所以事务管理也是这三者中最复杂的；
* 从第三点设计复杂度来看，SN需要对单机环境做大量改动，最为困难；load balancing方面，需要物理迁移数据，也是成本最高的；
* 接下来，在高可用，带宽，扩展行，多机远程能力这几方面，SN有明显优势。
* 而对于没有优势的几个点，设计复杂度、load balancing都有一些对策；大多数delightful的应用场景，不需要考虑消息数量的问题；
* 最后热点问题是3种架构都存在的；

| System Feature                           | shared nothing | shared memory | shared disk |
| ---------------------------------------- | -------------- | ------------- | ----------- |
| difficulty of concurrency control        | 2              | 2             | 3           |
| difficulty of crash recovery             | 2              | 1             | 3           |
| difficulty of data base design           | 3              | 2             | 2           |
| difficulty of load balancing             | 3              | 1             | 2           |
| difficulty of high availability          | 1              | 3             | 2           |
| number of messages                       | 3              | 1             | 2           |
| bandwidth required                       | 1              | 3             | 2           |
| ability to scale to large                | 1              | 3             | 2           |
| number of machines ability to have large distances | 1              | 3             | 2           |
| between machines susceptibility to critical sections | 1              | 3             | 2           |
| number of system images                  | 3              | 1             | 3           |
| susceptibility to hot spots              | 3              | 3             | 3           |

### Ref

* [The Case for Shared Nothing](http://db.cs.berkeley.edu/papers/hpts85-nothing.pdf)
* [Shared Nothing v.s. Shared Disk Architectures: An Independent View](http://www.benstopford.com/2009/11/24/understanding-the-shared-nothing-architecture/)
* [Architecture of a Database System](http://db.cs.berkeley.edu/papers/fntdb07-architecture.pdf)
