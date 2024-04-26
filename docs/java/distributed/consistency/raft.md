# [Raft](https://raft.github.io/)

**论文中文翻译**：https://github.com/maemual/raft-zh_cn/blob/master/raft-zh_cn.md。

**演示网站**：https://thesecretlivesofdata.com/raft/，配合官方的演示更容易理解。

**Raft协议实现**（Java，most star）：

官网列举了很多使用Raft协议的开源项目，但是大多并不是纯粹的Raft算法实现，这里了列举一些纯粹的算法实现项目。

+ [raft-java](https://github.com/wenweihu86/raft-java)

  Raft协议学习项目，适合学习使用。

+ [Apache Ratis](https://github.com/apache/ratis)

> 先看演示网站熟悉基本流程，然后结合论文、源码实现理解Raft的原理和细节。




## 工作原理

### 基本概念

+ 主要包含3个模块

  + **Leader选举**

  + **日志复制**

    领导人必须从客户端接收日志条目（log entries）然后复制到集群中的其他节点，并强制要求其他节点的日志和自己保持一致。

    日志条目即系统中的操作指令。类比MySQL中的redo log，MySQL保持多节点数据同步不也是通过复制日志么。

    复制的日志消息称为“Append Entry”。

  + **安全性**

    如果有任何的服务器节点已经应用了一个确定的日志条目到它的状态机中，那么其他服务器节点不能在同一个日志索引位置应用一个不同的指令。

+ 节点状态 State

  + **Follower** (节点初始状态)

  + **Candidate**

    候选人，可以参与Leader选举。

  + **Leader**

    经过选举后的主节点。

+ 节点中的持久性状态

  + **currentTerm**

    每个节点内会有一个参数 currentTerm，保存服务器已知最新的任期（在服务器首次启动时初始化为0，单调递增）。

    选举超时节点发起投票时会 + 1；Follower节点收到投票请求时如果currentTerm小于请求中的Term值会更新。 

  + **votedFor**

    即票投给了谁，如果没有投给任何候选人则为空，raft-java实现中由于使用 int 保存此数据，所以0即未投给任何人。

    代码中也通过这个值防止同一个任期内，节点重复投票，比如 raft-java votedFor != 0 说明当前任期已经投过票，不会再次投票。

  + **日志条目 log[]**

    每个条目包含了用于状态机的命令，以及领导人接收到该条目时的任期（初始索引为1）。

+ 节点中的易变状态

  + **commitIndex**	

    已知已提交的最高的日志条目的索引（初始值为0，单调递增）。

  + **lastApplied**

    已经被应用到状态机的最高的日志条目的索引（初始值为0，单调递增）。

+ Leader节点中的易变状态

  + **nextIndex[]**	

    对于每一台服务器，发送到该服务器的下一个日志条目的索引（初始值为领导人最后的日志条目的索引+1）。

  + **matchIndex[]**

    对于每一台服务器，已知的已经复制到该服务器的最高日志条目的索引（初始值为0，单调递增）。

+ **状态机**

  保存数据最新的值的容器。

  比如 raft-java 中的 ExampleStateMachine 本质是 RocksDB， 存储 KV, V是键为K的最新值。

  状态机与日志的关系：

  ![](https://github.com/maemual/raft-zh_cn/raw/master/images/raft-%E5%9B%BE1.png)

  > RocksDB是一个由Facebook开发的嵌入式键值存储系统。它是一个高性能、持久化的存储引擎，通常用于支持大规模的分布式系统。RocksDB基于Google的LevelDB项目，但在性能和功能上进行了一些改进。

+ **日志压缩（快照）**

  为了防止内存中日志文件过大，将旧的日志存储到快照中。

  所以  raft-java RaftNode 中有两个对象存储日志，snapshot + raftLog 才是节点完整的日志。

  raft-java 中通过RocksDB CheckPoint 实现快照，数据库的CheckPoint和[Raft快照](https://github.com/maemual/raft-zh_cn/blob/master/raft-zh_cn.md#7-%E6%97%A5%E5%BF%97%E5%8E%8B%E7%BC%A9)还是挺契合的。

  每个服务器独立地创建快照，只包括已经被提交的日志。

  ```java
  //存储新的日志
  private SegmentedLog raftLog;
  //存储旧的日志
  private Snapshot snapshot;
  ```

+ **日志匹配特性**

  + 如果在不同的日志中的两个条目拥有相同的索引和任期号，那么他们存储了相同的指令。

    > 第一个特性来自这样的一个事实，领导人最多在一个任期里在指定的一个日志索引位置创建一条日志条目，同时日志条目在日志中的位置也从来不会改变。

  + 如果在不同的日志中的两个条目拥有相同的索引和任期号，那么他们之前的所有日志条目也全部相同。

    > 第二个特性由附加日志 RPC 的一个简单的一致性检查所保证。**在发送附加日志 RPC 的时候，领导人会把新的日志条目前紧挨着的条目的索引位置和任期号包含在日志内。如果跟随者在它的日志中找不到包含相同索引位置和任期号的条目，那么他就会拒绝接收新的日志条目，在被跟随者拒绝之后，领导人就会减小 nextIndex 值并进行重试。**

+ **安全性**

  + **选举限制**

    Raft 使用投票的方式来阻止未包含所有已经提交日志条目的候选人赢得选举。

    这个很重要：**因为Leader可能通过日志复制覆盖Follower不一致的日志，确保了Leader包含所有已提交了的日志，就确保了不会覆盖掉已经提交了的日志**。

    > Raft协议的选举限制实现：Raft 通过比较两份日志中最后一条日志条目的索引值和任期号定义谁的日志比较新。
    > 如果两份日志最后的条目的任期号不同，那么任期号大的日志更加新。
    > 如果两份日志最后的条目任期号相同，那么日志比较长的那个就更加新。
    >
    > 比如 raft-java 中的实现：
    >
    > ```java
    > boolean logIsOk = request.getLastLogTerm() > raftNode.getLastLogTerm()
    > 	|| (request.getLastLogTerm() == raftNode.getLastLogTerm()
    > 		&& request.getLastLogIndex() >= raftNode.getRaftLog().getLastLogIndex());
    > ```

  + **提交之前任期内的日志条目**

    Raft规定Leader不会提交之前任期内的日志条目，只会提交当前任期内的日志条目，而提交当前任期内的日志条目会因为日志匹配特性间接提交之前任期内的日志条目，这样既实现了提交之前任期内的日志条目且不会产生论文图8中的漏洞。

    > 图8中展示的是如果Raft允许直接提交之前任期内的日志条目，则即使提交到多个节点，依然可能被覆盖，因为S5 Term=3 > 最新提交的日志 Term=2, S5是可能再次被选举为Leader的，然后通过日志复制就会覆盖最新提交的Term=2的日志。

### 工作流程

> 语言描述其实很容易落下很多细节，最好还是从Raft源码实现理解。

**主要流程**（简略）：

1. 系统初始化，各个节点刚初始化后都是Follwer状态；
2. 各个节点初始化后如果没有接收到Leader的消息就会认为当前没有Leader节点，会向配置表中的其他节点发送拉票信息，举荐自己为Leader;
3. 其他节点收到信息会投票，如果发起者收到多数的投票支持，就会称为Leader节点，所有会修改系统的请求都会经过Leader;
4. 每一条修改请求都会添加到Leader节点的记录中；但是还未提交，不会真地去修改数据；
5. 要提交修改数据，需要先将修改数据复制到其他所有Follower节点，并等待其他节点响应，当有多数节点响应已收到修改数据，主节点才会提交修改，将主节点的数据修改为目标值；
6. Leader提交修改后，需要向所有Follower节点同步提交要求，Follower节点收到同步提交请求后，保存自身的修改数据，从而达成一致；这个过程也称为“日志复制 Log Replication”；

> 实际的情况往往比上面说的复杂的多，可能多个节点同时发起拉票，可能存在部分节点通信故障等等。

**Leader选举**（详细）：

系统多个节点启动有先有后，有快有慢，可能出现第一个节点启动后发送拉票信息，其他节点还没完成启动无法响应等等情况；

Raft协议为了处理一些特殊情况，设置了两个超时时间：

+ 选举超时 Election Timeout

  选举超时是Follower在修改为Candidate状态之前等待的时间，选举超时随机设置为150毫秒到300毫秒之间。

+ 心跳超时 Heartbeat Timeout

  节点当选Leader后每个心跳周期内向其他节点发送一次自己当选Leader的消息，维持自己在其他节点的Leader地位。

这里再详细过一遍上面步骤1-3：

1. 系统初始化，各个节点刚初始化后都是Follwer状态, Term=0；然后随机等待 150ms - 300ms（推荐）;

   > 每个节点的选举超时时间每次都是随机生成的，为了尽量避免多个节点同时超时导致选票被瓜分的情况，有利于更快地选举出Leader。

2. 系统中总有一个节点最先选举超时，然后修改自己的状态为Candidate，Term=Term+1=1, 先给自己投1票，然后向节点配置表中的其他节点发起拉票请求；

   此时，其他节点的状态可能是：尚未启动、Follower状态、刚切换Candidate状态（虽然慢了一点点仍会向其他节点发起拉票请求，网络环境是复杂的，还不一定谁先获得多次支持呢）；

3. 尚未启动的节点无响应；已经启动的Follower节点接收到第一个拉票请求，检查请求中Term > 本地Term，Term更新为请求中的Term值，投赞成票并重置选举超时时间，如果节点在本轮选举中后续又接收到另一个节点拉票请求也不再投票；Candidate节点收到其他Candidate节点拉票请求，投反对票。

   > 注意：每轮选举（Term）每个节点只能投一次赞成票。

4. Candidate节点统计本轮选举得票数，可能由于多数节点还未启动完成等原因，直到超时也没有收到多数支持票；Term=Term+1，重置超时时间，再次向其他节点发起拉票请求；

   > 注意：可能Candidate选举超时重新发起拉票前，Follower先选举超时了，这是Follower会变成Candidate，向其他节点发起拉票请求。

5. 直到有一个节点收到多数支持票，向其他节点发布自己当选Leader的消息（Append Entries 消息），按心跳超时指定的时间间隔发送；其他节点记录Leader节点信息；

   > 疑问：节点收到多数支持票，向所有其他节点发送自己当选Leader的消息，还是只向自己的支持者Follwer发送消息？所有。
   >
   > 注意：当选后每个心跳周期内发送一次，维持自己在其他节点的Leader地位。

6. 一旦心跳超时，Follower 会知道 Leader 异常，会重新变为 Candidate 状态，参与下轮选举。

   > 疑问：异常的Leader会从节点列表中除名么？

**日志复制 Log Replication**：

对应主要流程的步骤：4-6。

**网络分区（脑裂）恢复后仍可保持一致性的原理**：

脑裂后，多数节点所在分区如果有Leader会正常继续运行，没有Leader会重新选举一个Leader然后继续运行；少数节点分区所有更新操作都无法完成；分区再次连接后（比如修复了网络故障），如果少数节点分区有Leader, Leader会从发送给其他节点的心跳响应中看到更高的任期Term，从而选择退位，然后回滚未提交的修改，并从新Leader节点同步最新数据。

### 源码实现流程图

#### [raft-java](https://github.com/wenweihu86/raft-java)

**源码debug启动**：

代码配置的JDK版本是1.7, 本地没有安装1.7修改为1.8，另外如果编译使用的JDK版本大于8，可能报编译错误：

```
Failed to execute goal org.apache.maven.plugins:maven-compiler-plugin:3.6.1:compile (default-compile) on project raft-java-core: Fatal error compiling: java.lang.ExceptionInInitializerError: Unable to make field private com.sun.tools.javac.processing.JavacProcessingEnvironment$DiscoveredProcessors com.sun.tools.javac.processing.JavacProcessingEnvironment.discoveredProcs accessible: module jdk.compiler does not "opens com.sun.tools.javac.processing" to unnamed module
```

是由于 Java 9+ 引入的模块化系统（Module System）导致的。在 Java 9 之后，不再允许直接反射访问模块中没有 opens 到所有模块的包。

**疑问**：

1. 论文5.3章节提到：“一旦一个领导人被选举出来，他就开始为客户端提供服务。客户端的每一个请求都包含一条被复制状态机执行的指令。领导人把这条指令作为一条新的日志条目附加到日志中去，然后并行地发起附加条目 RPCs 给其他的服务器，让他们复制这条日志条目。当这条日志条目被安全地复制（下面会介绍），领导人会应用这条日志条目到它的状态机中然后把执行的结果返回给客户端。**如果跟随者崩溃或者运行缓慢，再或者网络丢包，领导人会不断的重复尝试附加日志条目 RPCs （尽管已经回复了客户端）直到所有的跟随者都最终存储了所有的日志条目**。”

   但是看raft-java源码并没有重试操作。

2. 为何不包含已提交日志的节点不会被选举为Leader?

