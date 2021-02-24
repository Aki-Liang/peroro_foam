# Channel

不要通过共享内存的方式进行通信，应该通过通信的方式共享内存

## 并发模型

通信顺序进程（communicating sequential processes, CSP）

* Goroutine: CSP中的实体
* Channel: CSP中传递信息的媒介

## 先进先出

* 先从Channel读取数据的Goroutine会先接收到数据
* 先向Channel发送数据的Goroutine会得到先发送数据的权利

## 无锁管道

乐观锁：
* 不是真的锁，只是一种并发控制的思想
* 本质上是基于验证的协议

使用原子指令CAS（compare-and-swap或者compare-and-set）在多线程中同步数据

无锁队列的实现也依赖CAS指令

目前通过 CAS 实现11的无锁 Channel 没有提供先进先出的特性，所以该提案暂时被搁浅了

## 数据结构

```
type hchan struct {
	qcount   uint           // total data in the queue
	dataqsiz uint           // size of the circular queue
	buf      unsafe.Pointer // points to an array of dataqsiz elements
	elemsize uint16
	closed   uint32
	elemtype *_type // element type
	sendx    uint   // send index
	recvx    uint   // receive index
	recvq    waitq  // list of recv waiters
	sendq    waitq  // list of send waiters

	// lock protects all fields in hchan, as well as several
	// fields in sudogs blocked on this channel.
	//
	// Do not change another G's status while holding this lock
	// (in particular, do not ready a G), as this can deadlock
	// with stack shrinking.
	lock mutex
}
```

构建底层循环队列的相关字段
* qcount — Channel 中的元素个数；
* dataqsiz — Channel 中的循环队列的长度；
* buf — Channel 的缓冲区数据指针；
* sendx — Channel 的发送操作处理到的位置；
* recvx — Channel 的接收操作处理到的位置；

elemtype，elemsize表示当前Channel能够收发的元素类型和大小

sendq和recvq保存当前Channel因为缓冲区空间不足而阻塞的Goroutine列表

waitq等待队列类型，双向链表，链表中所有元素都是sudog；

sudog表示一个在等待列表中的Goroutine，该结构中保存了两个分别指向前后sudog的指针来构成链表

```
type waitq struct {
	first *sudog
	last  *sudog
}

type sudog struct {
	// The following fields are protected by the hchan.lock of the
	// channel this sudog is blocking on. shrinkstack depends on
	// this for sudogs involved in channel ops.

	g *g

	// isSelect indicates g is participating in a select, so
	// g.selectDone must be CAS'd to win the wake-up race.
	isSelect bool
	next     *sudog
	prev     *sudog
	elem     unsafe.Pointer // data element (may point to stack)

	// The following fields are never accessed concurrently.
	// For channels, waitlink is only accessed by g.
	// For semaphores, all fields (including the ones above)
	// are only accessed when holding a semaRoot lock.

	acquiretime int64
	releasetime int64
	ticket      uint32
	parent      *sudog // semaRoot binary tree
	waitlink    *sudog // g.waiting list or semaRoot
	waittail    *sudog // semaRoot
	c           *hchan // channel
}
```

## 创建管道

所有的管道都使用make关键字创建

编译器会将make(chan int, 10)表达式转换成OMAKE类型的节点，并在类型检查阶段将OMAKE类型的节点转换成OMAKECHAN类型

```
func typecheck1(n *Node, top int) (res *Node) {
	switch n.Op {
	case OMAKE:
		...
		switch t.Etype {
		case TCHAN:
			l = nil
			if i < len(args) { // 带缓冲区的异步 Channel
				...
				n.Left = l
			} else { // 不带缓冲区的同步 Channel
				n.Left = nodintconst(0)
			}
			n.Op = OMAKECHAN
		}
	}
}
```

OMAKECHAN类型的节点最终都会在SSA中间代码生成阶段之前被转换成调用makechan或者makechan64的函数

* makechan64 - 用于处理缓冲区大小超过2^32的情况

这两个函数会根据传入的参数类型和缓冲区大小创建一个新的Channel结构

```
func walkexpr(n *Node, init *Nodes) *Node {
	switch n.Op {
	case OMAKECHAN:
		size := n.Left
		fnname := "makechan64"
		argtype := types.Types[TINT64]

		if size.Type.IsKind(TIDEAL) || maxintval[size.Type.Etype].Cmp(maxintval[TUINT]) <= 0 {
			fnname = "makechan"
			argtype = types.Types[TINT]
		}
		n = mkcall1(chanfn(fnname, 1, n.Type), n.Type, init, typename(n.Type), conv(size, argtype))
	}
}
```
```
func makechan(t *chantype, size int) *hchan {
	elem := t.elem
	mem, _ := math.MulUintptr(elem.size, uintptr(size))

	var c *hchan
	switch {
	case mem == 0:
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		c.buf = c.raceaddr()
	case elem.kind&kindNoPointers != 0:
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default:
		c = new(hchan)
		c.buf = mallocgc(mem, elem, true)
	}
	c.elemsize = uint16(elem.size)
	c.elemtype = elem
	c.dataqsiz = uint(size)
	return c
}
```

* 如果当前Channel中不存在缓冲区，则只为hchan分配内存空间
* 如果当前Channel中存储的类型不是指针类型，则会为当前的Channel和底层数组分配一块连续的内存空间
* 在默认情况下会单独为hchan和缓冲区分配内存
* 最后统一更新elemsize，elemtype，dataqsiz几个字段

## 发送数据

发送数据使用的ch<-i语句会被编译器解析成OSEND节点并在gc.walkexpr中转换成runtime.chansend1

```
func walkexpr(n *Node, init *Nodes) *Node {
	switch n.Op {
	case OSEND:
		n1 := n.Right
		n1 = assignconv(n1, n.Left.Type.Elem(), "chan send")
		n1 = walkexpr(n1, init)
		n1 = nod(OADDR, n1, nil)
		n = mkcall1(chanfn("chansend1", 2, n.Left.Type), nil, init, n.Left, n1)
	}
}
```

chansend1调用chansend并传入Channel和需要发送的数据，如果在调用时将block参数设置成true则表示当前发送操作是阻塞的

```
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	lock(&c.lock)

	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("send on closed channel"))
	}
```

发送数据的逻辑会先为当前Channel加锁，防止并发问题。如果Channel已经关闭则回触发panic “send on closed channel”

### 直接发送

当存在等待的的接受者时，通过runtime.send直接将数据发送给阻塞的接受者

如果目标Channel没有被关闭且已经有处于度等待的Goroutine，chansend会从接收队列中取出最先陷入等待的Goroutine并直接向他发送数据

```
	if sg := c.recvq.dequeue(); sg != nil {
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}
```

发送数据时会调用runtime.send，该函数的执行分为两个部分

1. 调用runtime.sendDirect将发送的数据直接拷贝到x = <-c表达式中变量x所在的内存地址上
2. 调用goready将等待接收数据的Goroutine标记成可运行状态Grunnable并把该Goroutine放到发送方所在的处理器的runnext上等待执行，该处理器在下一次调度时会like唤醒数据的接收方

注意：发送数据的过程只是将接收方的Goroutine放到了处理器的runnext中，程序并没有立刻执行该Goroutine

```
func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	if sg.elem != nil {
		sendDirect(c.elemtype, sg, ep)
		sg.elem = nil
	}
	gp := sg.g
	unlockf()
	gp.param = unsafe.Pointer(sg)
	goready(gp, skip+1)
}
```

### 缓冲区

当缓冲区存在空余空间时，将发送的数据写入Channel的缓冲区

如果创建的Channel包含缓冲区并且Channel中的数据没有装满，会执行以下代码：

```
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	...
	if c.qcount < c.dataqsiz {
		qp := chanbuf(c, c.sendx)
		typedmemmove(c.elemtype, qp, ep)
		c.sendx++
		if c.sendx == c.dataqsiz {
			c.sendx = 0
		}
		c.qcount++
		unlock(&c.lock)
		return true
	}
	...
}
```

* 首先使用chanbuf计算出下一个可以存储数据的位置
* 然后通过typedmemmove将要发送的数据拷贝到缓冲区并增加sendx索引和qcount计数器
* 缓冲区buf是一个循环数组，当sendx等于dataqsiz时会重新回到数组开始位置

### 阻塞发送

当不存在缓冲区或者缓冲区已满时，等待其他Goroutine从Channel接收数据

当Channel没有接收者能够处理数据时，向Channel发送数据会被下游阻塞。向Channel阻塞地发送数据会执行如下代码：

```
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	...
	if !block {
		unlock(&c.lock)
		return false
	}

	gp := getg()
	mysg := acquireSudog()
	mysg.elem = ep
	mysg.g = gp
	mysg.c = c
	gp.waiting = mysg
	c.sendq.enqueue(mysg)
	goparkunlock(&c.lock, waitReasonChanSend, traceEvGoBlockSend, 3)

	gp.waiting = nil
	gp.param = nil
	mysg.c = nil
	releaseSudog(mysg)
	return true
}
```

* 调用getg获取发送数据使用的Goroutine
* 执行acquireSudog获取sudog结构并设置这一次阻塞发送的相关信息，如发送的Channel、是否在select中和待发送数据的内存地址等
* 将刚刚创建并初始化的sudog加入发送等待队列，并设置到当前Goroutine的waiting上，表示Goroutine正在等待该sudog准备就绪
* 调用goparkunlock将当前的Goroutine陷入沉睡等待唤醒
* 被调度器唤醒后执行一些收尾工作，并释放sudog结构体
* 最后返回true表示成功发送了数据

## 接收数据

Go语言中可以使用两种不同的方式去接收Channel中的数据

```
i <- ch                     // <1>
i, ok <- ch                 // <2>
```

这两种不同的方法经过编译器的处理都会变成ORECV节点，<2>方式会在类型检查阶段被转换成OAS2RECV类型。

```
 <- ch ------ORECV---->chanrecv1--|
               |                  --->chanrecv
           OAS2RECV--->chanrecv2--|
```

不同的接收方式会被转换成chanrecv1和chanrecv2两种不同的函数调用，但是这两个函数最终还是会调用chanrecv

```
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	if c == nil {
		if !block {
			return
		}
		gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}

	lock(&c.lock)

	if c.closed != 0 && c.qcount == 0 {
		unlock(&c.lock)
		if ep != nil {
			typedmemclr(c.elemtype, ep)
		}
		return true, false
	}
```

从一个空的Channel中接收数据时会直接调用gopark让出处理器的使用权

如果当前Channel已经被关闭且缓冲区不存在任何数据，那么会清除ep指针中的数据并立刻返回

除了上述两种特殊情况外，使用chanrecv从Channel接收数据时还包括以下三种不同的情况
* 当存在等待的发送者时，通过recv从阻塞的发送者或者缓冲区中获取数据
* 当缓冲区存在数据时，从Channel的缓冲区中接收数据
* 当缓冲区中不存在数据时，等待其他Goroutine向Channel发送数据

### 直接接收

Channel的sendq队列中包含处于等待状态的Goroutine时，取出队列头等待的Goroutine，调用runtime.recv函数

```
    if sg := c.sendq.dequeue(); sg != nil {
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true, true
	}
```

```
func recv(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	if c.dataqsiz == 0 {
		if ep != nil {
			recvDirect(c.elemtype, sg, ep)
		}
	} else {
		qp := chanbuf(c, c.recvx)
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		typedmemmove(c.elemtype, qp, sg.elem)
		c.recvx++
		c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz
	}
	gp := sg.g
	gp.param = unsafe.Pointer(sg)
	goready(gp, skip+1)
}
```

recv函数会根据缓冲区的大小分别处理不同的情况
* Channel不存在缓冲区的情况
    1. 调用recvDirect将Channel发送队列中Goroutine存储的elem数据拷贝到目标内存地址中
* Channel存在缓冲区的情况
    1. 将队列中的数据拷贝到接收方的内存地址
    2. 将发送方队列头的数据拷贝到缓冲区中，释放一个阻塞的发送方

最后都会调用goready将当前处理器的runnext设置成发送数据的Goroutine，在调度器下一次调度时将阻塞的发送方唤醒


### 缓冲区

Channel缓冲区中已经包含数据时，从Channel中接收数据会直接从缓冲区中recvx索引位置取出数据进行处理

```
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	...
	if c.qcount > 0 {
		qp := chanbuf(c, c.recvx)
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		typedmemclr(c.elemtype, qp)
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.qcount--
		return true, true
	}
	...
}
```

如果接收数据的内存地址不为空，则会使用typememmove将缓冲区的数据拷贝到内存中，清除队列中的数据并完成收尾工作

收尾工作包括递增recvx（循环队列，超过Channel容量时将其重置归零），还会减少qcount计数器并释放持有的Channel的锁

### 阻塞接收

Channel的发送队列不存在等待的Goroutine且缓冲区中也不存在任何数据时，从管道中接收数据的操作会变成阻塞的（与select语句结社使用时就可能会食道非阻塞的接收操作）

```
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	...
	if !block {
		unlock(&c.lock)
		return false, false
	}

	gp := getg()
	mysg := acquireSudog()
	mysg.elem = ep
	gp.waiting = mysg
	mysg.g = gp
	mysg.c = c
	c.recvq.enqueue(mysg)
	goparkunlock(&c.lock, waitReasonChanReceive, traceEvGoBlockRecv, 3)

	gp.waiting = nil
	closed := gp.param == nil
	gp.param = nil
	releaseSudog(mysg)
	return true, !closed
}
```

在该场景下会使用sudog将当前Goroutine包装成一个处于等待状态的Goroutine并将其加入到接收队列中

完成入队之后，上述代码还会调用goparkunlock立刻厨房Goroutine的调度，让出处理器的使用权并等待调度器的调度

## 关闭管道

编译器会将用于关闭管道的close关键字转换成OCLOSE节点以及runtime.closechan函数，当关闭的Channel是一个空指针或者已关闭的Channel时，GO语言运行时会直接崩溃并报出异常

```
func closechan(c *hchan) {
	if c == nil {
		panic(plainError("close of nil channel"))
	}

	lock(&c.lock)
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("close of closed channel"))
	}
    ...
    
    c.closed = 1

	var glist gList
	for {
		sg := c.recvq.dequeue()
		if sg == nil {
			break
		}
		if sg.elem != nil {
			typedmemclr(c.elemtype, sg.elem)
			sg.elem = nil
		}
		gp := sg.g
		gp.param = nil
		glist.push(gp)
	}

	for {
		sg := c.sendq.dequeue()
		...
	}
	for !glist.empty() {
		gp := glist.pop()
		gp.schedlink = 0
		goready(gp, 3)
	}
}
```

closechan函数会将recvq和sendq两个队列中的数据加入到Goroutine列表gList中，于此同时该函数会清除所有sudog上未被处理的元素

在函数的最后会为所有被阻塞的Groutine调用goready触发调度


-----
# 原文
https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-channel/















