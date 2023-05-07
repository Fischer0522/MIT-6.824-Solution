---
title: MIT-6.824 Lab3 KvRaft
date: 2023-03-03T00:13:51Z
lastmod: 2023-03-06T16:50:10Z
categories: [Distributed System,MIT 6.824]
---

# Lab3

## Lab3A

client通过调用`Clerk`​的`put`​ `append`​ `get`​方法与server进行交流

需要实现强一致性，即表现的像是只有一个副本，并且每个操作都应该能够看到之前的操作的结果

对于并发调用，应当遵循某种顺序（类似于隔离性）

为了实现强一致性，对于并发的

### 实现细节

* 每一个kvserver对应一个raft节点
* clerks发送RPC请求至和Raft Leader相关联的kvserver上
* kvserver 将操作提交至raft，raft将其存储于日志当中，kvserver只根据日志去执行操作，修改数据库状态
* clerk并不知道哪个节点为leader，如果发送rpc至错误的leader，需要重新发送请求至另外一个kvserver，
* commit之后leader需要通过rpc的方式将结果告知给clerk
* 如果没能成功commit，则需要报告一个错误至clerk，clerk再选择其他的server重试
* kvserver之间并不进行通信，彼此之间只通过raft进行通信

### server

先按照指导所说的先不考虑宕机问题，因此也不去处理clerk发送过来的重复请求，server层为基于raft的上层，逻辑上的请求流程为client-> server -> raft，返回时再反过来。server只负责接收client的请求然后传递给raft，同时接收raft commit了的信息，应用于自身，具体的决策由raft来完成，决定哪一条数据成功commit,然后写入。

为了保证线性一致性的问题，所有的操作无论读写都要写入到日志中，然后严格按照日志的顺序去执行，以防止读请求无法读取到在之前写入的数据。

server端的处理流程为：

1. 先通过`StartKVServer`​初始化服务端的对应状态
2. client通过rpc调用server的 put get append等函数
3. server先生成对应的日志，交给raft `put`​ `get`​ `append`​只应当修改raft层的日志，不应该修改server层的数据库存储信息
4. 设置一个对应的appliedHandler来处理来自raft层的commit消息，将其应用于自身，并通知至client

**applyMsgHandler**​

因此首先定义一个`applyMsgHandler`​用于接受来自底层raft上传的commit信息，不断的从raft层的channel当中获取消息，按照类型进行处理，处理完成之后通过一个channel来通知对应的server，告知其commit的消息

```go
func (kv *KVServer) applyMsgHandler() {
	for msg := range kv.applyCh {

		index := msg.CommandIndex
		op := msg.Command.(Op)
		kv.mu.Lock()
		if op.OpType == PUT {
			kv.memDb[op.Key] = op.Value
			Debug(dCommit, "S%d get PUT applyMsg from raft key:%s value:%s", kv.me, op.Key, op.Value)

		} else if op.OpType == APPEND {
			Debug(dCommit, "S%d get APPEND applyMsg from raft key:%s value:%s", kv.me, op.Key, op.Value)
			if _, ok := kv.memDb[op.Key]; !ok {
				kv.memDb[op.Key] = op.Value
			} else {
				kv.memDb[op.Key] += op.Value
			}
		}
		kv.mu.Unlock()
		// nothing to do for GET Op
		msgChan := kv.getMsgChan(index)
		msgChan <- op
	}

}

```

**Get/PutAppend**

只有Leader节点才会响应Client的请求信息，因此先对当前节点的身份进行判断，如果不是leader节点直接设置对应的错误即可返回

之后再将请求封装成一个Op，传到下层raft作为一条日志信息，为保证线性一致性，无论是只读的Get请求还是PutAppend请求都需要按照请求的顺序封装成一个日志，最后按照日志的顺序来确认commit，最终返回结果

因此kvserver在写入日志之后则需要监听对应日志的channel，以获取对应的commit信息，当确认到commit之后，再向client返回。

此外需要设置一个超时时间，来防止当前节点为Leader当时在接收请求之后宕机的情况，由于此时还未在server端实现对重复请求的处理，如果client向Leader发送重复的请求会被执行两次，因此最初采用的是延长超时时间来进行处理，经过尝试，对于我的底层Raft,Get请求超时时间为100ms，PutAppend请求为400ms时可以稳定通过第一个测试

```go
func (kv *KVServer) Get(args *GetArgs, reply *GetReply) {
	if kv.killed() {
		reply.Err = ErrWrongLeader
		return
	}

	_, state := kv.rf.GetState()
	if !state {
		reply.Err = ErrWrongLeader
		return
	}
	// Your code here.
	op := Op{
		Key:      args.Key,
		Value:    "",
		OpType:   GET,
		ClientId: args.ClientId,
		Index:    0,
	}
	Debug(dLog, "S%d send a Get log to raft key:%s", kv.me, args.Key)
	index, _, _ := kv.rf.Start(op)

	msgChan := kv.getMsgChan(index)
	defer func() {
		kv.mu.Lock()
		delete(kv.msgChan, index)
		kv.mu.Unlock()
	}()

	timer := time.NewTicker(100 * time.Millisecond)
	defer timer.Stop()
	select {
	case replyOp := <-msgChan:
		{
			if replyOp.ClientId != op.ClientId {
				reply.Err = ErrWrongLeader
				return
			}
			kv.mu.Lock()
			val, ok := kv.memDb[op.Key]
			if !ok {
				reply.Err = ErrNoKey
			} else {
				reply.Err = OK
				reply.Value = val
			}
			Debug(dInfo, "S%d server get the value in GetFunc from msgChan key: %s ,value:%s", kv.me, op.Key, reply.Value)
			kv.mu.Unlock()
			return
		}
	case <-timer.C:
		reply.Err = ErrWrongLeader
		return
	}

}

```

‍

![](https://pic-bed-1309931445.cos.ap-nanjing.myqcloud.com/blog/image-20230303113035-xokhls8.png)

![](https://pic-bed-1309931445.cos.ap-nanjing.myqcloud.com/blog/image-20230303113046-ha16k61.png)   

### 数据结构

**Op**​

表示一次操作,在client出先初始化出 key value clientId Optype

```go
type Op struct {
	// Your definitions here.
	// Field names must start with capital letters,
	// otherwise RPC will break.
	Key      string
	Value    string
	OpType   int
	ClientId int64
}
```

### 容错

当出现网络分区或者节点宕机时，如当Leader确认了commit信息之后宕机，但是未为给client响应，因此client会重复发送一个请求，如果该重新发送的请求不进行处理，最后就会被server层重新执行。主要分为在commit之前宕机和在commit之后宕机：

> * 当leader在确认commit之前宕机，client需要发现Leader宕机并且重新发送一次请求给其他的节点，直至找到一个新的Leader：
>
>   * 如果长时间未commit发生超时，即可认定Leader宕机正在进行重新选举，此时通过定时器返回Leader宕机的信息给client
>   * 如果当前的Leader宕机，已经选举了其他的节点为Leader，但是该迅速恢复，此时Client同样还在和该前Leader进行通信，但是server可以发现Start的term发生改变，此时即可认定自己不是Leader，向client响应
>   * 其他节点已经调用的Start处理了当前的日志
> * Leader确认comit之后宕机，server端已经进行了写入操作，但是client长时间未收到响应，尝试重新发送请求，此时需要防止同样的命令被重复执行，因此对于client的操作，需要设置一个独一无二的标识，保证只执行一次
> * client端应当记住当前的Leader的ServerId，之后向该Leader发送请求

而在Lab的实现上，在server端只需要检测是否超时即可，即当前Leader宕机后又迅速恢复，当时已经超时出发了LeaderElection，此时通过一个定时器进行检测，如果超时则证明出现了宕机，raft层正在重新选举而无法给予回应，因此此时应当告知client Leader发生变化，让其重试，向其他的server发送RPC，来确认新的Leader，之后和新的Leader进行通信。

而对于Lab提示中提及的关于Term的检测，最终我没有实现，Term的改变对于Leader的变化是充分不必要条件，如果当前Leader宕机，然后其他节点选举处理新的Leader，此时原本的Leader从宕机恢复，但是此时他还未收到心跳，因此自身的Term还未发生改变，但是此时原本的Leader应当向client响应自己并非节点。在这种实现方式下，也确实能够稳定通过测试。

```go
	select {
	case replyOp := <-ch:
		if op.ClientId != replyOp.ClientId || op.SeqId != replyOp.SeqId {
			Debug(dError, "S%d get the wrong Op", kv.me)
			reply.Err = ErrWrongLeader
		} else {
			reply.Err = OK
			Debug(dCommit, "S%d PutAppend OK key:%s value:%s", kv.me, args.Key, args.Value)
		}
	case <-timer.C:
		reply.Err = ErrWrongLeader
	}

```

对于client，在其中定义一个`seqId`​ 来记录当前请求的顺序，每次生成一个新的请求时自增，之后携带在请求当中发送给server，因此最终的client数据结构如下：

```go
type Clerk struct {
	servers []*labrpc.ClientEnd
	// You will have to modify this struct.
	clientId int64
	leaderId int
	seqId    int64
}
```

在调用`Get`​或者`PutAppend`​时`seqId++`

在Server端定义一个`isDuplicate`​来判断是否已经有其他的该op是否已经执行过，如果当前op的`seqId`​已经有记录，则证明之前有其他的server已经执行过该op，因此跳过执行，但是依旧需要通知client此次请求执行完毕

```go
func (kv *KVServer) isDuplicate(clientId int64, seqId int64) bool {
// 如果没有记录则是第一次执行
	lastSeqId, ok := kv.seqIdMap[clientId]
	if !ok {
		return false
	}
// 如果之前已经存在与当前相等或者更大的SeqId，则证明已经执行过一次，返回false
	return seqId <= lastSeqId
}

```

除此之外只需要修改`applyMsgHandler`​，跳过已经执行过的Op，当时最开始顺手写了个否定判断加return的写法，导致直接退出了循环，

```go

func (kv *KVServer) getMsgChan(index int) chan Op {
	kv.mu.Lock()
	defer kv.mu.Unlock()
	ch, ok := kv.msgChan[index]
	if !ok {
		kv.msgChan[index] = make(chan Op, 1)
		ch = kv.msgChan[index]
	}
	return ch
}

func (kv *KVServer) applyMsgHandler() {
	for msg := range kv.applyCh {

		index := msg.CommandIndex
		op := msg.Command.(Op)
		if !kv.isDuplicate(op.ClientId, op.SeqId) {

			kv.mu.Lock()
			kv.seqIdMap[op.ClientId] = op.SeqId
			if op.OpType == PUT {
				kv.memDb[op.Key] = op.Value
				Debug(dCommit, "S%d get PUT applyMsg from raft key:%s value:%s", kv.me, op.Key, op.Value)

			} else if op.OpType == APPEND {
				Debug(dCommit, "S%d get APPEND applyMsg from raft key:%s value:%s", kv.me, op.Key, op.Value)
				if _, ok := kv.memDb[op.Key]; !ok {
					kv.memDb[op.Key] = op.Value
				} else {
					kv.memDb[op.Key] += op.Value
				}
			}
			kv.mu.Unlock()
		}
		// nothing to do for GET Op
		msgChan := kv.getMsgChan(index)
		msgChan <- op
	}

}
```

在接收到commit的请求之后在通过ClientId和SeqId来确认唯一序列，即该Leader在接收到请求之后，还未进行commit就已经宕机，在对应的index没有写入日志，此时重新选举出一个Leader，在该Index的位置处写入了其他的不相关的日志，因此此时需要通过ClientId和SeqId确认该日志为该Client在对应序列出的Op，如果不是，则证明该节点经历了宕机并重启，变为了普通节点，需要告知client重新去寻找一个Leader

```go
	case replyOp := <-ch:
		if op.ClientId != replyOp.ClientId || op.SeqId != replyOp.SeqId {
			Debug(dError, "S%d get the wrong Op", kv.me)
			reply.Err = ErrWrongLeader
		} else {
			reply.Err = OK
			Debug(dCommit, "S%d PutAppend OK key:%s value:%s", kv.me, args.Key, args.Value)
		}
```

**超时时间**

而此时由于server端已经能够对重复的请求进行正确的处理，因此此时超时时间的设置便不再那么重要，并不会因为超时时间而导致bug的发生，只会影响RPC发送的频率个人感觉按照raft底层的`election timeout`​进行处理即可，`election timout`​为150-300ms，个人感觉设置为300ms较为合理，如果超过300ms基本上可以认定重新进行了Leader Election。如果设置为100ms的话，虽然100ms没反应的话也基本可以认定当前出现了问题，但是此时raft也没有选出新的节点，client再去发送RPC联系其他的server也是徒劳，还不如静静等待raft完成选举之后再去重试，能够减少一定的开销。

此时除了`TestSpeed3A`​以外其他的都可以稳定通过测试，对于请求速度的问题，当按照raft论文当中的提供的electIon timeout在150-300ms之间只能够跑到50-60ms之间，如果选举超时时间和Leader心跳的间隔过短，大约election timeout在80-130ms之间，而心跳间隔设置为25ms时，Lab3A可以在保证正确性的前提下运行，但是Lab2B当中一个测试的RPC发送次数过多，暂时还未找到一个较好的平衡，估计Lab3B的速度测试也是相同的情况，暂时先不去深究，之后再去想想合适的优化策略

## Lab3B

使用Lab2D当中的`snapshot()`​方法来优化宕机节点的重新复制日志的速度

* `maxraftstate`​​表示当前以字节为单位最大的日志体积，如果超出该体积，则应当生成一个快照，通过和`persisiter.RaftStateSize()`​​进行比较，如果maxraftstate为 -1则无需进行快照操作
* 需要能过跨过`checkpoint`​​来检测是否有重复的操作，因此无论何时检测重复操作时都应当考虑快照
* 在server层生成一个snapshot，之后传给raft层进行持久化存储

**applyMsgHandler**​

由于Raft层存在两个applier，一个在向上应用普通日志，另外一个在向上应用日志，因此应当顺序处理两个applier，在原本lab3A的基础上添加创建快照和处理applyMsg中的快照信息即可

```go

func (kv *KVServer) applyMsgHandler() {
	if kv.killed() {
		return
	}
	for msg := range kv.applyCh {

		if msg.CommandValid {
			index := msg.CommandIndex
			kv.mu.Lock()
			op := msg.Command.(Op)
			if index <= kv.lastApplied {
				kv.mu.Unlock()
				continue
			}

			kv.lastApplied = index
			if !kv.isDuplicate(op.ClientId, op.SeqId) {

				kv.seqIdMap[op.ClientId] = op.SeqId
				if op.OpType == PUT {
					kv.memDb[op.Key] = op.Value
					Debug(dCommit, "S%d get PUT applyMsg from raft key:%s value:%s", kv.me, op.Key, op.Value)

				} else if op.OpType == APPEND {
					Debug(dCommit, "S%d get APPEND applyMsg from raft key:%s value:%s", kv.me, op.Key, op.Value)
					kv.memDb[op.Key] += op.Value
				}
			}
			if kv.maxraftstate != -1 && kv.rf.GetRaftState() > kv.maxraftstate {
				snapshot := kv.encodeSnapshot()
				Debug(dSnap, "S%d creates a snapshot lastApplied:%d snapshotIndex:%d", kv.me, kv.lastApplied, index)
				kv.rf.Snapshot(index, snapshot)
			}
			kv.mu.Unlock()
			kv.getMsgChan(msg.CommandIndex) <- op

			// nothing to do for GET Op
		}
		if msg.SnapshotValid {
			kv.mu.Lock()
			if msg.SnapshotIndex > kv.lastApplied {
				kv.decodeSnapshot(msg.Snapshot)
				kv.lastApplied = msg.SnapshotIndex
			}
			kv.mu.Unlock()

		}
	}

}
```

**编解码**

和Lab2C中perisister的处理方法基本一致

```go
func (kv *KVServer) encodeSnapshot() []byte {
	w := new(bytes.Buffer)
	e := labgob.NewEncoder(w)
	e.Encode(kv.memDb)
	e.Encode(kv.seqIdMap)
	data := w.Bytes()
	return data
}
func (kv *KVServer) decodeSnapshot(data []byte) {
	if data == nil || len(data) < 1 {
		return
	}
	r := bytes.NewBuffer(data)
	d := labgob.NewDecoder(r)
	var memDb map[string]string
	var seqIdMap map[int64]int64

	if d.Decode(&memDb) != nil || d.Decode(&seqIdMap) != nil {
		Debug(dError, "S%d decode snapshot failed", kv.me)
	} else {
		kv.memDb = memDb
		kv.seqIdMap = seqIdMap
	}

}
```

‍

### 注意事项

> 目前会产生丢失key的情况，并且只有在多个客户端时才会发生，应该是日志相关的并发没有处理好
>
> 最初使用Append命令向 Key0之后追加 323，并且已经写入成功，applyMsgHandler将其写入到内存数据库当中，并且向msgChan发送了通知
>
> kvserver也收到了msgChan给予的通知，并且给予了客户端响应，
>
> 客户端也收到了响应，显示添加成功
>
> 后续进行了两次Get请求，一次查询到了323，另外一次没有查询到 323，
>
> 目前怀疑两次Get请求之间存在一次快照同步的问题，发生了状态机回退
>
> 目前只有TestSnapshotRecoverManyClients 和TestSnapshotUnreliable RecoverConcurrentPartition会挂掉，基本十次会挂掉一次，其他的都能够稳定过测试

* apply 日志时需要防止状态机回退：lab2 的文档已经介绍过, follower 对于 leader 发来的 snapshot 和本地 commit 的多条日志，在向 applyCh 中 push 时无法保证原子性，可能会有 snapshot 夹杂在多条 commit 的日志中，如果在 kvserver 和 raft 模块都原子性更换状态之后，kvserver 又 apply 了过期的 raft 日志，则会导致节点间的日志不一致。因此，从 applyCh 中拿到一个日志后需要保证其 index 大于等于 lastApplied 才可以应用到状态机中，而对于快照也需要保证其`lastIncludedIndex > lastApplied`​才能保证状态机不发生回退，再处理了日志和快照的状态机回退情况之后 missing element的情况**几乎**不再发生，在`applyMsgHandler`​出已经防止了状态机回滚的问题，但是依旧还会有极端情况发生，大约100次左右会发生一次missing key的问题，暂时已经没有精力去debug了，暂且先欠个债。
* 对于锁的话，在PutAppend Get applyMsgHandler等函数加一个函数级别的大锁即可，无需在`encodesnapshot`​ `decodesnapshot`​等小的函数分别加锁，`applyMsgHandler`​本身的执行也为连续的，中间不涉及到发送RPC等耗时且安全的操作，直接一个函数级的锁即可，频繁的上锁解锁反而会造成不必要的性能开销。
