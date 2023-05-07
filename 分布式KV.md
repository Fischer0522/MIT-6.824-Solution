---
title: 分布式KV
date: 2023-03-18T15:10:39Z
lastmod: 2023-03-19T11:23:53Z
categories: [Distributed System,MIT 6.824]
---

# 分布式KV

借助的MIT的6.824来谈一谈对于基于raft的分布式KV的理解，虽然非常遗憾没能够通过Lab4B的所有测试，在部分细节上没有能够处理好，但是对于如何通过Raft算法来构建一个分布式的KV存储服务也有了一定的理解。分别讨论一下Lab3的单Raft集群，无Shard的实现方式和Lab4的ShardKV的实现方式。

## ShardKV

### 整体架构

首先明确系统的运行方式：一开始系统会创建一个 shardctrler 组来负责配置更新，分片分配等任务，接着系统会创建多个 raft 组来承载所有分片的读写任务。此外，raft 组增删，节点宕机，节点重启，网络分区等各种情况都可能会出现。

* 对于集群内部，我们需要保证所有分片能够较为均匀的分配在所有 raft 组上，还需要能够支持动态迁移和容错。

* 对于集群外部，我们需要向用户保证整个集群表现的像一个永远不会挂的单节点 KV 服务一样，即具有线性一致性。

整个KV存储的整体架构和设计：

* 首先整个分布式KV的mini-raft依赖于在Lab2当中实现的底层的Raft：

  * 通过Raft的Leader选举来确定整个集群的Leader，来对外提供服务，client只能够与Leader server之间进行通信
  * 通过Raft的Log- Replication来完成各个节点之间的数据同步
  * 通过Raft的日志的持久化来保证节点宕机之后能够恢复至原来的状态，并通过Snapshot进行优化，节省存储空间并快速启动
  * 通过WAL的方式，任何操作先写入到Raft的日志当中，包括只读的Get请求，最终按照写入日志的顺序来执行，给予客户端响应，以保证线性一致性
  * 一个上层的server节点对应一个底层的raft，raft所选举出的Leader即为server集群中的Leader,server通过向raft 询问自己状态，来确定自己是否为Leader
  * 底层的Raft和上层的server通过一个channel进行通信，server通过`start`​来想Raft写入一条日志，`Snapshot`​将当前server状态的字节数组向下交给raft，请求其裁剪日志，存储快照。而在server当中有一个单独的后台协程去监听channel当中是否有commit了的日志，当监听到commit的日志，则读取其日志，将对应的操作应用于server层，并给予客户端以响应

* 其次整个KV存储为CS结构的，即在server端进行对数据的处理或者存储操作，client端负责与用户进行交流，接受用户所发出的请求。此处所描述的client端要区别于如redis客户端等独立的客户端。mini-raft的客户端应当作为整体存储的一部分，用户和client通过网络的方式进行通信，client再将对应的请求处理并发送至server端。

  ![](https://pic-bed-1309931445.cos.ap-nanjing.myqcloud.com/blog/image-20230318152429-53fb2z5.png)

* 整个mini-raft所提供的为分片服务，即不同的key会存储在不同的对应的节点上，整个DB分为多个shard，每个server节点负责维护一个或者多个shard（但是一个shard只能够由一台服务器进行服务），所有的shard共同组成一个完整的DB，当client需要向对应的group去发送请求，找到其中的key，如果client请求中的key并不是由当前的goup所负责的，则group需要拒绝掉此次请求。
* 既然进行进行分片操作，则需要一个shard controller，来生成一份配置，确定哪个group 去服务哪一个shard。

  * client去读取这个配置，来确定自己的请求应当发向哪一个group。
  * group也向shard controller去读取配置，来确定自己所服务的是哪个shard

  因此最终的client-shard controller -server的架构如下：

  ![](https://pic-bed-1309931445.cos.ap-nanjing.myqcloud.com/blog/image-20230318222116-tqbz2qv.png)

  ![](https://pic-bed-1309931445.cos.ap-nanjing.myqcloud.com/blog/image-20230318223359-an1bqst.png)

‍

### Client

Client在实现上比较简单，Lab4B原生所提供的代码就已经可以实现最基础的功能。

在Lab3的client的基础上只要添加读取配置，并且计算出该Key所对应的shard，再根据配置去向负责该shard的group发送RPC即可。每次发送完请求就去重新读取一遍最新的配置，保证始终向最新的配置去发送RPC

在提供的代码的基础之上，可以像Lab3那样去记住一个group的Leader，每次发送RPC只向Leader	去发送RPC，减少尝试的次数，进行一定的性能优化，可以维护一个Map，key为shard，value为LeaderId，而当因为配置更新或者Leader重新选举产生的变化再去重新进行遍历尝试，如果获取到了OK的响应则证明找到了正确group和正确的Leader，否则读取最新的配置的shard->group的关系，遍历新的group以找到该group的Leader。

总的来说，Client只需要满足以下的需求：

* 当RPC的响应为OK或者ErrNokey时，记录当前的LeaderId，并且直接返回
* 当RPC的响应为ErrWrongGroup时，需要结束对当前group的遍历，停止此次请求，去拉取最新的配置，重新发送请求
* 当收到ErrWrongLeader时，则证明当前的group为正确的group，继续遍历当前的group去获取到正确Leader
* 缓存每个shard的LeaderId以减少发送RPC的次数请求

而如果不追求性能只保证正确性的话，则整个请求的发送过程即为：根据当前自身拥有的最新的配置，不断的进行groups-servers的一个二维遍历，只有收到了OK或者ErrNoKey的响应才返回。每次请求结束后再去拉取一次最新的配置。

```go
type Clerk struct {
	sm       *shardctrler.Clerk
	config   shardctrler.Config
	make_end func(string) *labrpc.ClientEnd
	clientId int64
	seqId    int
}
```

```go
func (ck *Clerk) Get(key string) string {
	ck.seqId++
	args := GetArgs{
		Key:      key,
		ClientId: ck.clientId,
		SeqId:    ck.seqId,
	}
	for {
		shard := key2shard(key)
		args.KeyToShard = shard
		gid := ck.config.Shards[shard]
		if servers, ok := ck.config.Groups[gid]; ok {
			// try each server for the shard.
			for si := 0; si < len(servers); si++ {
				srv := ck.make_end(servers[si])
				var reply GetReply
				ok := srv.Call("ShardKV.Get", &args, &reply)
				if ok && (reply.Err == OK || reply.Err == ErrNoKey) {
					return reply.Value
				}
				if ok && (reply.Err == ErrWrongGroup) {
					ck.config = ck.sm.Query(-1)
					break
				}
				// ... not ok, or ErrWrongLeader
			}
		}
		time.Sleep(100 * time.Millisecond)
		// ask controler for the latest configuration.
		ck.config = ck.sm.Query(-1)
	}

	return ""
}
```

### Shard Controller

Shard Controller所负责的为决定哪一个group去负责哪一个Shard，当有一个Group加入到整个ShardKV的集群当中之后，为其分配一个Shard，而当一group离开整个集群之后，撤销为其分配的Shard，将该Shard分配给当前还存在于集群当中的其他的Group，保证每一个Shard始终有一个Group在为其提供服务。

在级别上Shard Controller和存储节点为同级的存在，同样使用Raft来进行复制，独立于存储节点运行，在实际的应用当中，应当部署在单独的服务器集群上，在多个服务器上部署server端和Client端。多个server形成一个Raft集群整体，而Client单纯是为了连接服务器，之间并不存在什么关系。因此在架构上也比较简单：

![](https://pic-bed-1309931445.cos.ap-nanjing.myqcloud.com/blog/image-20230318212513-065t146.png)

为实现此功能，需要四个API:

* `Join`:Join实现将一个group添加到集群当中，并且应当给其分配一个对应的Shard
* `Leave`:Leave实现将一个group移除集群，它之前所负责的Shard需要重新分配个其他的Group进行服务
* `Query`:查询当前Shard的分配情况，提供一个参数来代表想要拉去的版本，用于client和server来拉取配置使用
* `Move`:将一个Shard从一个group移交至另外一个group负责

Shard Controller本身的设计并不复杂，只是进行一个Shard的分配工作，将Shard分配给对应的Group进行处理，被分配至该Shard的Key都由对应的Group进行存储和查询的工作。

在实现上主要有以下的几个要求：

* 每个加入到集群当中的group都需要负责一至多个Shard，不能够存在空闲的Group的情况
* 一个Shard只能由一个group进行负责，但一个group可以负责多个Shard的存储
* Shard需要首先负载均衡，即负载最大的group和负载最小的group之间所负责的Shard的数量差距最大为1

### Client

在四个API的实现上比较简单，在逻辑上和Lab3基本一致，基本上直接照搬过来即可。记录Leader节点，向其发送RPC，得到正确回应后则返回，否则设置继续尝试下一个节点，直至找到正确的Leader

```go
func (ck *Clerk) Join(servers map[int][]string) {
	ck.seqId++
	args := &JoinArgs{}

	// Your code here.
	args.Servers = servers
	args.SeqId = ck.seqId
	args.ClientId = ck.clientId

	serverId := ck.leaderId
	for {
		// try each known server.
		var reply JoinReply

		ok := ck.servers[serverId].Call("ShardCtrler.Join", args, &reply)
		if !ok {
			Debug(dError, "C client send JoinRPC to server[%d] failed", serverId)
			serverId = (serverId + 1) % len(ck.servers)
		}
		if reply.WrongLeader {
			serverId = (serverId + 1) % len(ck.servers)
		}
		if ok && reply.WrongLeader == false {
			ck.leaderId = serverId
			return
		}

		time.Sleep(100 * time.Millisecond)
	}
}
```

### Server

先讨论一下单Raft集群的架构，适用于Lab3的KV和shard controller

Server通过一个单独的Raft集群实现，需要注意的是：

* 一个server节点对应一个Raft节点，server通过`Start`​ `Snapshot`​向Raft去写入日志，存储快照，server监听 channel以获取raft层日志的commit信息。
* 需要满足线性一致性：

  * 只有Leader才能够对外提供读写服务，以防止进行本地读从而读取到replica上陈旧的数据
  * 所有的操作包括读操作都需要写入到Raft当中，最终按照Raft日志写入的顺序去执行
* 需要在server层定义一个SeqId进行数据的去重，来保证同一条请求只会执行一次，考虑这样一种情况：server端接受了一条请求后将其写入日志，之后发生宕机，客户端长时间没有得到server的响应，重新向其发送了一条请求，而此时之前宕机的Leader重启，不仅重新接受了新的请求，同时之前写入到Raft当中的请求也没有丢失，此时如果不加以限制则会导致该请求被重复执行了两次。因此设置一个SeqIdMap进行去重。

Server端的逻辑也基本上和Lab3的server端基本一致，主要分为以下几部分：

* RPC函数，和Client端的API相对应 Join Query Leave Move，并且监听一个channel，等待请求按照raft的日志的写入顺序一次执行完成，以保证线性一致性。并且设置一个超时定时器，以保证如果底层的Raft发生了宕机，进行了重新的Leader选举，能够快速给Client响应。
* 后台协程`ApplyMsgHandler`​，负责监听Raft channel，以获取commit了的Raft日志，将其应用至当前server的状态机，并且通过一个channel告知RPC函数raft日志的处理结果。
* Join、Leave、Query、Move各个操作的对应处理函数，实现具体的逻辑

因此整个的处理流程如下：

1. client向server Leader发送一个请求
2. server确认自己为Leader能够响应请求后将请求封装成一个Op，写入到raft当中，监听对应的日志index的channel，等待该日志commit后由ApplyMsgHandler进行处理
3. ApplyMsgHandler监听raft层的channel，等待日志的commit信息,获取到commit的日志之后对其进行处理，调用JoinHandler/LeaveHandler（Lab3则是进行Put/Append),将处理完成的消息放入该日志index所对应的channel当中，通知server处理的结果
4. server拿到了ApplyMsgHandler处理的结果之后根据结果给Client以响应

![](https://pic-bed-1309931445.cos.ap-nanjing.myqcloud.com/blog/image-20230318222439-0odmn3k.png)

#### 负载均衡

通过负载均衡以实现各个Group所负责的Shard的数量大致上一致，从而保证不会所有client发来的请求都打到一个group上：

![](https://pic-bed-1309931445.cos.ap-nanjing.myqcloud.com/blog/image-20230318223601-pnyrjzy.png)

![](https://pic-bed-1309931445.cos.ap-nanjing.myqcloud.com/blog/image-20230318165117-790s48u.png)

对于Shard Controller核心功能就是在保证为每个group都分配至少一个shard的基础上去实现负载均衡，即各个Group的负载差值不超过1

在实现方式上最初所想的是维护一个类似LRU链表的数据结构，但是只能在Leave时去选取一个最久没有进行分配的group对其进行分配，该逻辑并不适用于Join操作，Join又需要去选取最新分配过的group去进行剥夺，并且之后看测试用例的要求时差值不超过1，因此LRU的形式也无法保证绝对的平均，正确性得不到保证，遂放弃。

因此最终的方案就是在Join和Leave时以相对平均的方式进行一次负载均衡，一次到位。统计当前Shard的分配情况，得到一个[gid -> count]的Map，避免一个group加入到集群当中一段时间后再额外进行负载均衡的问题。

对于负载均衡在实现上还有几个细节：

* Go本身的Map的遍历是不确定性，而在实现负载均衡时需要消除这种不确定性，即对于相同的集群状态，在不同的机器或者不同时刻Shard分配的方案要均为一致。可以通过先对Map取出后进行排序生成一个固定顺序的slice来消除这种不确定性。在实现上因为本身Shard也只有区区十个，Group的数量只会少于10，因此偷懒直接使用冒泡排序。
* 在排序上按照负载大小进行排序即可，并且由于负载的平均值通常不是整数，即无法做到完全平均，会有一部分比另外一部分多1的负载，由于涉及到Shard中数据的迁移，希望尽量少的进行Shard迁移，因此就让负载较高的保持+1的状态。负载较低的保持0的状态

负载均衡过程：

1. 在实现负载均衡上我采用的是求出平均值的整数部分和余数部分，余数如果不为零即代表无法进行均分，代表有几个group的负载需要大于平均值的整数部分。如果当前的group的坐标小于余数则该group属于大于平均值的那一类，将target设置为average + 1
2. 之后遍历所有的GroupMap，设置Target，确定是否需要 +1，如果当前的Group的负载大于target则遍历所有Shard，寻找分配给该group的shard，撤销掉Shard至 < target，在Shard数组中将其标记为0（Shard->gid)
3. 遍历所有GroupMap，设置Target，确定是否需要+1，如果当前Group的负载小于target则遍历所有Shard，寻找刚才撤销分配的Shard，将其重新分配给该Group，直至不再小于Target。

```go
func (sc *ShardCtrler) loadBalance(GroupMap map[int]int, lastShards [NShards]int) [NShards]int {
	length := len(GroupMap)
	Debug(dLog, "S%d before loadBalance groupMap is %v shard is %v", sc.me, GroupMap, lastShards)
	ave := NShards / length
	remainder := NShards % length
	sortGids := sortGroupShard(GroupMap)

	// 先把负载多的部分free
	for i := 0; i < length; i++ {
		target := ave

		// 判断这个数是否需要更多分配，因为不可能完全均分，在前列的应该为ave+1
		if !moreAllocations(length, remainder, i) {
			target = ave + 1
		}

		// 超出负载
		if GroupMap[sortGids[i]] > target {
			overLoadGid := sortGids[i]
			changeNum := GroupMap[overLoadGid] - target
			for shard, gid := range lastShards {
				if changeNum <= 0 {
					break
				}
				if gid == overLoadGid {
					lastShards[shard] = 0
					changeNum--
				}
			}
			GroupMap[overLoadGid] = target
		}
	}

	// 为负载少的group分配多出来的group
	for i := 0; i < length; i++ {
		target := ave
		if !moreAllocations(length, remainder, i) {
			target = ave + 1
		}

		if GroupMap[sortGids[i]] < target {
			freeGid := sortGids[i]
			changeNum := target - GroupMap[freeGid]
			for shard, gid := range lastShards {
				if changeNum <= 0 {
					break
				}
				if gid == 0 {
					lastShards[shard] = freeGid
					changeNum--
				}
			}
			GroupMap[freeGid] = target
		}

	}
	Debug(dLog, "S%d after loadBalance is groupMap %v shard:%v", sc.me, GroupMap, lastShards)
	return lastShards
}
```

### Shard Server

对于单集群的raft，Raft所提供的只有高可用性，即当存在节点宕机之后，依旧可以迅速选举出Leader来接替原本Leader的位置，对外提供服务，但是整个集群当中只有Leader在对外提供服务，所有的读写请求全部由Leader承担，并且Leader还要负责集群当中的Log Replication，发送心跳信息，等其他工作，在性能上相比于单机server只减不增。

ShardKV就是要解决这个单Raft集群的性能较低的问题，ShardKV进行了水平方向上的拓展，每个Shard负责一部分Key，例如Shard0负责所有以"a"开头的Key，Shard1负责所有 以“B”开头的Key，Shard只需要处理属于自己的Key即可，能够有效Leader的负载而一个Group又会去负责一个或者多个Shard，如果Shard负责的Key分配得当，各个Group又达到负载均衡，则能够有效降低Leader的负载。在最理想的情况下，一个Group只需要负责一个Shard，以获取最佳的性能。

ShardKV Server的大致架构如下：

* 一个Replica Group对应一个Raft集群，通过Raft来保证自身集群的高可用性
* Replica Group去向Shard Controller去拉取配置，确定自己所负责的Shard
* 一个Replica Group负责一个或者多个Shard，但是各个Group之间应当做到负载均衡，每个Group负责数量差距最大为1的Shard数量
* 整个集群为动态配置，会时不时有新的Group加入或者当前当前的Group因为自身原因，比如Raft节点宕机过多导致无法对外提供服务而下线
* 分片迁移：对于要下线的配置，需要将其上的数据重新分配给当前还在集群当中的Group，上线的Group需要在进行负载均衡时从其他的Group获取一个Shard以及该Shard对应的所有的数据
* 需要一个ApplyMsgHandler去监听Raft层的commit信息
* 一个MigrationAction去监听当前的所有的Shard状态，决定是否需要进行分片迁移
* 一个GcAction去监听当前的所有的Shard的状态，决定是否需要对不属于自己的Shard进行GC

#### 存储方式

在存储上，选择的方式为分片存储，即一个Shard使用一个`map[string]string`​去维护，并且绑定一个状态，封装成一个结构体，这样做主要有以下几种考虑：

* 如果只保证正确性的情况下，可以允许冗余存储，即允许存储不属于自己的Shard的KV，但是需要保证不影响读写和分片迁移。
* 在分片迁移时只发送当前属于自己负责的Shard当中的Key
* 为应对节点宕机的情况，需要一个SeqIdMap进行去重
* 如果不进行冗余存储，而是选择及时GC，那么也应当不阻塞当前正常读写请求，GC交给后台协程去检测和提交日志

因此最终所确定的存储结构为：

```go
type ShardComponent struct {
	ShardStatus int
	MemDB       map[string]string
	SeqIdMap    map[int64]int
}
statemachines map[int]*ShardComponent
```

可以保证在分片迁移时只传输自己当前负责的Shard的数据，GC时也不会影响正常的读写请求。

#### Shard迁移

之后再去考虑shard迁移的问题：

* 在哪进行检测，发现需要进行迁移
* 由谁检测，发送方还是接收方？
* RPC都需要发送什么
* 如何阻断新的请求

对于shard迁移，主要有两种实现方式，即为Pull和Push，使用Pull的方法时，当一个group检测到自己要服务于新的shard，但是自己还未获得新的shard的Map时，此时向其他的group去尝试pull数据，而Push操作，即group检测是否有之前服务于自己但是在新配置当中不服务于自己的shard，如果有，则需要将其push给其他的节点。

**Pull**

对于Pull方法，首先当group轮询检测到当前group处于pull状态时，发送RPC尝试拉取其他的shard。获取到shard之后，将其写入到raft当中，以达成共识，最终保证线性一致性的情况下改变状态机。如果要实现GC的话，之后在确认自己应用了相关的shard之后还需要再发送一次RPC，去告知对方的group将对应的shard的数据删除。因此一来一往对于一个shard而言就是两次RPC

**Push**

push即为轮询到push之后发送一次RPC，要求对方去应用自己的shard，当RPC获取到响应之后，即可确认将自己原本的shard删除，在RPC的结果上处理较为麻烦，但是可以节省一次RPC的开销，总体来说在实现难度上二者基本一致。

**RPC调用端**

最终采用了Pull的方法，在调用端的大致的逻辑顺序如下：

1. 一个协程在不断的轮询，向shard controller去请求最新的配置，当获取到新的配置之后，就将其写入到raft当中，以保证线性一致性的更新配置，防止配置和请求之间违反线性一致的问题。
2. 当`applyMsgHandler`​从raft当中获取到了对应的配置更新的请求，此时根据新的配置来判断当前的group所处于的状态，将group的状态从`normal`​转换为`Pull`​或者`Push`​，之后再将新日志进行应用
3. 另外一个协程在不断的轮询检测当前的group的状态，一旦检测到group处于pull的状态，即找到需要pull的所有shard，发送RPC来pull数据

    > shards应当为一个map[int][]int的数据结构，map的key为gid，即从哪个group去获取shard，而value即为该group所负责的所有的shardId，例如，当前g1负责s1 s2,g2负责 s3 s4,g3负责 s5，此时g1 g2下线，在新配置中就将 s1-s4全部重新分配给g3，此时的数据结构即为:[g1->[s1,s2],g2->[s3,s4]]，因此需要并行的从g1 g2处拉取对应的shard集合。
    >
4. 当RPC成功返回时，即可写入一条日志来请求将pull到的shard应用到自身的状态机之上，注意，为了保证各个shard之间的迁移过程互不干扰，因此需要保证分别写入日志，即对g1 g2拉取分两条写入到日志当中。这样在g1的shard应用到状态机之后如果g1之后重新分配了则g1即可重新对外提供服务，g1无需等待g2
5. 当从raft中获取到command之后，即可将shard应用到状态机当中，此时即可改变自身的状态为gcing，表明开始进入GC状态，而此时自身数据已经为最新的状态，已经可以对外提供读写服务，只是对方需要进行GC
6. 另外一个协程去单独检测GC相关的状态，如果检测到当前的状态为GCing，此时再去想对方group发送RPC，告知对方我已经完成了状态机的更新，此时你可以进行GC，删除掉之前的shard了，当得到正确的RPC响应之后，即可将自己的状态重新标记为normal，回归正常状态。

参数：发送当前的版本号和和需要获取的ShardIds

**RPC被调用端**

RPC被调用端负责将调用端需要的shard从自身获取出，并返回给调用端，

1. 先判断自身的config是否小于发送来的configNum，如果小于发送来的configNum，则证明自身的版本落后，可能存在脏数据，甚至当前需要拉去的分片都不是由自己所负责的，因此直接返回`ErrNotReady`​
2. 遍历发送来的ShardIds，深拷贝自身数据，放入到响应结构体当中，结构为一个Map，shardId->Shard，将kvDB和SeqIdMap一同拷贝

在 apply 分片更新日志时需要保证幂等性：

* 不同版本的配置更新日志：仅可执行与当前配置版本相同地分片更新日志，否则返回 ErrOutDated。
* 相同版本的配置更新日志：仅在对应分片状态为 Pulling 时为第一次应用，此时覆盖状态机即可并修改状态为 GCing，以让分片清理协程检测到 GCing 状态并尝试删除远端的分片。否则说明已经应用过，直接 break 即可。

#### 节点状态

根据上述shard迁移的需求，节点大致需要以下四种状态：

* **Normal**：正常且为默认状态，如果该shard处于自己的group的管理，则可以对外提供读写服务
* **Pull**: 即当前的group的正在管理此分片，但是此分片之前由其他的group负责，需要将数据从其他的group拉取过来之后才可对外提供正常服务
* **Push**：此分片在新配置当中已经不再受当前的group所管理，需要将其push给（被pull）其他的分片才可对外提供正常的读写服务
* **GCing**：当前的group已经获取到了最新的shard，完成了pull操作，可以对外提供读写服务，但是对方group（该shard的上一任group）还未将原本的shard数据删除，需要通过RPC告知对方将其删除之后才可将状态从GCing变回normal​

![](https://pic-bed-1309931445.cos.ap-nanjing.myqcloud.com/blog/image-20230319111209-s0w9gb8.png)

#### Operation

而对应的Op则有四种，分别为：

* ClientOp：用于处理Client端发送而来的Get/Put/Append请求，也只有ClientOp需要进行去重的操作：
* ConfigOp：从Shard Controller处拉取了新配置将其应用于自身
* ShardOp：从其他的group拉取到了Shard，将其应用于自己的状态机上
* GcOp：拉取方检测到GcOp后则可以清除自身的GC标记，变为Normal状态，推送方检测到GcOp则可以将自身的冗余Shard进行清除

**ApplyMsgHandler**​

```go
func (kv *ShardKV) applyMsgHandler() {
	for msg := range kv.applyCh {
		if msg.CommandValid {
			kv.commandHandler(msg)
		}
		if msg.SnapshotValid {
			kv.snapShotHandler(msg)
		}
	}
}
func (kv *ShardKV) commandHandler(msg raft.ApplyMsg) {

	//kv.mu.Lock()
	//defer kv.mu.Unlock()
	switch msg.Command.(type) {
	case ClientOp:
		kv.doClientCommand(msg)
	case ConfigOp:
		kv.doConfigCommand(msg)
	case ShardOp:
		kv.doShardCommand(msg)
	case GcOp:
		kv.doDeleteShardsCommand(msg)
	case EmptyOp:
		// nothing to do
	}
	if kv.needSnapshot() {
		kv.mu.Lock()
		snapshot := kv.encodeSnapshot()
		kv.mu.Unlock()
		//	Debug(dSnap, "S%d created a snpshot lastApplied:%d snapshotIndex:%d", kv.me, kv.lastApplied, msg.CommandIndex)
		kv.rf.Snapshot(msg.CommandIndex, snapshot)
	}
}
```

#### 配置更新

* 需要保证当前并不存在正在迁移的shard才可以进行配置更新，即所有的节点全部为normal状态
* 配置的更新需要逐步进行，每次去获取下一个配置，之后写入到raft当中以获取共识，需要在拉取配置和更新配置的两个阶段均需要进行校验，即新配置比旧配置的configNum 大1才进行更新
* 额外保存lastConfig用于计算需要拉去的组

#### 分片清理

**RPC发送端**

分片清理相关的协程不断轮询所有shard的状态，当检测到当前状态为GCing时，则遍历所有状态为GCing的shard，调用RPC告知对方已经完成了shard的迁移，请求对方将对应的shard删除，并且将自身的状态从push转变为pull，开始对外提供正常服务

**RPC接受端**

首先检测调用段的版本，如果自身版本高于调用者，则证明已经删除掉了，因此直接返回OK即可，否则写入一条日志来达到共识以进行删除

应用日志同样需要保证同配置版本，如果当前状态为GCing，则证明已经在正常对外提供服务，而如果为push，则将其恢复为默认状态，并且分配一个全新的kvDB和SeqIdMap

‍
