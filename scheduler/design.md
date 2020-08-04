## 可扩展的GO Scheduler设计文档 -2012

[原文](https://golang.org/s/go11sched)

本文档假定对GO语言和当前的Goroutine调度器实现有一定的先验知识。

#### 当前计划程序的问题

 当前的Goroutine调度器限制了用Go编写的并发程序，特别是高吞吐量服务器和并行计算程序的可扩展性。
 Vtocc服务器在8核box上的CPU最大使用率为70%！，而profile显示在runtime.futex()上花费的时间为14%。
 通常，在性能至关重要的情况下，调度程序可能会禁止用户使用惯用的细粒度并发。

###### 当前实现出了什么问题：
1. 单个全局mutex(Sched.Lock)和集中状态。
   mutex保护所有与goroutine相关的操作(创建、完成、重新调度等)。
2. Goroutine(G)切换(G.nextg)。
   工作线程(M‘s)经常在彼此之间切换可运行的Goroutine，这可能会导致延迟增加和额外的开销。
   每个M必须能够执行任何可运行的G，特别是刚刚创建G的M。
3. 每M内存缓存(M.mcache)。
   内存缓存和其他缓存(堆栈分配)与所有M相关联，而它们只需要与M的运行GO代码相关联(在syscall中阻塞的M不需要mcache)。
   M的运行围棋代码与所有M的之间的比率可以高达1：100。
   这会导致过度的资源消耗(每个MCache最多可以占用2M)和较差的数据局部性。
4. 积极的线程阻塞/解除阻塞。
   在系统调用存在的情况下，工作线程经常被阻塞和解锁。
   这增加了很多开销。

### 设计
#### 处理器

其总体思想是在运行时引入P(Processor)的概念，在处理器之上实现工作窃取调度器。
M表示操作系统线程(就像现在一样)。
p表示执行GO代码所需的资源。
当M执行GO代码时，它有一个关联的P。
当M空闲或在系统调用中时，它确实需要P。
确实有GOMAXPROCS P。
所有P都组织成一个数组，这是窃取工作的要求。
GOMAXPROCS更改涉及停止/启动世界以调整P数组的大小。
SCHED中的一些变量被分散并移到P。
将M中的一些变量移到P(与GO代码的活动执行相关的变量)。

```go
struct P
{
Lock;
G *gfree; // freelist, moved from sched
G *ghead; // runnable, moved from sched
G *gtail;
MCache *mcache; // moved from M
FixAlloc *stackalloc; // moved from M
uint64 ncgocall;
GCStats gcstats;
// etc
...
};

P *allp; // [GOMAXPROCS]

// 还有一个空闲P的无锁列表：
P *idlep; // lock-free list

```
1. 当M愿意开始执行GO代码时，它必须从列表中弹出一个P。
2. 当M结束执行GO代码时，它会将P推入列表。
因此，当M执行GO代码时，它必须有一个关联的P。
此机制取代了Schedul.atom(mcpu/mcumax)。

#### 调度

1. 当创建新的G或现有G变为可运行时，将其推入当前P的可运行Goroutine列表中。
2. 当P完成执行G时，它首先尝试从自己的可运行Goroutine列表中弹出一个G；
如果该列表为空，则P随机选择一个受害者(另一个P)，并试图从其中窃取一半可运行Goroutine。

####
类似地，当M进入syscall时，它必须确保有另一个M执行GO代码。
有两种选择，我们可以迅速阻塞和解除阻塞M，或者使用一些旋转。
这是性能和消耗不必要的CPU周期之间的内在冲突。
我们的想法是使用旋转并消耗CPU周期。
但是，它不应该影响使用GOMAXPROCS=1(命令行实用程序、appengine等)运行的程序。
旋转是两级的：(1)具有相关P个自旋的空闲M正在寻找新的G，(2)M没有相关的P个自旋等待可用的P个。
至多有GOMAXPROCS旋转的M((1)和(2))。
当存在类型(2)的空闲M时，类型(1)的空闲M不阻塞。
当产生新的G，或者M进入系统调用，或者M从空闲转换到忙碌时，它确保至少有1个旋转的M(或者所有P都忙碌)。
这确保了没有可以运行的可运行G；
同时避免了过多的M阻塞/解锁。
旋转主要是被动的(屈服于操作系统，sched_Year())，但也可能包括一些主动旋转(循环烧毁CPU)(需要调查和调优)。
