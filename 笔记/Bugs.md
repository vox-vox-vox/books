# Bugs

# 1. $ 解析

背景：workflow-def中，`${...}`中包含的东西被视为一种变量，解析的时候会对其进行处理：

```go
			dollarIndex := strings.IndexAny(def, "$")
			before := def[0:dollarIndex]
			middle := def[dollarIndex+2 : strings.IndexAny(def, "}")]
			after := def[strings.IndexAny(def, "}")+1:]
```

bug：传入的def字符串本身含有`$`但是不含有`${}`，如：`^((?!parking).)*$`，这样会导致在`middle`的地方出现数组越界![image-20210904133148510](/Users/huxiao/Library/Application Support/typora-user-images/image-20210904133148510.png)

解决方法：不能单纯的按照`$`来作为是否是变量的依据，而是要根据`${`。只有解析到了`${`才视为这之后要解析变量。

# 2. 加缓存导致的bug

背景：

每次从history中读取任务流程定义，读取完毕之后进行解析匹配

bug：

加了缓存，每次如果缓存命中就不会去查history，而选择去查缓存。这样造成的问题是，如果我之前读的是版本21的history，缓存没有命中，将其放入，这样我再读版本20的流程图，就会直接把版本21的拿出来，从而造成各种诡异的错误。

解决方法：

1. 去掉缓存
2. 加入版本号，一起拼成key



# 3. 优化前的流程图匹配会出现stackOverflow

![image-20210911201753077](/Users/huxiao/Library/Application Support/typora-user-images/image-20210911201753077.png)





# 4. error抛出的时候消失

```go
//在一段代码中，我插入了自己的一段代码并抛出了异常，如下所示

//修改之前：

var err error
...
if a>0{
  
  //这是一个闭包，里面是回调函数
  AddFunc(func(){
    err=funcB()
  })
}else{
  
}
if err!=nil{
  return err
}
...


//修改之后：
var err error
...
if a>0{
  //=============我插入的代码================Start
  err:=funA()//新声明了一个局部error
  if err!=nil{
    return err
  }
  //=======================================End
  AddFunc(func(){
    err=funcB()//所有闭包内的error会抛给局部error
  })
}else{
  
}
if err!=nil{
  return err//全局error自然无法抛出闭包内的error
}
...


//这样会造成的现象：funcB的错误抛不出来

//为什么？
/*
因为之前的err的作用域是整段代码，闭包中的err实际上也是这个全局err，funcB()的err实际上是抛到了全局err中
我加入了一段代码以后，if内新加了一个“err:=xxx”，这实际上会引入一个新的if内的局部err，闭包内的err就变成了局部err，所有funcB的东西都抛到了局部error中，导致最后全局的error无法抛出
*/

```

# 5. Gorm 空指针异常

```go
func (s *WorkflowOperationLogStore) List(req workflow.OperationLogsReq) (result *model.WorkflowOperationLog) {
	condition := workflow.OperationLogsReq{
		WorkflowId: req.WorkflowId,
		UserName:   req.UserName,
	}
	pre := s.db.Model(&model.WorkflowOperationLog{}).Where(&condition)
	pre.Where(" = ?", req.Operation).Scan(result)
	return result
}

```

因为此时result还没有初始化，只是指向model.workflowOperationLog的一个指针，所以在Scan的时候会出错，报空指针异常，正常的方法应该这么写：

```go
func (s *WorkflowOperationLogStore) List(req workflow.OperationLogsReq) (result model.WorkflowOperationLog) {
   condition := workflow.OperationLogsReq{
      WorkflowId: req.WorkflowId,
      UserName:   req.UserName,
   }
   pre := s.db.Model(&model.WorkflowOperationLog{}).Where(&condition)
   pre.Where(" = ?", req.Operation).Scan(&result)
   return result
}
```



# 6. Gorm migrate 深坑

gorm 在 migrate时，如果位于前面的迁移失败了，后面的会打印sql但是实际上不会执行！

所以要把想要migrate的表放在前面才可以！



# 7. 流程图加速加载的坑

之前自己写了这么一段代码：

```go
func (s *WorkflowService) GetFlowChartFromMultiple(wfID string, runID string, domain string) (resp *types.FlowChartResponse, err error) {
   historyChan := make(chan interface{})
   queryChan := make(chan interface{})
   type TransportData struct {
      Resp  *types.FlowChartResponse
      Error error
   }
   var transportInterface interface{}
   go func() {
      resp, err := s.GetFlowChart(wfID, runID, domain)
      if resp != nil {
         historyChan <- TransportData{
            Resp:  resp,
            Error: err,
         }
         return
      }
   }()
   go func() {
      resp, err := s.GetFlowChart3(wfID, runID, domain)
      if resp != nil {
         queryChan <- TransportData{
            Resp:  resp,
            Error: err,
         }
         return
      }
   }()

   select {
   case transportInterface = <-historyChan:
      println("==================history================")
   case transportInterface = <-queryChan:
      println("==================query================")
   }
   transportData, _ := transportInterface.(TransportData)
   resp = transportData.Resp
   err = transportData.Error
   return
}
```

意图很明显，就是开两个goroutine，每个goroutine都跑一个流程图解析，哪个运行的快返回哪个

但是这个代码内出现了2个重大bug

- goroutine中没有panic recover机制。
  处理请求的协程发生panic gin会自己处理，但是自己新加的协程一旦出现panic，就会导致整个进程崩掉，从而触发进程重启工具，中间出现一堆502
- 会造成死锁。
  select 条件如果两个chan都没有信号传进来，会发生阻塞，从而造成死锁现象。在上面的用例中，如果两个getflowChart模块同时抛出err，这样historyChan和queryChan都阻塞，整体协程将会在select处卡死

解决方案：

- 单独使用goroutine时采用手动recover