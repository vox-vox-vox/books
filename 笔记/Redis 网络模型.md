# Redis 网络模型

## 单worker线程+多路IO

这图啥意思

![Screenshot_20211126_190328](https://tva1.sinaimg.cn/large/008i3skNgy1gwsui7laxxj31uw0u0q9d.jpg)

![img](https://tva1.sinaimg.cn/large/008i3skNgy1gwv4df89wkj30qj0hiabl.jpg)

## redis为什么选择单线程

Redis 的核心网络模型选择用单线程来实现。

对于一个 DB 来说，CPU 通常不会是瓶颈，因为大多数请求不会是 CPU 密集型的，而是 I/O 密集型，redis的数据存取基于内存而不是硬盘所以很快，真正瓶颈在网络IO。

- 避免过多的上下文切换开销
- 避免同步机制的开销

## 单reactor模型

多路复用只返回一种是否可读的状态





