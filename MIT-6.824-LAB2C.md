---
title: MIT-6.824 LAB2C Persistent
date: 2023-01-27T16:53:38Z
lastmod: 2023-01-28T23:37:00Z
categories: [Distributed System,MIT 6.824]
---

# LAB2C

2A 2B写得好，2C简单写一下就能过了，按照figure2 和给的代码提示写就可以了，比较简单，十分钟拿下

![](https://pic-bed-1309931445.cos.ap-nanjing.myqcloud.com/blog/image-20230128232638-hejmc87.png)

需要进行持久化的字段：

* currentTerm
* votedFor
* log[]

因此只要涉及到这几个字段改变的地方就需要调用`persist()`​进行持久化写入操作

* `setTerm()`​ ：进行term的跟随，设置为follower，votedFor清空，因此需要进行持久化
* `RequestVote()`​：投票之后设置了`currentTerm`​和`votedFor`​
* `startElection()`​：开始选举之后投自己一票，currentTerm++
* `AppendEntries()`​：冲突进行覆写时日志改变，持久化，追加日志时日志改变，持久化
* `Start()`​：上层应用调用raft的`Start()`​时，新增一个命令，写入一条日志，持久化

`readPersist()`​在`Make()`​函数时进行调用，读取进行恢复

```go
func (rf *Raft) persist() {
	Debug(dPersist, "S%d: [%d] is persisted, STATE: %v ", rf.me, rf.me, rf.log.String())
	w := new(bytes.Buffer)
	e := labgob.NewEncoder(w)
	e.Encode(rf.currentTerm)
	e.Encode(rf.votedFor)
	e.Encode(rf.log)
	data := w.Bytes()
	rf.persister.SaveRaftState(data)
	// Your code here (2C).
	// Example:
	// w := new(bytes.Buffer)
	// e := labgob.NewEncoder(w)
	// e.Encode(rf.xxx)
	// e.Encode(rf.yyy)
	// data := w.Bytes()
	// rf.persister.SaveRaftState(data)
}
```

```go
func (rf *Raft) readPersist(data []byte) {
	if data == nil || len(data) < 1 { // bootstrap without any state?
		return
	}
	r := bytes.NewBuffer(data)
	d := labgob.NewDecoder(r)
	var currentTerm int
	var votedFor int
	var log Log
	if d.Decode(&currentTerm) != nil || d.Decode(&votedFor) != nil || d.Decode(&log) != nil {
		Debug(dError, "S%d failed to read persist")
	} else {
		rf.currentTerm = currentTerm
		rf.votedFor = votedFor
		rf.log = log
	}
	// Your code here (2C).
	// Example:
	// r := bytes.NewBuffer(data)
	// d := labgob.NewDecoder(r)
	// var xxx
	// var yyy
	// if d.Decode(&xxx) != nil ||
	//    d.Decode(&yyy) != nil {
	//   error...
	// } else {
	//   rf.xxx = xxx
	//   rf.yyy = yyy
	// }
}

```
