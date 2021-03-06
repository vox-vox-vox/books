# DNS

域名系统DNS能把互联网上的域名转换为IP地址

# 域名

```go
mail.cctv.com
  |		 |   |---顶级域名
	|    |---二级域名
	|---三级域名
```

# DNS服务器

域名服务器有多种类型

一个服务器的管辖范围称为一个区，每个服务器保存该区的所有域名到IP的映射

```
						根域名服务器
						/				\
			顶级域名服务器	顶级域名服务器
      	/
    权限域名服务器 ...	
```

- 根域名服务器：所有根域名服务器都知道所有顶级域名服务器的IP和域名。全世界台数不多
- 顶级域名服务器：根域名服务器的子结构
- 权限域名服务器：子结构
- 本地域名服务器：主机首先查询的DNS服务器

# DNS查询过程

DNS查询报文采用UDP，为了减少开销

## 迭代查询

![img](https://images2015.cnblogs.com/blog/464291/201707/464291-20170703113844956-354755333.jpg)

## 高速缓存

每个域名服务器都存有高速缓存。由于`(IP,域名)`的绑定关系不常改变，高速缓存是必要的，数据一致性常常能得到保证，超时时间可以设置的长一点

（10min->24h）。

