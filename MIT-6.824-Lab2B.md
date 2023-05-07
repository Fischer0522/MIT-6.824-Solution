---
title: MIT-6.824 Lab2B Log Replication
date: 2023-01-26T10:52:00Z
lastmod: 2023-01-26T18:42:25Z
categories: [Distributed System,MIT 6.824]
---

# Lab2B

主要分为两部分：

* 一部分为在lab2A的AppendEntries的基础上将普通的心跳信息拓展为传输日志

* 另一部分为根据论文5.4的safety对RequestVote进行改写，添加安全限制，防止随意的leader选举导致日志丢失

基本上还是遵循figure2进行实现

### 5.3

* leader将command转换为一个log entry，之后通过`AppendEntries`​发送给follower，当follower安全复制了日志之后，（半数以上都成功写入了日志），之后leader将其写入的状态机（应用层），并返回给client     

  `start`​当中实现
* follower如果没能够成功写入，则leader就一直进行重试操作，

  `appendEntries`​当中反复调用 leaderSendAppendEntries?
* 半数以上的成功写入了日志则认为该日志条目为commited状态

  `leaderSendAppendEntries`​当中实现
* log entry commit时，会提交leader日志之前所有的log entry

  单独的applier协程实现,再想想
* leader保存已知的提交了的最高的日志的索引`commitIndex`​,未来的RPC中会将该索引发送给其他节点
* 如果不同日志的两个条目具有相同的索引和term，则存储相同的命令，并且之前的log entry也都相同 ：If the follower does not find an entry in  
  its log with the same index and term, then it refuses the  
  new entries.

  `AppendEntries`​当中由follower去比较
* 如果AppendEntries返回成功值，那么就可以认为`is identical to its own log up through the new entries.`​

  `leaderSendAppendEntries`​实现
* leader通过强制让follower的日志和leader相同2来处理不一致性，leader找出和follower最近的不同，然后删除follower的日志，再把之后leader的日志复制给follower，

  同样`leaderSendAppendEntris`​
* leader对每个follower维护一个`nextIndex`​，即leader将要发送给follower的下一个日志，leader当选之后，nextIndex初始化为leader自己的最后一个log的Index，之后通过AppendEntries进行试探，如果不匹配就递减，直至leader和follower的日志匹配，如果匹配，AppendEntries会返回success，并且移除follower所有冲突的日志，并从leader的日志中进行追加

  **CandidateStartElection当中实现!!!!**
* 如果AppendEntries返回success，则证明leader和follower的日志一致
* leader永远不会重写或者删除 自己的日志

## AppendEntries

### Figure2

**state**：

* 定义logEntry结构体，包含command，term，index
* commitIndex：自身commit了的log的最高index
* lastApplied：应用到状态机的最高index
* nextIndex[]：下一个要发送给server的log entry 的index
* matchIndex[]：被follower复制了的最高的index 初始化为0  将 matchIndex 更新为你最初在 RPC 中发送的参数中 prevLogIndex + len( entries[]) 。

**AppendEntries**​

* 当prevLogTerm匹配时，但是prevLogIndex上却没有log Entry时，返回false，和term < currentTerm的情况合并(立刻返回)
* 如果存在的log entry和appendEntries携带的新的entry冲突时（相同索引但是不同term），删除旧的，写入新的

  如果跟随者拥有领导者发送的所有条目，跟随者必须不截断其日志。任何跟随领导者发送的条目的元素都必须被保留。这是因为我们可能从领导者那里收到了一个过时的AppendEntries RPC，而截断日志将意味着 "收回 "我们可能已经告诉领导者我们的日志中的条目。
* 只要不存在于日志中的新log Entry，全部应用
* leaderCommit > commitIndex ，set commitIndex = min(leaderCommit,index of last new entry)

**Rules**​

* 当commitIndex > lastApplied，lastApplied递增，应用log[lastApplied]至状态机 

  你要确保这种应用只由一个实体完成。具体来说，你需要有一个专门的 **&quot;applier&quot;**​，或者围绕这些应用进行锁定，这样其他的例程就不会检测到需要应用的条目，并且也试图进行应用。

  确保定期检查 commitIndex > lastApplied，或者在 commitIndex 更新后（即 matchIndex 更新后）检查。例如，如果你在检查commitIndex的同时向对等体发送AppendEntries，你可能不得不等到下一个条目被追加到日志中，然后再应用你刚刚发送并得到确认的条目。
* 发送AppendEntries除heartbeat以外的另一条件：last log index >=nextIndex for a follower

  `appendEntries`​ 当中实现
* AppendEntries从nextIndex开始发：

  * 成功则更新follower的nextIndex和matchIndex `AppendEntries`​
  * 因为不一致性导致的失败，nextIndex递减，重新发送 `leadersendAppendEntries`​
* 如果存在一个N，N > commitIndex，并且大多数的matchIndex[i] > N,log[N].term == currentTerm，则设置commitIndex = N ,没太想好放哪​​

## 实现

将上面总结的规矩shuffle一下:

### start

客户端调用的入口,请求写入一条日志:

* leader将command转换为一个log entry，之后通过`AppendEntries`​发送给follower，当follower安全复制了日志之后，（半数以上都成功写入了日志），之后leader将其写入的状态机（应用层），并返回给client

```go
func (rf *Raft) Start(command interface{}) (int, int, bool) {
	index := -1
	term := -1
	rf.mu.Lock()
	defer rf.mu.Unlock()
	if rf.status != leader {
		return index, term, false
	}

	newLogEntry := Entry{
		Command: command,
		Term:    rf.currentTerm,
		Index:   rf.log.lastLog().Index + 1,
	}
	rf.log.append(newLogEntry)

	rf.appendEntries(false)
	// Your code here (2B).
	Debug(dClient, "S%d receive a command from client", rf.me)
	return index, term, true
}

```

### appendEntries

leader调用,只在意leader相关的内容即可

在lab2A的基础上加上日志相关的内容:

* 发送AppendEntries除heartbeat以外的另一条件：last log index >=nextIndex for a follower

  ```go
  if lastLog.Index >= rf.nextIndex[peer] || heartbeat {
  ```

* 再把RPC的参数改一下，之前做2A的时候觉得不影响就直接全传的0，这次认真传一下：

  ```go
  			args := AppendEntriesArgs{
  				Term:         rf.currentTerm,
  				LeaderId:     rf.me,
  				PrevLogIndex: prevLog.Index,
  				PrevLogTerm:  prevLog.Term,
  				LogEntries:   make([]Entry, lastLog.Index-nextIndex+1),
  				LeaderCommit: rf.commitIndex,
  			}
  			copy(args.LogEntries, rf.log.slice(nextIndex))
  ```

‍

### leaderSendAppendEntries

* 半数以上的成功写入了日志则认为该日志条目为commited状态

  单独定义一个`leaderTryCommit()`​函数，负责在条件满足时进行commit，条件即为**rules for leader : 4**

  ![](https://pic-bed-1309931445.cos.ap-nanjing.myqcloud.com/blog/image-20230128170952-5e7g6gq.png)

  ```go
  func (rf *Raft) leaderTryToCommit() {
  	if rf.status != leader {
  		var stringStatus string
  		if rf.status == candidate {
  			stringStatus = "candidate"
  		} else {
  			stringStatus = "follower"
  		}
  		Debug(dWarn, "S%d,only leader can commit a log,but server[%d] is:%v", rf.me, rf.me, stringStatus)
  		return
  	}
  	// rules for leaders : rule4
  	for n := rf.commitIndex + 1; n <= rf.log.lastLog().Index; n++ {
  		if rf.log.at(n).Term != rf.currentTerm {
  			continue
  		}
  		counter := 1
  		for peer := range rf.peers {
  			if peer != rf.me && rf.matchIndex[peer] >= n {
  				//Debug(dCommit, "S%d commit counter++ counter:%d", rf.me, counter)
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
  }
  ```

* 如果AppendEntries返回成功值，那么就可以认为`is identical to its own log up through the new entries.`​，因此不需要去处理冲突问题，直接将`nextIndex`​和`matchIndex`​进行更新即可

  * 成功匹配的了日志则为发送过去的所有日志，因此`matchIndex`​加上长度即可，而next永远等于matchIndex + 1

  * **Fast Backup** ：为了加快处理冲突的速度，使用`XTerm`​ `XLen`​ `XIndex`​进行加速处理：

    * `XTerm = -1`​(初始值)的情况则为在follower当中未能找到匹配的Term，长度不足，对应下面的case3，此时直接使用长度作为Index即可，在后面进行追加
    * `XTerm != -1`​则存在冲突，

      * 而如果过`XIndex == 0`​则证明只有一种全部冲突的term，此时需要leader去找到该term在leader log当中的index，在此之后的全部进行覆盖，对应case2
      * `XIndex != 0`​ 最为普通的情况，通过follower即可完成定位，leader直接采用follower的建议即可，对应case1,定位到第一个5的Index

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
    >

    ‍

  ```go
  if reply.Success {
  			// 发送成功则为发送的所有日志条目全部匹配，match直接加上len(Entries)
  			// next 永远等于match + 1
  			match := args.PrevLogIndex + len(args.LogEntries)
  			next := match + 1
  			rf.matchIndex[server] = match
  			rf.nextIndex[server] = next

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
  					rf.nextIndex[server] = reply.XIndex
  				}

  			}

  		}
  		rf.leaderTryToCommit()
  ```

* leader通过强制让follower的日志和leader相同2来处理不一致性，leader找出和follower最近的不同，然后删除follower的日志，再把之后leader的日志复制给follower，

* ~~因为不一致性导致的失败，nextIndex递减，重新发送~~  采用XTerm进行优化

### AppendEntries

* 当prevLogTerm匹配时，但是prevLogIndex上却没有log Entry时，返回false(立刻返回)

  * 为了防止index out of range，先日志处理过短的情况
  * 返回前计算`XTerm`​ `XLen`​ `XIndex`​用于优化
* 如果存在的log entry和appendEntries携带的新的entry冲突时（相同索引但是不同term），删除旧的，写入新的

  如果跟随者拥有领导者发送的所有条目，跟随者必须不截断其日志。任何跟随领导者发送的条目的元素都必须被保留。这是因为我们可能从领导者那里收到了一个过时的AppendEntries RPC，而截断日志将意味着 "收回 "我们可能已经告诉领导者我们的日志中的条目。

  只要不存在于日志中的新log Entry，全部应用

  ```go
  	for i, entry := range args.LogEntries {
  		// figure 2 AppendEntries RPC Receiver implementation 3
  		idx := entry.Index
  		if idx <= rf.log.lastLog().Index && rf.log.at(idx).Term != entry.Term {
  			rf.log.truncate(idx)
  		}
  		// figure 2 AppendEntries RPC Receiver implementation 4
  		if entry.Index > rf.log.lastLog().Index {
  			// append all entries after idx
  			Debug(dLog, "S%d leader %d append entries to %d,entries is%v", rf.me, args.LeaderId, rf.me, args.LogEntries)
  			rf.log.append(args.LogEntries[i:]...)
  			break
  		}
  	}
  ```

* leaderCommit > commitIndex ，set commitIndex = min(leaderCommit,index of last new entry)

  ```go
  	// figure 2 AppendEntries RPC Receiver implementation 5
  	if args.LeaderCommit > rf.commitIndex {
  		rf.commitIndex = min(args.LeaderCommit, rf.log.lastLog().Index)
  		rf.wakeApplier()
  	}
  ```

### 其他细节

#### Fast Backup

如果日志过短（case3），直接使用长度即可：

```go
	if rf.log.lastLog().Index < args.PrevLogIndex {
		reply.Success = false
		reply.Conflict = true
		reply.XTerm = -1
		reply.XIndex = -1
		reply.XLen = rf.log.lastLog().Index
		Debug(dLog, "S%d follower's log is to short,index:%d,prevLogIndex%d", rf.me, rf.log.lastLog().Index, args.PrevLogIndex)
		Debug(dLog, "S%d Conflict Xterm %d,XIndex %d,XLen %d", rf.me, reply.XTerm, reply.XIndex, reply.XLen)
		return
	}
```

正常情况XIndex找到XTerm的第一个Index即可

```go
		for idx := args.PrevLogIndex; idx > 0; idx-- {
			// the first index of the conflict term
			if rf.log.at(idx-1).Term != reply.XTerm {
				reply.XIndex = idx
				break
			}

		} 
```

#### LogEntry

```go
type Entry struct {
	Command interface{}
	Term    int
	Index   int
}
type Log struct {
	Entries []Entry
}
```

Entry作为日志的载体，按照论文中定义`Command`​ `Term`​ `Index`​，由于后续涉及到读取和操作Index，因此Index单独定义，而不是直接使用数组下标，后续对Log封装了一些CRUD的操作，因此再用一个结构体套一层

由于涉及到`prevLogIndex`​ `prevLogTerm`​等操作，因此为了方便再单独定义一个dummy head，此时index的真正起始值也为1，符合论文中的要求

#### applier

> If commitIndex > lastApplied at any point during execution, you should apply a particular log entry. It is not crucial that you do it straight away (for example, in the AppendEntries RPC handler), but it is important that you ensure that this application is only done by one entity. Specifically, you will need to either have a dedicated “applier”, or to lock around these applies, so that some other routine doesn’t also detect that entries need to be applied and also tries to apply.

> Hint：Your code may have loops that repeatedly check for certain events. Don't have these loops execute continuously without pausing, since that will slow your implementation enough that it fails tests. Use Go's [condition variables](https://golang.org/pkg/sync/#Cond), or insert a time.Sleep(10 * time.Millisecond) in each loop iteration.

如指南和lab当中的hint当中所说，单独定义一个`applier`​协程去完成相关操作,然后通过条件变量来完成协程间的通信

当有需要commit的日志时，就唤醒该协程进行commit：

* **Rules for leader 4：**If there exists an N such that N > commitIndex, a majority  
  ofmatchIndex[i] ≥ N, and log[N].term == currentTerm:  
  set commitIndex = N (§5.3, §5.4).
* **AppendEntries RPC Receiver implementation 5 :**​If leaderCommit > commitIndex, set commitIndex =  
  min(leaderCommit, index of last new entry)

条件变量执行到`wait()`​时会释放掉锁，因此操作channel时解锁，在if的末尾又应当加锁

```go
func (rf *Raft) applier() {
	rf.mu.Lock()
	defer rf.mu.Unlock()

	for !rf.killed() {
		// rules for all servers:rule1
		if rf.commitIndex > rf.lastApplied && rf.log.lastLog().Index > rf.lastApplied {
			rf.lastApplied++
			applyMsg := ApplyMsg{
				CommandValid: true,
				Command:      rf.log.at(rf.lastApplied).Command,
				CommandIndex: rf.lastApplied,
			}
			//DPrintVerbose("[%v]: COMMIT %d: %v", rf.me, rf.lastApplied, rf.commits())
			Debug(dCommit, "S%d commit lastApplied:%d commits:%v", rf.me, rf.lastApplied, rf.commits())
			rf.mu.Unlock()
			rf.applyChan <- applyMsg
			rf.mu.Lock()
		} else {
			Debug(dCommit, "S%d: rf.applyCond.Wait()", rf.me)
			rf.applierCond.Wait()
			Debug(dInfo, "S%d wake up!!", rf.me)
		}
	}
}If there exists an N such that N > commitIndex, a majority
ofmatchIndex[i] ≥ N, and log[N].term == currentTerm:
set commitIndex = N (§5.3, §5.4).
```

‍
