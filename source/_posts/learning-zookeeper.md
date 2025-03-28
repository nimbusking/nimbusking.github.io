---
title: ZooKeeper学习
abbrlink: 324f2c0b
date: 2022-08-01 12:20:32
tags:
  - 中间件
  - ZooKeeper
updated: 2025-02-11 17:38:43
categories: ZooKeeper
---


## 前言
- 作为一款优秀的分布式协调软件，ZK的重要性毋庸置疑。
- 本文以《从Paxos到ZooKeeper 分布式一致性原理与实践》一书为主要参考，节选书中比较核心的一些知识点。下文简略为该书。
  - 本文沿袭该书中的脉络，先对一些理论介绍，再在介绍ZK的同时会重点说一些应用场景，最后会集中补充一些技术细节。对相应章节感兴趣可以直接跳转查阅。
- 源码部分以ZooKeeper 3.5.8版本为引子，部分内容会结合源码说明
- 源码注释相关，地址：https://github.com/nimbusking/zookeeper
<!-- more -->


---
## 知识点总结

### **一、Zookeeper 核心概念与底层原理**
1. **Zookeeper 的数据模型是什么？**  
   - 类似文件系统的树形结构（ZNode），每个 ZNode 存储数据（字节数组）和元信息（版本、权限等）。  
   - **ZNode 类型**：持久节点（Persistent）、临时节点（Ephemeral）、顺序节点（Sequential）。

2. **Zookeeper 的 Watcher 机制如何工作？**  
   - 客户端对 ZNode 注册 Watcher 监听事件（如节点创建、删除、数据变更），事件触发后服务端通知客户端（一次性通知，需重新注册）。

3. **ZAB 协议（Zookeeper Atomic Broadcast）是什么？**  
   - 专为 Zookeeper 设计的崩溃恢复原子广播协议，保证集群数据一致性。  
   - **两个阶段**：  
     - **选举 Leader**：通过投票机制（myid + zxid）选出 Leader。  
     - **消息广播**：Leader 将事务请求以 Proposal 广播给 Follower，收到半数 ACK 后提交。

4. **Zookeeper 的读写流程是怎样的？**  
   - **读请求**：直接由当前节点处理（可能读到旧数据）。  
   - **写请求**：转发给 Leader，通过 ZAB 协议广播到多数节点，确认后返回成功。

5. **Zookeeper 的会话（Session）机制如何保证高可用？**  
   - 客户端与 Server 建立 TCP 长连接，会话超时时间内未收到心跳则判定会话失效，自动清理临时节点。

---

### **二、应用实践与常见场景**
6. **Zookeeper 的典型应用场景有哪些？**  
   - 分布式配置管理、服务注册与发现、分布式锁、选主（Leader Election）、集群监控。

7. **如何用 Zookeeper 实现分布式锁？**  
   - **方案1（临时顺序节点）**：  
     1. 客户端创建临时顺序节点（如 `/lock/seq-0001`）。  
     2. 检查自己是否是最小序号节点，若是则获取锁；否则监听前一个节点的删除事件。  
     3. 释放锁时删除自身节点。  
   - **方案2（临时节点）**：所有客户端竞争创建同一个临时节点，创建成功者获得锁（需处理惊群效应）。

8. **如何用 Zookeeper 实现服务注册与发现？**  
   - 服务提供者启动时创建临时节点（如 `/services/serviceA/192.168.1.1:8080`）。  
   - 消费者监听该节点，获取可用服务列表。

---

### **三、常见故障与解决**
9. **客户端连接超时可能的原因？**  
   - 网络问题、Zookeeper 集群宕机、客户端未正确处理 Session 超时（需重连并重建临时节点）。

10. **Zookeeper 集群脑裂（Split-Brain）问题如何解决？**  
    - 通过 ZAB 协议保证只有一个 Leader，其他节点自动转为 Follower 或重新选举。

11. **节点删除失败（KeeperErrorCode = NoNode）的可能原因？**  
    - 节点已被删除、版本号不匹配、权限不足。

12. **Watcher 丢失事件的可能原因？**  
    - Watcher 是一次性的，事件触发后需重新注册；网络抖动导致事件未送达。

---

### **四、使用注意事项**
13. **Zookeeper 的部署建议**  
    - 集群节点数为奇数（至少3台），避免脑裂。  
    - 数据目录（dataDir）和日志目录（dataLogDir）分离，提升性能。

14. **ZNode 设计原则**  
    - 避免存储大文件（ZNode 数据上限默认 1MB）。  
    - 临时节点用于会话关联资源（如服务注册），持久节点用于配置信息。

15. **权限控制（ACL）**  
    - 使用 `digest` 或 `ip` 模式限制访问权限，避免未授权操作。

---

### **五、分布式锁方案对比**
| **方案**           | **实现原理**                          | **优点**                          | **缺点**                          | **适用场景**                |
|---------------------|---------------------------------------|-----------------------------------|-----------------------------------|---------------------------|
| **Zookeeper**       | 临时顺序节点 + Watcher                | 强一致性，可靠性高                | 性能较低，实现复杂                | 对一致性要求高的场景（如金融） |
| **Redis**           | SETNX + Redlock 算法                  | 高性能，简单易用                  | 需处理锁续期、脑裂问题            | 高并发、允许短暂不一致      |
| **数据库**          | 唯一索引/乐观锁                       | 实现简单                          | 性能差，无自动释放机制            | 低频操作，小规模系统        |
| **Etcd**            | Lease（租约） + Revision 版本号       | 高可用，支持强一致性              | 依赖 etcd 集群，学习成本较高      | Kubernetes 生态相关场景     |

#### **对比关键点**：
1. **一致性**：Zookeeper 和 Etcd 基于 CP 模型，保证强一致性；Redis 是 AP 模型，可能短暂不一致。  
2. **性能**：Redis 性能最高，Zookeeper 和 Etcd 次之，数据库最差。  
3. **可靠性**：Zookeeper 的临时节点和 Watcher 机制可靠性高；Redis 需额外处理锁续期和超时。  
4. **复杂度**：Zookeeper 实现复杂，Redis 简单但需解决锁续期问题。

---

### **六、Zookeeper 常见问题扩展**
16. **Zookeeper 的 Watch 机制为何设计为一次性？**  
    - 减少服务端压力，避免大量无效事件堆积。客户端需根据业务需求重新注册 Watcher。

17. **Zookeeper 的 zxid 是什么？**  
    - 事务 ID，由 Leader 分配，高 32 位为 epoch（Leader 任期），低 32 位为计数器，保证全局有序。

18. **Zookeeper 的 Follower 和 Observer 的区别？**  
    - Follower 参与选举和写请求的 ACK 投票；Observer 不参与投票，仅同步数据，用于扩展读性能。

---

### **附：Zookeeper 集群配置核心参数**
```properties
# zoo.cfg
tickTime=2000  # 心跳间隔（ms）
initLimit=10    # Follower 连接 Leader 的超时时间（tickTime * initLimit）
syncLimit=5     # Leader 与 Follower 同步数据的超时时间
dataDir=/data   # 数据存储目录
clientPort=2181 # 客户端连接端口
server.1=node1:2888:3888  # 集群节点配置（2888 为数据同步端口，3888 为选举端口）
server.2=node2:2888:3888
server.3=node3:2888:3888
```


---

## 一些分布式理论

### 分布式的特点
- **分布性**：分布式系统中的多台计算机都会在空间上随意分布。
- **对等性**：分布式系统中的计算机没有主／从之分， 既没有控制整个系统的主机，也没有被控 制的从机，组成分布式系统的所有计算机节点都是对等的。
- **并发性**：如果在并发场景下协调好共享资源
- **缺乏全局时钟**：在分布式系统中，很难定义两个事件究竟谁先谁后，原因就是因为分布式系统缺乏一个全局的时钟序列控制。 
- **故障总会发生**：组成分布式系统的所有计算机，都有可能发生任何形式的故障。

### 分布式环境的各种问题
- 通信异常：分布式节点之间需要网络通信，网络通信就带来，比如延迟等异常情况
- 网络分区：组成分布式系统的所有节点，因为网络异常，导致一部分节点能通信，一部分节点不能通信，这个现象称之为网络分区，俗称为“脑裂”。
- 三态：分布式系统的每一次请求与响应，存在特有的“三态”，即成功、失败和超时。
- 节点故障：

### 分布式事务
分布式事务场景下的数据一致性等问题

### 关于CAP与BASE
关于CAP的应用，我们要知道，分布式系统无法同时满足这三个需求的。往往只能满足其中两项，关于这点，在书中有个表格，这里贴出来，需要的时候可以看看缺失其中一项的应用场景是什么：
![CAP定理应用](324f2c0b/CAP定理应用.jpg)
**另外，在分布式场景下，分区容错性（P）是一定要解决的，不然怎么称之为分布式系统？**
这也从某种意义上说，目前分布式解决方案中，一般都是保证CP、或者AP场景下的实现

#### 关于BASE
BASE是由Basically Available(基本可用)、Soft state（软状态）和Eventually consistend（最终一致性）这三个短语构成的。
来自EBAY工程师Dan Pritchett在2008年发布的一篇文章中提到的概念，笔者从ACM网站上下载了下来，有需要可以翻阅看看。
![Introduce BASE different with ACID.pdf](324f2c0b\Introduce BASE different with ACID.pdf) 
原文文章连接：https://dl.acm.org/doi/10.1145/1394127.1394128
【备注】书中介绍了一下ACID背景以及带来的问题，通过引入交易转账的两张小表来阐述怎么做到最终一致性，以及最终一致性的目标。读者可以仔细看看，例子不难，对于理解最终一致性是很有帮助的。概括的说，BASE模型通过弱化过程中的强一致性，保证最终一致性来完成扩展性提升的。文中还举了另外一个例子，转账的例子，过程中作为用户而言是不关心到账过程的，只关心最终到账，因此在此过程中，我们可以放松关于到账的细节的一致性要求。

BASE是对CAP中一致性和可用性权衡的结果，其来源于对大规模互联网系统分布式实践的总结，是基于CAP定理逐步演化而来的，其核心思想是：**即使无法做到强一致性(Strong consistency)，但每个应用都可以根据自身的业务特点，采用适当的方式来使系统达到最终一致性(Eventual consistency)。**
【备注】关于最终一致性，书中还介绍了五类主要的变种，分别是：
- 因果一致性（Causal consistency）
- 读己之所写（Read your writes）
- 会话一致性（Session consistency）
- 单调读一致性（Monotonic /ˌmänəˈtänik/ read consistency）
- 单调写一致性（Monotonic write consistency）

### 一致性协议
【备注】这个章节中介绍了三种经典的分布式一致性算法，分别是二段提交（2PC, Two-Phase Commit）、三段提交（3PC, Three-Phase Commit）以及大名鼎鼎的Paxos算法。
在介绍二、三段提交算法之前，书中引入了“段式”提交的核心理念：
当一个事务操作需要跨越多个分布式节点的时候， 为了保持事务处理的ACID特性， 就需要引入 一个称为 ＂协调者(Coordinator)" 的组件来统一调度所有分布式节点的执行逻辑， 这些被调度的分布式节点则被称为 “参与者” (Participant)。 **协调者负责调度参与者的行为，并最终决定这些参与者是否要把事务真正进行提交。** 

#### 2PC
PC, 是Two-Phase Commit的缩写， 即二阶段提交，是计算机网络尤其是在数据库领域内， 为了使基于分布式系统架构下的所有节点在进行事务处理过程中能够保持原子性和一致性而设计的一种算法。
【备注】你可能也听过另外一个二段：二段锁（2PL, Two-Phase Locking），这俩个是完全不同的东西哦。二段锁，是解决单机事务中的数据的一致性和隔离性而诞生的。而，二段提交是处理的分布式事务。
二段提交示意图，如下图所示：
![二段事务提交示意图](324f2c0b/二段事务提交示意图.jpg)

##### 优缺点
**优点**：原理简单，实现方便。
**缺点**：同步阻塞、单点问题、脑裂、太过保守。
关于缺点：
- **同步阻塞**：在二阶段提交的执行过程中，所有参与该事务操作的逻辑都处千阻塞状态
- **单点问题**：**协调者**的角色在整个二阶段提交协议中起到了非常重要的作用。 一旦协调者出现问题， 那么整个二阶段提交流程将无法运转， 更为严重的是， 如果协调者是在阶段二中出现问题的话， 那么其他参与者将会一直处于锁定事务资源的状态中， 而无法继续完成事务操作。
- **数据不一致**：当协调者向所有的参与者发送Commit请求之后， 发生了局部网络异常或者是协调者在尚未发送完Commit请求之前自身发生了崩溃，导致最终只有部分参与者收到了Commit请求。 于是， 这部分收到了Commit请求的参与者就会进行事务的提交， 而其他没有收到Commit请求的参与者则无法进行事务提交， 整个分布式系统便出现了数据不一致性现象。
- **太过保守**：如果在协调者指示参与者进行事务提交询问的过程中，参与者出现故障而导致协调者始终无法获取到所有参与者的响应信息的话，这时协调者只能依靠其自身的超时机制来判断是否需要中断事务， 这样的策略显得比较保守。 换句话说， 二阶段提交协议没有设计较为完善的容错机制，任意一个节点的失败都会导致整个事务的失败。

【备注】在书中的这几个缺点的详细描述可以很好的理解二段事务提交的特点，衍生的场景，可以了解一下MySQL中事务是怎么使用二段提交这个特性的。
关于2PC和3PC这两个概念，有兴趣的看官，可以参阅1988年出版，由Philip A. Bernstein等人巨著的，《数据库并发控制理论》（暂译）(*CONCURRENCY CONTROL AND RECOVERY IN DATABASE SYSTEMS*)一书，该书中系统的阐述了在数据库系统设计过程中，并发场景的种种问题及挑战，其中2PC和3PC在书中的第7章有详细阐述。这本书，同样涵盖了，例如MVCC（熟悉MySQL的一定不陌生），2PL等数据库并发控制理论。

当然如果你还不过瘾，在研究完本篇Zookeeper相关概念之后，可以参考参考，知乎上一位博主总结的关于分布式的台前幕后：https://zhuanlan.zhihu.com/p/338161857
我这里截取一下该文章中提到的Paxos算法演进过程
![paxos算法演进过程](324f2c0b/paxos算法演进过程.png)

#### 3PC
3PC是由2PC演进而来，其过程主要是将二阶段提交协议的“提交事务请求”过程一分为二，形成了由CanCommit、PreCommit和doCommit三个阶段组成的事务处理协议。
其原理示意图（来自Google）：
![三段事务提交示意图](324f2c0b/三段事务提交示意图.jpg)

##### 优缺点
**优点**：相较于二阶段提交协议，三阶段提交协议最大的优点就是降低了参与者的阻塞范围，并且能够在出现单点故障后继续达成一致。
**缺点**：参与者接收到preCommit消息后，如果网络出现分区，此时协调者所在的节点和参与者无法进行正常的网络通信，在这种情况下，该参与者依然会进行事务的提交，这必然出现数据的不一致性。

### Paxos
Paxos算法是莱斯利·兰伯特( [Leslie Lamport](https://en.wikipedia.org/wiki/Leslie_Lamport)) （【备注】计算机界的一位巨佬，其在分布式领域的贡献不亚于Dijkstra这些巨佬的贡献，2013年图灵奖获得者）1990年提出的**一种基于消息传递且具有高度容错特性的一致性算法，是目前公认的解决分布式一致性问题最有效的算法之一**。
Paxos主要解决的问题就是在前面提到的，在分布式系统中，如果出现了诸如机器宕机或网络异常等情况。Paxos算法需要解决的问题就是**如何在一个可能发生上述异常的分布式系统中，快速且正确地在集群内部对某个数据的值达成一致，并且保证不论发生以上任何异常，都不会破坏整个系统的一致性。**

#### Paxos算法**角色划分**
Paxos算法中有三种角色：
- **Proposer**：提出提案（值）的节点。
- **Acceptor**：接受或拒绝提案的节点，负责存储已接受的提案。
- **Learner**：学习最终被选定的值，不参与决策过程。

#### **算法阶段**
Paxos算法分为两个阶段：**Prepare阶段**和**Accept阶段**。

##### **阶段1：Prepare阶段**
- **Proposer**选择一个全局唯一的提案编号（Proposal ID），并向所有**Acceptor**发送Prepare请求。
- **Acceptor**收到Prepare请求后：
  - 如果收到的提案编号比之前接受的任何提案编号都大，则承诺不再接受比该编号小的提案，并返回已接受的最高编号的提案（如果有）。
  - 否则，拒绝该Prepare请求。

##### **阶段2：Accept阶段**
- **Proposer**收到多数**Acceptor**的响应后：
  - 如果发现有**Acceptor**已经接受了某个值，则选择其中编号最大的值作为自己的提案值。
  - 如果没有值被接受，则可以使用自己提议的值。
- **Proposer**向所有**Acceptor**发送Accept请求，包含提案编号和值。
- **Acceptor**收到Accept请求后：
  - 如果提案编号不小于其承诺的最小编号，则接受该提案并返回响应。
  - 否则，拒绝该提案。

##### **Learner学习值**
- 一旦某个提案被多数**Acceptor**接受，**Learner**就可以学习到该值，算法达成一致。

#### 算法**关键点**
- **多数派原则（Quorum）**：Paxos要求多数**Acceptor**同意才能达成一致，确保即使部分节点故障，系统仍能正常运行。
- **提案编号的唯一性**：提案编号必须全局唯一且递增，用于区分提案的优先级。
- **二阶段提交**：通过Prepare和Accept两个阶段，确保只有一个值被选定。
- **安全性**：Paxos保证最终只有一个值被选定，且不会出现不一致的情况。

#### 关于Chubby
要知道：Chubby提供了粗粒度的分布式锁服务，开发人员不需要使用复杂的同步协议，而是直接调用Chubby的锁服务接口即可实现分布式系统中多个进程之间粗粒度的同步控制，从而保证了分布式数据的一致性。

## ZooKeeper与Paxos
### 初识ZooKeeper
ZooKeeper的设计目标是将那些复杂且容易出错的分布式一致性服务封装起来，构成一个高效可靠的原语集，并以一系列简单易用的接口提供给用户使用。
ZooKeeper是一个典型的分布式数据一致性的解决方案，分布式应用程序可以基于它实现诸如现诸如**数据发布／订阅、负载均衡、命名服务、分布式协调／通知、集群管理、Master选举、分布式锁和分布式队列**等功能。
ZooKeeper通过如下机制来保证分布式一致性特性：
1. **ZAB协议**  
   - **原子广播**：确保所有更新操作按顺序广播到所有节点，要么全部成功，要么全部失败。
   - **崩溃恢复**：故障节点恢复后，通过日志回放与主节点同步，确保数据一致性。
2. **Leader选举**  
   - 集群启动或Leader失效时，通过选举算法选出新Leader，确保只有一个节点负责协调更新。
3. **数据复制**  
   - 写操作由Leader处理并同步到多数Follower节点，确保数据在多数节点上一致。
4. **顺序一致性**  
   - 使用全局唯一的事务ID（zxid）保证所有操作的顺序一致性。
5. **会话管理**  
   - 客户端会话超时后，相关临时节点和锁会被自动清理，防止不一致。
6. **Watch机制**  
   - 客户端可监听节点变化，及时获取更新通知，保持数据一致性。
7. **持久化和日志**  
   - 所有更新操作先写入磁盘日志，确保数据持久化，故障后可通过日志恢复。

### ZooKeeper的基本概念
特别是服务器端集群的三种角色
#### 集群角色
ZooKeeper不同于传统的Master/Slave模式，将整个集群分为**Leader、Follower、Observer三种机器**。
- 关于**Leader机器**：ZooKeeper集群中的所有机器通过一个Leader选举过程来选定一台被称为"Leader"的机器，Leader服务器为客户端提供读和写服务。
- **关于Follower和Observer机器**：Follower和Observer都能够提供读服务，唯一的区别在干，Observer机器不参与Leader选举过程，也不参与写操作的“过半写成功”策略。

#### 会话（Session）
客户端启动的时候，首先会与服务器建立一个TCP连接， 从第一次连接建立开始，客户端会话的生命周期也开始了，通过这个连接，客户端能够通过心跳检测与服务器保持有效的会话，同时也能够向ZooKeeper服务器发送请求并接受响应，同时还能够通过该连接**接收来自服务器的Watch事件通知**。

#### 数据节点（Znode）
这里指的数据节点，是指的ZooKeeper的内存数据结构。在ZooKeeper中，ZNode可以分为持久节点和临时节点两类，其中：
- **持久节点**：指一且这个ZNode被创建了，除非主动进行ZNode的移除操作，否则这个ZNode将一直保存在ZooKeeper上。
- **临时节点**：它的生命周期和客户端会话绑定，一且客户端会话失效，那么这个客户端创建的所有临时节点都会被移除
数据节点还有另外一个额外的属性，**SEQUENTIAL** */səˈkwen(t)SHəl/* ，一旦被标记这个属性，那么这个节点将被创建的时候，会在节点名称后面追加上一个整型数字，这个整型数字是由**父节点维护的一个自增数字**。

#### 版本
每个ZNode，ZooKeeper都会为其维护一个叫作**Stat**的数据结构，Stat中记录了这个ZNode的三个数据版本，分别是**version（当前ZNode的版本）**、cversioo（当前ZNode子节点的版本）和**aversion（当前ZNode的ACL版本）**。

#### 事件监听器（Watcher）
ZooKeeper允许用户在指定节点上注册一些Watcher, 并且在一些特定事件触发的时候，ZooKeeper服务端会将事件通知到感兴趣的客户端上去，该机制是ZooKeeper实现分布式协调服务的重要特性。

#### ACL
ZooKeeper采用ACL (Access Control Lists)策略来进行权限控制，类似于UNIX文件系统的权限控制。其中ZooKeeper定义了如下5中权限：
- **CREATE**：创建**子节点**权限
- **READ**：获取节点数据和**子节点**列表的权限
- **WRITE**：更新节点数据的权限
- **DELETE**：删除子节点的权限
- **ADMIN**：设置节点ACL的权限

### ZooKeeper的ZAB协议

#### ZAB协议概述
要明白一点，ZooKeeper并没有完全采用Paxos算法，而是使用了一种称为 **Zookeeper Atomic Broadcast(ZAB，Zookeeper原子消息广播协议)**的协议作为其数据一致性的核心算法。

在ZooKeeper中，**主要依赖ZAB协议来实现分布式数据一致性**，基于该协议，ZooKeeper实现了一种主备模式的系统架构来保持集群中各副本之间数据的一致性。具体过程概括如下：

**首先：**ZooKeeper使用**一个单一的主进程**来接收并处理客户端的**所有事务请求**，并采用ZAB的原子广播协议，将服务器数据的状态变更以事务Proposal的形式广播到所有的副本进程上去。这个过程的特点是：ZAB协议的这个主备模型架构保证了同一时刻集群中只能够有一个主进程来广播服务器的状态变更，因此能够很好地处理客户端大量的井发请求。

**其次：**考虑到在分布式环境中，顺序执行的一些状态变更其前后会存在一定的依赖关系，有些状态变更必须依赖于比它早生成的那些状态变更。例如变更C需要依赖变更A和变更B。 这样的依赖关系也对ZAB协议提出了一个要求：**ZAB协议必须能够保证一个全局的变更序列被顺序应用**，也就是说，ZAB协议需要保证如果一个状态变更已经被处理了，那么所有其依赖的状态变更都应该巳经被提前处理掉了。

**最后**：考虑到主进程在任何时候都有可能出现崩溃退出或重启现象，因此，ZAB协议还需要做到在当前主进程出现上述异常情况的时候， 依旧能够正常工作。

上述步骤的概述，就是：

*所有事务请求必须由**一个全局唯一的服务器**来协调处理， 这样的服务器被称为 Leader 服务器， 而余下的其他服务器则成为 Follower 服务器。 Leader 服务器负责将一**个客户端事务请求**转换成**一个事务 Proposal**（提议）， 并将该 Proposal **分发**给集群中所有的Follower 服务器。 之后 Leader服务器需要等待所有 Follower 服务器的反馈， 一旦**超过半数**的 Follower 服务器进行了正确的反馈后， 那么 Leader 就会再次向所有的 Follower 服务器分发 Commit 消息要求其将前一个 Proposal 进行提交。*

#### ZAB协议介绍

ZAB协议包含两个基本的模式，分别是崩溃恢复和消息广播。

##### 消息广播

ZAB协议的消息广播过程使用的是一个原子广播协议，***类似于**一个二阶段提交过程*。针ZAB协议的消息广播过程使用的是对客户端的事务请求，Leader服务器会为其生成对应的事务Proposal, 并将其发送给集群中其余所有的机器， 然后再分别收集各自的选票， 最后进行事务提交，如下图所示：

![ZAB协议消息广播流程示意图](324f2c0b/ZAB协议消息广播流程示意图.jpg)

前文中提到这个过程仅仅是类似于一个2PC过程，但是又与2PC过从不同的是：

- 移除了中断逻辑，所有的Follower服务器要么正常反馈Leader 提出的事务Proposal，要么就抛弃Leader服务器。意味着我们可以在过半的Follower服务器已经反馈Ack之后就开始提交事务Proposal了。当然这个过程是解决不了Leader崩溃的问题的，所以由后面的恢复模式来解决这个问题。
- 整个消息广播协议是基于具有FFIO特性的TCP协议来进行网络通信的，因此能够很容易地保证消息广播过程中消息接收与发送的顺序性。

##### 崩溃恢复
崩溃恢复是为了解决Leader节点崩溃或者因为网络原因导致的，失去过半的Follower联系的，就会进入恢复模式。
根本解决的问题是，重新选举的同时，还能很好的通知到整个集群。

### Paxos算法与ZAB算法直接的差异
#### 1. **设计目标**
- **Paxos**：
  - 目标是在分布式系统中就某个值达成一致（即一致性协议）。
  - **更通用，适用于各种分布式一致性问题。**
- **ZAB**：
  - 目标是实现原子广播（Atomic Broadcast），确保所有节点的操作顺序一致。
  - **专为Zookeeper设计，主要用于实现分布式协调服务。**

#### 2. **角色划分**
- **Paxos**：
  - 分为Proposer、Acceptor和Learner三种角色。
  - 角色可以动态变化，灵活性较高。
- **ZAB**：
  - 分为Leader和Follower两种角色。
  - Leader负责协调所有写操作，Follower只负责同步数据，角色相对固定。

#### 3. **算法流程**
- **Paxos**：
  - 分为Prepare和Accept两个阶段。
  - 通过提案编号（Proposal ID）和多数派原则确保一致性。
  - 可能存在活锁问题（多个Proposer竞争）。
- **ZAB**：
  - 分为选举阶段（Leader Election）和广播阶段（Atomic Broadcast）。
  - 选举阶段通过快速选举算法选出Leader。
  - 广播阶段由Leader将操作按顺序广播给Follower，确保所有节点的操作顺序一致。

#### 4. **数据一致性** 【核心差异】
- **Paxos**：
  - 保证最终一致性，但不保证操作的全局顺序。
  - 适用于需要达成一致但不需要严格顺序的场景。
- **ZAB**：
  - 保证操作的严格顺序（通过zxid实现）。
  - 适用于需要强一致性和严格顺序的场景（如Zookeeper的分布式锁）。

#### 5. **性能优化**
- **Paxos**：
  - 通用性强，但实现复杂，性能可能较低。
  - Multi-Paxos通过选举Leader优化性能，减少Prepare阶段的次数。
- **ZAB**：
  - 专为Zookeeper优化，性能较高。
  - 通过快速选举和顺序广播减少通信开销。

#### 6. **应用场景**
- **Paxos**：
  - 适用于各种分布式一致性问题，如分布式存储、状态机复制等。
  - 例如：Google的Chubby、Spanner等系统。
- **ZAB**：
  - 专为Zookeeper设计，适用于分布式协调服务。
  - 例如：Zookeeper的分布式锁、配置管理、命名服务等。

#### 7. **容错性**
- **Paxos**：
  - 支持任意节点故障（包括Leader）。
  - 通过多数派原则确保一致性。
- **ZAB**：
  - 支持Follower故障，但Leader故障会触发重新选举。
  - 通过快速选举和日志同步确保一致性。

#### 总结
| 特性                | Paxos                          | ZAB                            |
|---------------------|--------------------------------|--------------------------------|
| **设计目标**        | 通用一致性协议                 | 原子广播（专为Zookeeper设计）  |
| **角色划分**        | Proposer、Acceptor、Learner    | Leader、Follower               |
| **算法流程**        | Prepare + Accept两阶段         | 选举 + 广播两阶段              |
| **数据一致性**      | 最终一致性                     | 严格顺序一致性                 |
| **性能优化**        | Multi-Paxos优化                | 快速选举 + 顺序广播优化         |
| **应用场景**        | 通用分布式系统                 | Zookeeper分布式协调服务         |
| **容错性**          | 支持任意节点故障               | 支持Follower故障，Leader故障触发选举 |

- **Paxos**更通用，适合解决各种分布式一致性问题，但实现复杂。
- **ZAB**专为Zookeeper优化，性能更高，适合需要严格顺序一致性的场景。

---

## ZooKeeper的典型应用场景
ZooKeeper是一个典型的发布/订阅模式的分布式数据管理与协调框架，开发人员可以使用它来进行分布式数据的发布与订阅。另一方面，通过对ZooKeeper中丰富的数据节点类型进行交叉使用，配合Watcher事件通知机制，可以非常方便地构建一系列分布式应用中都会涉及的核心功能，**如数据发布/订阅、负载均衡、命名服务、分布式协调/通知、集群管理、Master选举、分布式锁和分布式队列等。**
### 数据发布/订阅
数据发布/订阅（Publish/Subscribe）系统，即所谓的**配置中心**，顾名思义就是发布者将数据发布到ZooKeeper的一个或一系列节点上，供订阅者进行数据订阅，进而达到动态获取数据的目的，实现配置信息的集中式管理和数据的动态更新。
**ZooKeeper采用的是推拉相结合的方式**：客户端向服务端注册自己需要关注的节点，一旦该节点的数据发生变更，那么服务端就会向相应的客户端发送Watcher事件通知，客户端接收到这个消息通知之后，需要主动到服务端获取最新的数据。
### 负载均衡
### 命名服务
命名服务（NameService）也是分布式系统中比较常见的一类场景，在《Java网络高级编程》一书中提到，命名服务是分布式系统最基本的公共服务之一。
在分布式系统中，被命名的实体通常可以是集群中的机器、提供的服务地址或远程对象等一这些我们都可以统称它们为名字（Name），其中**较为常见的就是一些分布式服务框架（如RPC、RMI）中的服务地址列表**，**通过使用命名服务，客户端应用能够根据指定名字来获取资源的实体、服务地址和提供者的信息等**。

### 分布式协调/通知
### 集群管理
所谓集群管理，包括集群监控与集群控制两大块，前者侧重对集群运行时状态的收集，后者则是对集群进行操作与控制。在日常开发和运维过程中，我们经常会有类似于如下的需求：
- 希望知道当前集群中究竟有多少机器在工作
- 对集群中每台机器的运行时状态进行数据收集
- 对集群中机器进行上下线操作

### Master选举
Master选举是一个在分布式系统中非常常见的应用场景。分布式最核心的特性就是能够将具有独立计算能力的系统单元部署在不同的机器上，构成一个完整的分布式系统。而与此同时，实际场景中往往也需要在这些**分布在不同机器上的独立系统单元中选出一个所谓的“老大”，在计算机科学中，我们称之为Master。**
这里面有个典型场景，一些分布式密集计算型集群，往往不需要每个节点去计算出结果，只需要其中一台计算出结果之后，之后将结果同步到整个集群，这样旧大大减少重复工作，提升性能。
在该书中举例了一个Master选举来完成计算结果的场景，简单看下面的交互示意图。
![一个Master选举示例](324f2c0b/一个Master选举示例.jpg)
### 分布式锁
分布式锁是控制分布式系统之间同步访问共享资源的一种方式。如果不同的系统或是同一个系统的不同主机之间共享了一个或一组资源，那么访问这些资源的时候，往往需要通过一些互斥手段来防止彼此之间的干扰，以保证一致性，在这种情况下，就需要使用分布式锁了。
在ZooKeeper场景中，实现的过程主要分为两种，排它锁和共享锁。
- **排它锁：**排他锁（ExclusiveLocks，简称X锁），又称为写锁或独占锁，是一种基本的锁类型。如果事务T对数据对象O加上了排他锁，那么在整个加锁期间，只允许事务T对O进行读取和更新操作，其他任何事务都不能再对这个数据对象进行任何类型的操作直到T，释放了排他锁。
  在ZooKeeper中，通过ZooKeeper上的数据节点来表示一个锁，*例如/exclusive_lock/lock节点就可以被定义为一个锁*
  ![ZooKeeper场景排它锁流程](324f2c0b/ZooKeeper场景排它锁流程.jpg)
- **共享锁：**共享锁（SharedLocks，简称S锁），又称为读锁，同样是一种基本的锁类型。如果事务T对数据对象O加上了共享锁，那么当前事务只能对O进行读取操作，其他事务也只能对这个数据对象加共享锁-一直到该数据对象上的所有共享锁都被释放。
  和排他锁一样，同样是通过ZooKeeper上的数据节点来表示一个锁，是一个类似于`/shared_lock/[Hostname]-请求类型-序号`的临时顺序节点，例如`/shared_lock/192.168.0.1-R-0000000001`，那么，这个节点就代表了一个共享锁，如下图所示：
  ![ZooKeeper场景共享锁定义示例](324f2c0b/ZooKeeper场景共享锁定义示例.jpg)
  工作流程如下图所示：
  ![ZooKeeper共享锁工作流程](324f2c0b/ZooKeeper共享锁工作流程.jpg)

#### 羊群效应
上面讲解的这个共享锁实现，**大体上能够满足一般的分布式集群竞争锁的需求**，并且性能都还可以一—这里说的一般场景是指集群规模不是特别大，一般是在10台机器以内。但是如果机器规模扩大之后，会有什么问题呢？
客户端无端地接收到过多和自己并不相关的事件通知，如果在集群规模比较大的情况下，不仅会对ZooKeeper服务器造成巨大的性能影响和网络冲击，更为严重的是，如果同一时间有多个节点对应的客户端完成事务或是事务中断引起节点消失，**ZooKeeper服务器就会在短时间内向其余客户端发送大量的事件通知**，这就是所谓的**羊群效应**。
改进后的共享锁，流程如下图所示，改进的主要区别是：**此时只针对比自己小的节点注册Watcher，而不是所有节点**
![ZooKeeper改进后共享锁工作流程](324f2c0b/ZooKeeper改进后共享锁工作流程.jpg)

在具体的实际开发过程中，我们提倡根据具体的业务场景和集群规模来选择适合自己的分布式锁实现：
- 在集群规模不大、网络资源丰富的情况下，第一种分布式锁实现方式是简单实用的选择；
- 而如果集群规模达到一定程度，并且**希望能够精细化地控制分布式锁机制**，那么不妨试试改进版的分布式锁实现。
### 分布式队列
### 其它分布式系统应用
#### Hadoop
#### HBase
#### Kafka
### ZK在阿里实践中的应用
#### Dubbo

---

## ZooKeeper技术内幕
### 系统模型
- **数据模型**
  - 树结构
  - 事务ID
- **节点特性：4个常见的**
  - 持久节点（PERSISTENT）
  - 持久顺序节点（PERSISTENT_SEQUENTIAL）
  - 临时节点（EPHEMERAL）
  - 临时顺序节点（EPHEMERAL_SEQUENTIAL）
- **版本管理：保证分布式数据原子性操作**
  ZooKeeper中为数据节点引入了版本的概念，每个数据节点都具有三种类型的版本信息，对数据节点的任何更新操作都会引起版本号的变化，下表中对这三类版本信息分别进行了说明。
  | 版本类型        | 说明           |
  | ------------- |:-------------:|
  |  version      |当前数据节点数据内容的版本号 |
  | cversion      | 当前数据节点子节点的版本号      |
  | aversion      | 当前数据节点ACL变更版本号      |

  ZooKeeper中的版本概念和传统意义上的软件版本有很大的区别，**它表示的是对数据节点的数据内容、子节点列表，或是节点ACL信息的修改次数**，我们以其中的version这种版本类型为例来说明：
  在一个数据节点/zk-book被创建完毕之后，节点的version值是0，表示的含义是“当前节点自从创建之后，被更新过0次”。如果现在对该节点的数据内容进行更新操作，那么随后，version的值就会变成1。同时需要注意的是，在上文中提到的关于version的说明，其表示的是对数据节点数据内容的变更次数，强调的是变更次数，因此**即使前后两次变更并没有使得数据内容的值发生变化，version的值依然会变更。**

- **Watcher：数据变更的通知**
  - 一次性：
  - 客户端串行执行：
  - 轻量
- **ACL：保障数据的安全**
### 序列化与协议-Jute
【备注】知道有这么个东西，ZooKeeper官方团队曾经一直想换，但是种种原因没换成，最后沿用到现在，Jute也不是什么大问题，够用。
### 客户端
ZooKeeper的客户端主要由以下几个核心组件组成。
- ZooKeeper实例：客户端的人口。
- ClientWatchManager：客户端Watcher管理器。
- HostProvider：客户端地址列表管理器。
- clientCnxn：客户端核心线程，其内部又包含两个线程，即SendThread和EventThread。前者是一个I/O线程，主要负责ZooKeeper客户端和服务端之间的网络I/O通信；后者是一个事件线程，主要负责对服务端事件进行处理。
#### 一次会话创建的过程
一个图，了解一下
![ZooKeeper客户端一次会话创建过程](324f2c0b/ZooKeeper客户端一次会话创建过程.jpg)

### 服务器启动
ZooKeeper服务器的启动，大体可以分为以下五个主要步骤：配置文件解析、初始化数据管理器、初始化网络I/O管理器、数据恢复和对外服务。
参考如下流程图
![ZooKeeper服务器启动流程](324f2c0b/ZooKeeper服务器启动流程.jpg)

---

### Leader选举（核心）
选举的核心架构示意图如下图所示，本质上就是使用了若干缓存队列来加速处理过程
![ZooKeeper选举多层架构](324f2c0b/ZooKeeper选举多层架构.jpg)
#### 启动时的选举
在我们讲解Leader选举的时候，需要注意的一点是，隐式条件便是ZooKeeper的**集群规模至少是2台机器**，这里我们以3台机器组成的服务器集群为例。
在服务器集群初始化阶段，当有一台服务器（我们假设这台机器的myid为1，因此称其为Serverl）启动的时候，它是无法完成Leader选举的，是无法进行Leader选举的。
当第二台机器（同样，我们假设这台服务器的myid为2，称其为Server2）也启动后，此时这两台机器已经能够进行互相通信，每台机器都试图找到一个Leader，于是便进入了Leader选举流程：
1. **每个Server会发出一个投票**
    由于是初始情况，因此对于Server1和Server2来说，都会将自己作为Leader服务器来进行投票，每次投票包含的最基本的元素包括：所推举的服务器的myid和ZXID，我们以 *（myid，ZXID）* 的形式来表示（实际是Vote对象）。因为是初始化阶段，因此无论是Serverl还是Server2，**都会投给自己**，即Serverl的投票为 *（1，0）*，Server2的投票为 *（2，0）*，然后各自将这个投票发给集群中其他所有机器。
2. **接收来自各个服务器的投票**
  每个服务器都会接收来自其他服务器的投票。集群中的每个服务器在接收到投票后，首先会判断该投票的有效性，包括检查是否是本轮投票、是否来自LOOKING状态的服务器。
3. **处理投票**
  在接收到来自其他服务器的投票后，针对每一个投票，服务器都需要将别人的投票和自己的投票进行PK，PK的规则如下：
    - 优先检查ZXID。ZXID比较大的服务器优先作为Leader。
    - 如果ZXID相同的话，那么就比较myid。myid比较大的服务器作为Leader服务器。
4. **统计投票**
  每次投票后，服务器都会统计所有投票，**判断是否已经有过半的机器接收到相同的投票信息**。对于Server1和Server2服务器来说，都统计出集群中已经有两台机器接受了（2，0）这个投票信息。这里我们需要对“过半”的概念做一个简单的介绍。所谓“过半”就是指大于集群机器数量的一半，**即大于或等于（n/2+1）**。对于这里由3台机器构成的集群，大于等于2台即为达到“过半”要求。
  那么，当Server1和Server2都收到相同的投票信息（2，0）的时候，即认为已经选出了Leader。
5. **改变服务器状态**
  一旦确定了Leader，每个服务器就会更新自己的状态：如果是Follower，那么就变更为FOLLOWING，如果是Leader，那么就变更为LEADING。

那么上面的5步流程，我们简化一下，示意图如下图所示：
![启动时集群选举示意图](324f2c0b/3台服务器启动时集群选举示意图.jpg)
#### 运行期间的选举
只有一种情况，启动时选举的Leader节点挂了，此时需要重新选举来确定Leader节点，基本跟启动时一致，只不过内部投票Vote信息是不同的。
我们假设当前正在运行的ZooKeeper服务器由3台机器组成，分别是Serverl、Server2和Server3，**当前的Leader是Server2**。假设在某一个瞬间，Leader挂了，这个时候便开始了Leader选举。
1. 变更状态。
  当Leader挂了之后，余下的非Observer服务器都会将自已的服务器状态变更为LOOKING，然后开始进人Leader选举流程。
2. 每个 Server会发出一个投票。
  在这个过程中，需要生成投票信息（myid，ZXID）。**因为是运行期间，因此每个服务器上的ZXID可能不同**，我们假定Server1的ZXID为123，而Server3的ZXID为122。在第一轮投票中，Serverl和Server3都会投自己，即分别产生投票（1，123）和（3，122），然后各自将这个投票发给集群中所有机器。
3. 接收来自各个服务器的投票。
4. 处理投票。
  对于投票的处理，和上面提到的服务器启动期间的处理规则是一致的。在这个例子里面，由于Server1的ZXID为123，Server3的ZXID为122，那么显然，Serverl会成为Leader。
5. 统计投票。
6. 改变服务器状态。

#### Leader选举的算法分析
在ZooKeeper中，提供了**三种Leader选举的算法**，分别是LeaderElection、UDP版本的FastLeaderElection和TCP版本的FastLeaderElection，可以通过在配置文件zoo.cfg中使用electionAlg属性来指定，分别使用数字0～3来表示。
- 0代表LeaderElection，这是一种纯UDP实现的Leader选举算法；
- 1代表UDP版本的FastLeaderElection，并且是非授权模式；
- 2也代表UDP版本的FastLeaderElection，但使用授权模式；
- 3代表TCP版本的FastLeaderElection。

值得一提的是，**从3.4.0版本开始，ZooKeeper废弃了0、1和2这三种Leader选举算法，只保留了TCP版本的FastLeaderElection选举算法**。
这里面会牵扯到几个概念性的术语：
- **SID：服务器ID**
  SID是一个数字，用来唯一标识一台ZooKeeper集群中的机器，每台机器不能重复，和myid的值一致。
  PS: myid在配置集群的时候配置中解析出来的
- **ZXID：事务ID**
  **ZXID是一个事务ID，用来唯一标识一次服务器状态的变更。**在某一个时刻，集群中每台机器的ZXID值不一定全都一致，这和ZooKeeper服务器对于客户端“更新请求”的处理逻辑有关。
- **Vote：投票**
  Leader选举，顾名思义必须通过投票来实现。当集群中的机器发现自已无法检测到Leader机器的时候，就会开始尝试进行投票。
- **Quorum：过半机器数**
  这是整个Leader选举算法中最重要的一个术语，我们可以把这个术语理解为是一个量词，**指的是ZooKeeper集群中过半的机器数**，如果集群中总的机器数是n的话，那么可以通过下面这个公式来计算quorum的值：
  ```quorum = (n/2 + 1)```
  例如，如果集群机器总数是3，那么quorum就是2。

##### 算法过程
这个过程，可以简化的看上面的那个示意图中的显示。唯一不同的，示意图中只例举了3台集群，往往实际应用中会有很多，你可以在模拟阶段，譬如使用5台机器来模拟，这样不同阶段，同一条机器收到的Vote选票就是不止一个了，但是比较的规则依旧如此。
概括的，再来看看下面的流程图，帮助回忆：
![Leader选举算法实现流程示意图](324f2c0b/Leader选举算法实现流程示意图.jpg)
拆解开来看：
1. 自增选举轮次
  在FastLeaderElection实现中，有一个logicalclock属性，**用于标识当前Leader的选举轮次，ZooKeeper规定了所有有效的投票都必须在同一轮次中**。ZooKeeper在开始新一轮的投票时，会首先对logicalclock进行自增操作。
2. 初始化选票
  在开始进行新一轮的投票之前，每个服务器都会首先初始化自己的选票
  | 属 性        | 选票值          |
  | ------------- |:-------------:|
  |  id      |当前服务器自身的SID |
  | zxid      | 当前服务器最新的ZXID值      |
  | electionEpoch      | 当前服务器的选举轮次      |
  | peerEpoch      | 被推举的服务器的选举轮次      |
  | state      | LOOKING      |
3. 发送初始化选票
  在完成选票的初始化后，服务器就会发起第一次投票。**ZooKeeper会将刚刚初始化好的选票放入sendqueue队列中，由发送器WorkerSender负责发送出去。**
4. 接收外部投票
  每台服务器都会不断地从`recvqueue`队列中获取外部投票。
  如果服务器发现无法获取到任何的外部投票，那么就会立即确认自己是否和集群中其他服务器保持着有效连接。如果发现没有建立连接，那么就会马上建立连接。如果已经建立了连接，那么就再次发送自己当前的内部投票。
5. 判断选举轮次
  当发送完初始化选票之后，接下来就要开始处理外部投票了。在处理外部投票的时候，会根据选举轮次来进行不同的处理。
  - 外部投票的选举轮次大于内部投票:
    如果服务器发现自已的选举轮次**已经落后于**该外部投票对应服务器的选举轮次，那么就会**立即更新自己的选举轮次（logicalclock）**，并且**清空所有已经收到的投票**，然后使用初始化的投票来进行PK以确定是否变更内部投票（关于PK的逻辑会在步骤6中统一讲解），最终再将内部投票发送出去。
  - 外部投票的选举轮次小于内部投票:
    如果接收到的选票的选举轮次落后于服务器自身的，那么ZooKeeper就会直接忽略该外部投票，**不做任何处理**，并返回步骤4。
  - 外部投票的选举轮次和内部投票一致:
    这也是绝大多数投票的场景，如果外部投票的选举轮次和内部投票一致的话，那么就开始进行选票PK。

6. 选票PK
  在步骤5中提到，在收到来自其他服务器有效的外部投票后，就要进行选票PK了也就是`FastLeaderElection.totalOrderPredicate`方法的核心逻辑。
  选票PK的目的是为了确定当前服务器是否需要变更投票，主要从**选举轮次、ZXID和SID三个因素来考虑**，具体条件如下：**在选票PK的时候依次判断，符合任意一个条件就需要进行投票变更。**
  - 如果外部投票中被推举的Leader服务器的选举轮次大于内部投票，那么就需要进行投票变更。
  - 如果选举轮次一致的话，那么就对比两者的ZXID。如果外部投票的ZXID大于内部投票，那么就需要进行投票变更。
  - 如果两者的ZXID一致，那么就对比两者的SID。如果外部投票的SID大于内部投票，那么就需要进行投票变更。
  
7. 变更投票
  通过选票PK后，如果确定了外部投票优于内部投票（所谓的“优于”，是指外部投票所推举的服务器更适合成为Leader），那么就进行投票变更—使用外部投票的选票信息来覆盖内部投票。变更完成后，再次将这个变更后的内部投票发送出去。
8. 选票归档
  无论是否进行了投票变更，都会将刚刚收到的那份外部投票放入“选票集合”recvset中进行归档。
9. 统计投票
  完成了选票归档之后，就可以开始统计投票了。统计投票的过程就是为了统计集群中**是否已经有过半的服务器认可了当前的内部投票**。如果确定已经有过半的服务器认可了该内部投票，则终止投票。否则返回步骤4。
10. 更新服务器状态
  统计投票后，如果已经确定可以终止投票，那么就开始更新服务器状态。
  服务器会首先判断当前被过半服务器认可的投票**所对应的Leader服务器是否是自己**，
  - 如果是自己的话，那么就会将自己的服务器状态更新为LEADING。
  - 如果自己不是被选举产生的Leader的话，那么就会根据具体情况来确定自已是FOLLOWING或是OBSERVING。

#### Zookeeper选举过程中可能出现的问题
Zookeeper 的选举过程虽然设计精巧，但在实际应用中仍存在一些问题，主要包括以下几点：
1. **脑裂问题（Split-Brain）**
   - **问题描述**：在网络分区的情况下，可能会出现多个Leader，导致数据不一致。
   - **解决方案**：Zookeeper 通过 **Quorum机制**（多数派原则）来避免脑裂，只有获得多数节点认可的Leader才能进行写操作。
2. **选举性能问题**
   - **问题描述**：在大规模集群中，选举过程可能会比较耗时，尤其是在网络不稳定的情况下。
   - **解决方案**：可以通过优化网络配置、减少集群规模或使用更高效的选举算法（如Raft）来缓解。
3. **选举过程中的服务不可用**
   - **问题描述**：在选举过程中，Zookeeper 集群可能无法处理写请求，导致服务暂时不可用。
   - **解决方案**：可以通过配置 Observer 节点来分担读请求，减少选举对服务的影响。
4. **Zxid冲突**
   - **问题描述**：在极端情况下，可能会出现Zxid冲突，导致数据不一致。
   - **解决方案**：Zookeeper 通过严格的Zxid生成规则和日志回放机制来避免这种情况。
5. **网络分区的影响**
   - **问题描述**：网络分区可能导致部分节点无法参与选举，从而影响集群的可用性。
   - **解决方案**：可以通过配置多个机房、使用更可靠的网络设备以及设置合理的超时时间来减少网络分区的影响。
6. **选举算法的复杂性**
   - **问题描述**：ZAB协议相对复杂，实现和理解起来有一定难度。
   - **解决方案**：可以通过使用更简单的选举算法（如Raft）来替代ZAB，但需要权衡一致性和性能。
7. **资源竞争**
   - **问题描述**：在高并发场景下，选举过程中可能会出现资源竞争，影响选举效率。
   - **解决方案**：可以通过优化资源分配和锁机制来减少资源竞争。
8. **配置不当**
   - **问题描述**：如果集群配置不当（如超时时间设置不合理），可能会导致频繁选举，影响集群稳定性。
   - **解决方案**：需要根据实际业务需求和网络环境，合理配置超时时间和其他参数。


---

### 各种服务器角色
#### Leader
Leader服务器是整个ZooKeeper集群工作机制中的核心，其主要工作有以下两个：
- 事务请求的唯一调度和处理者，保证集群事务处理的顺序性：使用责任链模式来处理每一个客户端请求是ZooKeeper的一大特色，ZK请求中使用了7个组件来依次处理请求。
- 集群内部各服务器的调度者。

#### Follower
从角色名字上可以看出，Follower服务器是ZooKeeper集群状态的跟随者，其主要工作有以下三个：
- 处理客户端非事务请求，转发事务请求给Leader服务器。
- 参与事务请求 Proposal 的投票。
- 参与Leader 选举投票。

#### Observer
Observer是ZooKeeper自`3.3.0`版本开始引入的一个全新的服务器角色。
从字面意思看，该服务器充当了一个观察者的角色一其观察ZooKeeper集群的最新状态变化并将这些状态变更同步过来。**Observer服务器在工作原理上和Follower基本是一致的**，对于非事务请求，都可以进行独立的处理，而对于事务请求，则会转发给Leader服务器进行处理。
和Follower唯一的区别在于，Observer不参与任何形式的投票，包括事务请求Proposal的投票和Leader选举投票。简单地讲 **，Observer服务器只提供非事务服务，通常用于在不影响集群事务处理能力的前提下提升集群的非事务处理能力。**







