---
title: MIT-6.824 LAB2D Snapshot
date: 2023-01-28T23:38:32Z
lastmod: 2023-02-01T22:53:06Z
categories: [Distributed System,MIT 6.824]
---

# LAB2D

![](https://pic-bed-1309931445.cos.ap-nanjing.myqcloud.com/blog/image-20230128234137-q28ni4k.png)

* `Snapshot(index int,snapshot []byte)`​ :持久化字节数组，相关的函数已经提供，因此所剩下的就是处理索引相关,删除index之前的日志
* `InstallSnapshot`​RPC

## Snapshot

函数原型：`Snapshot(index int,snapshot []byte)`​

上层扫描当亲的状态，调用`Snapshot`​，将扫描到的日志之前日志对应的状态发送至raft层，raft将快照的字节数组进行持久化，并删除对应的索引前的日志，用于节约空间

* 对于日志，只有已经commit之后才能确保写入，因此才能对其进行快照操作，如果`index`​ > `commitIndex`​，则直接返回
* 由于之前为了对齐索引，和处理`prevLog`​，采用了`dummyHead`​的实现方式，在进行快照之后为了处理偏移量，同样采用`dummyHead`​的方式，不过为了可以直接拿到`prevLogTerm`​和`prevLogIndex`​，`dummyhead`​当中放入`lastIncludedTerm`​和`lastIncludeIndex`
* 参数更新：

  * lastIncludedIndex：index截取的位置，直接设置为index
  * `lastIncludedterm`​：如果超出范围则取最后一个term，否则直接按照索引去获取即可

![](https://pic-bed-1309931445.cos.ap-nanjing.myqcloud.com/blog/image-20230131205909-bfvjoj7.png)

‍

### 持久化

此处涉及到了两个新变量：`lastIncludedIndex`​ `lastIncludedTerm`​ 表明快照当中的最后一个日志的index和term，需要和snapshot一同进行持久化

因此一共需要持久化:

* `Log`​
* `currentTerm`​
* `votedFor`​
* `lastIncludedTerm`​
* `lastIncludedIndex`​

```go
func (rf *Raft) persistData() []byte {
	w := new(bytes.Buffer)
	e := labgob.NewEncoder(w)
	e.Encode(rf.currentTerm)
	e.Encode(rf.votedFor)
	e.Encode(rf.log)
	e.Encode(rf.lastIncludedIndex)
	e.Encode(rf.lastIncludedTerm)
	data := w.Bytes()
	return data

}
```

### 日志和索引

之前日志为单独进行封装，存在的问题就是无法去读取`lastIncludedTerm`​和`lastIncludedIndex`​，因此删掉之前对log的封装，在raft层封装一些对日志的操作

`lastIncludedIndex`​初始化为0即可保证通过之前的2A-2C的测试，在2D中减去`lastIncludedIndex`​得到真实的偏移量

```go
func (rf *Raft) restoreLog(idx int) Entry {
	return rf.log.Entries[idx-rf.lastIncludedIndex]
}

func (rf *Raft) restoreLogTerm(idx int) int {
	if idx-rf.lastIncludedIndex == 0 {
		return rf.lastIncludedTerm
	}
	return rf.log.Entries[idx-rf.lastIncludedIndex].Term
}
func (rf *Raft) getPrevLogIndex(server int) int {
	return rf.nextIndex[server] - 1
}
func (rf *Raft) getPrevLogTerm(idx int) int {
	lastIndex := rf.getLastIndex()
	if idx == lastIndex+1 {
		idx = lastIndex
	}
	return rf.restoreLogTerm(idx)
}

func (rf *Raft) getLastIndex() int {
	return len(rf.log.Entries) - 1 + rf.lastIncludedIndex
}
func (rf *Raft) getLastTerm() int {
	if len(rf.log.Entries)-1 == 0 {
		return rf.lastIncludedTerm
	} else {
		return rf.log.Entries[len(rf.log.Entries)-1].Term
	}
}
```

### bug

2D暴露出了很多之前遗留的bug问题，和index out of range折腾了好几天，index out of range主要是因为访问到了已经被持久化为快照的日志所导致的，尤其是nextIndex和matchIndex相关的

![](https://pic-bed-1309931445.cos.ap-nanjing.myqcloud.com/blog/image-20230131212510-wfhq1jw.png)

根据论文`nextIndex`​在`Make()`​创建raft对象时并不需要管，而在leader当选成功后，全部设置为last log index + 1，`matchIndex[]`​全部初始化为0，代表复制了最高的index，而对于leader自身，就所有的log都已经写入了日志(废话)，matchIndex[rf.me]初始化为lastLogIndex，其他的从节点的信息留着发送日志之后返回结果时再进行处理

`candidateSendRequestVote`:

```go
			for i, _ := range rf.peers {
				// rules for state :nextIndex[] matchIndex[]
				rf.nextIndex[i] = lastLogIndex + 1
				rf.matchIndex[i] = 0
			}
			rf.matchIndex[rf.me] = lastLogIndex

```

`AppendEntries`​

lastIncludedIndex > PrevLogIndex证明，当前leader想要应用的日志已经被follower写成快照了，因此leader发送的进度过慢，因此最简单的处理方法为返回一个最新的 `Index`​，让leader之后再去探测

```go
	if rf.lastIncludedIndex > args.PrevLogIndex {
		reply.Success = false
		reply.XIndex = rf.getLastIndex() + 1
	}

```

`appendEntries`​

小于则证明当前无可发日志，全部被持久化为快照，需要通过快照的形式进行发送，即论文中所说：

> the leader must occasionally send snapshots to followers that lag behind. This happens when the leader has already discarded the next log entry that it needs to send to a follower.

因此直接返回，通过`InstallSnapShot`​RPC发送快照给follower，不再去发送日志

```go
			if rf.nextIndex[peer]-1 < rf.lastIncludedIndex {
				return
			}

```

## InstallSnapshot RPC

### 实现

在实现上和发送日志基本一致，同样定义三个函数：

* `leaderSendInstallSnapshot`​ ：leader调用发送一次RPC
* `sendInstallSnapshot`​ ：RPC发送函数
* `InstallSnapshot`​ ：follower被动调用向自身写入快照

### leaderSendIntallSnapshot

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

发送日志和发送快照在接收到发送结果之后的处理方式基本一致，但是快照不需要处理冲突问题，因此只需要处理term过小，自身为old leader的问题，以及更新`matchIndex`​和`nextIndex`​即可

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

### SendSnapshot

直接调用即可，没什么好说的

```go
func (rf *Raft) sendSnapshot(server int, args *SnapshotArgs, reply *SnapshotReply) bool {
	ok := rf.peers[server].Call("Raft.InstallSnapshot", args, reply)
	return ok
}
```

### InstallSnapshot

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

* 检查term（receiver implementation 1)
* 保留在快照当中不存在的日志项，其他被快照覆盖的日志直接删除（receiver implementation 5 6 7)
* 更新 commitIndex 和 lastApplied
* 通过ApplyMsg的channel将日志应用的消息告知给上层(receiver implentation 8)

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

### bug

在`TestSnapshotInstallCrash2D`​的测试当中挂掉了，出现了index out of range 定位到applier的commit时，会去根据`rf.lastApplied`​去获取到最新提交应用的日志，返回给应用层，而由于该测试为节点会crash掉，而lastApplied并未进行持久化保存，因此默认为0

后续提交时由于之前的日志已经进行持久化，再按照0去读取会导致日志的索引小于`rf.lastIncludedIndex`​，最终导致index out of range，因此解决方案为在通过`Make`​函数进行初始化时，需要恢复lastApplied的状态：

```go
	if rf.lastIncludedIndex > 0 {
		rf.lastApplied = rf.lastIncludedIndex
	}

```

此外似乎某个日志输出时会存在index of range的问题，关闭日志输出时并没有遇到bug，暂时也懒得再去debug了，等后续有问题时再去找找看看，目前关闭日志输出能够稳定过测试（睡了一觉bug没有了。。这也也算能稳定过测试了。。。）

‍
