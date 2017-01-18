---
layout: post
title: "Paxos and Raft"
date: 2016-09-23 17:23
comments: true
category: "Big Data"
tags: [consensus]
---

{:toc}

## 1. Distributed Consensus

分布式一致性是指在一组process里对一个value达成的一致。这个value可以是任何操作，例如“修改某个变量的值”，“设置某个节点为primary”等等。为什么需要分布式一致性呢，因为在分布式系统里，组件是不可靠的，无论是网络问题还是组件自身的问题，我们希望在这些不可靠组件构成的系统中得到一个可靠的系统。

一致性协议就是为了解决这些不可靠组件如何对外提供一致的value。具体说来可以认为多个process可以提供不同的value，一致性协议能迫使这些组件互相协作得到一个一致的结论，并且允许有少量的process出现失败或重启。

一致性协议可以应用的场景包括节点中状态机的复制，分布式的键值存储，分布式序号生成等等，任意节点都可以接收消息，但是对外提供的确是一致的value。

<!--more-->

在分布式系统中，基础复制方法有以下几种：

* 主从异步复制（磁盘在复制前损毁，则造成数据丢失）
* 主从同步复制（一个失联节点会造成整个系统不可用）
* 主从版同步复制（可能任何从库都不完整，需要多数派读写）
* 多数派读写

## 2. Paxos

Paxos算法是Lamport于1990年提出的一种基于消息传递的一致性算法，一开始并未引起人们注意，直到06年google的chubby锁服务使用paxos作为chubby cell中的一致性算法，paxos的人气从此一路狂飙。

Paxos有两个原则：

* 安全原则---保证不能做错的事
	* 只能有一个值被批准，不能出现第二个值把第一个覆盖的情况
	* 每个节点只能学习到已经被批准的值，不能学习没有被批准的值
* 存活原则---只要有多数服务器存活并且彼此间可以通信最终都要做到的事
	* 最终会批准某个被提议的值
	* 一个值被批准了，其他服务器最终会学习到这个值 

Paxos有两个角色：

* Proposer：提议发起者，处理客户端请求，将客户端的请求发送到集群中，以便决定这个值是否可以被批准。
* Acceptor：提议批准者，负责处理接收到的提议，他们的回复就是一次投票。会存储一些状态来决定是否接收一个值。

<img src="http://7xqfqs.com1.z0.glb.clouddn.com/16-9-20/80935469.jpg" width="700px"/>

### 2.1 Paxos过程

下面介绍Paxos协议过程，以及解决的一些问题。

**1、第一阶段 Prepare**

**P1a：Proposer 发送 Prepare**

Proposer 生成全局唯一且递增的提案 ID（Proposalid，以高位时间戳 + 低位机器 IP 可以保证唯一性和递增性），向 Paxos 集群的所有机器发送 PrepareRequest，这里无需携带提案内容，只携带 Proposalid 即可。

**P1b：Acceptor 应答 Prepare**

Acceptor 收到 PrepareRequest 后，做出“两个承诺，一个应答”。

两个承诺：

* 第一，不再应答 Proposalid `小于等于`（注意：这里是 <= ）当前请求的 PrepareRequest；
* 第二，不再应答 Proposalid `小于`（注意：这里是 < ）当前请求的 AcceptRequest

一个应答：

* 返回自己已经 Accept 过的提案中 ProposalID 最大的那个提案的内容，如果没有则返回空值;

**注意：这“两个承诺”中，蕴含两个要点：**

* 就是应答当前请求前，也要按照“两个承诺”检查是否会违背之前处理 PrepareRequest 时做出的承诺；
* 应答前要在本地持久化当前 Propsalid。

**2、第二阶段 Accept**

**P2a：Proposer 发送 Accept**

“提案生成规则”：Proposer 收集到多数派应答的 PrepareResponse 后，从中选择proposalid最大的提案内容，作为要发起 Accept 的提案，如果这个提案为空值，则可以自己随意决定提案内容。然后携带上当前 Proposalid，向 Paxos 集群的所有机器发送 AccpetRequest。

**P2b：Acceptor 应答 Accept**

Accpetor 收到 AccpetRequest 后，检查不违背自己之前作出的“两个承诺”情况下，持久化当前 Proposalid 和提案内容。最后 Proposer 收集到多数派应答的 AcceptResponse 后，形成决议。

<img src="http://7xqfqs.com1.z0.glb.clouddn.com/16-9-20/9823122.jpg" width="600px"/>

### 2.2 核心思想

* Optimistic concurrency control. Hold a `preemptible lock` first, try updating, restart on denial.
* Quorum as a logical unit of acceptor for Choose operation. A value is chosen iff it's accepted by a quorum, which implies from the proposer's perspective the Choose operation is `atomic`, it's all or nothing, it's either accepted by a quorum or it isn't.

### 2.3 协议推导

Paxos 协议利用了 Quorum 机制，选择的 W=R=N/2+1。简单而言，协议就是 Proposer 更新 Acceptor 的过程，一旦某个 Proposer 成功更新了超过半数的 Acceptor，则更新成功。Learner 按 Quorum 去读取 Acceptor，一旦某个 value在超过半数的 Acceptor 上被成功读取，则说明这是一个被批准的 value。协议通过引入轮次，使得高轮次的议抢占低轮次的提议来避免死锁。

协议的关键点是如何满足“在一次Paxos算法实例过程中只批准一个Value”，称这个约束为“约束条件 1”，推到过程就是不断推到出“约束条件 1”的充分不必要条件。

* “约束条件 2” => “约束条件 1”：一旦一个 value 获得超过半数的 Acceptor 批准,之后 Paxos 协议实例只能批准这个 value。
* “约束条件 3” => “约束条件 2”：一旦一个 value 获得超过半数的 Acceptor 批准,之后任何 Acceptor 只能批准这个 value。
* “约束条件 4” <=> “约束条件 3”: 一旦一个 value v 获得超过半数的 Acceptor 批准,之后 Proposer  议的 value 只能是 v。
* “约束条件 5” => “约束条件 4”：Proposer 提议一个 value v 前,要么之前没有任何一个 value 被批准,要么存在一个大小为 N/2+1 的 Acceptor 集合,这个集合内的各个 Acceptor 批准过的轮数最大的 value 是 v。

可以用反证法证明“约束条件 5”.

### 2.4 问题

1. 这里可以看到paxos使用了多acceptor，为什么？
	
	为了解决Acceptor crash的问题，必须要用到一种多数选择的方法。
	
2. 如何保证批准的提案无法改变？
	
	假设Proposer以更高的序号发提案，但已经批准的提案必然被一半以上的Acceptor接受，那么Acceptor返回应答必然包含这个已经批准的提案，所以此时Proposer发起的Accept消息必然是（高序号，已经批准的提案）这么一个组合。所以一旦一个提案被批准，以后永远只能批准这个提案。
	
3. paxos中是否存在死锁和活锁？
	
	虽然paxos协议过程类似于”占坑“，需要value抢占超过半数的”坑“，但是高轮数的提案可以抢占低轮数的提案，所以可以避免死锁的发生。但是这种设计可能导致”活锁“，即Proposer互相不断以更高的轮数提出提案，使得每轮paxos过程都无法完成。一种解决方案是Proposer重新提案之前等待随机时间，让上一个提案有时间进行第二阶段的accept。
	
	<img src="http://7xqfqs.com1.z0.glb.clouddn.com/16-9-27/70209540.jpg" width="500px"/>

4. 如果某个提案之后又另外一个提案跟进会发生什么情况？

	如果前一个提案已经被多数派接受，那新的提案会看到和使用前一个提案作为它的value；如果前一个提案没有被多数派接受，但新提案发现这个提案，那新提案也会使用它作为value；如果前一个提案没有被多数派接受，并且新提案没有发现，则新提案会使用自己的value，旧的提案不会被block。
	
    <img src="http://7xqfqs.com1.z0.glb.clouddn.com/16-9-27/38064842.jpg" width="500px"/>
        
    <img src="http://7xqfqs.com1.z0.glb.clouddn.com/16-9-27/41211772.jpg" width="500px"/>
        
    <img src="http://7xqfqs.com1.z0.glb.clouddn.com/16-9-27/4176188.jpg" width="500px"/>
	
5. server如何知道提案的value？

	只有Proposer自己知道自己选择的提案，别的server如果想要知道必须发起一个paxos instance。
	
6. 在 P2a 阶段，为什么Proposer发起的提案需要使用旧的value和更高的proposalID？
   
   是受“在一次paxos instance中只批准一个value”的约束。即使读取了N/2+1的acceptor状态，由于没有读取所有acceptor，Proposer也无法判断value是否批准了。这时候选择proposalID更高的旧value，可以保证，要么此时Paxos还没有批准任何一个value，要么只能是旧的value。

7. 如果提案被多数派接受后，一个acceptor宕机了，导致多数派不成立，会有什么问题？ 

	不会有问题，因为新的Proposer选择的quorum一定包括接受上一次提案的那台机器(接受的机器数至少是N/2)，这时候会使用更高的proposalID提交旧的value。

### 2.5 Zab

Zookeeper 使用一种修改后的 Paxos 协议，称为 Zab。

在 zookeeper 中，始终分为两种场景：

* Leader activation：leader 选举，数据同步
* Active messaging：leader 接收 client 更新请求，同步到各个 follower

在两种场景中，zk都依赖于一个全局版本号：zxid。zxid 由 (epoch, count) 组成，`epoch` 是选举编号，每次提议进行leader选举时 epoch 都会增加，`count` 是 leader 为每个更新操作决定给的序号。从全局看，**一个 zxid 代表了一个更新操作的全局序号（版本号）**。

每个 zookeeper 节点都有各自最后 commit 的 zxid，表示这个 zookeeper 节点上最近成功执行的更新操作，也代表了这个节点的数据版本。在 Leader activation 阶段，**每个 zk 节点都以自己的 zxid 作为 proposalID 发起 paxos instance，设置自己为leader (value)**。每个节点既是 Proposer 也是 Acceptor。通过 Paxos 协议，某个超过 quorum 半数的节点中持有最大的 zxid 节点会成为新的leader。实际上 proposalID 会是 (zxid，nodeid)，这是当 zxid 相同时，zk 会选择节点编号较大的成为 leader。成为新的 leader 需要与 follower 进行同步，数据同步过程可能会涉及删除 follower 上的最后一条脏数据。

当与至少半数节点完成数据同步后，leader 更新 epoch，在各个 follower 上以 (epoch + 1, 0) 为 zxid 写一条没有数据的更新操作。这个更新操作称为 `NEW_LEADER` 消息，是为了在各个节点上更新 leader 信息，当收到超过半数的 follower 对 NEW_LEADER 的确认后，leader 发起对 NEW_LEADER 的 COMMIT 操作，并进入 Active messaging 阶段。

进入 active messaging 状态的 leader 会接收从客户端发来的更新操作，为每个更新操作生成递增的 count，组成递增的 zxid。Leader 将更新操作以 zxid 的顺序发送给各个 follower (包括leader本身, 一个 leader 同时也是 follower，当收到超过半数的 follower 的确认后，Leader 发送针对该更新操作的 COMMIT 消息给各个 follower。这个更新操作的过程很类似`两阶段提交`，只是 leader 永远不会对更新操作做 abort 操作。

如果 leader 不能更新超过半数的 follower，此时可以发起新的 leader 选举。最后一条更新操作处于“中间状态”，其是否生效取决于选举出的新 leader 是否有该条更新操作。

Zookeeper 通过 zxid 将两个场景阶段较好的结合起来，且能保证全局的强一致性。

* 由于同一时刻只有一个 zookeeper 节点能获得超过半数的 follower，所以同一时刻最多只存在唯一的 leader；
* 每个 leader 利用 TCP 带来的消息 FIFO 特点以 zxid 顺序更新各个 follower，只有成功完成前一个更新操作的才会进行下一个更新操作；
* 在同一个 leader 任期内，数据在全局满足 quorum 约束的强一致，即读超过半数的节点一定可以读到最新已提交的数据；
* 每个成功的更新操作都至少被超过半数的节点确认，使得新选举的 leader 一定可以包括最新的已成功提交的数据。

## 4. Raft

Paxos 相比 Raft 比较复杂和难以理解。角色扮演和流程比 Raft 都要啰嗦。比如 Agreement 这个流程，在 Paxos 里边：Client 发起请求举荐 Proposer 成为 Leader，Proposer 然后向全局 Acceptors 寻求确认，Acceptors 全部同意 Proposer 后，Proposer 的 Leader 地位得已承认，Acceptors 还得再向Learners 进行全局广播来同步。鉴于 paxos 的难以理解，standford 开发另外一个一致性算法 Raft。

一般来说一致性算法有两种：

* 节点同质，leader-less
* 节点异构，leader-based

容易理解上面的这两种方法，Paxos 是节点同质的，Raft 采用节点异构，那对比与同质的有什么好处？

* 问题得到分解（normal operation、leader change），可以类比 zab。
* normal operation 得到简化，没有冲突
* 比 leader-less 来的更高效，相当于一次 prepare 阶段可以有多次 accept 阶段。

那下面就来说说 Raft 是怎么做的，为什么会比 paxos 容易理解。

### 4.1 Server states

Server 被分为 Leader、Follower、Candidate。这个类似于分布式副本管理中的 primary 和 secondary。Leader 负责与 client 交互并发送同步 RPC 命令给 Follower；Follower 是完全被动的接收和反馈 RPC 命令；Candidate 用来选举 Leader。状态图如下：

<img src="http://7xqfqs.com1.z0.glb.clouddn.com/public/16-12-7/40396396.jpg" width="600px"/>

Raft 中 Follower 长时间没有接受到心跳就会转为 Candidate 状态，收到多数投票应答之后可以转为 Leader，Leader 会定期向其他节点发送心跳。当 Leader 和 Candidate 接收到更高版本的消息后，转为 Follower。

<img src="http://7xqfqs.com1.z0.glb.clouddn.com/public/16-12-7/64890697.jpg" width="700px"/>

### 4.2 Leader Election

Raft 中时间被分为 Terms 这么一个概念，在 Terms 中完成 Leader Election 和 Normal operation。每个 term 最多只有一个 leader，term 允许选举失败，导致 term 没有 leader（未达到多数投票应答而超时）。

<img src="http://7xqfqs.com1.z0.glb.clouddn.com/public/16-12-7/22131457.jpg" width="600px"/>


Raft 的选主过程中，每个 Candidate 节点先将本地的 Current Term 加一，然后向其他节点发送RequestVote 请求，其他节点根据本地数据版本、长度和之前选主的结果判断应答成功与否。具体处理规则如下：

1.  如果 now – lastLeaderUpdateTimestamp < elect_timeout，忽略请求
2.  如果 req.term < currentTerm，忽略请求。
3.  如果 req.term > currentTerm，设置 req.term 到 currentTerm 中，如果是 Leader 和Candidate 转为 Follower。
4.  如果 req.term == currentTerm，并且本地 voteFor 记录为空或者是与 vote 请求中 term 和CandidateId 一致，req.lastLogIndex  > lastLogIndex，即 Candidate 数据新于本地则同意选主请求。
5.  如果 req.term == currentTerm，如果本地 voteFor 记录非空或者是与 vote 请求中 term 一致 CandidateId 不一致，则拒绝选主请求。
6. 如果 lastLogTerm > req.lastLogTerm，本地最后一条 Log 的 Term 大于请求中的 lastLogTerm，说明 candidate 上数据比本地旧，拒绝选主请求
 上面的选主请求处理，符合Paxos的“少数服从多数，后者认同前者”的原则。按照上面的规则，选举出来的 Leader，一定是多数节点中 Log 数据最新的节点。

下面来分析一下**选主的时间和活锁问题**。

设定 Follower 检测 Leader Lease 超时为 HeartbeatTimeout，Leader 定期发送心跳的时间间隔将小于 HeartbeatTimeout，避免 Leader Lease 超时，通常设置为小于 HeartbeatTimeout/2。当选举出现冲突，即存在两个或多个节点同时进行选主，且都没有拿到多数节点的应答，就需要重新进行选举，这就是常见的选主活锁问题。类似于 Paxos，Raft 中引入随机超时时间机制，有效规避活锁问题。

注意上面的 Log 新旧的比较，是基于 `lastLogTerm` 和 `lastLogIndex` 进行比较，而不是基于 `currentTerm` 和 `lastLogIndex` 进行比较。**currentTerm 只是用于忽略老的Term的vote请求，或者提升自己的currentTerm，并不参与Log新旧的决策**。

考虑一个非对称网络划分的节点，在一段时间内会不断的进行vote，并增加currentTerm，这样会导致网络恢复之后，Leader会接收到AppendEntriesResponse中的term比currentTerm大，Leader就会重置currentTerm并进行StepDown，这样Leader就对齐自己的Term到划分节点的Term，重新开始选主，最终会在上一次多数集合中选举出一个term>=划分节点Term的Leader。

Raft中收到任何term高于currentTerm的请求都会进行StepDown，前提是必须经过前期必要的检查，就像VoteRequest处理中时间戳的检查。


### 4.3 Log Recovery

Log Recovery 就是要保证一定已经 Committed 的数据不会丢失，未 Committed 的数据转变为 Committed，但不会因为修复过程中断又重启而影响节点之间一致性。

>Committed的定义: 持久的，最终会被状态机执行的Log Entry。

Log Recovery 这里分为 `Leader-Alive` 和 `Leader-Change`。

* Leader-Alive 修复主要是解决某些 Follower 节点重启加入集群，或者是新增 Follower节点加入集群。
    * Leader需要向Follower节点传输漏掉的Log Entry，如果Follower需要的Log Entry已经在Leader上Log Compaction清除掉了，Leader需要将上一个Snapshot和其后的Log Entry传输给Follower节点。
    * Leader-Alive模式下，只要Leader将某一条Log Entry复制到多数节点上，Log Entry就转变为Committed。
    * Leader-Change修复主要是在保证Leader切换前后数据的一致性。
    * 通过上面Raft的选主可以看出，每次选举出来的Leader一定包含已经committed的数据（**抽屉原理**：选举出来的Leader是多数中数据最新的，一定包含已经在多数节点上commit的数据），新的Leader将会覆盖其他节点上不一致的数据。
    * 虽然新选举出来的Leader一定包括上一个Term的Leader已经Committed的Log Entry，但是*可能也包含上一个Term的Leader未Committed的Log Entry*。这部分Log Entry需要转变为Committed，相对比较麻烦，需要考虑Leader多次切换且未完成Log Recovery。解决方案如下：
    
    Raft中增加了一个约束：**对于之前Term的未Committed数据，修复到多数节点，且在新的Term下至少有一条新的Log Entry被复制或修复到多数节点之后，才能认为之前未Committed的Log Entry转为Committed**。下图就是一个Leader-Change场景下的Log Recovery过程：
    
<img src="http://7xqfqs.com1.z0.glb.clouddn.com/public/16-12-7/63638971.jpg" width="600px"/>

1. S1是Term2的Leader，将LogEntry部分复制到S1和S2的2号位置，然后Crash。2. S5被S3、S4和S5选为Term3的Leader，并只写入一条LogEntry到本地，然后Crash。3. S1被S1、S2和S3选为Term4的Leader，并将2号位置的数据修复到S3，达到多数；并在本地写入一条Log Entry，然后Crash。4. 这个时候2号位置的Log Entry虽然已经被复制到多数节点上，但是并不是Committed。这时候可能会产生两种情况：    * S5被S3、S4和S5选为Term5的Leader，将本地2号位置Term3写入的数据复制到其他节点，覆盖S1、S2、S3上Term2写入的数据    * S1被S1、S2、S3选为Term5的Leader，将3号位置Term4写入的数据复制到S2、S3，使得2号位置Term2写入的数据变为Committed
    
选出Leader之后，Leader运行过程中会进行副本的修复，这个时候只要多数副本数据完整就可以正常工作。Leader为每个Follower维护一个nextId，标示下一个要发送的logIndex。过程如图：

<img src="http://7xqfqs.com1.z0.glb.clouddn.com/public/16-12-7/9644050.jpg" width="600px"/>上图中Follower a与Leader数据都是一致的，只是有数据缺失，直接通知Leader从logIndex=5开始进行重传，只需一次回溯。Follower b与Leader有不一致性的数据，需要回溯7次才能找到需要进行重传的位置.
### 4.4	Membership Management

分布式系统运行过程中节点总是会存在故障报修，需要支持节点的动态增删。节点增删过程不能影响当前数据的复制，并能够自动对新节点进行数据修复，如果删除节点涉及Leader，还需要触发*自动选主*。直接增加节点可能会导致出现新老节点结合出现两个多数集合，造成冲突。

下图是3个节点的集群扩展到5个节点的集群，直接扩展可能会造成Server1和Server2构成老的多数集合，Server3、Server4和Server5构成新的多数集合，两者不相交从而可能导致决议冲突。 <img src="http://7xqfqs.com1.z0.glb.clouddn.com/public/16-12-7/36189950.jpg" width="600px"/>
 Raft采用的策略相对简单一些，每次只增删一个节点，这样就不会出现两个多数集合，不会造成决议冲突的情况。按照如下规则进行处理：
1.	Leader收到AddPeer/RemovePeer的时候就进行处理，而不是等到committed，这样马上就可以使用新的peer set进行复制AddPeer/RemovePeer请求。 2.	Leader启动的时候就发送AddPeer请求，防止上一轮AddPeer没有完成commit。 3.	Leader在删除自身节点的时候，会在RemovePeer被Committed之后，进行关闭。因为节点动态调整跟Leader选举是两个并行的过程，可以实现安全的动态节点增删。另外节点需要一些宽松的检查来保证选主和AppendEntries的多数集合：* 节点可以接受不是来自于自己Leader的AppendEntries请求* 节点可以为不属于自己节点列表中的Candidate投票为了避免同时有两个节点变更正在进行，在有未committed的change正在进行的时候，不允许进行节点变更。节点变更有一个问题，对一个只有两个节点的Cluster，发起RemovePeer。这个时候一个节点挂掉，另外一个节点没有收到RemovePeer请求，这样系统将停止工作。因此强烈建议集群节点数>=3个。

## 3. Multi Paxos

终于到了最后的 `Multi Paxos`，这篇文章断断续续写了也快3个月了。在实践中验证的分布式一致性算法包括，Paxos、ZAB、Multi Paxos，其实还有 `Viewstamped`，不过再这篇文章中将不会再补充，因为用的实在是太少了。Multi Paxos也一样，还没有有影响力的开源实现，据我说知，阿里的 OceanBase 和 Google 的不少产品都有用到。下图是一个总结，可以看到开源的软件大多数都使用 Zookeeper 作为满足一致性要求。

<img src="http://7xqfqs.com1.z0.glb.clouddn.com/16-12-13/96999766-file_1481634011735_c025.png" width="600px"/>

下面就介绍 Multi Paxos。我们向来分析下 Paxos 的缺点。

* Paxos 中每选中一个 value，都需要经过两轮 RPC (prepare, accept)。
* 当并发的 Proposer 数量增加的时候，冲突和重启数也会增加。

那么参照 Raft，我们可以用 leader-based 的方法来解决。这里的 multi 也就代表一次 prepare 可以进行多次 accept，从而提高性能。

## 5. Discussion

## Ref

[A Beginner's Guide to Paxos](https://docs.google.com/presentation/d/1y2lbLzSmZdd3OVzXpxAvmvWt4Hhn_WljQiU4x30Cst4/edit?usp=sharing)

[Raft user sutdy](https://ramcloud.stanford.edu/~ongaro/userstudy/)

[The Raft Consensus](https://raft.github.io/)

[Can’t we all just agree?](https://blog.acolyer.org/2015/03/01/cant-we-all-just-agree/)


