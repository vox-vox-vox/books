# 数据库与gorm

# 1. 事务与并发

事务块：BEGIN--COMMIT

```golang
transactionKindNone transactionKind = iota
transactionKindNormal
transactionKindInternal
```

这三种事务有什么区别



for update怎样实现行锁的功能？

众所周知锁都需要上锁和解锁，那我用for update是进行上锁么，那什么时候解锁呢

`for update`是一种`行级锁`，又叫`排它锁`。

一旦用户对某个行施加了行级加锁，则该用户可以查询也可以更新被加锁的数据行，其它用户只能查询但不能更新被加锁的数据行。也就是可以读但是不能写。

> for update仅适用于InnoDB，且必须在事务块(BEGIN/COMMIT)中才能生效。在进行事务操作时，通过“for update”语句，MySQL会对查询结果集中每行数据都添加排他锁，其他线程对该记录的更新与删除操作都会阻塞。

解锁时间：

**只有当出现如下之一的条件，才会释放共享更新锁：**

1. 执行提交（COMMIT）语句
2. 退出数据库（LOG　OFF）
3. 程序停止运行



删除的时候要加锁吗？我是加了，这样就只有这一个用户可以操作这一行，其他用户会阻塞，避免查询到已经被删除的

# 2. Model

虽然不知道为什么，但是删除哪张表是由dbDef来确定的，可以把数据库的model看成一个表，每一个model对应数据库中的一张表

```go
dbDef := &model.WorkflowDef{}
err := t.Where("name=?", "hx_workflowForDelete").Delete(dbDef).Error
```

```go
type WorkflowDef struct {
   Model
   Name        string `json:"name" gorm:"unique_index:uix_workflow_defs_name_version"`
   Version     int    `json:"version" gorm:"unique_index:uix_workflow_defs_name_version"`
   Description string `json:"description"`
   // Disable, Enable
   BuryPoint string `json:"bury_point"`
   // NotifyNoop, NotifyError
   NotifyPolicy       string `json:"notify_policy"`
   SkipTask           bool   `json:"skip_task"`
   UserDef            string `json:"user_def"`
   WrappedDef         string `json:"wrapped_def"`
   FailDef            string `json:"fail_def"`
   AutoscalerPriority *uint  `json:"autoscaler_priority" gorm:"default:2"`
}
```

gorm使用snake_cases，`SkipTask`就默认对应表中的`skip_task`



# 3. Sql句子

这个什么意思啊

```go
result []model.CurrentWorkflowDef
err = s.db.Model(model.WorkflowDef{}).Where("id in (?)", s.db.Model(model.WorkflowDef{}).Select("max(id)").Group("name").QueryExpr()).Scan(&result).Error

由内往外翻译一下：
1. select max(id):	找到id的最大值
2. Group(name):			根据name进行分组，和1组合起来就是找到每个name的最大id的行
3. QueryExpr():			整个句子作为查询表达式，便于嵌套
4. where id in (?):	里面是查询表达式，in语句相当于select范围规定在后面这个表里，而这个表就是1+2+3我们构造出来的那个表
5. Scan(&result):		将结果映射成result所规定的数据结构
```



选择所有的内容

```go
err = s.db.Find(& []model.CurrentWorkflowDef{}).Scan(&result).Error
//select * from current_workflow_def
//所谓的current_workflow_def 实际上就是结构体切片
```



怎样删除 name=xxx中id最大的



22936

大的要来了：

```sql
Select substring(workflow_uuid,1,char_length(workflow_uuid)-37) as workflowName , count(*) as count_num from(
--成功的任务原始数据
    select segment_task.*,segment_task.updated_at-segment_task.created_at as running_time from
  (
      select segment.*,par.value from
      (
          select seg.*,sta.name,sta.description,sta.workflow_def_id,sta.status as status1
          from segment_tasks as seg
          left join status_defs as sta
          on seg.status_def_id=sta.id
          where seg.created_at>'2021-05-01 00:00:00.000000+08'
          order by seg.id desc
      )
      as segment
      left join segment_task_params as par
      on segment.id=par.segment_task_id
      --where par.name='log'
  )
  as segment_task
  where segment_task.status1>=
  (
      select status.status from status_defs as status
      where status.workflow_def_id = segment_task.workflow_def_id
      and status.name like '%manu%'
      order by status.status
      limit 1
  )
  or segment_task.status1=4
  )
as temp
group by substring(workflow_uuid,1,char_length(workflow_uuid)-37)


```



```sql
--Select substring(workflow_uuid,1,char_length(workflow_uuid)-37) as workflowName , count(*) as count_num from(

select distinct id , workflow_uuid from
(
--成功的任务原始数据
    select segment_task.*,segment_task.updated_at-segment_task.created_at as running_time from
  (
      select segment.*,par.value from
      (
          select seg.*,sta.name,sta.description,sta.workflow_def_id,sta.status as status1
          from segment_tasks as seg
          left join status_defs as sta
          on seg.status_def_id=sta.id
          where seg.created_at>'2021-05-01 00:00:00.000000+08'
          order by seg.id desc
      )
      as segment
      left join segment_task_params as par
      on segment.id=par.segment_task_id
      --where par.name='log'
  )
  as segment_task
  where segment_task.status1>=
  (
      select status.status from status_defs as status
      where status.workflow_def_id = segment_task.workflow_def_id
      and status.name like '%manu%'
      order by status.status
      limit 1
  )
  or segment_task.status1=4
  ) as distinct_seg 
  --)as temp
  --group by substring(workflow_uuid,1,char_length(workflow_uuid)-37)
```



## JOIN

![clipboard.png](https://segmentfault.com/img/bVbk2mR?w=966&h=760)





关于`left join`的重复问题：本质在于，表A和表B相等的列不是一一对应的，就会出现重复。

比如连接条件：a.id=b.segment_id，而segment_id有3个都为1，那么a表id为1的行肯定要分别匹配这三个segment_id才行



## distinct 不能自由选择不同的维度

count(distinct(xxx))能完成整个表的去重吗

count 是聚合函数，count只和group by 

为什么`count(distinct id)`可以修复问题，完成去重。首先，count id就是普通的计数，但是经过distinct(id)包装后本身就是去重后的id，这样count就绝对不会重复

Select   __ , __ , __ , ... ,只要这中间有聚合函数，就一定要把其他所有列上加上group by，一个范例：

```sql
select A，B，count(*)
from
group by A
```

这时sql并不知道A组里面到底显示哪一个B，就会提示

column "B" must appear in the GROUP BY clause or be used in an aggregate function

B必须放到Group BY里面，形成上下两两对称，或者B放入聚合函数里面

group by 多个时，GROUP BY X, Y意思是将所有具有相同X字段值和Y字段值的记录放到一个分组（通过聚合函数聚合）里

group by 不加聚合函数没有意义

count 这个聚合函数里面不同的列数有区别吗？count(A)和count(B)难道不是完全取决于Group by？反正count只是求个数而已



> 任何操作想把表中的信息压缩（聚合函数+group by / distinct / ...）都不应影响到其他无关列的显示，不能让sql去猜没有被聚合的那列到底要选择哪个，否则就会失败





## 表的拼接

平均时间

avgtime      count     workflow_def_id    name



平均长度和数量

avgtime      count     workflow_def_id    name

拼接 avgTIme，avgLen和其他两列

```sql
select table3.*,wd.updated_at,wd.version from 
	(
		select table2.*,workflow_defs.name from workflow_defs join
			(
			select avg(seg_table.length)as avgLen,avg(seg_table.running_time) as avgTime ,count(distinct seg_table.id),seg_table.workflow_def_id from
				(
				select 	avgLen.*,	avgTime.running_Time as running_Time from 
					(
						select segment_task.* from
						(
							select segment.*,par.value from
							(
								select seg.*,sta.name,sta.description,sta.workflow_def_id,sta.status as status1
								from segment_tasks as seg
								left join status_defs as sta
								on seg.status_def_id=sta.id
								where seg.created_at>'2021-07-01 00:00:00.000000+08' and seg.created_at<'2021-08-01 00:00:00.000000+08'
								order by seg.id desc
							)as segment
							left join segment_task_params as par
							on segment.id=par.segment_task_id
						)as segment_task
						where 
						segment_task.status1>=
						(
							select status.status from status_defs as status
							where status.workflow_def_id = segment_task.workflow_def_id
							and status.name like '%manu%'
							order by status.status
							limit 1
						)
						or segment_task.status1=4
    				)as avgLen 
					left join
					(
						select segment_task.*,segment_task.updated_at-segment_task.created_at as running_time from
							(
								select segment.*,par.value from
								(
									select seg.*,sta.name,sta.description,sta.workflow_def_id,sta.status as status1
									from segment_tasks as seg
									left join status_defs as sta
									on seg.status_def_id=sta.id
									where seg.created_at>'2021-07-01 00:00:00.000000+08' and seg.created_at<'2021-08-01 00:00:00.000000+08'
									order by seg.id desc
								)
								as segment
								left join segment_task_params as par
								on segment.id=par.segment_task_id
							)as segment_task
							where segment_task.status1=
							(
								select status.status from status_defs as status
								where status.workflow_def_id = segment_task.workflow_def_id
								and status.name like '%manu%'
								order by status.status
								limit 1
							)
					)as avgTime 
					on avgTime.id=avgLen.id
				)as seg_table
			group by seg_table.workflow_def_id
			)as table2
		on workflow_defs.id=table2.workflow_def_id
	)as table3
join workflow_defs as wd on wd.id=table3.workflow_def_id
order by table3.name,table3.workflow_def_id
```



按照workflow_uuid 的前几个字符分组聚合，并进行去重的结果

```sql
Select substring(workflow_uuid,1,char_length(workflow_uuid)-37) as workflowName , count(distinct id) as count_num from(
--成功的任务原始数据
    select segment_task.*,segment_task.updated_at-segment_task.created_at as running_time from
  (
      select segment.*,par.value from
      (
          select seg.*,sta.name,sta.description,sta.workflow_def_id,sta.status as status1
          from segment_tasks as seg
          left join status_defs as sta
          on seg.status_def_id=sta.id
          where seg.created_at>'2021-07-01 00:00:00.000000+08' and seg.created_at<'2021-08-01 00:00:00.000000+08'
          order by seg.id desc
      )
      as segment
      left join segment_task_params as par
      on segment.id=par.segment_task_id
      --where par.name='log'
  )
  as segment_task
  where segment_task.status1>=
  (
      select status.status from status_defs as status
      where status.workflow_def_id = segment_task.workflow_def_id
      and status.name like '%manu%'
      order by status.status
      limit 1
  )
  or segment_task.status1=4
  )
as temp
group by substring(workflow_uuid,1,char_length(workflow_uuid)-37)
```



查询特定的

```sql
select * from task_checks where task_name = 'iw_auto_lane_topology'
order by (updated_at-created_at)
desc
```





# 4. 批量update

```sql
update segment_tasks 
SET region_length=tmp.region_length
FROM
	(Values
		(246,100),
	 	(247,101),
	 	(248,102)
	)As tmp (id,region_length)
Where
	segment_tasks.id=tmp.id
```



使用unnest

# 5. 模糊查询 like

```sql
SELECT * FROM "workflow_operation_logs"  WHERE ("workflow_operation_logs"."user_name" = 'huxiao') AND ( operation like '%terminate%')  
```



# 6. 分页 offset + limit

传入page pagesize

```sql
err = cur.Count(&count).Offset((page - 1) * pageSize).Limit(pageSize).Find(&result).Error


SELECT * FROM "current_workflow_defs" LIMIT pageSize OFFSET (page-1)*pageSize  

```





# Redis

## 1. once

从现象和逻辑来看，redis的once如果成功找到就不调用item中的Func，如果没有找到就调用Func去找数据库里的东西

```go
item := &cache.Item{
  Key:    key,
  Object: res,
  Func: func() (interface{}, error) {
    result, err := s.Get(name, version)
    if err != nil {
      return types.WorkflowDefRes{}, err
    }
    return *result, err
  },
  Expiration: time.Minute * 10,
}


if err := store.GredisManager.Once(item); err != nil {
  logger.Infow("WorkflowDefService:GetCached once cache fail",
               "error", err)
  return nil, err
}
```

## 2. Redis 分布式锁

## 3. Redis 广播配合go-cache实现多实例缓存功能

go-cache 只在单实例上有用，也就是说，当请求通过负载均衡打在单实例上的时候，任何添加go-cache或移除go-cache的操作都不会对其他实例上的go-cache造成影响，这样就会造成分布式系统上的数据一致性问题。

可以利用redis的发布订阅机制一定程度上解决这个问题：

Redis 的发布订阅模式需要 发送方（pub） 和 接收方（sub）

发送方发送广播，所有接收方实例收到广播，并根据广播的标签找到相应的handler()来执行广播的内容。

在我们的逻辑中，发送方应当在添加和删除go-cache时都进行广播。接收方应当在收到广播后进行go-cache的添加和删除

# go-cache

使用go-cache的一个主要原因是因为redis里面的东西必须变成json才能缓存，这样我们每次还是要unmarshal，失去了加缓存的意义



























