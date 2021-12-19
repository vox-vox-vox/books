Oh no

# 解析tcpdump抓包输出 

查看tcp报文的方法： sudo tcpdump -i any -X port 9999 或wireshark

Tcpdump 管道 输入无法实时处理的问题：https://unix.stackexchange.com/questions/15989/how-to-process-pipe-tcpdumps-output-in-realtime





# IO

Strerror 返回指向静态内存的指针。以后学线程库时我们会看到，有些函数的错误码并不保存在errno中，而是通过返回值返回，就不能调用perror打印错误原因了，这时strerror就派上了用场