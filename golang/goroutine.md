# 线程&goroutine

* 线程是系统调用  
goroutine调用由go.runtime实现  
调度消耗小  
* m:n模型  
m个线程，n个goroutine  


# GMP
* G  
goroutine
* M  
Machine有线程栈 系统线程
* P 
Processor 虚拟cpu

## GM  
> go1.2前  

* 一个全局的goroutine队列  
M从全局队列中取一个goroutine来运行  
* 问题
    * goroutine全局队列互斥锁，锁征用严重
    * M需要加载缓存G的数据，每个M都有自己的缓存  
    每次调度G不一定分配给哪个M，每次都要重新加载  
    亲缘性差

## GMP  

* 缓存从M挂到P上
* goroutine全局队列global queue  
P上有本地队列local queue   
local queue 采用CAS算法避免加锁

> GOMAXPROCS 限制的是P的数量

# 调度算法
* P执行完G local queue空时  
去global queue取G （当前数量/P的数量）  
global queue为空时 随机去其他local queue抢一半
* local queue 数量达256时推一部分到global queue  
MG执行完阻塞任务时找不到空闲P，G推到global queue  
* 有1/61的概率去global queue取一些

* 线程首次创建时会执行一个特殊的G(g0)  
g0执行调度local queue，在两个goroutine之间切换

## syscall
> 调syscall时解绑P，G阻塞  
P暂时不调度给其他M  
超过sysmon tick(10ms)，P设置为idle(空闲)可以被其他M绑定
syscall结束后M优先找原来的P  
找不到就找空闲的P  
空闲P没有M来绑定执行剩下的G，则创建一个M  

## sysmon
> 监控线程  
无需P运行
死循环 

* 释放5min以上闲置物理内存 
* 2min垃圾回收
* 长时间未处理的netpoll添加到全局队列
* 抢占调度长时间运行的goroutine 
* 收回阻塞状态的P

## 信号抢占
M注册sigurg信号  
出发注册的hangler  
往运行指针PC中插一条指令  
把G推到global queue中

## channel调度
* goroutine因channel被阻塞G被放在runnext中  
G调度时不从local queue中取，先运行runnext中的G

## goroutine recycle
* 每一个P都维护一个local freelist
* 全局维护两个  
    * 一个包含栈 (2K)
    * 一个不包含栈(没被GC扫描过)
