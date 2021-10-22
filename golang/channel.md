# channel结构
```
chan struct {
    lock // 互斥锁

    buf // 数组 实现的队列
    sendx
    recvx
    
    sendq // 等待的发送者
    recvq // 等待的接收者

    closed // 是否关闭
}
```

* 如果发送者先被阻塞  
接收者goroutine唤醒发送者设成runable  
从P的runnext中唤醒

* 如果接收者先阻塞  
不通过buf，发送者直接把数据给接收者  
不需要加锁