# GeeRPC

## 异步调用

调用方不会被阻塞，



- client --> server 的信息，server怎么保证有序处理
- server --> client 的信息，client怎么保证有序处理



可能有多个client，多个client开多个goroutine，一个client对应server中的一个goroutine

可能一个client有多个rpc调用，一个client也可以开启并发，我们的client是支持异步和并发的

server 读request时for循环处理，handle时新开一个goroutine，并用waitgroup处理

**所有的开多个可能的goroutine都要利用waitgroup**







