---
title: MIT-6.824 Lab2A Leader Election
date: 2023-01-23T00:12:07Z
lastmod: 2023-01-26T16:31:39Z
categories: [Distributed System,MIT 6.824]
---

# Lab2A

‍

2A部分只需要实现raft的选举机制，因此主要看论文的5.2部分即可

* 通过心跳机制来检测leader是否正常运行，来确保无需重新选举，心跳复用`AppendEntries`​，发送时不带log即可
* 当follower一段时间没有收到leader的心跳，则认为leader宕机，自己尝试申请称为leader，为保证不出现多个follower同时开始选举导致选票平分的情况，令各个节点的超时时间为大于150ms-300ms（根据lab的提示当中所说）的一个随机数
* 通过`RequestVote`​来发送RPC请求让别的节点来为自己投票
* 当收到了大多数的投票后就会从candidate转变为leader
* 当多个节点同时申请称为leader时，没有产生一个大多数的结果，则定义一个`electionTimeOut`​，过期后开启下一轮的选举
* 称为leader之后就向其他的节点去发送心跳来保证其不会再过期从而成为leader

### State

![](https://pic-bed-1309931445.cos.ap-nanjing.myqcloud.com/blog/image-20230123160218-2ek68p9.png)

节点需要维护的状态如上：

* 需进行持久化：currentTerm，votedFor用于在宕机恢复后确认自己在当前term的状态，防止在一个term当中选出多个leader，log显然需要持久化，无需多解释（Lab2A只涉及到leader election，并且没有要求根据日志去限制，因此当中log可以先不实现）
* 其他易失性的数据都是用于进行日志同步时使用，也可以先不实现

Lab2A中只实现`currentTerm`​ `votedFor`​即可

### Rules

![](https://pic-bed-1309931445.cos.ap-nanjing.myqcloud.com/blog/image-20230123161054-molr6yu.png)

* all server：

  Lab2A中依旧不用管log，因此对于所有sever中的log只需要注意第二条即可，即如果rpc中回应的term号大于自己的term号，就将自己的term号进行更新

* follower：

  * 响应candidate和leader的rpc请求，提供一个rpc来响应leader的心跳，一个rpc用于给candidate投票
  * 当follower长时间没收到leader的心跳则超时转变为candidate
* candidate：

  * 选举流程：自增currentTerm，给自己投票，重设election timer，发送rpc让其他为自己投票
  * 收到大多数的选票后成为leader
  * 如果收到了一个比自己term要大的leader，即自己不是最新的leader，则将自己转变为follower
  * 超时则重新选举，即当前term没有选出leader，开启下一个term

* leader：

  * 向follow发送心跳以防止其超时
  * log相关的统统不管

‍

### RequestVote

![](https://pic-bed-1309931445.cos.ap-nanjing.myqcloud.com/blog/image-20230123165517-cx40s4u.png)

arguments：

* term
* candidateId
* log相关的依旧不用管

result：

* term：投票者的term
* voteGranted

收到投票请求后，如果自身的term大于candidate的term，则拒绝投票，

如果voteFor为空则证明该term还未投过票，同意投票，论文中写的为voteFor为candidateId依旧同意投票，可以防止rpc返回时丢包，即投票者已经投票，但是candidate未收到，candidate最好对于重复投票的情况进行处理，使用一个map进行去重，防止一票统计多次的情况（正常情况下应该不会发生candidate接收到多票的情况，等实现时再试试怎么处理）

‍

**term处理**​

1. currentTerm < args.Term : 更新，（是否需要转变为follower?),但是仍给对方投票
2. currentTerm > args.Term：拒绝投票，将自己的term携带在reply当中，令对方更新term
3. leader检测到自身的term小于他人：转变为follower，在heartbeat中实现
4. 只要term增加，voteFor要么投自己，要么清空

**candidate停止**：在startElection当中处理

1. 票数超半，赢，变为leader，重置一下计时器
2. 平：啥都不干，重置一下计时器，等待超时
3. 他人为leader：

    1. heartbeat当中通过心跳检测（首选），收到了heartbeat就证明已经有leader了
    2. 投票结果带状态（感觉没必要）

log依旧不用管

### AppendEntries

![](https://pic-bed-1309931445.cos.ap-nanjing.myqcloud.com/blog/image-20230123171030-26h2zyl.png)

在lab2A当中只需要负责完成心跳相关的操作即可，因此log相关的一改忽略即可

**Arguments：**

* term：
* leaderId
* entries：null

**Result**​

* term
* success

当参数中的term（调用者） < currentTerm return false，其他全是log相关的，不管了

需要定义一个`AppendEntriesHandler`​的方法来在接收到心跳之后对计时器进行重置

‍

‍

**别忘了实现GetState()**

![](https://pic-bed-1309931445.cos.ap-nanjing.myqcloud.com/blog/image-20230123205738-uo5jyy2.png)

‍

### ticker

**timeout**​

分为election timeout和 heartbeat timeout：

* heartbeat timeout：These messages are sent in intervals specified by the heartbeat timeout，应该理解为一个时间间隔，即leader多久发送一次心跳信号
* election timeout用于检测在由follower转变为candidate，以及candidate多久没有选出一个新的节点

个人认为heartbeat可以设置为一个固定的间隔，多久发送一次，election timeout为一个随机值，防止同时选举，此外heartbeat timeout 应当小于 election timeout

## 实现

实现上对着figure2一条条捋即可，做一个"MapReduce"，将论文中的rules shuffle给对应的函数，最后满足一条条规则给拼接满足

首先定义一个ticker，用于计时处理各种任务，理论上为程序的入口函数，后续的所有的操作都通过ticker定时来触发，最简单的实现方式为以心跳为周期，发送心跳，并且不断去检测是否需要选举

```go
func (rf *Raft) ticker() {
	for rf.killed() == false {

		// Your code here to check if a leader election should
		// be started and to randomize sleeping time using
		// time.Sleep().
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

这样存在的问题是如果心跳周期为100ms，则200-300ms内超时的均会在300ms时被检测到，从而导致一同选举的问题，解决方案为单独开一个周期10-30ms的协程去单独探测`electionTimeout`​，不过目前并没有因为一同选举而导致的选举超时问题，因此暂且先用最懒的实现方式

lab2A主要实现的有两个功能，发送心跳和请求投票，分别对应`AppendEntries`​,`RequestVote`​,定义的相关函数有：

* 发送心跳：

  * `appendEntries`:入口函数，由ticker调用，负责遍历所有的节点，调用`leaderSendAppednEntries`​发送心跳
  * `leaderSendAppendEntries`​：负责一次心跳的发送，收到响应后检查，处理term等
  * `sendAppendEntries`:rpc调用
  * `AppendEntries`​：被rpc调用，从节点响应心跳信息
* 选举：

  * `startElection`:入口函数，由ticker调用，负责遍历所有节点，调用`candidateSendRequestVote`​请求投票
  * `candidateSendRequestVote`:负责一次投票的处理，term，票数的处理，决定是否成为leader等
  * `sendRequestVote`:rpc调用
  * `RequestVote`:被rpc调用，决定是否投票

调用关系：

![](https://pic-bed-1309931445.cos.ap-nanjing.myqcloud.com/blog/C534484440E247816F4FE92E3167B488-20230125235843-b7ly099.png)

### 锁相关

go中不存在可重入锁的概念，因此需要保证对于一个锁只lock一次，防止死锁

![](https://pic-bed-1309931445.cos.ap-nanjing.myqcloud.com/blog/BA4B00506CA00A7D75C0436C9E164BC7-20230126000421-gt0b8k2.png)

‍

使用`time.Ticker`​确实是容易出错，到处reset有点把自己reset晕了，单独的`time.Sleep`​虽然不严格，但是也能满足功能上的要求，能够稳定通过测试

再就是锁相关，多层调用加上rpc之后有点混乱，最好捋清楚都在哪进行调用，此外，用于进行测试的`getState()`​由于涉及到读取状态，因此也需要加锁，之前`getState()`​没加锁反而能通过一定的测试，加锁之后直接死锁

### 重置定时器：

根据学生指南中所说：

> 确保你在图2描述的地方准确地重置你的选举计时器。具体来说，你只应该在以下情况下重置你的选举计时器：a）你从当前的领导者那里得到一个AppendEntries RPC（即，如果AppendEntries参数中的任期已经过时，你不应该重启你的计时器）；b）你正在开始一个选举；或者c）你授予另一个对等体一个投票。

‍
