---
title: 6.824 Lab2通关指南
date: 2023-02-02T11:25:22Z
lastmod: 2023-02-27T11:53:17Z
categories: [Distributed System,MIT 6.824]
---

# Raft

## 谈一谈对raft的理解？

> Raft是一种共识算法，设计得很容易理解。它在容错和性能方面与Paxos相当。不同的是，它被分解成相对独立的子问题，并干净地解决
>
> 了实际系统所需的所有主要部分。我们希望Raft能让更多的人了解共识，而这些人将能够开发出比现在更高质量的基于共识的系统

raft最重要的一个特点就是其通过leader来保证强一致性，client只和leader进行通信，发送给follower的消息也会被转发给leader，最终的决策也是由leader进行的，如何时进行是否写入某条日志，何时将日志进行commit，从而实现了强一致性，而这样的问题就是并没有办法像Zookeeper那样通过分布式的方式来实现性能的提升，尤其是IO吞吐量的提升。

并且由于需要进行leader和follower之间的协调问题，因此在性能上会有额外的开销，但是通过额外开销提供了容错性，在有2n + 1个节点的情况下，允许有 n个节点宕机

raft本身有三个核心：leader Election Log Replication Log Compation 。raft在日志层级上形成分布式共识，即leader和follower之间由leader写入一份所有节点都认同的日志，之后节点之间通过日志来保证数据的一致性，整个集群（其实就是leader进行决策）认为可以commit时，就讲消息上传至应用层，之后应用层就可以将该操作或者数据进行写入。

## Raft是如何进行leader 选举的

raft的选举简单来说就是一个投票达到大多数的过程，即只要有一个节点能够拿到超过1 / 2 的票（包括自己投自己），那么他就可以成为leader

而选举的触发和结束是靠心跳和超时时间来完成的，所有节点的初始状态全为follower，当超时之后，节点便会变成candidate，之后candidate向其他的所有节点发送RPC，请求其他节点为自己投票，如果在这一轮内有一个candidate能拿到超过1 / 2的选票，之后该节点就会转换为leader，之后就会想其他节点发送心跳信息（AppendEntries RPC），重置其他节点的超时时间，阻止其超时进行选举，也就是说，只要一个leader没有宕机，那么他可以一直维持自己的任期。而为了保证所有的不会在同一时间进行选举，从而导致谁也无法拿到大多数选票的情况，超时时间设置为一个随机的时间，通常在150ms-300ms之间

由于leader会决定整个集群的状态，因此如果随意进行选举会产生安全问题，例如一个落后的节点成为了leader，则会导致其他已经commit了的日志丢失。因此主要有两个限制条件：

* `condition1 := args.LastLogTerm > rf.getLastTerm()`​

* `condition2 := args.LastLogTerm == rf.getLastTerm() && args.LastLogIndex >= rf.getLastIndex()`​

条件一保证不会有落后的节点成为leader，条件二保证即使是同一任期内candidate的日志也为最新的

## Raft如何进行日志复制

对于日志复制，首先需要明确的一个概念是Raft的日志并不像如Bitcask那样通过日志来提供存储功能，Raft的日志并不提供任何的存储功能，只是作为数据同步来使用，通过Raft完成了数据同步之后，就可以将日志进行"commit"，将其提供给状态机（state machine)，此时由上层的状态机（存储层）来完成具体的日志存储的工作。

就像之前所说的Raft的一切操作都由Leader来进行安排，因此日志选举的整个过程也是先从Leader开始，当Leader收到了来自上层应用的一个日志写入请求，此时生成一个对应的日志条目（Log Entry)，此时便在Leader节点上写入了一个日志条目，后续再将该日志条目同步给其他节点。

Leader从上层接收到的为一条命令，如：`set x = 5`​，并且同时记录Leader当前的任期以及该日志的顺序或者位置，还需一个Index，因此一个日志条目一共包含三个信息，分别是`Command`​ `logIndex`​ `logTerm`​，可视化的结果如下：

![](https://pic-bed-1309931445.cos.ap-nanjing.myqcloud.com/blog/image-20230222003246-emx5d63.png)

**复制流程**​

当Leader产生了一个新的日志之后，Leader就会立刻尝试通过AppendEntry RPC来将日志发送给Follower来进行日志的复制，而follower如果认为该日志条目并不存在冲突问题，即将该日志条目进行写入操作，追加到自己的日志当中，给Leader以写入成功的返回，而当Leader接收到了半数以上的follower成功将日志进行写入，此时Leader就可以认为该日志条目为"commit"的状态，因此Leader就可以将其应用到状态机上，即向上告诉应用层该条命令已经执行成功，可以将其真正持久化下来，并且可以给客户端写入成功的响应。即只要"commit"了的日志，都可以认为其成功写入。

**事与愿违**​

但是通常情况下事与愿违，日志复制的过程当中很难保证不存在节点宕机或者网络分区等错误，因此此时就需要Raft来提供容错能力（Fault Tolerance)，来保证日志的正确写入。会产生额外的未提交的日志、日志遗失等问题：

![](https://pic-bed-1309931445.cos.ap-nanjing.myqcloud.com/blog/image-20230222005445-gmhkmkw.png)

之前在领导选举中已经讨论过，如果整个集群中不存在网络分区的情况下，通过Raft领导者选举策略选举出来的领导一定是能够正确代表整个集群的状态的，即在日志上一定领先于Follower，因此在日志上主要解决两个问题，一是当前的Leader正确，Leader发送了最新的日志，但是Follower自身落后于Leader，另一种情况是出现了网络分区，网络分区恢复之后那个落后的Leader还在尝试向其他的follower发送日志（follower已经领先于这个错误的Leader）：

* 当Leader存在问题时，如果一个Follower发现自己的任期大于心跳中携带的Leader的任期，直接拒绝日志的写入，return掉
* 而对于Follower落后的情况，在心跳中额外携带一个`PrevLogIndex`​，如果Follower当前的最后一个日志的`Index`​仍小于`PrevLogIndex`​，则证明Follower存在落后的情况，此时拒绝写入，请求Leader发送一个更早的日志来尝试写入，以此消除掉日志上差距，此处最简单方式就是Leader回退一格，发送上一条日志，通过一次次尝试最终可以将进度对齐

**日志同步优化**​

一次回退一格日志显然会带来大量的RPC开销，因此可以通过跳跃试探的方式来减少RPC的发送次数。

Follower向Leader返回冲突时额外携带三个信息：

> ### Fast Backup
>
> AppendEntry添加额外的返回信息：
>
> * XTerm:冲突的term号
> * XIndex：XTerm的第一个Index
> * Xlen：follower的日志的长度
>
> **case**：
>
> |s1|4|5|5||
> | --| -| -| -| -|
> |s2|4|6|6|6|
>
> |s1|4|4|4||
> | --| -| -| -| -|
> |s2|4|6|6|6|
>
> |s1|4|<br />|||
> | --| -| -| -| -|
> |s2|4|6|6|6|
>
> * 对于第一个case：s2第一次发送AppendEntry后接收到的XTerm为5，Index为2（index从1开始），leader当中并没有term = 5，因此后续的AppendEntry可以直接跳转到Index = 2开始发送
> * 对于第二个case：leader当中含有Xterm，因此leader就从Xterm的最后一个Entry开始进行同步
> * 对于第三个case：follower当中不存在leader的entry，通过Xlen来找到最后一个Entry进行同步

## 持久化

持久化为raft提供了宕机恢复的功能，总的来说用于保存raft节点的当前状态和日志，而在没有日志压缩的情况下，只需要保存`votedFor`​ `currentTerm`​ `log`​三个变量即可:

* `commitIndex`​ 和`lastApplied`​ `nextIndex[]`​ `matchIndex[]`​ 可以通过一次日志发送达成共识之后重新恢复
* `status`​如果节点原本为follower只需再接受一次心跳信息即可重新变为follower，而如果为leader则只需要再进行一次超时选举选出新的leader即可

`votedFor`​ `currentTerm`​ 两个变量即可保证节点在恢复之后能有正常的节点选举能力并且能够正常的接受日志信息，之后所有的数据都可以用过选举和日志同步完成恢复

再加入了日志压缩之后，防止重启之后访问到被压缩的日志，因此需要将`lastIncludedIndex`​ `lastIncludedTerm`​进行持久化，`commitIndex`​ `lastApplied`​也通过`lastIncludedIndex`​ 和`lastIncludedTerm`​进行初步初始化，防止`index out of range`​。

## 日志压缩

Raft通过日志压缩主要解决两个问题：

* 大量日志占用存储空间
* 落后较多的节点需要执行大量的日志才能跟上Leader的进度

Raft采用的日志压缩的方法为拍快照的方式，即上层存储层获取一个当前状态的快照，保存当前的各个KV的值，因此Raft层只需将该快照进行持久化，并且可以删除之前所有的日志，后续如果有落后的Follower需要进行日志复制，只需要将该快照发送给他，即可将Follower的状态直接同步到和日志相同的状态。

![](https://pic-bed-1309931445.cos.ap-nanjing.myqcloud.com/blog/image-20230226113754-8n8rdhd.png)

#### 快照创建

整个日志压缩的过程是由存储层（service）发起的，即存储层将当前的状态压缩成一个字节数组，和对应的一个index，raft节点就将index对应的日志删除，并且将快照进行持久化，并且处理由于删除日志和创建快照带来的索引偏移问题

之前的日志为了方便获取prevLogIndex prevLogTerm，创建了一个dummyhead，因此日志压缩后依旧可以创建一个dummyhead，不过里面可以存储`lastIncludedIndex`​和`lastIncludedTerm`​，这样如果要获取之前的最后一个，可以直接通过索引0找到（其实都差不多）。

上层传来的快照一定是commit并且应用了的，因此可以通过index更新一下`commitIndex`​和`lastApplied`​

#### 日志发送

日志发送是通过心跳`appendEntries`​一同发送的，当发现一个follower落后太多（即要给follower发送的日志已经被压缩成快照）此时不再发送日志，而是选择直接发送快照给follower

调用follower的`IntallSnapshotRPC`​，follower删除快照包含的所有的日志，保留快照之后的日志，并更新对应的`lastIncludedIndex`​ `lastIncludedTerm`​ `commitIndex`​ `lastApplied`​，最后持久化快照

![](https://pic-bed-1309931445.cos.ap-nanjing.myqcloud.com/blog/image-20230226151316-oj977vr.png)

## 日志状态

引入日志压缩之后会存在Index的偏移量问题，如果不细心处理会出现大量的`index out of range`​的问题

由于通过快照来进行日志压缩，会删除掉之前的日志，因此需要引入两个量`lastIncludedIndex`​和`lastIncludedTerm`​来记录当前日志的状态，即被删除的最后的一个日志条目的`Index`​和`Term`​，因此在进行日志压缩之后，真实的Index和逻辑上的Index出现了偏差，需要通过LastIndexdIndex进行偏移，来获取到真实的索引量，而如果需要获取的索引量如果小于`lastIncludedIndex`​，则证明该日志条目已经被日志压缩进行删除，在进行日志同步时则直接发送快照进行同步

因此需要提供对应的偏移形式的计算`lastIndex`​ `lastTerm`​ `prevIndex`​ `prevTerm`​的函数

与日志相关的还有`commitIndex`​ 和`lastApplied`​  ：

* `lastApplied`​表示最后进行应用（向上层通知正式写入）
* `commitIndex`​表示Leader认为应当写入的（commit但还未写入）

因此如果二者之间存在差值，即`commitIndex > lastApplied`​则证明有还未写入的日志，需要写入并通知上层。

这两个变量无需进行持久化，如果节点发生宕机之后leader只需要再发送一次日志并且达成共识即可获取到新的commitIndex，此时就会重新上层提交一遍，最终恢复到宕机前的状态，而使用日志压缩之后，宕机刚恢复的lastApplied为0，此时通过发送日志达成共识获取了一个新的commitIndex，此时开始通过applier进行日志的应用，会从0开始应用，最终触发`index out of range`​。

因此在节点进行恢复读取持久化的数据时，需要对`commitIndex`​和`lastApplied`​进行更新，防止去读取已经被压缩的日志。

```go
	rf.readPersist(persister.ReadRaftState())
	if rf.lastIncludedIndex > 0 {
		rf.lastApplied = rf.lastIncludedIndex
		rf.commitIndex = rf.lastIncludedIndex
	}
```

至此，由于日志压缩引起的`index of out range`​​已经全部解决

## 安全性保证

主要讨论一下已经commit了的日志如何在leader重新选举时而不丢失的问题

先列一下之前讨论过的Leader约束：

```go
condition1 := args.LastLogTerm > rf.getLastTerm()
condition2 := args.LastLogTerm == rf.getLastTerm() && args.LastLogIndex >= rf.getLastIndex()
```

raft最基本即为只有大多数（超过 1/2）赞同时才会取得一个共识，从而可以保证至少有一个节点跨过了两个Leader的任期，从而可以消除两个任期之间的冲突。

假设有5个节点，因此只有获取了三个投票才能成为新的leader

* 在Term0时，假设 s1 s2 s3达成了共识，并且s1为Leader。将日志复制给s2 s3。s4 s5因为网络分区没能和s1 达成共识，成为s1 的follower
* 之后Leader s1发生宕机，并且s4 s5的网络分区问题得以解决，此时重新选出一个Leader，s4 s5由于版本落后无法成为Leader，因此最终只能是s2 或者s3成为Leader，
* 因此此时的Leader就可以保证可以跨过两个Term，之前commit了的日志消息就可以得以保留

![](https://pic-bed-1309931445.cos.ap-nanjing.myqcloud.com/blog/image-20230226193630-9j39yo1.png)

## 集群成员迁移

考虑如何修改raft集群的配置，这一部分在 6.824的Lab当中并未实现，即对一部分节点进行停机或者添加新的节点。因此就会涉及到两份配置文件，一份配置文件中有五个节点，另一份配置文件中可能会有六个节点。

因此如果不停机就直接向其中添加节点，并对新的节点使用新的配置文件（新节点指的是存在于新的配置文件中的节点）一部分节点会认为当前有5个节点，拿到3票即可成为Leader，而另一部分认为有6个节点，拿到4票才会成为Leader，因此在节点添加进去之后，最终经过投票可能会产生两个Leader，出现脑裂的问题。

Raft所给出的解决方案为两阶段方法，Raft首先切换到一个过渡态的配置（joint consensus)，一旦过渡态的配置被提交了，集群在开始向新配置切换。

过渡态配置可以视为新旧配置的结合：

* 日志会被复制到所有存在于新旧配置中的节点上
* 新旧配置的节点都有可能成为Leader
* 对于共识的达成（Leader选举和日志的commit），需要旧配置和新配置的两个集群分别达成单独的共识

### 迁移过程

1. 最初为$C_{old}$单独起作用，之后Leader收到了配置迁移的请求，Leader便开始创建过渡态的配置文件$C_{old,new}$，该配置文件作为一个日志条目进行存储并且复制给其他的节点，一个节点只要将该配置文件添加到日志当中，则会使用该配置进行决策，即Leader创建之后就会以该配置文件来决定该配置文件是否commit。
2. 如果$C_{old,new}$ commit了，则$C_{new} \ C_{old}$都无法单独起作用，需要两个集群单独决定后再取结果。而$C_{old,new}$commit 之后，之前讨论的**Leader Completeness Property**可以保证再次选举的Leader一定包含 $C_{old,new}$​
3. 当$C_{old,new}$ commit之后，整个集群就可以进行下一阶段，Leader再创建$C_{new}$，并且以相同的方式复制给其他节点，最终再commit
4. 当$C_{new}$ commit之后则可以视为整个集群进入了新的配置当中，此时旧配置会被认为是过时的并且不在新配置当中的节点也应当被关闭

最终保证再整个配置迁移的过程当中，不存在新配置和旧配置能够单独起作用的时间，从而避免了脑裂问题

![](https://pic-bed-1309931445.cos.ap-nanjing.myqcloud.com/blog/image-20230226204203-jxm0iwk.png)

### 细节问题

* 当一个新节点需要加入到集群当中，如果日志较为多的话，需要话一段时间才能追上原本的集群，因此采用的解决方案为新的节点作为一个不投票的成员，它只会接受Leader向其发送日志，自己默默补齐进度
* 如果新的配置当中不包括当前的Leader，则Leader会在新的配置$C_{new}$ commit时转换为 follower，而像之前所说，Leader在创建了$C_{new}$之后就会按照$C_{new}$进行决策，因此创建到最终commit的一段时间内，Leader在管理着一个自己不是大多数的集群
* 此外已经被移除的节点可能还会影响到集群，被移除的节点发现自己超时之后还会去请求投票（新的Term），可能会导致当前的Leader又变回follower ,对应的解决方案为如果服务器在听取当前领导者的最小选举超时内接收到RequestVote RPC，则它不更新其期限或授予其投票。

## 实现

扯了一大堆的理论，再讲讲怎么怎么实现

### Lab2A

感觉按照执行流程顺起来会比较方便。

首先看一下Raft节点的结构，我在实现上基本遵守了论文当中的定义。与选举有关的主要有currentTerm votedFor status以及两个超时时间。

```go
type Raft struct {
	mu        sync.Mutex          // Lock to protect shared access to this peer's state
	peers     []*labrpc.ClientEnd // RPC end points of all peers
	persister *Persister          // Object to hold this peer's persisted state
	me        int                 // this peer's index into peers[]
	dead      int32               // set by Kill()

	// Your data here (2A, 2B, 2C).
	// Look at the paper's Figure 2 for a description of what
	// state a Raft server must maintain.
	currentTerm int
	votedFor    int
	log         Log

	// volatile state on all servers
	commitIndex int
	lastApplied int
	// volatile state on leaders
	nextIndex  []int
	matchIndex []int

	status                Status
	heartbeatIntervalTime time.Duration
	electionTime          time.Time
	applyChan             chan ApplyMsg

	applierCond *sync.Cond

	lastIncludedIndex int
	lastIncludedTerm  int

	snapshot []byte
}
```

整个程序的入口即为ticker，通过ticker来触发AppendEntries和RequestVote。而对于领导者选举来说，即为通过ticker来触发election timeout。在ticker的实现上我只定义了一个ticker，他会分别以50ms的频率发送AppendEntries和以150-300ms的频率去发送RequestVote，这种实现方式其实并不是很严格，即超时时间为210ms和250ms的在第250ms是才去发送RequestVote，这样就导致容易产生冲突，各拿到一半的票数，最终导致选举不出Leader节点，这种情况在AppendEntries的情况下尤为明显，因此最终将其设置为了50ms。但是这种实现方式下确实也能够稳定通过当前的所有测试，带来的影响就是会多发送几次RPC，对后续的Lab3 4也没有影响，姑且就先这样了。

较为严格的实现方式应当为单独再设置一个ticker，以10ms-30ms去检测是否超时，如果超时则发送RequestVote。或者每次休眠一个150-300ms的时间，一旦不休眠就是要发送RequestVote。不过如果随机睡眠的话后续还要去考虑被重置的情况，实现起来相对来说较为复杂一点，当苏醒之后，先判断被重置之后的时间是否到了，如果没到，则继续休眠。

```go
func (rf *Raft) ticker() {
	for rf.killed() == false {
		time.Sleep(rf.heartbeatIntervalTime)
		rf.mu.Lock()
		if rf.status == leader {
			rf.appendEntries(true)
		}
		if time.Now().After(rf.electionTime) {
			rf.startElection()
		}
		rf.mu.Unlock()
	}
}
```

election timeout超时之后，触发选举，将自己的status转换为candidate，为自己投票，term++，之后再尝试向其他节点获取投票

```go
func (rf *Raft) startElection() {
	// rules for candidates:rule1
	rf.status = candidate
	rf.currentTerm++
	rf.votedFor = rf.me
	rf.persist()
	rf.setRandomElectionTimeout(rf.me)
	Debug(dInfo, "S%d start election in term %d", rf.me, rf.currentTerm)
	voteCounter := 1
	args := RequestVoteArgs{
		Term:         rf.currentTerm,
		CandidateId:  rf.me,
		LastLogIndex: rf.getLastIndex(), // not necessary for 2A
		LastLogTerm:  rf.getLastTerm(),
	}
	var becomeLeader sync.Once
	for peer := range rf.peers {
		if peer == rf.me {
			continue
		}
		go func(peer int) {
			rf.candidateSendRequestVote(peer, &args, &voteCounter, &becomeLeader)
		}(peer)
	}

}
```

发送完RPC之后则对响应进行统计，如果超出的半数以上的节点（算上自己），那么就成为Leader，需要通过一个标志位或者sync.Once来保证称为Leader的动作只执行一次。成为Leader之后立刻对nextIndex进行初始化，然后发送appendEntires。

对于term如果接受到的Term大于自身，那么证明自己已经落后，直接中止选举过程即可。

```go
	if (*voteCounter > len(rf.peers)/2) && rf.currentTerm == args.Term && rf.status == candidate {
		becomeLeader.Do(func() {
			Debug(dLeader, "S%d become a leader", rf.me)
			rf.status = leader
			lastLogIndex := rf.getLastIndex()
			for i, _ := range rf.peers {
				// rules for state :nextIndex[] matchIndex[]
				rf.nextIndex[i] = lastLogIndex + 1
				rf.matchIndex[i] = 0
			}
			rf.matchIndex[rf.me] = lastLogIndex
			rf.appendEntries(true)
		})
```

至此，选举过程结束。对于2A还需要实现AppendEntries来发送心跳信息来阻止其他的节点成为Leader，不断的去重制其定时器。

接收端

接收端主要还是按照描figure2描述的对Term做一下限制即可，如果自己的term小，则跟随新term并投票，自己的term大则拒绝投票，并且需要保证当前周期并没有投过票或者投过相同的候选人，可以重新在对其投一票（针对于超时重传的问题）。

#### 心跳信息

在Lab2A当中AppendEntries只负责心跳的作用，并不需要携带任何的日志，每次都发送一个空日志即可，然后如果接收端的term更新则将自己置为follower。而在接收端同样是跟随新的term，拒绝旧的term，并且维持follower和重置election timeout即可。

至此2A基本上能够通过2A的三个测试了。总的来说光实现选举的逻辑并不复杂，主要是需要设计一个较好的ticker方案，在Lab当中也提示并不推荐使用go内置的timer和ticker，虽然逻辑上没有问题，但是很容易写出bug，固定的ticker去检测虽然理论上存在一定的缺陷，但是正确性确实可以得到保证，无非就是多发几次RPC，但是对于代码功底的要求会低不少。

### Lab2B

2B可以说是整个Lab2的难点了，2B如果能够写好，后续的2C 2D会轻松很多。

2B主要是实现日志同步，大致分为三部分：

- AppendEntries当中发送时携带日志发送，接受时探测差异，向前发送。
- RequestVote当中保证Leader Election选举的安全性，防止日志的覆盖。
- 实现Start()从上层接受命令生成日志，通过applier协程将提交了日志传递到上层的state machine

#### AppendEntries

在AppendEntries发送成功后对nextIndex还有matchIndex进行更新，而发送失败后则通过需要解决日志Term冲突的问题，重新发送日志，最简单的方法为不断往前递减直至找到不存在冲突的Term，但是会发送大量的RPC来进行探测，造成大量的网络开销，并且速度也不理想，因此可以按照课上所讲的Fast Backup的方式进行探测。

```go
		if reply.Success {
			// 发送成功则为发送的所有日志条目全部匹配，match直接加上len(Entries)
			// next 永远等于match + 1
			match := args.PrevLogIndex + len(args.LogEntries)
			next := match + 1
			rf.commitIndex = rf.lastIncludedIndex
			rf.matchIndex[server] = match
			rf.nextIndex[server] = next
			Debug(dLog2, "S%d update server is %d nextIndex:%d lastIncludedIndex is:%d preLogIndex is %d lenLog is %d", rf.me, server, next, rf.lastIncludedIndex, args.PrevLogIndex, len(args.LogEntries))

		} else if reply.Conflict {
			// 如果存在冲突则通过XTerm XLen XIndex进行快速定位
			// XTerm == -1则证明 follower当中日志过短，不存在和发送的同term的entry，直接按长度从最末尾进行重写
			if reply.XTerm == -1 {
				rf.nextIndex[server] = reply.XLen
			} else {
				// xIndex为0则证明 follower当中没找到，即只有一个term的情况,需要leader进行定位
				// follower : 4 4 4
				// leader : 4 4 5 5 5
				if reply.XIndex == 0 {
					last := rf.findLastLogByTerm(reply.XTerm)
					rf.nextIndex[server] = last
				} else {
					// XIndex找到了则直接使用XIndex的即可
					// 也有可能是follower的日志已经快照化，让leader重新探查
					rf.nextIndex[server] = reply.XIndex
				}

			}

		}
```

最后Leader再尝试一次commit，是否有index > commitIndex（即还未commit），但是又有半数以上的节点macthIndex >= index（超过1/2的节点完成了该日志的写入），此时即可认为该日志已经commit，条件在论文当中也已经给出（rules for leaders：rule4）

```go
	for n := rf.commitIndex + 1; n <= rf.getLastIndex(); n++ {
		if rf.restoreLogTerm(n) != rf.currentTerm {
			continue
		}
		counter := 1
		for peer := range rf.peers {
			// if leader and 1/2 follower match this log,commit it
			if peer != rf.me && rf.matchIndex[peer] >= n {
				counter++
			}
			if counter > len(rf.peers)/2 {
				rf.commitIndex = n
				Debug(dCommit, "S%d leader commit:commitIndex,%d totalLog:%s", rf.me, rf.commitIndex, rf.log.String())
				rf.wakeApplier()
				break
			}

		}
	}
```

接收端

在接收端主要根据prevLogIndex prevLogTerm来判断是否存在冲突，如果存在，则通过fast backup的方式找到存在冲突的前一条日志。

如果prevLog不存在冲突时，则删除自身之后所有的日志，和Leader强制达成同步。

```go
	for i, entry := range args.LogEntries {
		// figure 2 AppendEntries RPC Receiver implementation 3
		// delete conflict entry
		idx := entry.Index
		if idx <= rf.getLastIndex() && rf.restoreLogTerm(idx) != entry.Term {
			rf.log.Entries = rf.log.Entries[:idx-rf.lastIncludedIndex]
			rf.persist()
		}
		// figure 2 AppendEntries RPC Receiver implementation 4
		// save entry from leader
		if entry.Index > rf.getLastIndex() {
			// append all entries after idx
			Debug(dLog, "S%d leader %d append entries to %d,entries is%v", rf.me, args.LeaderId, rf.me, args.LogEntries)
			rf.log.append(args.LogEntries[i:]...)
			rf.persist()
			break
		}
	}
```

fast backup

```go
	if rf.restoreLogTerm(args.PrevLogIndex) != args.PrevLogTerm {
		reply.Success = false
		reply.Conflict = true
		// set XTerm XLen XIndex
		reply.XTerm = rf.restoreLogTerm(args.PrevLogIndex)
		for idx := args.PrevLogIndex; idx > rf.lastIncludedIndex; idx-- {
			// the first index of the conflict term
			if rf.restoreLogTerm(idx-1) != reply.XTerm {
				reply.XIndex = idx
				break
			}

		}
		reply.XLen = len(rf.log.Entries)
		Debug(dLog, "S%d Conflict Xterm %d, XIndex %d,XLen %d", rf.me, reply.XTerm, reply.XIndex, reply.XLen)
		return
	}
```

最后，还需要判断一下自身是否需要commit，由于自身已经和Leader达成的日志的同步，因此在commitIndex上也需要和Leader达成同步，将日志commit，并之后应用于状态机

```go
	// figure 2 AppendEntries RPC Receiver implementation 5
	if args.LeaderCommit > rf.commitIndex {
		rf.commitIndex = min(args.LeaderCommit, rf.getLastIndex())
		rf.wakeApplier()
	}
```

#### RequestVote

在日志部分RequestVote所承担的相比于AppendEntires就要少很多了，只需要完成论文当中对于安全性的保证即可，即如果candidate在日志上落后于自己那么就不能为其投票，

表现为在Term上落后和同一Term但是Index上落后

```go
condition1 := args.LastLogTerm > rf.getLastTerm()
condition2 := args.LastLogTerm == rf.getLastTerm() && args.LastLogIndex >= rf.getLastIndex()
```

#### 状态机

在Lab2B当中需要和上层的状态机进行通信，从上层接受命令，并将其创建给日志，立刻发送一次心跳尝试将其同步到follower上

交互上可以参考这张图

![](https://pic-bed-1309931445.cos.ap-nanjing.myqcloud.com/blog/image-20230303113035-xokhls8.png)

```go
func (rf *Raft) Start(command interface{}) (int, int, bool) {
	index := -1
	term := rf.currentTerm
	rf.mu.Lock()
	defer rf.mu.Unlock()
	if rf.status != leader {
		return index, rf.currentTerm, false
	}
	index = rf.getLastIndex() + 1
	newLogEntry := Entry{
		Command: command,
		Term:    rf.currentTerm,
		Index:   index,
	}
	rf.log.append(newLogEntry)
	rf.persist()
	rf.appendEntries(false)
	Debug(dClient, "S%d receive a command from client", rf.me)
	return index, term, true
}
```

而对于日志的提交，变量上主要有commitIndex和lastApplied，commitIndex代表当前raft达成共识的能够提交的日志，lastApplied代表传递给上层状态机的日志的Index。如果二者之间存在差值那么则代表需要将提交了日志向上层传递。

在Lab的学生指南当中提倡使用一个单独的applier协程去检测是否需要提交，如果需要则通过channel将消息传递给上层，此外，为了防止该协程一直轮询占用CPU的资源，可以采用条件变量或者sleep的方式，这里我才用的为条件变量的方式来实现，如果没有需要应用的日志则wait，等待唤醒，而唤醒主要有两个位置：

- 对于Leader节点，在LeaderTryToCommit当中判断出了超出1/2达成commit的时候就应当一并通知applier苏醒，进行commit
- 而对于Follower节点，当通过AppendEntries完成和Leader的日志同步时，如果发现自己之前的commit进度落后于Leader，此时就需要唤醒applier写成，将commit的日志进行applied。

```go
	if counter > len(rf.peers)/2 {
		rf.commitIndex = n
		Debug(dCommit, "S%d leader commit:commitIndex,%d totalLog:%s", rf.me, rf.commitIndex, rf.log.String())
		rf.wakeApplier()
		break
	}
	// figure 2 AppendEntries RPC Receiver implementation 5
	if args.LeaderCommit > rf.commitIndex {
		rf.commitIndex = min(args.LeaderCommit, rf.getLastIndex())
		rf.wakeApplier()
	}
```

applier

```go
func (rf *Raft) applier() {
	rf.mu.Lock()
	defer rf.mu.Unlock()

	for !rf.killed() {

		if rf.commitIndex > rf.lastApplied && rf.getLastIndex() > rf.lastApplied {
			rf.lastApplied++
			Debug(dCommit, "S%d commit lastApplied:%d lastIndex %d commits:%v", rf.me, rf.lastApplied, rf.getLastIndex(), rf.commits())
			applyMsg := ApplyMsg{
				CommandValid:  true,
				SnapshotValid: false,
				Command:       rf.restoreLog(rf.lastApplied).Command,
				CommandIndex:  rf.lastApplied,
			}
			rf.mu.Unlock()
			rf.applyChan <- applyMsg
			rf.mu.Lock()
		} else {
			Debug(dCommit, "S%d: rf.applyCond.Wait()", rf.me)
			rf.applierCond.Wait()
			Debug(dInfo, "S%d wake up!!", rf.me)
		}
	}
}
```

### Lab2C

2C本身并没有较多的细节，如果2B写得好的话，按照论文上的描述把对应的字段持久化，就可以通过所有的测试了，不需要改动之前的代码，并且在Lab的注释当中已经给出了他的persister的用法，大约十分钟就可以拿下。

没有必要展示代码了，按照论文描述的实现即可，为什么需要持久化这些字段在上面也进行了分析。

### Lab2D

Lab2D比较棘手的就是对于日志Index的重构，添加一个lastIncludedIndex和一个lastIncluedTerm

lastIncludedIndex代表被快照的最后一个Index，对于日志的读取需要通过lastIncludedIndex进行偏移，即将所有从0开始读取的全部偏移至从lastIncludedIndex开始读取，并且提供偏移版本的lastIndex lastTerm prevIndex prevTerm，以及一个偏移版本的获取日志。

并且这两个量需要和快照一同进行持久化。

在实现上和发送日志基本一致，同样定义三个函数：

- leaderSendInstallSnapshot ：leader调用发送一次RPC
- sendInstallSnapshot ：RPC发送函数
- InstallSnapshot ：follower被动调用向自身写入快照

#### leaderSendIntallSnapshot

发送时机

当日志过于落后时，如论文中所说：

> the leader must occasionally send snapshots to followers that lag behind. This happens when the leader has already discarded the next log entry that it needs to send to a follower.

此时leader自身的日志已经被写成快照的形式，再使用nextIndex 已经无法访问，因此应当把leader的快照发送给follower，这也是leader唯一发送快照的条件

等到RPC发送之后再加锁，一点点的性能优化并且不会出错

```go
			if rf.nextIndex[peer]-1 < rf.lastIncludedIndex {
				go func(peer int) {
					rf.leaderSendSnapshot(peer)
				}(peer)
				return
			}
```

发送日志和发送快照在接收到发送结果之后的处理方式基本一致，但是快照不需要处理冲突问题，因此只需要处理term过小，自身为old leader的问题，以及更新matchIndex和nextIndex即可

```go
func (rf *Raft) leaderSendSnapshot(server int) {
	args := SnapshotArgs{
		Term:              rf.currentTerm,
		LeaderId:          rf.me,
		LastIncludedIndex: rf.lastIncludedIndex,
		LastIncludedTerm:  rf.lastIncludedTerm,
		Data:              rf.persister.ReadSnapshot(),
	}
	reply := SnapshotReply{}
	ok := rf.sendSnapshot(server, &args, &reply)
	if !ok {
		Debug(dError, "S%d send InstallSnapshot rpc to %d failed", rf.me, server)
		return
	}
	rf.mu.Lock()
	defer rf.mu.Unlock()
	if rf.status != leader || rf.currentTerm != args.Term {
		return
	}
	if reply.Term > rf.currentTerm {
		rf.setNewTerm(reply.Term)
		rf.setRandomElectionTimeout(rf.me)
		return
	}
	rf.matchIndex[server] = args.LastIncludedIndex
	rf.nextIndex[server] = args.LastIncludedIndex + 1
}
```

#### SendSnapshot

直接调用即可，没什么好说的

```go
func (rf *Raft) sendSnapshot(server int, args *SnapshotArgs, reply *SnapshotReply) bool {
	ok := rf.peers[server].Call("Raft.InstallSnapshot", args, reply)
	return ok
}
```

#### InstallSnapshot

按照论文所说进行实现即可

![](https://pic-bed-1309931445.cos.ap-nanjing.myqcloud.com/blog/image-20230201132413-2wekg3s.png)

lab的讲义上提示无需实现offset，因此和offset相关的done以及 2 3 4三条也一同无需实现，结构体简化为：

```go
type SnapshotArgs struct {
	Term              int
	LeaderId          int
	LastIncludedIndex int
	LastIncludedTerm  int
	Data              []byte
}
type SnapshotReply struct {
	Term int
}
```

最终需要实现的功能：

- 检查term（receiver implementation 1)
- 保留在快照当中不存在的日志项，其他被快照覆盖的日志直接删除（receiver implementation 5 6 7)
- 更新 commitIndex 和 lastApplied
- 通过ApplyMsg的channel将日志应用的消息告知给上层(receiver implentation 8)

```go
func (rf *Raft) InstallSnapshot(args *SnapshotArgs, reply *SnapshotReply) {
	rf.mu.Lock()
	defer rf.mu.Unlock()
	reply.Term = rf.currentTerm
	// receiver implementation 1
	if args.Term < rf.currentTerm {
		return
	}
	rf.setNewTerm(args.Term)
	// leader发送来的快照比自身的进度还落后，直接return
	if rf.lastIncludedIndex >= args.LastIncludedIndex {
		return
	}
	index := args.LastIncludedIndex
	temp := make([]Entry, 0)
	temp = append(temp, Entry{
		Command: nil,
		Term:    args.LastIncludedTerm,
		Index:   args.LastIncludedIndex,
	})
	// 保留快照不包含的所有的entry，快照之前的全部删除
	// receiver implementation 5 6 7
	for i := index + 1; i <= rf.getLastIndex(); i++ {
		temp = append(temp, rf.restoreLog(i))
	}
	rf.lastIncludedIndex = args.LastIncludedIndex
	rf.lastIncludedTerm = args.LastIncludedTerm
	rf.log.Entries = temp
	rf.snapshot = args.Data

	if index > rf.commitIndex {
		rf.commitIndex = index
	}
	if index > rf.lastApplied {
		rf.lastApplied = index
	}
	rf.persister.SaveStateAndSnapshot(rf.persistData(), rf.snapshot)
	// receiver implementation 8
	msg := ApplyMsg{
		SnapshotValid: true,
		Snapshot:      rf.snapshot,
		SnapshotTerm:  rf.lastIncludedTerm,
		SnapshotIndex: rf.lastIncludedIndex,
	}
	rf.mu.Unlock()
	rf.applyChan <- msg
	rf.mu.Lock()
}
```

AppendEntires

在Append Entries当中需要防止follower已经对于prevLogIndex进行了快照化，注释掉的话第一个测试就过不了。

```go
	if rf.lastIncludedIndex > args.PrevLogIndex {
		reply.Success = false
		reply.XIndex = rf.getLastIndex() + 1
		reply.Conflict = true
		return
	}
```
