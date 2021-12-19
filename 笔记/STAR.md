# STAR

## 1. 流程图

### 背景

1. dev环境流程图展示很慢
2. prod环境流程图展示存在大量bug，影响生产判断，每次查日志浪费很多时间

### 目标

1. 提高流程图匹配准确性
2. 提高流程图匹配效率

### 行动

#### 优化1.0 读取history接口，进行流程图匹配算法简化

history线性表是流程图真正的历史执行记录，具有可靠性

匹配过程由深搜workflow-chart、同时匹配history线性表改为根据线性表匹配对应workflow-chart节点，对于并行任务尤其有效（原先的并行任务会扫描线性表直至匹配，造成很大浪费）。

这样做会带来一个问题：逻辑节点的连接问题。因为history中不会记录逻辑节点，而只会记录真正有意义的算法模块，没有逻辑节点的连接，单凭非逻辑节点无法得知真正的运行流，也无法展示出完整的流程图。比如遇到很多循环嵌套时，整个流程图全部节点的连接就会出问题，但是业务逻辑暂时不会那么复杂

优化完匹配方式后，速度并没有预想中的那么快，继续排查，利用火焰图分析瓶颈点，优化了：1.频繁unmarshal（go-cache解决） 2.用户owner频繁查db（redis解决） 3. 频繁查时区（代码变量暂存解决）

模块CPU占用率下降很多，但是存在以下问题：1. gocache单机，无法分布式，加入广播机制过于复杂 2. **获取History接口本身很慢**，返回时间30ms-9s不等

优化提升不明显，有必要彻查history接口慢的原因

#### 插曲

cadence对于我们来说就是黑盒，中间干了什么很不好查。抱着尝试的态度，与开发人员交流。得到的反馈是，**history接口是一个重量级接口，本身速度就很慢**，体现在：1. history实际上是cadence内部使用的一个接口，用于历史重放，用户调用不合适 2. 夹杂了大量信息，这些信息并不是我们所需要的。想要的没有，不想要的一大堆，无法定制化。  

这时面临两条选择：1. 自己做数据持久化 2. 利用cadence开发人员建议的query

最后使用query代替history，进行查询，本质上是利用了cadence的history-replay机制，但是更加轻量灵活。

#### 优化2.0 利用query-handler，进行流程图展示重构

query原理：利用history-replay机制，将workflow的各个状态重新记录至map中，然后利用之前注册的query-handler去查询内存中的信息。

query优点：

1. 轻量，可以只将想查询的内容放入内存中，不关注也不用过滤其他信息
2. 可定制，可以包括逻辑节点，将流程图展示变为100%准确
3. 性能更好，支持缓存。对于同一个decisionTask，cadence不会重新进行历史重放，而是会选择一个已经保存着历史重放对应内存的workflow-worker直接进行查询

利用query机制，流程图无需匹配，直接进行节点连接即可。

#### 优化2.5 query-handler存在的一些问题

##### **stickyScheduledToStartTimeout**

**stickyScheduledToStartTimeout**指decisionTask在stickExecute过程中**所能接受的耗费的最长时间**，如果过了这个时间（可能是网卡了之类的），decisionTask**就视为原来的sticky-worker已经宕机**，就不会再进行sticky策略。

- 如果设置的**stickyScheduledToStartTimeout**时间较短（如1s），可能会出现由于workflow-worker并发Query过多，延时过高，导致某次响应decisionTask时间超过1s，此时cadence-server会清除decisionTask的sticky，此时视为workflow-worker已经宕机（实际可能没有宕机），从而造成之后所有的有关该workflow的请求都不会执行stick策略，每一次Query都会新开一个goroutine，造成goroutine泄漏、高latency等现象。

  压测时出现了workflow-worker内存暴涨的现象就是此原因

- 如果设置的**stickyScheduledToStartTimeout**时间较长（如1h），可以避免上述问题，但会出现另一种问题。时间设的过长，如果workflow-worker真的宕机，decisionTask仍将等待1h，此时会造成卡死现象，如果没有新的decisionTask出现来更新sticky绑定关系，当前decisionTask会一直等待，直至1h超时。外部的具体现象表现为，如果一个workflow block在一个activity上很久，此时对应的workflow-worker宕机，之后的decisionTask(包括Query，activityCompleted，都会卡顿1h)

  staging环境上出现了query一次30秒的情况，是因为我将**stickyScheduledToStartTimeout**设置为30s，staging环境的workflow-worker经过CICD重新部署时，workflow-worker视为宕机，再进行query的时候会默认等待30s，然后才进行history-replay，给用户的观感很差

最终，**stickyScheduledToStartTimeout**设为5s

##### 外部Redis

为了进一步提高接口并发性能，加入Redis层。

### 反思

1. 找到真正的业务瓶颈，才能对症下药。这里的业务瓶颈不是json unmarshal之流，而是慢接口
2. 要多与开发者交流



## 2. SKIP

### 背景

生产环境中有很多算法模块占用大量资源，时常存在想要跳过某模块的想法，但是之前的流程过于死板，缺少鲁棒性。

### 目标

跳过某模块运行，同时保证整个系统逻辑自洽。包括：跳过scheduled/running/init的task，reset类操作时不重新进行操作。

### 行动

#### skip 1.0

skip属于异步操作，利用cadence提供的信号机制实现。

有两种不同的skip

1. 跳过scheduled/running 的task，此类task正在阻塞。
2. 跳过init的task，此类task还没开始。

处理逻辑如下：

```go
====================old=======================
...
f = workflow.ExecuteActivity() // 放入队列
err = f.Get() // 阻塞
...
====================new=======================
...
// taskname：本次处理的task的名字
new skipChan //新建接受skip信号的chan
new skiptasks//新建记录待skip信号的list
if taskname in skiptasks{
  skiptasks.delete(taskname)
}else{
  for{
    select:
    case: signalname<-skipchan //signalname:应当跳过的taskname
    	if taskname = signalname break;// 适用于跳过正在scheduled/running的task
    	else skiptasks.add(signalname)// 适用于正在init的task，将signalname插入skiptasks，然后由于for死循环再次进入阻塞
    case: f.Get()
  }
}
```

#### skip 2.0

增强鲁棒性，兼容reset，每一个skipped的任务都必须被放入队列中，否则无法拿到eventId，也就无法reset

### 反思

暂无





## 3. workflow 钩子

### 背景

人工任务重置时需要手动清理状态，耗时耗力，增加沟通交流成本

### 目标

设计一种方案，在人工任务回退时自动清除状态，减少人力消耗

### 行动

流程图中植入钩子，在定义中加入，用户可选择加钩子或不加钩子。进行回退操作时，找到reset cursor 和当前起始点，向上线性遍历，一直发送请求

### 反思

http请求的设计主要是考虑可扩展性，将任务编排系统和业务关系解耦，后续考虑将其加入消息队列































