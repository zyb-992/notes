# Go

---

## go命令执行原理

> https://halfrost.com/go_command/
>
> https://hyper0x.github.io/go_command_tutorial/#/0.0

## go/ast

### go/type

`type.Config.Check(...)`方法是将传入的`files []*ast.File`解析处理，写入`type.Package`的`type.Scope`中

## GMP

> https://blog.devtrovert.com/p/goroutine-scheduler-revealed-youll
>
> https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part2.html
>
> https://medium.com/@ankur_anand/illustrated-tales-of-go-runtime-scheduler-74809ef6d19b
>
> https://nghiant3223.github.io/2025/04/15/go-scheduler.html

##### 旧调度器

旧的调度器只有G、M，而新建且尚未运行的G都是存放在调度器的一个全局队列中，存放/获取时都需要加锁以对G进行操作，存在对锁粒度的大量资源消耗

首个版本的调度器存在的问题

- `blocking threads`存在大量无意义的`memory cached`，在线程系统调用时没有被释放
- 仅有`全局运行队列(GRQ)`，导致从`GRQ`中插入/获取`G`时都需要加锁，消耗`CPU`资源
- 某个`G`绑定在`M`上执行时，对从该`M`新创建的`G`都是放入`GRQ`中，不太高效

**Context-Switching**

​	用户态/内核态

Go运行时调度器宏观上存在G、M、P三种数据结构，

- “**自旋线程**“：如果工作线程(M)没有本地工作（即M附属的P中没有G可运行），且全局运行队列(`Global Queue`)或者网络轮询器中（`NetPoller`)没有符合`GRunable`运行状态的G

  `m.spining`：该线程是否进入“自旋”

  `schedt.nmspinning`：系统当前”自旋“线程数

#### 源码相关函数

> go version: 1.21.6
>
> path: runtime/proc.go

##### 全局变量

```go
gomaxprocs
allg
// len(allp) = gomaxprocs
allp
allm


type sched struct{
 gFree // 全局的g缓存列表   
 
 runq
 runqsize
}

type g struct{
	stack       stack   // offset known to runtime/cgo
    _panic    	*_panic // innermost panic - offset known to liblink
	_defer    	*_defer // innermost defer
	m         	*m      // current m; offset known to arm liblink
	schedlink    guintptr
}

type p struct{
    runnext guintptr
    runq [256]guintptr // 每个p的本地队列最多支持存储256个g
    runqhead, runqtail uint32	
}

type m struct {
    mOS 
}

```



##### m

```go
stopm

// 实际是个单向链表组成的栈，新来的数据插入链表头
mput(mp *m)
//1. 获取sched.midle，设置m.schedlink = sched.midle
//2. 设置sched的midle=mp
mget() *m
//1. 获取sched.midle,即即将运行的m
//2.设置sched.midle为m.schedlink

mPark()
// 1. causes a thread to park itself, returning once woken.

// 新创建的工作线程的入口
mstart0()
//1. 调用mstart1()，等待函数返回
//2. 调用mexit()结束

mstart1()
//1. minit()初始化m
//2. 阻塞调用schedule()一直获取g执行

minit()
//1. 执行系统调用stdcall7创建线程句柄
```



##### g

```go
schedule()

execute(gp *g, inheritTime bool)
//1. 执行g在当前的工作线程m上
//2. g状态由_Grunnable->_Grunning
//3. 调用gogo函数执行协程

findRunnable()
//1. 当前是否在stw
	// 1.1 是，调用gcstopm()解绑p，阻塞当前线程工作
//2. 尝试从GCworker中获取g执行
//3. p.schedtick % 61 == 0 且调度器全局队列数 > 0: globrunqget(pp,1) 只能取一个g
//4. 从g附属的p的本地运行队列中获取g
//5. 从调度器的全局运行队列中获取g(globrunqget(pp,0))
//6. 从net poller获取g
//7. 调用stealWork从其他p里获取g

globrunqget(pp *p, max int32)
//1. 根据调度器当前的全局运行队列数与runtime.gomaxprocs计算出每个p平均能获取到的g个数
//2. n必须<=全局运行队列数
//3. max>0 且 n>max -> n = max
//4. 如果n超过了当前g附属的p的本地运行队列(local queue)的一半，那么n=len(p.runq) / 2
//5. 从调度器的全局运行队列中取出n个g，放入当前p中，返回首个取出的g

newproc(fn *funcval)
//1. 调用newproc1创建新的g
//2. 将新g放入当前的g的p中(runqput)

runqput(p,g,next)
//1. next=true，g设置为p.runnext
//2. next=false,g放置p.runq尾部
```



## 内存管理

> https://medium.com/@ankur_anand/a-visual-guide-to-golang-memory-allocator-from-ground-up-e132258453ed
>
> https://go.dev/ref/mem

## Channel

**源码阅读**

| Code                                                    | Action | Source Code                                                  | Block            |
| ------------------------------------------------------- | ------ | ------------------------------------------------------------ | ---------------- |
| `make(chan int64)`                                      | 初始化 | `runtime.makechan`                                           |                  |
| `ch <- v`                                               | 发送   | `runtime.chansend1` -> `runtime.chansend(block: true)`       | 阻塞             |
| `select`<br />`case ch <- v: ...`<br />`default`        | 发送   | `runtime.selectnbsend` -> `runtime.chansend(block: false)`   | 非阻塞           |
| `reflect.Value.Send`<br />`reflect.Value.TrySend`       | 发送   | `reflect.chansend` -> `reflect.chansend0` -> `runtime.reflect_chansend(nb bool)` | 阻塞<br />非阻塞 |
| `v  <- ch`                                              | 接收   | `runtime.chanrecv1` -> `runtime.chanrecv`                    | 阻塞             |
| `select`<br />`case v, ok = <- ch : ...`<br />`default` | 接收   | `runtime.selectnbrecv` -> `runtime.chanrecv`                 | 非阻塞           |
| `reflect.Value.Recv`<br />`reflect.Value.TrvRecv`       | 接收   | `reflect.chanrecv` -> `runtime.reflect_chanrecv(nb :bool)` -> `runtime.chanrecv(nb:bool)` | 阻塞<br />非阻塞 |

#### 发送：chansend函数

1、channel为空

- 非阻塞：返回
  - `reflect.Value.TrySend`
  - `select`<br />`case c <- v: ...`<br />`default`：执行default分支逻辑
- 阻塞：**panic**
  - `v <- ch`
  - `reflect.Value.Send`
  - `select`<br />`case c <- v: ...`<br />

```go
if c == nil {
    if !block {
        return false
    }
    gopark(nil, nil, waitReasonChanSendNilChan, traceBlockForever, 2)
    throw("unreachable")
}
```

2、非阻塞情况 && 管道未关闭 && 管道缓冲区已满：直接返回

```go
if !block && c.closed == 0 && full(c) {
    return false
}
```

3、上锁，检测管道已关闭，解锁并抛出`panic`

```go
lock(&c.lock)

if c.closed != 0 {
    unlock(&c.lock)
    panic(plainError("send on closed channel"))
}
```

4、管道接收队列不为空（即管道缓冲区当前没有数据，所以接收队列里才有值

发送的元素数据直接通过send方法发送给接收方（内存拷贝）

```go
if sg := c.recvq.dequeue(); sg != nil {
    // Found a waiting receiver. We pass the value we want to send
    // directly to the receiver, bypassing the channel buffer (if any).
    send(c, sg, ep, func() { unlock(&c.lock) }, 3)
    return true
}
```

5、管道缓冲区未满，获取当前缓冲区中的`发送索引`，并通过`typedmemmove`将数据写入缓冲区，更新`发送索引`；解锁，不阻塞地返回

```go
if c.qcount < c.dataqsiz {
    // Space is available in the channel buffer. Enqueue the element to send.
    qp := chanbuf(c, c.sendx)
    if raceenabled {
        racenotify(c, c.sendx, nil)
    }
    typedmemmove(c.elemtype, qp, ep)
    c.sendx++
    if c.sendx == c.dataqsiz {
        c.sendx = 0
    }
    c.qcount++
    unlock(&c.lock)
    return true
}
```

6、

非阻塞情况：（缓冲区满了且接收队列中没有接收方），直接返回

阻塞情况：

- 获取当前运行的`goroutine`与`sudog`
- 结合`gp`封装`sudog`结构
- 将包含当前待发送数据的`goroutine`加入管道的生产队列
- 调用`gopark`方法阻塞该`goroutine`

```go
if !block {
    unlock(&c.lock)
    return false
}

/* 阻塞的情况：*/

// Block on the channel. Some receiver will complete our operation for us.
gp := getg()
mysg := acquireSudog()
mysg.releasetime = 0
if t0 != 0 {
    mysg.releasetime = -1
}
// No stack splits between assigning elem and enqueuing mysg
// on gp.waiting where copystack can find it.
mysg.elem = ep
mysg.waitlink = nil
mysg.g = gp
mysg.isSelect = false
mysg.c = c
gp.waiting = mysg
gp.param = nil
c.sendq.enqueue(mysg)
// Signal to anyone trying to shrink our stack that we're about
// to park on a channel. The window between when this G's status
// changes and when we set gp.activeStackChans is not safe for
// stack shrinking.
gp.parkingOnChan.Store(true)
gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceBlockChanSend, 2)
// Ensure the value being sent is kept alive until the
// receiver copies it out. The sudog has a pointer to the
// stack object, but sudogs aren't considered as roots of the
// stack tracer.
KeepAlive(ep)

```

7、当前`goroutine`被唤醒，证明有接收方通过管道唤醒了该`goroutine`，并已经将数据直接拷贝给了接收方，所以唤醒当前`goroutine`继续执行其原有的逻辑，不阻塞下去了

```go
// someone woke us up.
if mysg != gp.waiting {
    throw("G waiting list is corrupted")
}
gp.waiting = nil
gp.activeStackChans = false
closed := !mysg.success
gp.param = nil
if mysg.releasetime > 0 {
    blockevent(mysg.releasetime-t0, 2)
}
mysg.c = nil
releaseSudog(mysg)
if closed {
    if c.closed == 0 {
        throw("chansend: spurious wakeup")
    }
    panic(plainError("send on closed channel"))
}
return true
```

#### 接收：chanrecv函数

1、管道为空，非阻塞情况：直接返回，阻塞情况：抛出`panic`

```go
if c == nil {
    if !block {
        return
    }
    gopark(nil, nil, waitReasonChanReceiveNilChan, traceBlockForever, 2)
    throw("unreachable")
}
```

2、非阻塞情况，管道缓冲区且发送队列都为空，返回管道类型零值

```
if !block && empty(c) {
    if atomic.Load(&c.closed) == 0 {
        return
    }

    if empty(c) {
        if raceenabled {
            raceacquire(c.raceaddr())
        }
        if ep != nil {
            typedmemclr(c.elemtype, ep)
        }
        return true, false
    }
}
```

3、上锁
管道已关闭，当前缓冲区数据为空，返回管道类型零值

管道未关闭，且发送队列取到某个`goroutine`，唤醒该协程并拷贝数据

```go
lock(&c.lock)

if c.closed != 0 {
    if c.qcount == 0 {
        if raceenabled {
            raceacquire(c.raceaddr())
        }
        unlock(&c.lock)
        if ep != nil {
            typedmemclr(c.elemtype, ep)
        }
        return true, false
    }
    // The channel has been closed, but the channel's buffer have data.
} else {
    // Just found waiting sender with not closed.
    if sg := c.sendq.dequeue(); sg != nil {
        // Found a waiting sender. If buffer is size 0, receive value
        // directly from sender. Otherwise, receive from head of queue
        // and add sender's value to the tail of the queue (both map to
        // the same buffer slot because the queue is full).
        recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
        return true, true
    }
}
```

4、此时管道可能关闭，也可能未关闭

- *管道已关闭但管道缓冲区队列还有数据*：从缓冲区中获取数据
- *管道未关闭且发送队列无数据，但管道缓冲区队列还有数据*：从缓冲区中获取数据

```go
if c.qcount > 0 {
    // Receive directly from queue
    qp := chanbuf(c, c.recvx)
    if raceenabled {
        racenotify(c, c.recvx, nil)
    }
    if ep != nil {
        typedmemmove(c.elemtype, ep, qp)
    }
    typedmemclr(c.elemtype, qp)
    c.recvx++
    if c.recvx == c.dataqsiz {
        c.recvx = 0
    }
    c.qcount--
    unlock(&c.lock)
    return true, true
}
```

5、如果管道缓冲区和发送队列都为空，但非阻塞的情况，此时直接返回；

阻塞情况：**加入接收队列等待生产者来唤醒**

```

if !block {
    unlock(&c.lock)
    return false, false
}

// no sender available: block on this channel.
gp := getg()
mysg := acquireSudog()
mysg.releasetime = 0
if t0 != 0 {
    mysg.releasetime = -1
}
// No stack splits between assigning elem and enqueuing mysg
// on gp.waiting where copystack can find it.
mysg.elem = ep
mysg.waitlink = nil
gp.waiting = mysg
mysg.g = gp
mysg.isSelect = false
mysg.c = c
gp.param = nil
c.recvq.enqueue(mysg)
// Signal to anyone trying to shrink our stack that we're about
// to park on a channel. The window between when this G's status
// changes and when we set gp.activeStackChans is not safe for
// stack shrinking.
gp.parkingOnChan.Store(true)
gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceBlockChanRecv, 2)

```

6、已获取到数据，取消阻塞逻辑，当前协程已被唤醒，继续执行原有的逻辑

```go
// someone woke us up
if mysg != gp.waiting {
    throw("G waiting list is corrupted")
}
gp.waiting = nil
gp.activeStackChans = false
if mysg.releasetime > 0 {
    blockevent(mysg.releasetime-t0, 2)
}
success := mysg.success
gp.param = nil
mysg.c = nil
releaseSudog(mysg)
return true, success
```



#### QA

- Q: `runtime.makechan`中初始化`hchan`时，在元素包含指针时为什么要分开分配hchan和缓冲区的内存
  - A:
    - 如果hchan和指针元素内存连续，GC会扫描整个连续内存块，分开分配可以让GC更精确地只扫描包含指针的缓冲区部分
    - 指针元素通常需要特殊的内存对齐要求，分开分配可以确保指针元素获得正确的内存对齐
    - 指针元素的写操作需要写屏障(Write Barrier)，分开分配可以限制写屏障的作用范围
- Q: `select`语句中`nil`通道的特殊处理
  - A:在select语句的实现中(runtime/select.go)，编译器会为每个case生成特殊代码。在评估各个case是否可执行时，会先检查通道是否为nil：
    1. 如果通道是nil，该case会被标记为"不可执行"
    2. select机制会跳过所有不可执行的case
    3. 这个过程不会导致panic，而是简单地忽略该case

#### 延伸阅读

1. **GMP模型**
2. **垃圾回收机制**

## 编译过程
