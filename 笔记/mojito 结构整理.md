# mojito 结构整理

worker端发送polled请求给mojito，mojito回复polled，所以mojito的存在意义就是告诉worker什么时候可以开始工作，如果worker一直收到的polled都是null，那就无法运行里面的execute指令，也就无法执行本来的业务。

# 1. gin框架

## 1.1 输入流的处理和解析函数

```go
func (s *TaskDefServer) Get(c *gin.Context) {
// 1. Param  解析参数
// Param returns the value of the URL param.
// It is a shortcut for c.Params.ByName(key)
//     router.GET("/user/:id", func(c *gin.Context) {
//         // a GET request to /user/john
//         id := c.Param("id") // id == "john"
//     })
// 注意，:后面跟着的是参数的名字，但是不一定在最后，可以在中间，比如
// router.GET("/user/:abcd/details",func(c *gin.Context) {
//     id := c.Param("abcd") // id == "john"
// })
// 那么我此时访问“user/hx/detail”，就可以得到id = hx,而abcd只是一个中转

  taskName := c.Param("name")

// 2. Query 查询参数
// Query returns the keyed url query value if it exists,
// otherwise it returns an empty string `("")`.
// It is shortcut for `c.Request.URL.Query().Get(key)`
//     GET /path?id=1234&name=Manu&value=
// 	   c.Query("id") == "1234"
// 	   c.Query("name") == "Manu"
// 	   c.Query("value") == ""
// 	   c.Query("wtf") == ""  
  versionName := c.Query("version")
}
```

## 1.2 router

```
taskDef := e.Group("api/v1/task-defs")
{
   taskDef.POST("", server.GTaskDefServer.Create)
   taskDef.PUT("", server.GTaskDefServer.Update)
   taskDef.GET("/:name", server.GTaskDefServer.Get)
   taskDef.GET("", server.GTaskDefServer.List)
}
```

1. **怎样区分POST，GET，PUT ？**
2. ` taskDef.POST("", server.GTaskDefServer.Create)` 后面装的是handler，handler可以用匿名函数，也可以调用其他地方的函数。
3. 不同的http请求底层统一调用：

```go
(请求种类，相对路径，handler)
func (group *RouterGroup) handle(httpMethod, relativePath string, handlers HandlersChain) IRoutes {
   absolutePath := group.calculateAbsolutePath(relativePath)
   handlers = group.combineHandlers(handlers)
   group.engine.addRoute(httpMethod, absolutePath, handlers)
   return group.returnObj()
}
```



## 1.3 c.JSON()

`c.JSON(http.StatusOK, polled)`是向前端发还是向worker发 ----------谁调用了接口就向谁返回，这是面向一次http请求的，一次http请求是点对点的，所以这个只是返回而已。



# 2. global

global.go全局参数初始化

```go
func init() {
   // init config and store
   store.InitStore()

   // init mojito: CadenceClient workflowClient
   mojito.InitClient()
}
```



# 3. 任务调度过程

## 3.1 一次简单的任务调度过程

1. 前端开启一次workflow，task1被systemworker放入cadence的队列中
2. 负责task1的worker（简称worker1）一直在访问mojito的接口，mojito将其转发，进行cadence client的调用，查看cadence队列中有没有task，在前端触发workerflow开始时，mojito终于查到了。
3. mojito将拉到的任务信息封装为`polled`发送给worker1，worker1的while循环被打破，进入并执行相应业务代码，执行业务代码前开启子进程，对mojito发送心跳，保持通信
4. worker1任务结束，向mojito发送http的delete请求，请求内包装了运行结束的output等众多信息（日后补）
5. mojito接收到请求，对其进行解析，调用cadence client，更新cadence队列中的task信息。cadence队列中的task是什么时候更新的，如果此时才更新，就说明之前cadence队列中一直是处于schedule，那worker岂不是可以一次性拉到很多task？
6. 更新完毕后，更改数据库的内容，保证数据一致



前端的schedule状态什么时候更新为started？cadence内部task如果被拉到，就自己把task的状态更改为started，此时也就回答了上面的问题4，也就是保证了并发的worker进程并不会造成冲突，在cadence的task被其中一个worker访问到时，状态已经改变了，其他的请求就不能再拉到队列中的任务了。

## 3.2 拉任务的细节

worker端一直发起轮询，mojito读取请求信息并进行`polltask`尝试

1. 解析worker端发来的请求参数【taskName，pollerID，Domain】
2. 转入multiplexerClient，选择相应的Client，这里因为初始化采用了Cadence，所以由CadenceClient来执行poll
3. `t, cadenceErr = cadenceClient.PollTask(ctx, req)`

4. 返回的是什么东西？polledData{inputdata，workflowID，TaskID，WorkflowName，TaskName，...}
5. 第4步返回的input是怎么来的？存在cadence中，每次取出来交给worker的时候就已经配好了，**但是是怎么把input放进cadence的呢**？
6. 怎样把返回的东西“发放”给worker？通过http请求，也就是`c.JSON(http.StatusOK, polled)`

## 3.3 心跳机制

心跳机制是定时发送一个自定义的[结构体](https://baike.baidu.com/item/结构体/3709485)(心跳包)，让对方知道自己还活着，以确保连接的[有效性](https://baike.baidu.com/item/有效性/4463171)的机制。

谁向谁发送心跳，查看谁的存活

猜想：在运行业务程序的时候同时发送心跳，让mojito知道自己还活着

如果不进行心跳，会发生什么？业务时间过长导致连接断开？哪里会触发连接断开的代码？









# 动态子流程

涉及到的参数：

- makeOutputTask：专门用来产生output，作为下一个的input。output主要包括

  - **successPerCent** wait阈值

  - **dynamicTasks**   [ {} , {} , ... ] 提供了subWorkflow的名字和标识符等**外部信息**
  - **dynamicTasksInput**  { {} , {} , ... }提供了subWorkflow 的输入

- 产生子流程的task
  - **task['inputData']** 就相当于makeOutputTask 的output
  - **dynamic_task** 定义子流程的所有参数，包括
  - **input** 子流程的输入





子流程和父流程的交互问题，目前猜测子流程结束之后会调用mojito接口，mojito调用并更新数据库查看相应子流程对应的父流程麾下的所有子流程的完成情况，根据此完成情况决定waitTask是否进行





1. 父流程wait逻辑怎么实现的
2. 动态子流程失败以后，怎么通知父流程
3. 子流程需要提前定义好还是在父流程中实现(提前定义好，父流程还是根据子流程的名字去触发子流程的)
4. success percent 怎么实现
5. 运行一个小demo





# http Task

一次愚蠢的，，

把http的put请求地址写成localhost，但是http请求本身是dev端的服务器发的，他根本找不到localhost:7900

解决方法：put到自己的IP/在本地运行systemworker







# 由删除任务引申出的数据库操作



**涉及到网络请求的一些方面**

1. warpWorkflowDef为什么要包装，不知道有啥用，update里面用到了



**业务相关**

现在的情况是，有两个表，一个是current_workflow_def，一个是workflow_def

我们平时显示的都是workflow_def，这里面存储了所有版本，会挑选一个最新版本（id最大的）用于显示。而current_workflow_def 并不知道有啥用

到底要删除哪个表









## 删除涉及到两张表

历史记录不删，**每次都往上加**

用户能看到的所有版本都由current来读取





核心冲突：软删除会导致create语句创建原有的workflow时会失败，出现duplicate key

create时：

如果能查出来数据，说明之前删了（软删），这时把current version+1并置为null（也可以用update），把history 中重新插入一条新的记录即可

如果查不出来数据，说明不存在，这时在current和history中直接插入新的记录即可



update时：

没有任何区别，直接update



delete时：

直接软删除current，history不管



可能出现的问题：创建一个新的结果版本号却不为1，这是因为之前已经创建过，怎样提示前端



# 并行任务(暂缓)

动态并行任务 retry 会执行全部的任务，现在想只执行失败的任务

1. 动态并行任务的子任务和父任务的交互

   - 会被pod拉走，然后pod再往cadence中放入subtask吗 【是的】

   - 通过怎样的数据结构才能将子任务和父任务结合起来？在流程的哪个步骤进行子任务和父任务的交互？【无论是子流程还是子任务，最后的结尾都是一个http任务，统一走`api/v1/sub-workflow`】

   

2. retry

   retry 怎样将所有subTask都retry

3. 父任务开子流程和父任务开子任务的区别到底有多大呢？可能体现在以下几点：
   1. 子流程所需要的定义参数更多，相当于一群任务
   2. 子任务最后要通知父任务，子流程最后也要通知父任务
   3. ⚠️ 没有父流程，只有所谓的父任务的概念，父流程中只有父任务是有意义的





​     A             重新启动所有的                  

b  b.... b      定点启动Failed的     

​     C			 重跑所有Failed的



玉神：

A什么时候算完成

A起10000个过程中（黄），b运行完第一个（绿），C还在等待（灰）



不使用动态并行任务，暂时用子流程代替，一个子流程里放一个任务。





# 流程图匹配

流程图：仅仅是一张图

流程：真实的运行流程，可能不会经过流程图中的每一个节点

流程图匹配：来将流程的真实执行状态和流程图对应起来

## 代码整体逻辑：

```go
1. 预处理
2. 建立流程图
3. 解析historyEvents，生成scheduleEvents
4. 根据流程图和scheduleEvents进行匹配
5. 进行善后
```

## 变量解释：

```go
historyEvents 				cadence-server中生成的记录所有events的表，包含所有event信息，我们只需要其中的一部分
resp.flowNodes 				流程图的各个节点
edges									记录了各个节点连接情况的有向图。map[A]map[B]C  表示A到B存在连接，C=0时表示这里建图时连接但是实际没有连接，C=1时表示这里确实连接
scheduleEvents				真正被activity-worker拉的event记录，也是真正被执行的event。后面流程图匹配就是匹配的这个表
taskInfo							存储整个workflow的所有信息
  type TaskInfo struct {
    taskEidToName map[int64]string
    taskEventID   map[string]int64
    taskStartTime map[string]int64
    taskEndTime   map[string]int64
    taskStatus    map[string]string						//存储了各个task的完成状态
    taskResult    map[string]interface{}
    taskAttempt   map[string]string						//映射				taskNmae--->尝试次数
    taskErrors    map[string]string
    taskErrorsMap map[string]map[string]string //chart
    taskEidToTask map[int64]string             //chart
    taskIdMap       []map[int64]workflow.TaskID //chart    ”存储了表征流程整体状态变化的节点“ 的切片 ，装了map。每一个map 只装一个键值对，映射 eventId--->taskID 
    // 什么样的eventId？ 1. 状态为DecisionFailed的eventId 2. historyEvents中最后一个eventId 【len(historyEvents)】
    // 每一次状态重启都会有不同的RunID，taskIdMap记录这些标志着状态重启的event。
    // 比如一次流程中有两个decisionFailed，taskIdMap记录3个eventId，分别表征三种状态，这三种状态的RunId是不一样的。
    // 
    // taskID 是一个结构体，如下：
    /*
      TaskID struct {
      WorkflowID string `json:"workflow_id"`
      RunID      string `json:"run_id"`
      TaskName   string `json:"task_name"`
      ActivityID string `json:"activity_id"`
      Attempt    string `json:"attempt"`
      }
      用于拼起来，作为整个的taskID，放到前端显示
    */


    taskIsCompleted bool										//workflow是否完成，在开始图匹配之前就已经设定好了
    failedTaskOwner string
    ResetEventId    int
    TaskIdStr       string
    // Input        *map[string]interface{}
  }
```

## 函数解释：

```go
建图函数：
AddSingleNode(resp.flowNodes, mataworkflow.tasks[i]) 										向resp.flowNodes尾部追加节点
addTemplateEdge(edges, nodeID_a, nodeID_b) 															向edges 中追加边，连接a-->b的边(向a的邻接表中加入b)
AddTemplateNode(resp.flowNodes, edges, mataworkflow.tasks[i])						=AddSingleNode+addTemplateEdge
																																				往当前流程图尾部追加单个节点,并从新节点引一条边指向虚空
GenTemplates(resp.flowNodes, edges, mataworkflow.tasks)  								往当前流程图尾部追加多个节点，追加至两个数据结构，1.resp.flowNodes   																																				 2. edges

解析函数：
scheduleEvents, childEvents = s.GetTaskInfo(historyEvents, &taskInfo, flowNodes, wfID, req.RunID)  由historyEvents解析出scheduleEvents

匹配函数：
matchTemplate(resp.flowNodes , edges , scheduleEvents , childEvents , taskInfo , eventIndex, nodeIndex, visitGain bool, taskCountBegin bool) (visitedEventIdx, visitedNodeIdx)
//将事件编号为eventIndex的event，从图中编号为nodeIndex的node开始进行匹配
//visitGain：能否更新当前递归所匹配到的数据
//taskCountBegin
//最终更新resp.flownode，返回visitedEventIdx, visitedNodeIdx
//visitedEventIdx:当前分支能匹配到的最深的event表index
//visitedNodeIdx:当前分支能匹配到的最深的节点index
//e.g.
/*
更新每个节点的：
1. status
2. NodeConf
3. TaskID（如果已经成功的task，会获得taskID，将taskInfo的taskId这个struct转换成string拼起来送到前端显示）
4. ResetEventID（不知道有什么用）！！！一般都是activity的前一个dicisionTaskCompleted的index，不知道有什么用
*/


解决了loop的诸多bug，例如
1. 无法更新循环节点状态
2. 只用Node.conf append，导致新的startTime不能及时覆盖老的startTime

```

## 建立流程图GenTemplates：

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gvkhxjsksnj60u0140ado02.jpg" alt="F47203B065B4FC9C2B2E7559A72FE3DD" style="zoom:30%;" />

## 匹配流程图matchTemplate：

resp.flowNode更新前

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gvkhxp9yqkj61000b675e02.jpg" alt="image-20210809135843466" style="zoom:50%;" />

resp.flowNode更新后

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gvkhxs4owwj613y0m2jve02.jpg" alt="image-20210809140006627" style="zoom:50%;" />

每个`event`都是historyEvent中的一个

`node`就是resp.workflowNode中的一个node

`scheduleEvents`: 已经处理的Events：绿色，黄色。不会加入诸如while节点之类的哑节点

`eventIndex`：scheduleEvents中的index，表征**scheduleEvents中要匹配的记录**

`nodeIndex`：node在图中的位置

`nextNodes`:当前匹配的Node在流程图中所连接的所有node的index组成的数组

`dstNodes`: 当前匹配的Node在流程中真正连接的所有node的index组成的数组

`runID`: 流程重启RunID会改变

流程图的node分为 simple和非simple，simple和http是会被记录到scheduled里面的，这两种节点的状态(node.resp)会在匹配的时候会发生变化，其他种类节点的状态都是在switch节点后才发生变化

只有匹配到simple节点event表才会往后+1，其余情况event都不会变化

## 关注两个event

decisionTaskCompleted     event

decisionFailed       				event

## BUG

![image-20210810191752987](https://tva1.sinaimg.cn/large/008i3skNly1gvkhxv9dqsj60l004a0t302.jpg)

1. 这样只要最后是completed，任意节点走到这一步就一定会匹配成COMPLETED，不管这个节点有没有真的匹配，但是对最后结果不影响
2. 流程图匹配是线性扫描，如果并行任务可能会发生任务的加入顺序和流程图中的节点顺序不同，匹配时会发生混乱【！！！】
3. 关于分支问题：1. 循环时无法更新循环的状态，而只是当成一个分支去遍历深度，实际上LOOPEND节点不应该是选择结构
   						   2. 如果底下长度过长，则一定要dfs所有的分支，找到里面最深的，可以尝试只寻找当前node的nextNodes中和event匹配的部分，而不进行dfs
4. 需求：展示dynamic_fork_join

## 流程图代码重构

### 1.数据结构

采用自顶向下的数据结构重构法

```go
resp，返回的就是这个：
      type FlowChartResponse struct {
        Summary          interface{}            `json:"summary"`
        SearchAttributes map[string]interface{} `json:"searchAttributes"`
        RunID            string                 `json:"runId"`
        Input            interface{}            `json:"input"`
        FlowNode         []*FlowNode            `json:"nodes"`
        FlowEdges        []*FlowEdge            `json:"edges"`
      }

Summary，好解决
SearchAttributes，好解决
Input，好解决

FlowNode，每个节点的信息
      type FlowNode struct {
        ID           string          `json:"id"`
        ResetEventID int             `json:"resetEventId"`								ok
        DataType     string          `json:"dataType"`										ok
        Status       string          `json:"status"`											ok
        Name         string          `json:"name"`												ok
        RefName      string          `json:"refName"`											ok
        NodeConf     []*FlowNodeConf `json:"conf"`												仅simple
        TaskID       string          `json:"taskId"` 											ok
        Attempt      int             `json:"attempt"`											仅simple
        CallbackUrl  string          `json:"callback_url"`								仅simple
      }
FlowEdges，节点之间的连接信息
      type FlowEdge struct {
        Source string `json:"source"`
        Target string `json:"target"`
        Used   bool   `json:"used"`
      }

```

### 2.性能分析

压测过程中采样。压测是为了看接口性能，采样是为了看接口瓶颈点

压测：

Go-wrk(未使用)

Vegeta

```shell
压测接口指令：
 echo "GET http://local.hdmap.momenta.works:7900/api/v1/workflows/package-lane-match:3d3fad71-943c-475d-aa25-f3c1c77dbcac/chart?workflowId=package-lane-match%3A3d3fad71-943c-475d-aa25-f3c1c77dbcac&run_id=77a8eb50-8a63-4a12-80ce-6d51ece14aee" | vegeta attack  -rate=10  -duration=30s |vegeta report
 
 rate：			1s发多少请求
 duration：	持续多少秒


压测报告：
Requests      [total, rate, throughput]  300, 10.03, 8.53
Duration      [total, attack, wait]      35.17998577s, 29.900097564s, 5.279888206s
Latencies     [mean, 50, 95, 99, max]    4.177504777s, 4.037548922s, 9.199635195s, 12.207337568s, 13.794506709s
Bytes In      [total, mean]              6161700, 20539.00
Bytes Out     [total, mean]              0, 0.00
Success       [ratio]                    100.00%
Status Codes  [code:count]               200:300

```



采样：

```shell
1. 引入采样工具，gin框架直接import："github.com/DeanThompson/ginpprof"
2. 开启服务器，进行压测	
3. 压测时对httpServer进行采样：go tool pprof http://localhost:7900/debug/pprof/profile\?seconds\=20
4. 查看采样报告 ：go tool pprof -http=:8082 ~/pprof/pprof.samples.cpu.005.pb.gz
```



分析（火焰图）：



https://zhuanlan.zhihu.com/p/7152906



### 3. 瓶颈与解决方案

瓶颈1：频繁的unmarshal

![image-20210827150804901](https://tva1.sinaimg.cn/large/008i3skNly1gvkhxzni85j61ko0u0aob02.jpg)

![image-20210827150416883](https://tva1.sinaimg.cn/large/008i3skNly1gvkhy59civj61260b8mym02.jpg)

加缓存，缓存workflow-meta

1. 解决workflowMeta的查询问题，key 只存 workflowID即可，因为workflowID就可以唯一表示流程图定义的用户信息，甚至只存workflowName就行，好吧只存workflowName就可以唯一标识流程图定义了。有一个问题就是如果用户改了流程图定义，这时候就要删除缓存了。

2. 使用go-cache的一个主要原因是因为redis里面的东西必须变成json才能缓存，这样我们每次还是要unmarshal，失去了加缓存的意义，加缓存的本意是为了让它能直接得到数据结构而不去进行繁杂的unmarshal

3. 小问题，如果更改了task定义，要去remove缓存吗，这里暂时设计成不要，因为任务定义并不在流程定义的userDef中

加缓存后性能如下

![image-20210827190643570](https://tva1.sinaimg.cn/large/008i3skNly1gvkhy9id5dj61gj0u015n02.jpg)

瓶颈2：getTaskOwner频繁查数据库

![image-20210827193006806](https://tva1.sinaimg.cn/large/008i3skNly1gvkhyf8qfwj61go0u0nai02.jpg)

同样是加缓存，更改任务定义时要删除缓存

优化后性能如下：

![image-20210827193743032](https://tva1.sinaimg.cn/large/008i3skNly1gvkhykf2s3j623q0u0wwi02.jpg)



### 4. 火焰图显示前后性能差别

优化前：

主要由四部分组成，总占比17.21%

![image-20210830115019128](https://tva1.sinaimg.cn/large/008i3skNly1gvkhypjrp5j61ki0u0h1i02.jpg)

优化后：

优化了其中的三部分，总占比10.77%

![image-20210830114911414](https://tva1.sinaimg.cn/large/008i3skNly1gvkhyu3ih5j61i60u0dsg02.jpg)

### 5. 压测报告

压测时有两个参数， -rate：每秒钟发送的请求数量  -duration：压测的时间

rate太大服务器会崩掉，具体体现为：

![image-20210830104537698](https://tva1.sinaimg.cn/large/008i3skNly1gvkhyzb2qkj60hs044t9a02.jpg)

如果读端关闭连接，写端继续写，第一次写，会收到RST，再写，报错`broken pipe`





### 6. 最终（暂定）结论

1. 优化了三处，1. 递归改为迭代 2. 为避免unmarshal加缓存 3. 为避免查数据库加缓存

2. 最终性能还是卡在 getDescription上，这个函数会调用cadence server，每次消耗时间50ms~9s不等

3. 单从CPU占用率上已经有了很大的提升，如下图所示

   ![image-20210831095144829](https://tva1.sinaimg.cn/large/008i3skNly1gvkhzjzlebj625c0u04in02.jpg)



# 钩子

1. 利用historyEvents的方法解析之前有多少已经运行的
2. 只要activityCompleted/activityFailed就要清空
3. 怎样拿出钩子的接口信息
4. 调用以后结束时判断标志
5. 怎样把api值封装到event中



最终代码：

```go
// 通过回调钩子清除状态
//	wfID：workflowID
//	runID：流程运行的runID，为空则表示当前流程
//	eventID：清除状态的起点，CleanStatus清除从eventID-->流程结束的所有状态
func (s *WorkflowTasksService) CleanStatus(wfID string, runID string, eventID int64) error {
	// 得到historyEvents
	historyEvents, err := mojito.GcadenceClient.GetWorkflowHistory(wfID, runID)
	if err != nil {
		return err
	}

	// 调用流程图匹配，得到每一个节点的真实数据
	// activityNameToNode：key：activityName value ：flowNode
	activityNameToNode := make(map[string]*types.FlowNode)
	resp, err := GworkflowService.GetFlowChart(wfID, runID)
	for _, flowNode := range resp.FlowNode {
		if flowNode.TaskID != "" {
			stringTemp := strings.Split(flowNode.TaskID, ":")
			activityName := stringTemp[len(stringTemp)-2]
			activityNameToNode[activityName] = flowNode
		}
	}
	// 从表中的reset起点开始线性向下扫描，回调每一个activityStarted的任务对应的url
	for id, event := range historyEvents {
		if id+1 >= int(eventID) && (event.EventType.String() == taskCompleted || event.EventType.String() == taskFailed || event.EventType.String()== taskTimeOut ){

			var scheduledID *int64
			switch event.EventType.String() {
				case taskCompleted:
					scheduledID = event.ActivityTaskCompletedEventAttributes.ScheduledEventId
				case taskFailed:
					scheduledID = event.ActivityTaskFailedEventAttributes.ScheduledEventId
				case taskTimeOut:
					scheduledID = event.ActivityTaskTimedOutEventAttributes.ScheduledEventId
			}
			activityName := historyEvents[*scheduledID-1].ActivityTaskScheduledEventAttributes.ActivityId
			if callbackNode , ok := activityNameToNode[*activityName]; !ok{
				continue
			}else if callbackNode.CallbackUrl=="" {
				// 该节点没有定义CallbackUrl，不调
				continue
			} else{
				data := requests.Json{
					"activityName": *activityName,
					"input":        callbackNode.NodeConf[2].Value,
					"output":       callbackNode.NodeConf[3].Value,
				}
				resp, err := requests.Post(callbackNode.CallbackUrl, data)
				if err != nil {
					return err
				}
				if resp.R.StatusCode >=400 {
					err :=  fmt.Errorf("bad request:url=%s,response:%s", callbackNode.CallbackUrl, resp.Text())
					logger.Error( err)
					return  err
				}else{
					logger.Infof("task "+ *activityName +" success")
				}
			}
		}
	}

	return nil
}

```

自己写了一个JSON解析，不过没啥用，尽量不要自己解析JSON，直接传interface即可

```go
自定义outputJSON解析，废弃
func (s *WorkflowTasksService) JsonHandler(input interface{}) string {
if output, ok := input.(string); ok {
   return output
} else if outputMap, ok := input.(map[string]interface{}); ok {
   outputBuffer := new(bytes.Buffer)
   fmt.Fprintf(outputBuffer, "{")
   for key, value := range outputMap {
      fmt.Fprintf(outputBuffer, "\"%s\"=\"%s\",\n", key, s.JsonHandler(value))
   }
   fmt.Fprintf(outputBuffer, "}")
   return outputBuffer.String()
} else if outputList, ok := input.([]interface{}); ok {
   outputBuffer := new(bytes.Buffer)
   fmt.Fprintf(outputBuffer, "[")
   for _, value := range outputList {
      fmt.Fprintf(outputBuffer, "%s,", s.JsonHandler(value))
   }
   output = strings.TrimRight(outputBuffer.String(), ",")
   output = output + "]"
   return output
}
return ""
}
```



# 权限接入与管理

## 1. 通过sdk访问权限系统

权限接入系统的sdk，连接权限控制系统，主要提供两个功能（通过向权限系统发请求得到响应来实现功能）：

1. 权限中间件，加在路由层为接口提供权限。调用接口前先通过此中间件向权限系统发请求，得到此次接口调用的权限，再判断能否调用接口。

   Golang中间件的工作方式：

2. 涉及到用户方面的函数，主要是获取token和username等

   如：`GetRequestUserName()`:根据token，放到header中变成cookie，发给权限系统，得到用户名

https://devops.momenta.works/Momenta/hdmap-workflow/_git/account-client-go?path=%2F&version=GBdev&_a=contents



## 2. 权限系统使用

权限域相当于是一个权限命名空间。有了域之后，每个域管理员只能管理自己所在域的资源权限、所在域的角色组、所在域的用户。具体来说，域有三个概念：

1. 用户域（管理域）——即用户可以管理的域。用户域分四种
   1. 普通用户——没有用户域。比如小红，用户域为空——没有任何域的管理权 
   2. 普通管理员——拥有用户域。假设张三，只拥有两个管理域——dwh(datawarehouse)、 osm ，他就可以有权限给大家要分配datawarehouse、osm两种域内资源的访问权限。
   3. 高级管理员——拥有用户域＋域最高负责人。假设李四，拥有两个管理域——dwh(datawarehouse)、 osm，他就是这两个域的最高负责人。
      1. 李四拥有让普通用户小红，拥有osm普通管理域的能力
      2. 但李四没有dwh、osm之外，其它域的管理权
   4. **超级管理员——拥有admin用户域。这是一个特殊的用户域。表示它有所有域的管理权（就是超级管理员）**
2. 角色域——角色可以关联一组用户，这组用户可以拥有多种策略，角色也有域的概念。
   1. 如果该角色只有osm 域，那么该角色就只能添加osm内资源访问权限，不能添加其它域资源的访问权
   2. **admin是一个特殊的角色域，它可以添加任何策略域资源的访问权限**
3. 策略域——所有资源都分属于不同的域。将所有的访问对象（如某个接口，某个数据库。。。）统一抽象成资源，策略域就是管理这些资源权限的钥匙
   1. 比如osm资源(`osm:*`)，属于osm域;
   2. 比如datawarehouse(dwh:*) 资源，属于datawarehouse域;
   3. 比如production-management-ui 页面访问　属于pmui 域(pm:*)
   4. **`\*:\**`是**一个特殊的域策略：它包含了所有域的策略

4. 权限域——每一个项目对应一个权限域，如mojito域，pm域



将用户添加到角色中，如hx和lly均属于admin角色，zqy属于user角色

将角色和策略绑定，就可以配置该角色允许访问的资源，如admin角色允许访问所有接口，user角色只能访问GET接口



# Cadence system worker

1. 所有的workflow 都是存储在cadence中？所有**被mojito start的**workflow都放在cadence中
2. system worker做的所有事情都是向cadence中发送消息，包括side-effect，包括activity



1. 

2. 注册并启动workflowWorker【register and start】【这一步仅仅是启动workflow的worker，并没有启动workflow。】

   1. 通过mojito查到所有的workflow-def，对每个workflow-def生成相应的workflow-def-func。每个func目前的区别只是name不同，func生成但没有调用。
   2. register所有的workflow-def-func。
   3. start workflowWorker
   4. 启动了，然后呢？是一直在监听workflow什么时候添加到cadence中吗?是的。我创了一个workflow以后立马就被拉取到了，直接进入了workflow-def-func

3. workflow-def-func对拉取到的workflow做了什么

   1. 获取wfInfo【wfInfo是cadence内部定义的】

      ```go
      // WorkflowInfo information about currently executing workflow
      type WorkflowInfo struct {
      	WorkflowExecution                   WorkflowExecution
      	WorkflowType                        WorkflowType
      	TaskListName                        string
      	ExecutionStartToCloseTimeoutSeconds int32
      	TaskStartToCloseTimeoutSeconds      int32
      	Domain                              string
      	Attempt                             int32 // Attempt starts from 0 and increased by 1 for every retry if retry policy is specified.
      	lastCompletionResult                []byte
      	CronSchedule                        *string
      	ContinuedExecutionRunID             *string
      	ParentWorkflowDomain                *string
      	ParentWorkflowExecution             *WorkflowExecution
      	Memo                                *s.Memo
      	SearchAttributes                    *s.SearchAttributes
      	BinaryChecksum                      *string
      }
      
      ```

      

   2. 利用side-effect方法调用mojito，查询到workflow-def，包括

      ```go
      WorkflowDefRes struct {
      		WorkflowDefReq
      		ID        uint      `json:"id"`
      		CreatedAt time.Time `json:"created_at"`
      		UpdatedAt time.Time `json:"updated_at"`
      	}
      	WorkflowDefReq struct {
      		Name        string `json:"name" binding:"required"`
      		Version     int    `json:"version"`
      		Description string `json:"description"`
      		// Disable, Enable
      		BuryPoint string `json:"bury_point"`
      		// default false
      		SkipTask bool `json:"skip_task"`
      		// NoNotify, NotifyError
      		NotifyPolicy string `json:"notify_policy"`
      		UserDef      string `json:"user_def" binding:"required"`
      		WrappedDef   string `json:"wrapped_def"`
      		FailDef      string `json:"fail_def"`
      	}
      ```

      

   3. 解析获得的workflow-def，获得tasks
      1. 解析前准备，workflowTaskMeta 和 workflowMeta

      ```go
      WorkflowMeta struct {
      		Name            string              `json:"name"`
      		Description     string              `json:"description"`
      		Tasks           []*WorkflowTaskMeta `json:"tasks"`
      		SchemaVersion   int                 `json:"schemaVersion"`
      		FailureWorkflow string              `json:"failureWorkflow"`
      }	
      
      WorkflowTaskMeta struct {
      		Name                           string                         `json:"name"`
      		TaskReferenceName              string                         `json:"taskReferenceName"`
      		Description                    string                         `json:"description"` // used to set status
      		Type                           string                         `json:"type"`
      		InputParameters                json.RawMessage                `json:"inputParameters"`
      		CaseValueParam                 string                         `json:"caseValueParam,omitempty"`
      		DecisionCases                  map[string][]*WorkflowTaskMeta `json:"decisionCases,omitempty"`
      		DefaultCase                    []*WorkflowTaskMeta            `json:"defaultCase,omitempty"`
      		ForkTasks                      [][]*WorkflowTaskMeta          `json:"forkTasks,omitempty"`
      		Optional                       bool                           `json:"optional"`
      		JoinOn                         []string                       `json:"joinOn,omitempty"`
      		DynamicForkTasksParam          string                         `json:"dynamicForkTasksParam,omitempty"`
      		DynamicForkTasksInputParamName string                         `json:"dynamicForkTasksInputParamName,omitempty"`
      		LoopCondition                  string                         `json:"loopCondition,omitempty"`
      		LoopOver                       []*WorkflowTaskMeta            `json:"loopOver,omitempty"`
      		SubWorkflowParam               SubWorkflowParam               `json:"subWorkflowParam,omitempty"`
      }
      
      
      ```

   4. 将mojito端传来的workflow-def中的warpped-def  unmarshal至workflowMeta中
   
   3. 按照递归的顺序，将workflowMeta中的Tasks依次加入tasks数组中，目的就是要将每一个task节点都拆开【这里的每一个task可以类比一个node】
   
   4. 157-172 不会，不知道干啥的
   
   5. 174-188 wait signal ，跳
   
   6. 188-196 skip signal ，跳
   
   7. 将workflow转换为dsl workflow
      1. dsl workflow的数据结构
   
         实现了Execute的都是Statement，比如Sequence，Parallel，Choice等等	
   
         
   
      ```go
        type DslWorkflow struct {
          name      string
          processor Statement
        }
      
        type Statement interface {
          Execute(ctx workflow.Context, ao func(string) ActivityOptions, bindings *sync.Map) error
        }
      
        type (
          // Sequence consist of a collection of Statements that runs in sequential.
          Sequence struct {
            Elements []Statement
          }
      
          // Parallel can be a collection of Statements that runs in parallel.
          Parallel struct {
            Branches []Statement
          }
      
          DynamicParallel struct {
            Tasks  interface{}
            Inputs interface{}
          }
      
          // Choice can make choice using the last output
          Choice struct {
            KeyBase       string
            Name          string
            TaskRefName   string
            Arguments     map[string]interface{}
            Branches      map[string]Statement
            DefaultBranch Statement
          }
      
          // Loop can implement do-while with loop body and condition
          Loop struct {
            Condition string
            Body      *Sequence
          }
      
          // ActivityInvocation is used to express invoking an Activity. The Arguments defined expected arguments as input to
          // the Activity, the result specify the name of variable that it will store the result as which can then be used as
          // arguments to subsequent ActivityInvocation.
          ActivityInvocation struct {
            Name            string
            TaskRefName     string
            Arguments       map[string]interface{}
            Result          string
            Optional        bool
            Type            string
            SubWorkflowName string
          }
      
          Wait struct {
            SignalName  string
            Name        string
            TaskRefName string
            Arguments   map[string]interface{}
          }
      
          Join struct {
            JoinOn []string
          }
      
          SubWorkflow struct {
            Name string
          }
        )
      
      ```
   
   8. 将workflow的input解析成数据结构
   
   9. 从wfInfo解析searchAttributes
   
   12. `wf.Execute()` 真正执行任务编排？？？（判断什么时候哪个task应当被放在什么位置）
   
       ```go
       // GoNamed creates a new coroutine with a given human readable name.
       // It has similar semantic to goroutine in a context of the workflow.
       // Name appears in stack traces that include this coroutine.
       func GoNamed(ctx Context, name string, f func(ctx Context)) {
       	internal.GoNamed(ctx, name, f)
       }
       ```



## worker

```go
// Worker represents objects that can be started and stopped.
Worker interface {
   // Start starts the worker in a non-blocking fashion
   Start() error
   // Run is a blocking start and cleans up resources when killed
   // returns error only if it fails to start the worker
   Run() error
   // Stop cleans up any resources opened by worker
   Stop()
}

```

activityWorker和workflowWorker都是worker



### work.start()

```go
func (bw *baseWorker) Start() {
  // 如果已经start，退出
	if bw.isWorkerStarted {
		return					
	}
  // 暂时不知道干啥的，可以理解为 WorkerStartCounter+1
	bw.metricsScope.Counter(metrics.WorkerStartCounter).Inc(1)

	for i := 0; i < bw.options.pollerCount; i++ {
		bw.shutdownWG.Add(1) //shutdownWaitGroup
		go bw.runPoller() //开启poller （见下）
	}

	bw.shutdownWG.Add(1)
	go bw.runTaskDispatcher()//开启dispatcher

	bw.isWorkerStarted = true
	traceLog(func() {
		bw.logger.Info("Started Worker",
			zap.Int("PollerCount", bw.options.pollerCount),
			zap.Int("MaxConcurrentTask", bw.options.maxConcurrentTask),
			zap.Float64("MaxTaskPerSecond", bw.options.maxTaskPerSecond),
		)
	})
}
```

#### poller：

```go

func (bw *baseWorker) runPoller() {
   defer bw.shutdownWG.Done()
   bw.metricsScope.Counter(metrics.PollerStartCounter).Inc(1)

   for {// 死循环，除非shutdown，否则一直pollask
      select {
      case <-bw.shutdownCh:  //shutdownCh 发来信号就停止，否则阻塞
         return
      case <-bw.pollerRequestCh: // bw.pollerRequestCh（poller请求的信道） 发来信号就进入
         if bw.sessionTokenBucket != nil {
            bw.sessionTokenBucket.waitForAvailableToken()
         }
         bw.pollTask()// 拉任务（见后）
      }
   }
}
```

```go
func (bw *baseWorker) pollTask() {
   var err error
   var task interface{}
   bw.retrier.Throttle()
   if bw.pollLimiter == nil || bw.pollLimiter.Wait(bw.limiterContext) == nil {
      task, err = bw.options.taskWorker.PollTask() //拉任务（见后）
      if err != nil && enableVerboseLogging {
         bw.logger.Debug("Failed to poll for task.", zap.Error(err))//失败了，task=nil，但不返回
      }
      if err != nil && isServiceTransientError(err) {
         bw.retrier.Failed()
      } else {
         bw.retrier.Succeeded()
      }
   }

   if task != nil {
      select {
      case bw.taskQueueCh <- &polledTask{task}:
      case <-bw.shutdownCh:
      }
   } else {
      bw.pollerRequestCh <- struct{}{} // poll failed, trigger a new poll，poll失败，触发新的poll，直到成功
   }
}
```

```go
// PollTask polls a new task
func (wtp *workflowTaskPoller) PollTask() (interface{}, error) {
   // Get the task.
   workflowTask, err := wtp.doPoll(wtp.poll)
   if err != nil {
      return nil, err
   }

   return workflowTask, nil
}
```

```go
// doPoll runs the given pollFunc in a separate go routine. Returns when either of the conditions are met:
// - poll succeeds, poll fails or worker is shutting down
func (bp *basePoller) doPoll(pollFunc func(ctx context.Context) (interface{}, error)) (interface{}, error) {
   if bp.shuttingDown() {
      return nil, errShutdown
   }

   var err error
   var result interface{}

   doneC := make(chan struct{})
   ctx, cancel, _ := newChannelContext(context.Background(), chanTimeout(pollTaskServiceTimeOut))

   go func() {// 单独开一个goroutine 去进行网络请求
      result, err = pollFunc(ctx)//实际上是运行传入的pollFunc，这里传入的Func见下
      cancel()
      close(doneC)
   }()

   select {
   case <-doneC:
      return result, err
   case <-bp.shutdownC:
      cancel()
      return nil, errShutdown
   }
}
```

```go
// 终于到了最终的poll，实际上就是发网络请求给cadence，
// Poll for a single workflow task from the service
func (wtp *workflowTaskPoller) poll(ctx context.Context) (interface{}, error) {
   startTime := time.Now()
   wtp.metricsScope.Counter(metrics.DecisionPollCounter).Inc(1)

   traceLog(func() {
      wtp.logger.Debug("workflowTaskPoller::Poll")
   })

   request := wtp.getNextPollRequest()
   defer wtp.release(request.TaskList.GetKind())

   response, err := wtp.service.PollForDecisionTask(ctx, request, yarpcCallOptions...)
   if err != nil {
      if isServiceTransientError(err) {
         wtp.metricsScope.Counter(metrics.DecisionPollTransientFailedCounter).Inc(1)
      } else {
         wtp.metricsScope.Counter(metrics.DecisionPollFailedCounter).Inc(1)
      }
      wtp.updateBacklog(request.TaskList.GetKind(), 0)
      return nil, err
   }

   if response == nil || len(response.TaskToken) == 0 {
      wtp.metricsScope.Counter(metrics.DecisionPollNoTaskCounter).Inc(1)
      wtp.updateBacklog(request.TaskList.GetKind(), 0)
      return &workflowTask{}, nil
   }

   wtp.updateBacklog(request.TaskList.GetKind(), response.GetBacklogCountHint())

   task := wtp.toWorkflowTask(response)
   traceLog(func() {
      var firstEventID int64 = -1
      if response.History != nil && len(response.History.Events) > 0 {
         firstEventID = response.History.Events[0].GetEventId()
      }
      wtp.logger.Debug("workflowTaskPoller::Poll Succeed",
         zap.Int64("StartedEventID", response.GetStartedEventId()),
         zap.Int64("Attempt", response.GetAttempt()),
         zap.Int64("FirstEventID", firstEventID),
         zap.Bool("IsQueryTask", response.Query != nil))
   })

   wtp.metricsScope.Counter(metrics.DecisionPollSucceedCounter).Inc(1)
   wtp.metricsScope.Timer(metrics.DecisionPollLatency).Record(time.Now().Sub(startTime))

   scheduledToStartLatency := time.Duration(response.GetStartedTimestamp() - response.GetScheduledTimestamp())
   wtp.metricsScope.Timer(metrics.DecisionScheduledToStartLatency).Record(scheduledToStartLatency)
   return task, nil
}
```

#### dispatcher（派遣，调度员）

```go
func (bw *baseWorker) runTaskDispatcher() {
	defer bw.shutdownWG.Done()

	for i := 0; i < bw.options.maxConcurrentTask; i++ {
		bw.pollerRequestCh <- struct{}{}//向poller goroutine 发信号，触发poller 去拉task
	}

  //触发完以后for循环等待拉任务的结果
  
	for {
		// wait for new task or shutdown
		select {
		case <-bw.shutdownCh:
			return
		case task := <-bw.taskQueueCh:
			// for non-polled-task (local activity result as task), we don't need to rate limit
			_, isPolledTask := task.(*polledTask)
			if isPolledTask && bw.taskLimiter.Wait(bw.limiterContext) != nil {
				if bw.isShutdown() {
					return
				}
			}
			bw.shutdownWG.Add(1)
			go bw.processTask(task)// 新开一个goroutine 执行拉到的task
		}
	}
}
```

process workflow时首先要解析它的history

根据拉到的信息：

![image-20210903100044627](https://tva1.sinaimg.cn/large/008i3skNly1gvki61iij5j60m50hpaca02.jpg)

通过解析函数，解析成这个：

```go
// workflowExecutionContextImpl is the cached workflow state for sticky execution
workflowExecutionContextImpl struct {
   mutex             sync.Mutex
   workflowStartTime time.Time
   workflowInfo      *WorkflowInfo
   wth               *workflowTaskHandlerImpl

   // eventHandler is changed to a atomic.Value as a temporally bug fix for local activity
   // retry issue (github issue #915). Therefore, when accessing/modifying this field, the
   // mutex should still be held.
   eventHandler atomic.Value

   isWorkflowCompleted bool
   result              []byte
   err                 error

   previousStartedEventID int64

   newDecisions        []*s.Decision
   currentDecisionTask *s.PollForDecisionTaskResponse
   laTunnel            *localActivityTunnel
   decisionStartTime   time.Time
}
```

## 回调函数 GetCadenceWorkflowFunc

定义了一个回调函数，在workflow被拉取到的时候会

region_produce2:fd5700fe-4bad-4959-8917-67c7cdd18e55?runId=a3d68b6f-b115-4874-a43d-8934f8ab20c3



## 系统疑问

1. workflow start 后开了几个gorutine，各个都有什么作用
2. LocalActivityWorker是啥
3. workflow-worker开启-poll-dispatch-execute workflowFunc的整个过程再梳理
4.  workflowFunc到底是start之后执行然后结束，还是在workflow整个过程中都停留在里面？
5. 其他已经拉到的还会执行workflowFunc嘛



# history Query重构性能分析

![image-20210924120841656](https://tva1.sinaimg.cn/large/008i3skNly1gvki094ex9j61hk08qgn202.jpg)

![image-20210924133236272](https://tva1.sinaimg.cn/large/008i3skNly1gvki5jkkb0j61d20263yh02.jpg)

1. choice 节点：choice节点的真正选择（case1，case2，defaultcase）应当被记录，status应当被置为completed

```shell
 echo "GET http://localhost:7900/api/v1/workflows/hx_testQuery_complex3_hhxx:07b7481d-b3b1-4cda-afce-9769dcd2b34c/chart?workflowId=hx_testQuery_complex3_hhxx%3A07b7481d-b3b1-4cda-afce-9769dcd2b34c&run_id=1420d94e-16dc-4cba-b9ad-fde8abce559d" | vegeta attack  -rate=30  -duration=30s |vegeta report
```



压测结果（本地）：

长history（100个activity）

延时同为70ms的情况下：

history接口，并发数100

![image-20210916152131231](https://tva1.sinaimg.cn/large/008i3skNly1gvki0flnppj62l40amq5w02.jpg)

query接口

![image-20210916152332715](https://tva1.sinaimg.cn/large/008i3skNly1gvki0ij0ukj61am0lujww02.jpg)



若将history接口调制300并发，结果：

![image-20210916152429956](https://tva1.sinaimg.cn/large/008i3skNly1gvki0n4120j61as0dctcn02.jpg)



Query接口压测

单机 32G内存 8核Ubuntu



压测请求：

```shell
echo "GET http://localhost:7900/api/v1/workflows/hx_testQuery_complex3_hhxx:0955c5c0-3adb-4dea-9ecf-a55251ac9fe0/chart?workflowId=hx_testQuery_complex3_hhxx%3A0955c5c0-3adb-4dea-9ecf-a55251ac9fe0&run_id=d21dab97-f749-49d0-9576-1eb61b081202" | vegeta attack  -rate=450  -duration=3s |vegeta report
```





```shell
echo "GET http://localhost:7900/api/v1/workflows/hx_testQuery_complex3_hhxx:3777ccac-6de1-42da-b0be-1aa0642f468b/chart?workflowId=hx_testQuery_complex3_hhxx%3A3777ccac-6de1-42da-b0be-1aa0642f468b&run_id=bef0e2f5-0890-4e21-8990-ee4c563ab6d5" | vegeta attack  -rate=450  -duration=3s |vegeta report
```



性能：

![image-20210918140842227](/Users/huxiao/Library/Application Support/typora-user-images/image-20210918140842227.png)



压测报告(均为30s)

300

![image-20210918132202426](https://tva1.sinaimg.cn/large/008i3skNly1gvki0rjwjlj60hs067dgp02.jpg)

400

![image-20210918120706972](https://tva1.sinaimg.cn/large/008i3skNly1gvki0v41e0j60ht064aax02.jpg)

450 

![image-20210918120615373](https://tva1.sinaimg.cn/large/008i3skNly1gvki1043mvj60ho067q3s02.jpg)

500

![image-20210918120731002](https://tva1.sinaimg.cn/large/008i3skNly1gvki13utehj60hq066my002.jpg)

550

![image-20210918120908255](https://tva1.sinaimg.cn/large/008i3skNly1gvki18ok7oj60hn080gn102.jpg)

550之后，cadence崩溃，从此之后性能骤降



怀疑system-worker出问题，尝试先测试最简单Query

```shell
echo "GET http://localhost:5088/domains/samples-domain/workflows/query_b4791fd0-8ac5-4648-9b2e-dc5846ef072c/5a192f3a-81d8-444d-9af6-4b32d4805525/query" | vegeta attack  -rate=450  -duration=3s |vegeta report
```





写了一个小demo，起worker：

```shell
type=worker go run ./cmd/samples/recipes/query
```

起query gin框架，4500端口监听，调用Query：

```go
go run ./cmd/samples/recipes/query
```

压测：

```shell
echo "POST http://localhost:4500/echo?wfid=query_f8ee4bf0-c299-4620-b09f-3149cfd189f8&runid=fae79a5d-1838-4077-bdfc-e86ee3d1755e" | vegeta attack  -rate=450  -duration=3s |vegeta report
```

256G 64核机器

1000

![image-20210918194640790](https://tva1.sinaimg.cn/large/008i3skNly1gvki1sefe0j60hs08l3zv02.jpg)



2000

![image-20210918194429113](https://tva1.sinaimg.cn/large/008i3skNly1gvki1x2262j60hs08labe02.jpg)

2000后，700性能也下降了

![image-20210918194933600](https://tva1.sinaimg.cn/large/008i3skNly1gvki213qfcj60hq073t9q02.jpg)

说明这个接口本身性能就不高，峰值QPS 单机2000左右



再在新的服务器上测一下最终的性能对比吧

```shell
echo "GET http://localhost:7900/api/v1/workflows/hx_testQuery_complex3_hhxx:3777ccac-6de1-42da-b0be-1aa0642f468b/chart?workflowId=hx_testQuery_complex3_hhxx%3A3777ccac-6de1-42da-b0be-1aa0642f468b&run_id=bef0e2f5-0890-4e21-8990-ee4c563ab6d5" | vegeta attack  -rate=450  -duration=3s |vegeta report
```

256G 64核 长度50 history

Query：

600

![image-20210918201644065](https://tva1.sinaimg.cn/large/008i3skNly1gvki29oerjj60hq06cmya02.jpg)

700

![image-20210918202228571](https://tva1.sinaimg.cn/large/008i3skNly1gvki2dg6aaj60hu087jss02.jpg)

800

![image-20210918201845972](https://tva1.sinaimg.cn/large/008i3skNly1gvki2i6clij60hp08075p02.jpg)

QPS 700左右





# Query 内存泄漏分析

符号约定：

Func 代表Query的回调函数

## 测试1：Func中插入大块内存，观察其是否可以释放

Query确实存在内存泄漏问题：

![image-20210926162349814](https://tva1.sinaimg.cn/large/008i3skNly1gvki2o6yfjj60kx0autdd02.jpg)

连续一个多小时未能释放。

分析内存：![image-20210926162828759](https://tva1.sinaimg.cn/large/008i3skNly1gvki2sfik4j62fy0e4wik02.jpg)

分析goroutine：

![image-20210926162915828](https://tva1.sinaimg.cn/large/008i3skNly1gvki2w1spqj62di0h0n2x02.jpg)

500多个goroutine：<img src="https://tva1.sinaimg.cn/large/008i3skNly1gvki31asg6j60hg0yw40a02.jpg" alt="image-20210926163009434" style="zoom:50%;" />

为什么Query查询结束后，goroutine却还在等待？推测goroutine的等待造成了内存无法释放，有必要查清楚这么多goroutine的成因。

## 测试2：Func中不插入内存，单纯观测是否由于query而存在大量goroutine未能释放

直接打1000个query，goroutine变成了1000个，而且没有释放：

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gvki3i57q0j60li14y40f02.jpg" alt="image-20210926165652389" style="zoom:20%;" />

过10min，再打1000个Query，goroutine变成了2000个，这已经是相当于goroutine泄漏了：

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gvki3p1qqzj60m015c40o02.jpg" alt="image-20210926165826969" style="zoom:25%;" />

有关这个问题，继续找cadence的工作人员进一步核实

## 结论

确实存在go-routine泄漏问题，准确的说并不是goroutine泄漏，而是缓存没有命中，导致大量query请求每次都进行history-replay，大量goroutine卡在程序的阻塞位置。至于缓存没有命中的原因是缓存失效时间的设置问题，接下来主要分析如何加缓存以及缓存存在的一些性能分析。



# StickyExecution 的执行机理与存在的矛盾

cadence的缓存机制叫stickyExecution，具体表现为：

1. 相同的workflow用相同的worker来执行，这样可以去掉history-replay的环节，节省时间。Query方法比直接查询history方法要快一些的原理也是这个stickyExecution

2. 何时触发：“New decision task contains the new history events will be dispatched to the same worker.” 也就是说每次有新的decision task时，缓存会生效，请求会被转发至相同的worker，整个过程表现为一种sticky执行。具体来说，触发decisionTask的时机有：query，activityCompleted等。也就是workflow-worker拉取到workflow时，先去检查缓存。

3. 由缓存引申出的高可用问题：decisionTaskA被workflow-workerA拉取到，然后workflow-workerA宕机，此时如果仍使用sticky策略，会导致decisionTaskA无法被任何workflow-worker拉取，此时decisionTask所对应的workflow工作会停止，呈现一种卡死的状态。为了保证高可用，设置一个StickyScheduleToStartTimeout时间，poll decisionTask的时间超过这个阈值时便不执行stickyExecute策略，缓存效果也就不复存在。此时实际上是重新生成一个不含有cache的decisionTask，这个decisionTask可以被任意workflow-worker拉取，而不含有cache也代表着整个过程要进行history-replay，此时耗时较长。

4. **stickyScheduledToStartTimeout**指decisionTask在stickExecute过程中**所能接受的耗费的最长时间**，如果过了这个时间（可能是网卡了之类的），decisionTask**就视为原来的sticky-worker已经宕机**，就不会再进行sticky策略。

   - 如果设置的**stickyScheduledToStartTimeout**时间较短（如1s），可能会出现由于workflow-worker并发Query过多，延时过高，导致某次响应decisionTask时间超过1s，此时cadence-server会清除decisionTask的sticky，此时视为workflow-worker已经宕机（实际可能没有宕机），从而造成之后所有的有关该workflow的请求都不会执行stick策略，每一次Query都会新开一个goroutine，造成goroutine泄漏、高latency等现象。
   - 如果设置的**stickyScheduledToStartTimeout**时间较长（如1h），可以避免上述问题，但会出现另一种问题。时间设的过长，如果workflow-worker真的宕机，decisionTask仍将等待1h，此时会造成卡死现象，如果没有新的decisionTask出现来更新sticky绑定关系，当前decisionTask会一直等待，直至1h超时。外部的具体现象表现为，如果一个workflow block在一个activity上很久，此时对应的workflow-worker宕机，之后的decisionTask(包括Query，activityCompleted，都会卡顿1h)

5. 需要注意的是，stickyExecution只与decisionTask有关。具体可表现为，现存在workflow-worker1，workflow-worker2。如果一个decisionTask被workflow-worker1拉到，而此时workflow-worker1宕机，理所应当sticky消失。但是重新来一个新的decisionTask时，会重新与workflow-worker2建立sticky联系，此时重新建立缓存。

    sticky消失以后还能重新获得sticky么，还是永久消失 。sticky不会永久消失。当新来一个decisionTask时，sticky继续生效

   

# mojito端利用describe-workflow进行cache

1. 调用describe-workflow，得到history-length，如果长度改变，缓存失效
2. 如果长度不变，可以放心获取缓存



# Job模式

## job模式如何执行一个task

1. 初始化时启动k8s-client
2. 利用k8s-client启动Global-jobScheduler
3. 调用jobScheduler.start，查表，得到需要用job模式启动的task列表
4. 为task列表中每一个task开启一个goroutine，以1s为周期，定时pollTask()
5. 拉到任务以后进行jobScheduler.schedule，配置需要的内存CPU等信息，调用k8sJob.start()开启一个Pod，工作模式为Job



## 如何写表

job模式的表提前定义好，采用CICD的方式写表。

比如unimesh，我们修改它devops中的azure_pipeline.yml，每次unimesh的业务代码有更改都会触发CICD，自动进行docker的build。build完docker后，原有worker模式是直接deploy到线上pod中，而job模式是将image名字和taskname等信息通过curl，调用mojito接口，然后mojito写表，这样每次unimesh的代码更改后，mojito的表都是最新的镜像。

表结构如下：

![image-20211015144020414](https://tva1.sinaimg.cn/large/008i3skNly1gvki3ywjvvj60zv08racy02.jpg)





## Job模式和worker模式的相同和不同

相同：

1. 都需要用mojito.sdk 
2. 都要通过k8s的Pod

不同：

1. worker本身长期运行在Pod中（Deployment模式），资源固定无法动态分配；Job 短期运行在Pod中（Job模式），在创建Pod的时候给出定制的资源限制 

2. worker的拉取任务本身在各自的pod中来进行，worker在pod中定时通过sdk方位mojito的poll接口（然后mojito再调用cadence）拉任务；Job拉取任务通过统一的mojito来进行，mojito自己调用cadence。

   worker：workerPod--->mojito--->cadence

   job：mojito--->cadence





# retry 机制

retry 是通过cadence提供的机制实现的，mojito-sdk只向mojito传递失败信息，但是不进行retry有关的任何操作。

candence-system-worker中定义了一个

![image-20211019205533918](https://tva1.sinaimg.cn/large/008i3skNly1gvkwzve6q0j30hr0850tm.jpg)

除了以上名字的两种Error，其余都会被Retry

另外，存在一个问题，retry的时候不会自动发信号（包括startTime和Msubstatus），所以外面状态还是running，里面状态已经变成了Queueing
