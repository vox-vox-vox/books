# decision 与 cadence&workflow-worker&activity-worker 运转流程

workflow-worker中的代码实际上等价于state machine

```java
public string sampleWorkflowMethod(...){
  var result = activityStubs.activityA(...)
  if(result.startsWith("x"){
     Workflow.sleep(...)
  }else{
     result = activityStubs.activityB(...)
  }
  
  return result
}
```

等价于

<img src="https://i.stack.imgur.com/yxuAG.png" alt="enter image description here" style="zoom: 33%;" />

定义：

1. decision：传给cadence的指令，cadence根据decision决定下一步工作。在多分支的state - machine中相当于实际执行的边（上图红色箭头）
2. decision task：workflow-worker中产生decision的代码块



1. 向cadence server的workflow队列中加入workflow信息，等待workflow-worker拉取
2. workflow-worker拉取到匹配的workflow，找到对应的worker，进行执行，发送给cadence server一个（scheduled activityA）的指令（decision），之后自行阻塞
3. cadence server收到decision后，创建activityA，放入activity队列中，等待activity worker拉取并执行
4. activity worker拉取并执行，在此期间acticity worker阻塞
5. activity worker执行完毕，返回cadence server并更新workflow信息，等待workflow-worker拉取
6. 同2，workflow-worker继续拉取







1. 监听到什么样的信息，queue中装有什么样的信息，前一个activity的output是怎样放在这个信息里从而继续被下一个任务拉走的？和cadence server交互采用什么样的数据结构

   答：上一个activity的output被记录在cadence的historyEvent，workflow-worker分配下一个任务时直接查historyEvent来找到其中的output

2. 











> cadence中的概念
>
> 1. workflow
> 2. activity
> 3. event
>
> 
>
> Cadence 核心抽象是一个无故障的有状态工作流。
>
> 学习 Cadence 的开发人员常问的问题是“如何处理工作流中的工作流工作进程失败/重启”？ 答案是你没有。 工作流代码完全没有注意到工作人员甚至 Cadence 服务本身的任何故障和停机时间。
>
> <img src="https://i.stack.imgur.com/yxuAG.png" alt="enter image description here" style="zoom:50%;" />



> 工作人员将结果报告回 Cadence 服务，后者又会通知工作流有关完成情况
>
> the worker reports the result back to the Cadence service which in turn notifies the workflow about completion



> excution
>
> - Execution must be deterministic
> - Execution must be idempotent

>  workflow worker 中的 `Get()` method will block until the activity completes and results are available.



# side effect

`workflow.SideEffect` It executes the provided function once and records its result into the workflow history.

仅执行一次提供的函数，并将其结果放入workflow history，例如：

![image-20210821111910096](/Users/huxiao/Library/Application Support/typora-user-images/image-20210821111910096.png)

The recorded result on history will be returned without executing the provided function during replay. This guarantees the deterministic requirement for workflow as the exact same result will be returned in replay.

流程replay的时候sideEffect部分也不会重新执行，为了保证确定性



every sideEffect call in non-replay mode results in a new marker event recorded into the history.

每次一调用sideEffect call 都会在historyEvents中创建一个新的marker event



do not use sideEffect function to modify any workflow state. Only use the SideEffect's return value.

尽量用sideEffect的return Value，而不是利用sideEffect函数本身，因为他retry时将不会被执行。可以这么说，sideEffect 唯一的作用就是执行一段动态的代码，然后产生一个return value，作为本次执行的状态。下次我们将用这个return value 作为状态的标识。











## mutableSideEffect

# signal

Signals provide a mechanism to send data directly to a running workflow：提供了一种向正在运行的workflow传递信息的机制





# execute activities

将activity 放入cadence队列中，使其变为scheduled，等待被拉取

The most straightforward way to do this is via the library method `workflow.ExecuteActivity`

```go
ao := cadence.ActivityOptions{
    TaskList:               "sampleTaskList",
    ScheduleToCloseTimeout: time.Second * 60,
    ScheduleToStartTimeout: time.Second * 60,
    StartToCloseTimeout:    time.Second * 60,
    HeartbeatTimeout:       time.Second * 10,
    WaitForCancellation:    false,
}
ctx = cadence.WithActivityOptions(ctx, ao)

future := workflow.ExecuteActivity(ctx, SimpleActivity, value)
var result string
if err := future.Get(ctx, &result); err != nil {
    return err
}
```



# search attributes

Search attributes data are indexed so you can search workflows by querying on these attributes. 

Search attributes use Elasticsearch



Search attributes 用来 查找 workflows



UpsertSearchAttributes is used to add or update search attributes from within the workflow code.，用来更新SeaarchAttributes的状态



SearchAttr被广泛用于前端的搜索



# task lists

There are multiple advantages of using a task list to deliver tasks instead of invoking an activity worker through a synchronous RPC:

- Worker doesn't need to have any open ports, which is more secure.
- Worker doesn't need to advertise itself through DNS or any other network discovery mechanism.
- When all workers are down, messages are persisted in a task list waiting for the workers to recover.
- A worker polls for a message only when it has spare capacity, so it never gets overloaded.
- Automatic load balancing across a large number of workers.
- Task lists support server side throttling. This allows you to limit the task dispatch rate to the pool of workers and still supports adding a task with a higher rate when spikes happen.
- Task lists can be used to route a request to specific pools of workers or even a specific process.



# LocalActivity

Some of the activities are very **short lived** and do not need the queing semantic, flow control, rate limiting and routing capabilities. For these Cadence supports so called *local activity* feature. Local activities are executed in the same worker process as the workflow that invoked them. Consider using local activities for functions that are:

- no longer than a few seconds
- do not require global rate limiting
- do not require routing to specific workers or pools of workers
- can be implemented in the same binary as the workflow that invokes them

The main benefit of local activities is that they are much more efficient in utilizing Cadence service resources and have much lower latency overhead comparing to the usual activity invocation







# workflowInterceptor

```go
type WorkflowInterceptor interface {
	// Intercepts workflow function invocation. As calls to other intercepted functions are done from a workflow
	// function this function is the first to be called and completes workflow as soon as it returns.
	// workflowType argument is for information purposes only and should not be mutated.
	ExecuteWorkflow(ctx Context, workflowType string, args ...interface{}) []interface{} //拦截工作流函数调用。 由于对其他拦截函数的调用是从工作流函数完成的，因此该函数首先被调用，并在返回后立即完成工作流。 工作流类型参数仅供参考，不应改变。
  
	ExecuteActivity(ctx Context, activityType string, args ...interface{}) Future
	ExecuteLocalActivity(ctx Context, activityType string, args ...interface{}) Future
	ExecuteChildWorkflow(ctx Context, childWorkflowType string, args ...interface{}) ChildWorkflowFuture
	GetWorkflowInfo(ctx Context) *WorkflowInfo
	GetLogger(ctx Context) *zap.Logger
	GetMetricsScope(ctx Context) tally.Scope
	Now(ctx Context) time.Time
	NewTimer(ctx Context, d time.Duration) Future
	Sleep(ctx Context, d time.Duration) (err error)
	RequestCancelExternalWorkflow(ctx Context, workflowID, runID string) Future
	SignalExternalWorkflow(ctx Context, workflowID, runID, signalName string, arg interface{}) Future
	UpsertSearchAttributes(ctx Context, attributes map[string]interface{}) error
	GetSignalChannel(ctx Context, signalName string) Channel
	SideEffect(ctx Context, f func(ctx Context) interface{}) Value
	MutableSideEffect(ctx Context, id string, f func(ctx Context) interface{}, equals func(a, b interface{}) bool) Value
	GetVersion(ctx Context, changeID string, minSupported, maxSupported Version) Version
	SetQueryHandler(ctx Context, queryType string, handler interface{}) error
	IsReplaying(ctx Context) bool
	HasLastCompletionResult(ctx Context) bool
	GetLastCompletionResult(ctx Context, d ...interface{}) error
}
```



workflow 拦截器，可以对workflow执行过程进行拦截，并做相应处理





# TaskHandler

有两种task，一种是workflow，一种是activity，相应的，他们的handler名字也不一样

```go
// WorkflowTaskHandler represents decision task handlers.
WorkflowTaskHandler interface {
   // Processes the workflow task
   // The response could be:
   // - RespondDecisionTaskCompletedRequest
   // - RespondDecisionTaskFailedRequest
   // - RespondQueryTaskCompletedRequest
   ProcessWorkflowTask(
      task *workflowTask,
      f decisionHeartbeatFunc,
   ) (response interface{}, err error)
}

// ActivityTaskHandler represents activity task handlers.
ActivityTaskHandler interface {
   // Executes the activity task
   // The response is one of the types:
   // - RespondActivityTaskCompletedRequest
   // - RespondActivityTaskFailedRequest
   // - RespondActivityTaskCanceledRequest
   Execute(taskList string, task *s.PollForActivityTaskResponse) (interface{}, error)
}
```





# HelloWorld（下周末再看吧！）

helloworld的运行机制（最简单的 “单Activity” workflow）：

## 1. **register** 

workflow-worker 和 activity-worker【将配置写入worker但不加载，等start时才加载】

## 2. **start** 

start 会在新的协程里开启 dispatcher 和 poller。 【workflow-worker 和 activitty-worker 同时start，本质相同，都是开启dispatcher 和 poller】【poller的开启数量和拉取速率可以设置】

### dispatcher

作为消息中转，通过信道向poller发送maxConcurrentTask数目的信号，poller收到以后开始pollTask()【这边的数量有点对不上，为什么不是开maxConcurrentTask个goroutine去拉？】发送完以后，接收poller拉来的Task，并且运行processTask(task)

### poller

作为拉取cadence server队列信息的协程，不断在后台运行PollTask()，如果拉到的为nil，会通过信道再次向poller发信号，再次触发一次新的pollTask()

## 3. **process**

process过程分为processWorkflowTask【对拉到的task为workflow来说】和 Execute【对拉到的task为activity来说】 

### workflow：

workflow要根据当前history，对history中的每个event都进行处理，具体要处理的event的有：

1. workflowflowStarted 利用反射调用定义的Func，运行Func，直至遇到ExecuteActivity，阻塞于此（第4步讲）
2. decisionTaskStarted 利用反射回到Func阻塞部分【和future有什么样的联系目前不知道，要搞清楚】
3. activityTaskScheduled 做了一些标记
4. activityTaskCompleted 利用反射调用Callback

### activity：

activity则不需要像workflow那样考虑那么多，主要执行Execute，利用反射调用定义的activityFunc，执行完毕后调用client，向cadence-server发送请求，cadence知道activity处理完之后就继续decisionTaskStarted，重新将workflow装入队列中。

## 4. 为什么以started为分界

history中的decisionTask对应workflow，ActivityTask对应Activity，每一次poll workflow的时候都会拉取分片的history，截取范围：(上次decisionTaskStarted，本次decisionTaskStarted]



# history replay

但是因为这是一个**分布式操作系统**，工作流所有权可以转移给其他工作人员并从以前的状态继续运行。与其他操作系统不同，这**不是从头重新启动**。这就是工作流对某些主机故障的容错方式。

正因为history的存在，只要启动了相同的workflow-worker，就可以实现基于history分布式处理同一workflow

DecisionTaskScheduled : workflow被放入队列（一切scheduled都是cadence-server做出的决策）

DecisionTaskStarted : workflow被poller拉走

DecisionTaskCompleted ：workflow运行结束，调用ExecuteActivity/workflow.Sleep，后面就要将activity放入队列中，也就是ActivityTaskScheduled

ActivityTaskScheduled：cadence-server根据workflow-worker做出的决策（ExecuteActivity），将相应的activity装入队列，这个过程叫scheduled

**Activities/ChildWorkflows/Timers/etc will not be re-executed during history replay.**

怎么理解？只是不会执行，future不会阻塞，但是拿着已经执行完的output，宛如已经执行了一样，真实np

ActivityStarted: activity 被执行了，后面一定接着completed





研究history缓存

为什么缓存越来越多？

replay的机制，replay对信号有什么影响

retry的时候会replay吗

query的时候replay会进行到什么程度？为什么我断点打不进去



# sticky Execution

workflow只被执行过的worker执行，因为有cache

Sticky Execution is to run the decision tasks for one workflow execution on same worker host. This is an optimization for workflow execution. When sticky execution is enabled, worker keeps the workflow state in memory. **New decision task contains the new history events will be dispatched to the same worker.** If this worker crashes, the sticky decision task will timeout after StickyScheduleToStartTimeout, and cadence server will clear the stickiness for that workflow execution and automatically reschedule a new decision task that is available for any worker to pick up and resume the progress.

具体的cache表现为`workflowExecutionContext`，这个虽然叫context，但实际上和golang的context没任何关系，顶多算一个缓存。

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

# selector

selector实现了golang中的select功能，即channel多选一

selector支持多种select，如下：

```go
Selector interface {
   // AddReceive adds a ReceiveChannel to the selector. f is invoked when the channel has data or closed.
   // ok == false indicates the channel is closed
   AddReceive(c Channel, f func(c Channel, ok bool)) Selector
   // AddSend adds a SendChannel to the selector. f is invoke when the channel is available to send
   AddSend(c Channel, v interface{}, f func()) Selector
   // AddFuture adds a Future to the selector f is invoked when future is ready
   AddFuture(future Future, f func(f Future)) Selector
   // AddDefault adds a default branch to the selector.
   // f is invoked when non of the other conditions(ReceiveChannel, SendChannel and Future) is met for one call of Select
   AddDefault(f func())
   // Select waits for one of the added conditions to be met and invoke the callback as described above.
   // When none of the added condition is met:
   //        if there is no Default(added by AddDefault) and , then it will block the current goroutine
   //        if Default(added by AddDefault) is used, when Default callback will be executed without blocking
   // When more than one of added conditions are met, only one of them will be invoked.
   // Usually it's recommended to use a for loop to drain all of them, and use AddDefault to break out the
   // loop properly(e.g. not missing any received data in channels)
   Select(ctx Context)
}
```





具体的demo：

1. 多个future并发竞争：

```go
selector := workflow.NewSelector(ctx)
var activityErr error
for _, s := range p.Branches {
   f := executeAsync(s, childCtx, inputs,bindings)
   selector.AddFuture(f, func(f workflow.Future) {
      err := f.Get(ctx, nil)
      if err != nil {
         // cancel all pending activities
         cancelHandler()
         activityErr = err
      }
   })
}

for i := 0; i < len(p.Branches); i++ {
   selector.Select(ctx) // this will wait for one branch <=================================block here
   if activityErr != nil {
      return activityErr
   }
}

return
```

selector中，AddFuture中的func是什么时候才会被调用，具体的阻塞发生是发生在future.Get中，还是发生在AddFuture的func中？

具体的block在外部的func中，类似于一种双重保险。更准确地说，会在`selector.Select(ctx)`的地方阻塞。如果有完成的future才会进入对应的func，其余的在这一次select中被过滤掉了。一共进行`len(p.Branches)`次过滤。







# Query

1. query的结果包在completedRequest里面

```go
completedRequest, err := wtp.taskHandler.ProcessWorkflowTask(
   task,
   func(response interface{}, startTime time.Time) (*workflowTask, error) {
      wtp.logger.Debug("Force RespondDecisionTaskCompleted.", zap.Int64("TaskStartedEventID", task.task.GetStartedEventId()))
      wtp.metricsScope.Counter(metrics.DecisionTaskForceCompleted).Inc(1)
      heartbeatResponse, err := wtp.RespondTaskCompletedWithMetrics(response, nil, task.task, startTime)
      if err != nil {
         return nil, err
      }
      if heartbeatResponse == nil || heartbeatResponse.DecisionTask == nil {
         return nil, nil
      }
      task := wtp.toWorkflowTask(heartbeatResponse.DecisionTask)
      task.doneCh = doneCh
      task.laResultCh = laResultCh
      return task, nil
   },
)
```



2. query会触发pollTask-->processTask，在process的过程中对整个workflow进行replay，但是replay并不是一定要进行的，见下：

3. **Query的并发情况**

一个Query，一个更改状态，会发生什么





# workflow context



# Future 阻塞机制

首先看future的Get，下面是官方对Get行为的描述：

```go
// Get blocks until the future is ready. When ready it either returns non nil error or assigns result value to
// the provided pointer.
// Example:
//  var v string
//  if err := f.Get(ctx, &v); err != nil {
//      return err
//  }
//
// The valuePtr parameter can be nil when the encoded result value is not needed.
// Example:
//  err = f.Get(ctx, nil)
```

下面是Get行为的具体实现，具体的阻塞被封装到了`Receive`中

```go
func (d *decodeFutureImpl) Get(ctx Context, value interface{}) error {
	more := d.futureImpl.channel.Receive(ctx, nil)//**********************看到这里是用channel的Receive来实现的
	if more {
		panic("not closed")
	}
	if !d.futureImpl.ready {
		panic("not ready")
	}
	if d.futureImpl.err != nil || d.futureImpl.value == nil || value == nil {
		return d.futureImpl.err
	}
	rf := reflect.ValueOf(value)
	if rf.Type().Kind() != reflect.Ptr {
		return errors.New("value parameter is not a pointer")
	}

	err := deSerializeFunctionResult(d.fn, d.futureImpl.value.([]byte), value, getDataConverterFromWorkflowContext(ctx), d.channel.env.GetRegistry())
	if err != nil {
		return err
	}
	return d.futureImpl.err
}
```

具体的Receive函数如下：

```go
func (c *channelImpl) Receive(ctx Context, valuePtr interface{}) (more bool) {
	state := getState(ctx)
	hasResult := false
	var result interface{}
	callback := &receiveCallback{							//定义callback函数，如果收到就会触发
		fn: func(v interface{}, m bool) bool {
			result = v
			hasResult = true
			more = m
			return true
		},
	}

	for {
		hasResult = false
		v, ok, m := c.receiveAsyncImpl(callback) //开启异步接受

		if !ok && !m { //channel closed and empty
			return m
		}

		if ok || !m {
			err := c.assignValue(v, valuePtr)
			if err == nil {
				state.unblocked()
				return m
			}
			continue //corrupt signal. Drop and reset process
		}
		for {
			if hasResult {
				err := c.assignValue(result, valuePtr)
				if err == nil {
					state.unblocked()
					return more
				}
				break //Corrupt signal. Drop and reset process.
			}
			state.yield(fmt.Sprintf("blocked on %s.Receive", c.name))  //调用yield，阻塞在某处，等待共享变量hasResult的改
		}
	}
}

```





# 待解决

1. replay机制
2. 反射执行机制
4. workflow真正执行的dispatcher机制
5. signal 的 replay机制。经常多次收到signal，signal的消费机制究竟是怎样的？
6. add future 和 add receive 那么future和receive有什么区别？

