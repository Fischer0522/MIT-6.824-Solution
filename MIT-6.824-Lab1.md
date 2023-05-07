---
title: MIT-6.824 Lab1 MapReduce
date: 2023-01-13T18:22:06Z
lastmod: 2023-03-26T22:41:25Z
categories: [Distributed System,MIT 6.824]
---

# Lab1

测试数据：

```
pg-being_ernest.txt pg-dorian_gray.txt pg-frankenstein.txt pg-grimm.txt pg-huckleberry_finn.txt pg-metamorphosis.txt pg-sherlock_holmes.txt pg-tom_sawyer.txt
```

map 和reduce的函数通过插件的形式加载，而文件的处理可以从顺序执行的内容抄一部分过来，因此需要实现的就是MapReduce本身的架构，将`mrsequential.go`​集成在一起的功能拆分给master和worker（一部分worker负责map task，一部分负责reduce task）

![](https://pic-bed-1309931445.cos.ap-nanjing.myqcloud.com/blog/image-20230114215900-2gtlr2g.png)

## 任务处理

**master**

负责读取user传出来数据，将对应的文件分发给map worker，打开文件读取之后直接将读取到的[]byte数组发送给map进行处理

```go
	intermediate := []mr.KeyValue{}
	for _, filename := range os.Args[2:] {
		file, err := os.Open(filename)
		if err != nil {
			log.Fatalf("cannot open %v", filename)
		}
		content, err := ioutil.ReadAll(file)
		if err != nil {
			log.Fatalf("cannot read %v", filename)
		}
		file.Close()
		// use rpc to assign a map task
		// use hash to choose a worker
		worker := hash(filename)
		rpc.call(worker,content,reply)

	}

```

**worker**

根据论文，worker本身为同质的，并不分为 mapworker或者reduceworker，只根据master进行rpc调用分配任务时来确定究竟执行maptask还是reducetask。因此worker的大致结构如下：

```go
func worker(content,string,reply *string,task) {
    switch task:
	case "map":
	    doMapTask(content string,reply *string)
	case "reduce":
	    doReduceTask(content string,reply *string)
    }
}
```

```go
func doMapTask(content string,reply *string) {
    kva := mapf(filename, string(content))
    file.Write(kva)
    *reply = fliepath
}
```

```go
func doReduceTask(content string,reply *string) {
    intermediate := file.Read(content)
	i := 0
	for i < len(intermediate) {
		j := i + 1
		for j < len(intermediate) && intermediate[j].Key == intermediate[i].Key {
			j++
		}
		values := []string{}
		for k := i; k < j; k++ {
			values = append(values, intermediate[k].Value)
		}
		output := reducef(intermediate[i].Key, values)

		// this is the correct format for each line of Reduce output.
		fmt.Fprintf(ofile, "%v %v\n", intermediate[i].Key, output)

		i = j
	}
  
  
}
```

## MapReduce框架



![](https://pic-bed-1309931445.cos.ap-nanjing.myqcloud.com/blog/image-20230114230911-vujy1up.png)

1. 当master和worker都启动之后，master维护一份worker的列表，来表明worker当前的工作状态，是否为空闲，正在执行的任务等，初始时master通过与worker进行一次rpc通信来确认worker的状态，如果没问题则初始化为空闲状态
2. 初始化完成之后，用户调用MapReduce，即向master节点发送数据，将需要进行work count的文本发送给master，这里直接按照一个worker一个文本的方式进行分配，通过hash函数来确定对应的worker，创建task，每一个txt文本对应一个map task，分配给一个worker，此时task列表当中只有map task，
3. task创建好之后，master通过rpc的方式去调用`worker()`​​​，并指定类型为mapTask，使其执行`doMapTask()`​​​
4. 分配到map task的worker 通过`Map()`​​​产生一系列的k-v键值对，k为单词，v为1，将结果写入到本地进行存储，将文件的名称返回给master节点用于创建reduce task

    一共有m个map worker和nReduce个redcue worker，一个map worker会产生n个中间文件，通过 `ihash(key) % nReduce`​​的方式确定key究竟分发给那个reduce worker去处理，通过hash的方式可以保证相同的key会发送给同一个reduce worker，因此产生中间文件共有m * n个，tmp-i-j，代表为第i个map worker产生并发送给第j个reduce worker。
5. master接收到 map worker的返回结果之后则证明分配的map task已经完成，修改当前对应的数据表示，等待所有map任务全部执行完毕。
6. 等待所有的map任务全部分配完毕之后就可以进行到reduce任务的阶段了，master向nReduce个worker去分配reduce任务，master向reduce worker传递需要传给他的文件名称
7. 后续reduce worker根据文件名称进行读取，读取到不同map产生的中间文件，对所有的key进行一次排序，对于相同的key传入一个reduce函数当中，计算key的数量，即为数组的长度，后续将所有的reduce的结果都输出到一个`mr-out-*`​​当中，作为该reduce worker的输出结果，

### master

1. 自身初始化
2. 从user处接受一系列的文件名，每一个文件对应创建为一个map task，加入到task队列当中
3. 等待worker向master请求任务，此时master只分配 map task或者wait task
4. 处理map worker的处理结束的响应，将对应的task从slice中移除，表示该任务已经完成，同时添加一个reduce任务到队列当中
5. 当队列中已经不存在map task时，将阶段转换为 reduce阶段
6. 接受worker请求task的rpc，分配reduce task或者wait task
7. 处理worker发送来的心跳信息（暂时不考虑）

## master与worker的关系

主要需要考虑的是master和worker究竟谁是主动谁是被动的关系，即通过谁来调用谁

* 一种是首先worker上线后向master去进行注册，之后master在处维护一个worker列表，后续有任务时master从任务列表中取出任务分配给空闲的worker去完成，即master去调用worker完成任务，论文中所写的也就是这种方式，

* 另外一种方式为worker自身处于一个循环当中，worker不断的向master去请求任务，master接受到了worker的任务请求后就去检查任务队列，选择一个任务去进行分配，此时worker变为了主动的一方，而master被worker去被动调用，master本身并不去维护worker的状态，即worker为stateless的，只有worker发送了请求master才知道worker的状态，master只需要记录有哪些任务需要分配即可

  worker为无状态的则master无法去检测worker是否已经挂掉了，但是超时时间到了worker依旧没有返回结果就可以认为该节点已经挂掉了，（可以采用一个后台协程去几秒一轮循来确认是否有任务超时了，也可以等到worker到了之后在去判断task是否过期，就像redis中ttl的两种实现方式一样）此时如果还有其他的worker来请求任务就可以将该任务重新分配给新的worker

  ‍

暂时能想到的就这么多，可以先尝试写一下试试

**crash**​

由于worker为无状态的模式，因此master只去监控task的完成情况，，即task超时之后会重新进行分配，在worker方，如果直接挂掉倒还好，但是如果没挂掉的话依旧会产生同名的中间文件，在真实的分布式环境下不在同一个主机节点上则不会去读取，而对于lab的在同一个节点上运行的话，如果超时了后续又恢复了，会对之前的结果产生影响，因此worker本身去进行超时检测，如果超时主动放弃任务，不向master做出完成任务的应答

此外，对于中间文件的起名问题，不应当用task的id，以防止多个文件之间相互干扰，可以在master维护一个全局的id，每次分配一个任务就对其自增1，即可保证超时的任务不会对其产生影响

## 实现

### 实现Word Count

#### 串行

对于串行执行的情况，无非就是首先将所有文章的单词全部变为`word-1`​的键值对形式，然后，排序后对单词进行归类，相同的单词作为一个数组交给reduce函数，reduce函数将其的长度统计出来即可，写为伪代码的形式

```go
words := os.Open(file)
kvs := mapf(words)
sort(ByKey(kvs))
result := make(map[string]int)
for ..... {
    get the same keys as a array;
    output := reducef(arrKv)
    result[key] = output
}
return result;
```

map：

```go
map(String key, String value):
// key: document name
// value: document contents
for each word w in value:
EmitIntermediate(w, "1");
```

单独的map本身并不涉及到任何的分布式相关的理念，只计算word的出现次数，每出现一个word就像其中添加一个[word] 1的键值对

将不同的word发送给对应的reduce函数，reduce数1的个数即可，即统计一下长度

```go
reduce(String key, Iterator values):
// key: a word
// values: a list of counts
int result = 0;
for each v in values:
result += ParseInt(v);
Emit(AsString(result));
```

#### 分布式环境

在分布式的情况下，Map和Reduce两个处理函数本身不需要又任何变化，只需将原本的串行执行变为master接收到任务之后，分配给多个worker并行执行，之后经过`shffule`​过程，交给对应的reduce worker，在让reduce worker来并行执行，最终汇总出结果

**master**

因此master只需要提供一个`AssignTask`​对worker进行任务分配和一个`ConfirmTaskDone`​对worker完成任务的结果进行接受即可，在worker无状态的情况下，master对worker的情况并不知晓，因此二者均为被动被worker rpc调用

**worker**​

worker只需根据在master分配的任务做出对应的动作即可，根据任务的类型来判断究竟是执行map 还是执行reduce 或者休眠和退出

```go
func Worker(mapf func(string, string) []KeyValue,
	reducef func(string, []string) string) {

	// Your worker implementation here.

	// uncomment to send the Example RPC to the coordinator.
	//CallExample()
	count := 0
	for {
		res := AskForTask()

		fmt.Printf("---------------------------------handing the %d work----------------------------\n", count)
		count++
		switch res.TaskType {
		case mapTask:
			doMapTask(mapf, res)
		case reduceTask:
			doReduceTask(reducef, res)
		case waitTask:
			fmt.Println("waiting here...")
			time.Sleep(1 * time.Second)
		case exitTask:
			return
		default:
			panic(fmt.Sprintf("unknown task type"))
		}

	}

}

```

```go
func AskForTask() Response {
	isAssigned := false

	response := Response{}

	ok := call("Coordinator.AssignTask", &isAssigned, &response)
	if ok {
		fmt.Printf("get the task:%v\n", response)
	} else {
		fmt.Printf("call failed!\n")
	}
	return response
}

```

### 任务分配机制

首先定义master相关的结构体：

```go
type Coordinator struct {
	// Your definitions here.
	assignedId     int
	files          []string
	phase          int // 0 for map phase 1 for reduce phase, 2 means all works finished
	nMap           int
	nReduce        int
	mapTasks       []Task
	mapTaskLeft    int
	reduceTasks    []Task
	reduceTaskLeft int
	muTasks        sync.Mutex
	interFiles [][]string
}

const (
	mapPhase    = 0
	reducePhase = 1
	finishPhase = 2
)


```

代码写的比较丑陋，基本就是刚刚能够通过test的程度，毫无美感可言

* assignedId为一个自增的Id，用于分配给task作为task的id，后续用于解决 worker crash的问题，
* files和interFiles分别为最初输入给master的文件和map worker产生的中间k-v文件的名称，由于都在同一台物理机上，因此直接传输名称进行读取即可，无需发送文件
* mapTasks和reduceTasks存放需要进行任务
* mapTaskLeft和 reduceTaskLeft为我偷懒的产物，懒得去优化代码事实从队列中移除已经完成的task，所以直接使用两个变量去统计
* phase表明当前整个MapReduce所处的阶段：

  * MapPhase则只分配Map Task
  * ReducePhase只分配Reduce Task
  * FinishPhase即先通知各个worker终止进程，当所有的worker均终止进程后自身也终止进程，结束整个MapReduce任务

**Task**

```go
type TaskStatus struct {
	TaskType   int
	isTimeOut  bool // lazy detect
	isAssigned bool
	isFinished bool
}
type Task struct {
	fileName  []string
	id        int
	status    TaskStatus
	startTime int64
}

```

对于task来表示分配的worker的任务，绑定该task需要的文件，开始时间，最后在定义一个结构体来表明当前task的状态和类型：表明是否超时、是否已经分配、以及是否完成

由于master为被动的分配任务，即master进行分配一定是因为有worker来进行索取任务，因此正常情况下任务在分配任务的前后只能处于以下几种状态：

||超时|分配|完成|
| ---| ------| ------| ------|
|1|F|F|F|
|2|F|T|F|
|3|F|T|T|
|4|T|T|F|
|5|T|T|T|

对于超时，采用了`lazy detect`​的方式，即当worker进行索取task时再检测是否有超时的任务，如果有，则将其分配的标志位设置为`F`​，撤销对其的分配，进行重新分配，就和redis对过期的键的处理方式一样（不过redis为了节省内存后台同时还会有一个线程在轮循检测是否有key过期），而Task即便过期了也不能被移除还要继续进行分配，因此使用`lazy detect`​在我看来是较好的解决方案

**任务分配**

对于无状态的worker，主要采取的方案为worker索取 -> master分配的方案，master不清楚worker当前的状态，只等待worker来获取task，

* 如果在MapPhase，则遍历mapTask队列，分配任务并检测超时，每次分配一个任务即终止遍历，对于其他的超时的task等到后续遍历到了再去进行处理，如果遍历完队列也未能找到可以分配的Task，则证明所有的task已经全部分配完毕，但分配出去的task也可能有因为worker宕机而执行失败的风险，因此让当前的worker休眠等待后续进行MapTask的重新分配或者等待ReduceTask即可
* 如果为ReducePhase，则遍历reducePhase队列，方案和上面一致，reduce task同样存在因宕机而执行失败的可能，因此同样休眠等待所有的task全部执行成功
* 当所有的reduce task也全部执行完毕后，则证明整个MapReduce任务已经执行完成，master通知所有的worker结束进程，按照论文中master应当最后再向用户去返回结果，但是lab中为了测试方便master直接退出即可，通过脚本来聚合所有reduce worker产生的结果去验证即可

### 信息交换

> The master keeps several data structures. For each map  
> task and reduce task, it stores the state (idle, in-progress,  
> or completed), and the identity of the worker machine  
> (for non-idle tasks).

按照论文中所写的实现方式应当为master维护这一个关于map task和 reduce task的两个task队列，以及worker的状态的数据结构，因此当一个worker上线时，向master进行一次通信，在master处进行一次登记，注册一个该worker的id，交给master来管理，后续应当master去根据worker的状态去主动调度worker来给worker来分配task，而当master发现worker执行task超时时master可以主动向worker发送一个消息来主动终止掉worker剩余的task（如果worker是超时而不是彻底宕机的话）

而由于我的实现方式采用了`stateless worker`​的方式，在通信方式上就简单了很多：

* worker只在申请task和task完成时才主动向master进行通信，用于在master端去对任务的状态进行更新

  * 申请task：

    * id为分配的taskId，index为task在队列当中的位置，用于后续返回时master来寻找task
    * nReduce用于在map task中进行hash取模运算

    ```go
    type Response struct {
    	FileName []string
    	TaskType int
    	NMap     int
    	NReduce  int
    	Id       int
    	Index    int
    }
    ```
  * 任务结束确认

    * TaskIndex用于master节点在寻找task
    * InterFile用于map task结束后将中间的k-v文件返回给master，用于后续分配个reduce worker

    ```go
    type TaskResult struct {
    	TaskIndex int
    	InterFile []string
    	TaskType  int
    }
    ```
* master从不主动向worker进行通信，master需要传递给worker的信息只在worker申请task时携带传递给worker，通过`Response`​中的`TaskType`​执行任务还是休眠或者终止

### fault tolerance

#### worker失效

**超时检测**

按论文中所述，master会周期性的去ping每个worker，规定时间内如果收不到worker的回应则可以将worker标记为失效，后续需要分配task时则不选择该worker，并且对分配给该worker的task进行重新分配。

而由于我的实现在master出采用了`stateless worker`​的实现方式，master也无法去ping worker，因此只需对超时的task进行标记即可，每次worker要求分配task时就会遍历整个队列，检测过期，如果过期的就进行重新分配

而使用这种`lazy detect`​+ `stateless`​我认为是没有什么问题的，就算master监控到了有任务过期但是却没有worker能去给分配task又有什么用呢，就像redis那样，如果不考虑内存占用的问题，如果没有客户端来读取就算key过期了又能怎样呢？并且确实这种实现方式确实能够通过测试。

```go
if !c.mapTasks[i].status.isFinished && c.mapTasks[i].startTime != 0 && time.Now().Unix()-c.mapTasks[i].startTime > 10 {
	c.mapTasks[i].status.isTimeOut = true // try to assign it to a new worker
	c.mapTasks[i].status.isAssigned = false
	fmt.Println("map task", c.mapTasks[i], ":is timeout!----------------------------------")
}
```

**垃圾任务**

对于出问题的worker节点要么为直接宕机，要么因为某些因素延迟执行完成，worker产生的残缺文件就会影响到最终的结果读取，这里我想到了两种解决方案，：

* 一是通过自增的`assignId`​的方法，每次分配任务时都给该任务指定一个`assignId`​，后续task完成之后返回该id交给master，用于进行shuffle过程，这样只会去读取成功返回的task产生的文件，出错map worker产生的残缺文件就会被略过，不会影响结果
* 如果最终的结果是由reduce worker返回给 master去聚合产生结果的话通过该方案同样可以将错误的结果给略过，如果最终结果是自己手动进行聚合，那么reduce也可以这样进行，但是测试用例当中的reduce worker的`mr-out*`​为通过脚本通配符进行匹配最终聚合出正确的结果，因此该方法就没法正确解决了
* 因此最佳方案为使用临时文件的方式，最开始先写入一个临时文件，如果执行过程中挂掉该文件也就不存在，最终确保能够正确输出时再将其改名为`mr-out-*`​的格式，供脚本进行聚合操作

  ```go
  	oname := "mr-tmp-out-" + strconv.Itoa(res.Id)
  	dir, _ := os.Getwd()
  	ofile, err := ioutil.TempFile(dir, oname)
  
  ```

  ```go
  	ofile.Close()
  	os.Rename(ofile.Name(), "mr-out-"+strconv.Itoa(res.Id))
  ```

map节点最好也改为使用临时文件的格式，防止产生的残缺文件留在磁盘上占用空间，但是能够稳定过测试了也懒得改了（还是懒）

![](https://pic-bed-1309931445.cos.ap-nanjing.myqcloud.com/blog/image-20230116200743-bxnnhyc.png)

## 并发控制

由于存在data race的部分基本上都是单独写或者读写混合，使用读写锁也没法做任何优化，因此就使用了最简单的`Sync.Mutex`​做并发控制，函数开始时加锁，defer解锁，后续可以重构为更加go风格的channel来实现，master单独开一个协程去监听一个channel，所有存在并发风险的操作全部在改协程中完成。

## summary

至此6.824的lab1也算全部完成了，读论文加写代码大约用了差不多4天的时间，从最开始仅仅能看懂到各方面的都设计的差不多大约用了两天左右的时间，写代码加debug又用了两天，代码部分实际上没有多大的工作量，主要还是不太熟悉这样纯靠日志来debug的方式花费了不少的时间，没有IDE也没有gdb，master节点能通过ide跑起来但是rpc调用断点不起作用，worker直接没法用IDE启动，使用IDE则插件加载失败。一行行去看日志确实是花费了不少的时间（有个变量手残穿错了，看了好几个小时。。。）

我的实现基本上就是属于堪堪能通过测试的写法，很多地方都写的简陋至极，等到保研前夕再考虑进行重构一下顺带着复习复习，现在只能马不停蹄的继续往下赶了。
