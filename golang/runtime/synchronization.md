# Synchronization

## 基本原语

* sync.Mutex
* sync.RWMutex
* sync.WaitGroup
* sync.Once
* sync.Cond

这些基本原语提供了基础的同步功能，在多数情况下应该使用抽象层级更高的Channel实现同步

### Mutex

Go语言的sync.Mutex由两个字段state和sema组成。其中state表示当前互斥锁的状态，sema是用于控制锁状态的信号量

```
type Mutex struct {
	state int32
	sema  uint32
}
```

state低三维分别表示mutexLocked、mutexWoken和mutexStarving，剩下的位置表示当前有多少个Goroutine在等待互斥锁释放

默认情况下互斥锁的所有状态位都是0

```
|-------------waitersCount---------|starving|woken|locked|
```

* mutexLocked 锁定状态
* mutexWoken 从正常模式被唤醒
* mutexStarving 进入饥饿状态
* waitersCount 当前互斥锁上等待的Goroutine个数

#### 正常模式和饥饿模式

正常模式下锁的等待者会按照先进先出的顺序获取锁。

刚被唤起的Goroutine与新创建的Goroutine竞争时，大概率会获取不到锁，为了减少这种情况出现，一旦Goroutine超过1ms没有获取到锁就会将当前互斥锁切换到饥饿模式

饥饿模式在GO 1.9中引入[sync: make Mutex more fair](https://github.com/golang/go/commit/0556e26273f704db73df9e7c4c3d2e8434dec7be)，用于保证互斥锁的公平性

在饥饿模式中，互斥锁会直接交给等待队列最前面的Goroutine。新的Goroutine在该状态下不能获取锁，也不会进入自旋状态，而是在队列末尾等待。

如果一个Goroutine获得了互斥锁，并且在队列末尾，或者等待时间少于1ms，互斥锁就会切换回正常模式

#### 加锁

加锁，当锁的状态是0时，将mutexLocked位置成1
```
func (m *Mutex) Lock() {
	if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
		return
	}
	m.lockSlow()
}
```

如果互斥锁状态不为0，就会调用`sync.Mutex.lockSlow`尝试通过自旋等方式等待锁的释放。

该方法主体是一个大循环

1. 判断当前Goroutine能否进入自旋
2. 通过自旋等待互斥锁的释放
3. 计算互斥锁的最新状态
4. 更新互斥锁的状态并获取锁

```
func (m *Mutex) lockSlow() {
	var waitStartTime int64
	starving := false
	awoke := false
	iter := 0
	old := m.state
	for {
		if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
			if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
				atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
				awoke = true
			}
			runtime_doSpin()
			iter++
			old = m.state
			continue
		}
```


自旋是一种多线程同步机制，当前的进程在进入自旋的过程中会一直保持 CPU 的占用，持续检查某个条件是否为真。在多核的 CPU 上，自旋可以避免 Goroutine 的切换，使用恰当会对性能带来很大的增益，但是使用的不恰当就会拖慢整个程序，所以 Goroutine 进入自旋的条件非常苛刻：

1. 互斥锁只有在普通模式才能进入自旋
2. `runtime.sync_runtime_canSpin`返回true
    * 运行在多CPU的机器上
    * 当前Goroutine为了获取该锁进入自旋的次数小于四次
    * 当前机器上至少存在一个正在运行的处理器P并且处理的运行队列为空

```
// Active spinning for sync.Mutex.
//go:linkname sync_runtime_canSpin sync.runtime_canSpin
//go:nosplit
func sync_runtime_canSpin(i int) bool {
	// sync.Mutex is cooperative, so we are conservative with spinning.
	// Spin only few times and only if running on a multicore machine and
	// GOMAXPROCS>1 and there is at least one other running P and local runq is empty.
	// As opposed to runtime mutex we don't do passive spinning here,
	// because there can be work on global runq or on other Ps.
	if i >= active_spin || ncpu <= 1 || gomaxprocs <= int32(sched.npidle+sched.nmspinning)+1 {
		return false
	}
	if p := getg().m.p.ptr(); !runqempty(p) {
		return false
	}
	return true
}
```

一旦当前Goroutine能够进入自旋就会调用`runtime.sync_runtime_doSpin`和`runtime.procyield`并执行30次PAUSE指定，该指令只会占用CPU并消耗CPU的时间

```
func sync_runtime_doSpin() {
	procyield(active_spin_cnt)
}

TEXT runtime·procyield(SB),NOSPLIT,$0-0
	MOVL	cycles+0(FP), AX
again:
	PAUSE
	SUBL	$1, AX
	JNZ	again
	RET
```

处理自旋相关逻辑后，互斥锁会根据上下文计算当前互斥锁最新的状态。根据不同的条件分别更新state字段中存储的信息
* mutexLocked
* mutexStarving
* mutexWoken
* mutexWaiterShift

```
		new := old
		if old&mutexStarving == 0 {
			new |= mutexLocked
		}
		if old&(mutexLocked|mutexStarving) != 0 {
			new += 1 << mutexWaiterShift
		}
		if starving && old&mutexLocked != 0 {
			new |= mutexStarving
		}
		if awoke {
			new &^= mutexWoken
		}
```
计算新的互斥锁状态后会使用CAS函数`sync/atomic.CompareAndSwarpInt32`更新状态
```
		if atomic.CompareAndSwapInt32(&m.state, old, new) {
			if old&(mutexLocked|mutexStarving) == 0 {
				break // 通过 CAS 函数获取了锁
			}
			...
			runtime_SemacquireMutex(&m.sema, queueLifo, 1)
			starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
			old = m.state
			if old&mutexStarving != 0 {
				delta := int32(mutexLocked - 1<<mutexWaiterShift)
				if !starving || old>>mutexWaiterShift == 1 {
					delta -= mutexStarving
				}
				atomic.AddInt32(&m.state, delta)
				break
			}
			awoke = true
			iter = 0
		} else {
			old = m.state
		}
	}
}
```
如果没有通过CAS获得锁，会调用`runtime.sync_runtime_SemacquireMutex`通过信号量保证资源不会被两个Goroutine获取。`runtime.sync_runtime_SemacquireMutex`会在方法中不断尝试获取并陷入休眠等待信号量的释放，一旦当前Goroutine可以获取信号量就会立刻返回

* 正常模式下，这段代码会设置唤醒和饥饿标记、重置迭代次数并重新执行获取锁的循环
* 饥饿模式下，当前Goroutine会获得互斥锁，如果等待队列中只存在当前Goroutine，互斥锁还会从饥饿模式中退出

#### 解锁

解锁过程`sync.Mutex.Unlock`会先使用`sync/atomic.AddInt32`函数快速解锁，会有两种情况：
* 如果该函数返回的新状态等于0，当前Goroutine就成功解锁了互斥锁
* 如果该函数返回的新状态不等于0，这段代码就会调用`sync.Mutex.unlockSlow`开始慢速解锁
```
func (m *Mutex) Unlock() {
	new := atomic.AddInt32(&m.state, -mutexLocked)
	if new != 0 {
		m.unlockSlow(new)
	}
}
```

`sync.Mutex.unlockSlow`先校验锁状态的合法性，如果当前互斥锁已经被解锁过了会直接抛出异常“sync：unlock of unlocked mutex”中止当前程序

```
func (m *Mutex) unlockSlow(new int32) {
	if (new+mutexLocked)&mutexLocked == 0 {
		throw("sync: unlock of unlocked mutex")
	}
	if new&mutexStarving == 0 { // 正常模式
		old := new
		for {
			if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
				return
			}
			new = (old - 1<<mutexWaiterShift) | mutexWoken
			if atomic.CompareAndSwapInt32(&m.state, old, new) {
				runtime_Semrelease(&m.sema, false, 1)
				return
			}
			old = m.state
		}
	} else { // 饥饿模式
		runtime_Semrelease(&m.sema, true, 1)
	}
}
```
* 正常模式下
    * 如果互斥锁不存在等待者，或者互斥锁的mutexLocked、mutexStarving、mutexWoken状态不都为0，则当前方法可以直接返回，不需要唤醒其他等待者
    * 如果互斥锁存在等待者，会通过`sync.runtime_Semrelease`唤醒等待者并移交锁的所有权
* 在饥饿模式下，则会直接调用`sync.runtime_Semrelease`将当前锁交给下一个正在尝试获取锁的等待者，等待者被唤醒后会得到锁，这时互斥锁还不会退出饥饿状态

### RWMutex

```
type RWMutex struct {
	w           Mutex
	writerSem   uint32
	readerSem   uint32
	readerCount int32
	readerWait  int32
}
```

* w 复用互斥锁提供的功能
* witerSem和readerSem 分别用于写等待读和读等待写
* readerCount 存储当前正在执行的读操作数量
* readerWait 表示当写操作被阻塞时等待的读操作个数

#### 写锁

获取写锁调用sync.RWMutex.Lock方法
```
func (rw *RWMutex) Lock() {
	rw.w.Lock()
	r := atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders) + rwmutexMaxReaders
	if r != 0 && atomic.AddInt32(&rw.readerWait, r) != 0 {
		runtime_SemacquireMutex(&rw.writerSem, false, 0)
	}
}
```

* 调用结构体持有的sync.Mutex结构体的sync.Mutex.Lock阻塞后续的写操作
    * 其他Goroutine在获取写锁时会进入自旋或者休眠
* 调用sync/atomic.AddInt32函数阻塞后续的读操作
* 如果有其他的Goroutine持有互斥锁的读锁，该Goroutine会调用runtime.sync_runtime_SemacquireMutex进入休眠状态等待所有读锁所有者执行结束后释放writerSem信号量将当前协程唤醒

释放写锁调用sync.RWMutex.unlock

```
func (rw *RWMutex) Unlock() {
	r := atomic.AddInt32(&rw.readerCount, rwmutexMaxReaders)
	if r >= rwmutexMaxReaders {
		throw("sync: Unlock of unlocked RWMutex")
	}
	for i := 0; i < int(r); i++ {
		runtime_Semrelease(&rw.readerSem, false, 0)
	}
	rw.w.Unlock()
}
```

* 调用sync/atomic.AddInt32函数将readCount变回正数，释放读锁
* 通过for循环释放所有因为获取读锁而陷入等待的Goroutine
* 调用sync.Mutex.unlock释放写锁

获取写锁时会先阻塞写锁的获取，后阻塞读锁的获取，这种策略能够保证读操作不会被连续的写操作“饿死”

#### 读锁

读锁的加锁方法sync.RWMutex.RLock会通过sync/atomic.AddInt32将readerCount加一

```
func (rw *RWMutex) RLock() {
	if atomic.AddInt32(&rw.readerCount, 1) < 0 {
		runtime_SemacquireMutex(&rw.readerSem, false, 0)
	}
}
```

* 如果该方法返回负数， 其他Goroutine获得了写锁，当前Goroutine就会调用runtime.sync_runtime_SemacquireMutex陷入休眠等待锁的释放
* 如果该方法的结果为非负数， 没有Goroutine或缺的写锁，当前方法会成功返回

释放读锁调用sync.RWMutex.RUnlock方法

```
func (rw *RWMutex) RUnlock() {
	if r := atomic.AddInt32(&rw.readerCount, -1); r < 0 {
		rw.rUnlockSlow(r)
	}
}
```

* 如果返回值大于等于0，读锁直接解锁成功
* 如果返回值小于0， 有一个正在执行的写操作，调用sync.RWMutex.rUnlockSlow方法

```
func (rw *RWMutex) rUnlockSlow(r int32) {
	if r+1 == 0 || r+1 == -rwmutexMaxReaders {
		throw("sync: RUnlock of unlocked RWMutex")
	}
	if atomic.AddInt32(&rw.readerWait, -1) == 0 {
		runtime_Semrelease(&rw.writerSem, false, 1)
	}
}
```

sync.RWMutex.rUnlockSlow 会减少获取锁的写操作等待的读操作数 readerWait 并在所有读操作都被释放之后触发写操作的信号量 writerSem，该信号量被触发时，调度器就会唤醒尝试获取写锁的 Goroutine。

### WaitGroup

sync.WaitGroup结构体中只包含两个成员变量

```
type WaitGroup struct {
	noCopy noCopy
    // 64-bit value: high 32 bits are counter, low 32 bits are waiter count.
	// 64-bit atomic operations require 64-bit alignment, but 32-bit
	// compilers do not ensure it. So we allocate 12 bytes and then use
	// the aligned 8 bytes in them as state, and the other 4 as storage
	// for the sema.
	state1 [3]uint32
}
```

* noCopy 保证sync.WaitGroup不会被开发者通过再复制的方式拷贝
* state1 存储着状态还信号量

sync.noCopy 是一个特殊的私有结构体，tools/go/analysis/passes/copylock 包中的分析器会在编译期间检查被拷贝的变量中是否包含 sync.noCopy 或者实现了 Lock 和 Unlock 方法，如果包含该结构体或者实现了对应的方法就会报出以下错误：

```
func main() {
	wg := sync.WaitGroup{}
	yawg := wg
	fmt.Println(wg, yawg)
}

$ go vet proc.go
./prog.go:10:10: assignment copies lock value to yawg: sync.WaitGroup
./prog.go:11:14: call of fmt.Println copies lock value: sync.WaitGroup
./prog.go:11:18: call of fmt.Println copies lock value: sync.WaitGroup
```

state1 一个占用12字节的数组，存储当前结构体的状态，在64位和32位的机器上表现不同

```
64-bit |waiter|counter|sema|
32bit  |sema|waiter|counter|
```
sync.WaitGroup 提供的私有方法 sync.WaitGroup.state 能够帮我们从 state1 字段中取出它的状态和信号量。

#### 接口

sync.WaitGroup.Done
```
func (wg *WaitGroup) Done() {
	wg.Add(-1)
}
```

sync.WaitGroup.Add

```
func (wg *WaitGroup) Add(delta int) {
	statep, semap := wg.state()
	state := atomic.AddUint64(statep, uint64(delta)<<32)
	v := int32(state >> 32)
	w := uint32(state)
	if v < 0 {
		panic("sync: negative WaitGroup counter")
	}
	if v > 0 || w == 0 {
		return
	}
	*statep = 0
	for ; w != 0; w-- {
		runtime_Semrelease(semap, false, 0)
	}
}
```

sync.WaitGroup.Add 可以更新 sync.WaitGroup 中的计数器 counter。虽然 sync.WaitGroup.Add 方法传入的参数可以为负数，但是计数器只能是非负数，一旦出现负数就会发生程序崩溃。当调用计数器归零，即所有任务都执行完成时，才会通过 sync.runtime_Semrelease 唤醒处于等待状态的 Goroutine。

sync.WaitGroup.Wait 在计数器大于 0 并且不存在等待的 Goroutine 时，调用 runtime.sync_runtime_Semacquire 陷入睡眠。

```
func (wg *WaitGroup) Wait() {
	statep, semap := wg.state()
	for {
		state := atomic.LoadUint64(statep)
		v := int32(state >> 32)
		if v == 0 {
			return
		}
		if atomic.CompareAndSwapUint64(statep, state, state+1) {
			runtime_Semacquire(semap)
			if +statep != 0 {
				panic("sync: WaitGroup is reused before previous Wait has returned")
			}
			return
		}
	}
}
```

当 sync.WaitGroup 的计数器归零时，陷入睡眠状态的 Goroutine 会被唤醒，上述方法也会立刻返回。

### Once

```
func main() {
    o := &sync.Once{}
    for i := 0; i < 10; i++ {
        o.Do(func() {
            fmt.Println("only once")
        })
    }
}

$ go run main.go
only once
```

每一个 sync.Once 结构体中都只包含一个用于标识代码块是否执行过的 done 以及一个互斥锁 sync.Mutex：

```
type Once struct {
	done uint32
	m    Mutex
}
```

sync.Once.Do 是Once结构体对外唯一暴露的方法
* 如果传入的函数已经执行过，会直接返回；
* 如果传入的函数没有执行过，会调用 sync.Once.doSlow 执行传入的函数

```
func (o *Once) Do(f func()) {
	if atomic.LoadUint32(&o.done) == 0 {
		o.doSlow(f)
	}
}

func (o *Once) doSlow(f func()) {
	o.m.Lock()
	defer o.m.Unlock()
	if o.done == 0 {
		defer atomic.StoreUint32(&o.done, 1)
		f()
	}
}
```

1. 为当前 Goroutine 获取互斥锁；
2. 执行传入的无入参函数；
3. 运行延迟函数调用，将成员变量 done 更新成 1；

sync.Once 会通过成员变量 done 确保函数不会执行第二次。

### Cond

sync.Cond可以让一组Goroutine在满足特定条件时被唤醒。每一个sync.Cond结构体在初始化时都需要传入一个互斥锁。

例：
```
var status int64

func main() {
	c := sync.NewCond(&sync.Mutex{})
	for i := 0; i < 10; i++ {
		go listen(c)
	}
	time.Sleep(1 * time.Second)
	go broadcast(c)

	ch := make(chan os.Signal, 1)
	signal.Notify(ch, os.Interrupt)
	<-ch
}

func broadcast(c *sync.Cond) {
	c.L.Lock()
	atomic.StoreInt64(&status, 1)
	c.Broadcast()
	c.L.Unlock()
}

func listen(c *sync.Cond) {
	c.L.Lock()
	for atomic.LoadInt64(&status) != 1 {
		c.Wait()
	}
	fmt.Println("listen")
	c.L.Unlock()
}

$ go run main.go
listen
...
listen
```

* 10 个 Goroutine 通过 sync.Cond.Wait 等待特定条件的满足；
* 1 个 Goroutine 会调用 sync.Cond.Broadcast 唤醒所有陷入等待的 Goroutine；

sync.Cond结构体

```
type Cond struct {
	noCopy  noCopy
	L       Locker
	notify  notifyList
	checker copyChecker
}
```

* noCopy — 用于保证结构体不会在编译期间拷贝；
* copyChecker — 用于禁止运行期间发生的拷贝；
* L — 用于保护内部的 notify 字段，Locker 接口类型的变量；
* notify — 一个 Goroutine 的链表，它是实现同步机制的核心结构；

```
type notifyList struct {
	wait uint32
	notify uint32

	lock mutex
	head *sudog
	tail *sudog
}
```

sync.Cond 对外暴露的 sync.Cond.Wait 方法会将当前 Goroutine 陷入休眠状态，它的执行过程分成以下两个步骤：

1. 调用 runtime.notifyListAdd 将等待计数器加一并解锁；
2. 调用 runtime.notifyListWait 等待其他 Goroutine 的唤醒并加锁：

```
func (c *Cond) Wait() {
	c.checker.check()
	t := runtime_notifyListAdd(&c.notify) // runtime.notifyListAdd 的链接名
	c.L.Unlock()
	runtime_notifyListWait(&c.notify, t) // runtime.notifyListWait 的链接名
	c.L.Lock()
}

func notifyListAdd(l *notifyList) uint32 {
	return atomic.Xadd(&l.wait, 1) - 1
}
```

runtime.notifyListWait 会获取当前 Goroutine 并将它追加到 Goroutine 通知链表的最末端：

```
func notifyListWait(l *notifyList, t uint32) {
	s := acquireSudog()
	s.g = getg()
	s.ticket = t
	if l.tail == nil {
		l.head = s
	} else {
		l.tail.next = s
	}
	l.tail = s
	goparkunlock(&l.lock, waitReasonSyncCondWait, traceEvGoBlockCond, 3)
	releaseSudog(s)
}
```

除了将当前 Goroutine 追加到链表的末端之外，我们还会调用 runtime.goparkunlock 将当前 Goroutine 陷入休眠，该函数也是在 Go 语言切换 Goroutine 时经常会使用的方法，它会直接让出当前处理器的使用权并等待调度器的唤醒。

sync.Cond.Signal 和 sync.Cond.Broadcast 就是用来唤醒陷入休眠的 Goroutine 的方法，它们的实现有一些细微的差别：

* sync.Cond.Signal 方法会唤醒队列最前面的 Goroutine；
* sync.Cond.Broadcast 方法会唤醒队列中全部的 Goroutine；

```
func (c *Cond) Signal() {
	c.checker.check()
	runtime_notifyListNotifyOne(&c.notify)
}

func (c *Cond) Broadcast() {
	c.checker.check()
	runtime_notifyListNotifyAll(&c.notify)
}
```
runtime.notifyListNotifyOne 只会从 sync.notifyList 链表中找到满足 sudog.ticket == l.notify 条件的 Goroutine 并通过 runtime.readyWithTime 唤醒：

```
func notifyListNotifyOne(l *notifyList) {
	t := l.notify
	atomic.Store(&l.notify, t+1)

	for p, s := (*sudog)(nil), l.head; s != nil; p, s = s, s.next {
		if s.ticket == t {
			n := s.next
			if p != nil {
				p.next = n
			} else {
				l.head = n
			}
			if n == nil {
				l.tail = p
			}
			s.next = nil
			readyWithTime(s, 4)
			return
		}
	}
}
```

runtime.notifyListNotifyAll 会依次通过 runtime.readyWithTime 唤醒链表中 Goroutine：

```
func notifyListNotifyAll(l *notifyList) {
	s := l.head
	l.head = nil
	l.tail = nil

	atomic.Store(&l.notify, atomic.Load(&l.wait))

	for s != nil {
		next := s.next
		s.next = nil
		readyWithTime(s, 4)
		s = next
	}
}
```

在一般情况下，我们都会先调用 sync.Cond.Wait 陷入休眠等待满足期望条件，当满足唤醒条件时，就可以选择使用 sync.Cond.Signal 或者 sync.Cond.Broadcast 唤醒一个或者全部的 Goroutine。

sync.Cond 不是一个常用的同步机制，但是在条件长时间无法满足时，与使用 for {} 进行忙碌等待相比，sync.Cond 能够让出处理器的使用权，提供 CPU 的利用率。使用时我们也需要注意以下问题：

* sync.Cond.Wait 在调用之前一定要使用获取互斥锁，否则会触发程序崩溃；
* sync.Cond.Signal 唤醒的 Goroutine 都是队列最前面、等待最久的 Goroutine；
* sync.Cond.Broadcast 会按照一定顺序广播通知等待的全部 Goroutine；

## 扩展源语

除了标准库中提供的同步原语之外，Go 语言还在子仓库 sync 中提供了四种扩展原语，

* golang/sync/errgroup.Group
* golang/sync/semaphore.Weighted
* golang/sync/singleflight.Group 
* golang/sync/syncmap.Map

其中的 golang/sync/syncmap.Map 在 1.9 版本中被移植到了标准库中

### ErrGroup

golang/sync/errgroup.Group 为我们在一组 Goroutine 中提供了同步、错误传播以及上下文取消的功能，

例：
```
var g errgroup.Group
var urls = []string{
    "http://www.golang.org/",
    "http://www.google.com/",
    "http://www.somestupidname.com/",
}
for i := range urls {
    url := urls[i]
    g.Go(func() error {
        resp, err := http.Get(url)
        if err == nil {
            resp.Body.Close()
        }
        return err
    })
}
if err := g.Wait(); err == nil {
    fmt.Println("Successfully fetched all URLs.")
}
```

golang/sync/errgroup.Group.Go 方法能够创建一个 Goroutine 并在其中执行传入的函数，而 golang/sync/errgroup.Group.Wait 会等待所有 Goroutine 全部返回，该方法的不同返回结果也有不同的含义：

* 如果返回错误 — 这一组 Goroutine 最少返回一个错误；
* 如果返回空值 — 所有 Goroutine 都成功执行；

```
type Group struct {
	cancel func()

	wg sync.WaitGroup

	errOnce sync.Once
	err     error
}
```
* cancel — 创建 context.Context 时返回的取消函数，用于在多个 Goroutine 之间同步取消信号；
* wg — 用于等待一组 Goroutine 完成子任务的同步原语；
* errOnce — 用于保证只接收一个子任务返回的错误；

我们能通过 golang/sync/errgroup.WithContext 构造器创建新的 golang/sync/errgroup.Group 结构体：
```
func WithContext(ctx context.Context) (*Group, context.Context) {
	ctx, cancel := context.WithCancel(ctx)
	return &Group{cancel: cancel}, ctx
}
```

运行新的并行子任务需要使用 golang/sync/errgroup.Group.Go 方法，这个方法的执行过程如下：

* 调用 sync.WaitGroup.Add 增加待处理的任务；
* 创建新的 Goroutine 并运行子任务；
* 返回错误时及时调用 cancel 并对 err 赋值，只有最早返回的错误才会被上游感知到，后续的错误都会被舍弃：

```
func (g *Group) Go(f func() error) {
	g.wg.Add(1)

	go func() {
		defer g.wg.Done()

		if err := f(); err != nil {
			g.errOnce.Do(func() {
				g.err = err
				if g.cancel != nil {
					g.cancel()
				}
			})
		}
	}()
}

func (g *Group) Wait() error {
	g.wg.Wait()
	if g.cancel != nil {
		g.cancel()
	}
	return g.err
}
```
golang/sync/errgroup.Group.Wait 方法只是调用了 sync.WaitGroup.Wait，在子任务全部完成时取消 context.Context 并返回可能出现的错误。

### Semaphore 

Go 语言的扩展包中提供了带权重的信号量 golang/sync/semaphore.Weighted，我们可以按照不同的权重对资源的访问进行管理，这个结构体对外只暴露了四个方法

* golang/sync/semaphore.NewWeighted 用于创建新的信号量；
* golang/sync/semaphore.Weighted.Acquire 阻塞地获取指定权重的资源，如果当前没有空闲资源，会陷入休眠等待；
* golang/sync/semaphore.Weighted.TryAcquire 非阻塞地获取指定权重的资源，如果当前没有空闲资源，会直接返回 false；
* golang/sync/semaphore.Weighted.Release 用于释放指定权重的资源；

golang/sync/semaphore.NewWeighted 方法能根据传入的最大权重创建一个指向 golang/sync/semaphore.Weighted 结构体的指针
```
func NewWeighted(n int64) *Weighted {
	w := &Weighted{size: n}
	return w
}

type Weighted struct {
	size    int64
	cur     int64
	mu      sync.Mutex
	waiters list.List
}
```

golang/sync/semaphore.Weighted 结构体中包含一个 waiters 列表，其中存储着等待获取资源的 Goroutine，除此之外它还包含当前信号量的上限以及一个计数器 cur，这个计数器的范围就是 [0, size]：

信号量中的计数器会随着用户对资源的访问和释放进行改变，引入的权重概念能够提供更细粒度的资源的访问控制，尽可能满足常见的用例。

golang/sync/semaphore.Weighted.Acquire 方法能用于获取指定权重的资源，其中包含三种不同情况：

* 当信号量中剩余的资源大于获取的资源并且没有等待的 Goroutine 时，会直接获取信号量；
* 当需要获取的信号量大于 golang/sync/semaphore.Weighted 的上限时，由于不可能满足条件会直接返回错误；
* 遇到其他情况时会将当前 Goroutine 加入到等待列表并通过 select 等待调度器唤醒当前 Goroutine，Goroutine 被唤醒后会获取信号量；

```
func (s *Weighted) Acquire(ctx context.Context, n int64) error {
	if s.size-s.cur >= n && s.waiters.Len() == 0 {
		s.cur += n
		return nil
	}

	...
	ready := make(chan struct{})
	w := waiter{n: n, ready: ready}
	elem := s.waiters.PushBack(w)
	select {
	case <-ctx.Done():
		err := ctx.Err()
		select {
		case <-ready:
			err = nil
		default:
			s.waiters.Remove(elem)
		}
		return err
	case <-ready:
		return nil
	}
}
```

另一个用于获取信号量的方法 golang/sync/semaphore.Weighted.TryAcquire 只会非阻塞地判断当前信号量是否有充足的资源，如果有充足的资源会直接立刻返回 true，否则会返回 false：

```
func (s *Weighted) TryAcquire(n int64) bool {
	s.mu.Lock()
	success := s.size-s.cur >= n && s.waiters.Len() == 0
	if success {
		s.cur += n
	}
	s.mu.Unlock()
	return success
}
```
因为 golang/sync/semaphore.Weighted.TryAcquire 不会等待资源的释放，所以可能更适用于一些延时敏感、用户需要立刻感知结果的场景。

当我们要释放信号量时，golang/sync/semaphore.Weighted.Release 方法会从头到尾遍历 waiters 列表中全部的等待者，如果释放资源后的信号量有充足的剩余资源就会通过 Channel 唤起指定的 Goroutine：

```
func (s *Weighted) Release(n int64) {
	s.mu.Lock()
	s.cur -= n
	for {
		next := s.waiters.Front()
		if next == nil {
			break
		}
		w := next.Value.(waiter)
		if s.size-s.cur < w.n {
			break
		}
		s.cur += w.n
		s.waiters.Remove(next)
		close(w.ready)
	}
	s.mu.Unlock()
}
```

当然也可能会出现剩余资源无法唤起 Goroutine 的情况，在这时当前方法在释放锁后会直接返回。

通过对 golang/sync/semaphore.Weighted.Release 的分析我们能发现，如果一个信号量需要的占用的资源非常多，它可能会长时间无法获取锁，这也是 golang/sync/semaphore.Weighted.Acquire 引入上下文参数的原因，即为信号量的获取设置超时时间。

带权重的信号量确实有着更多的应用场景，这也是 Go 语言对外提供的唯一一种信号量实现，在使用的过程中我们需要注意以下的几个问题：

* golang/sync/semaphore.Weighted.Acquire 和 golang/sync/semaphore.Weighted.TryAcquire 都可以用于获取资源，前者会阻塞地获取信号量，后者会非阻塞地获取信号量；
* golang/sync/semaphore.Weighted.Release 方法会按照先进先出的顺序唤醒可以被唤醒的 Goroutine；
* 如果一个 Goroutine 获取了较多地资源，由于 golang/sync/semaphore.Weighted.Release 的释放策略可能会等待比较长的时间；

### SingleFlight 

golang/sync/singleflight.Group 是 Go 语言扩展包中提供了另一种同步原语，它能够在一个服务中抑制对下游的多次重复请求。一个比较常见的使用场景是：我们在使用 Redis 对数据库中的数据进行缓存，发生缓存击穿时，大量的流量都会打到数据库上进而影响服务的尾延时。

但是 golang/sync/singleflight.Group 能有效地解决这个问题，它能够限制对同一个键值对的多次重复请求，减少对下游的瞬时流量。

在资源的获取非常昂贵时（例如：访问缓存、数据库），就很适合使用 golang/sync/singleflight.Group 优化服务。我们来了解一下它的使用方法：

```
type service struct {
    requestGroup singleflight.Group
}

func (s *service) handleRequest(ctx context.Context, request Request) (Response, error) {
    v, err, _ := requestGroup.Do(request.Hash(), func() (interface{}, error) {
        rows, err := // select * from tables
        if err != nil {
            return nil, err
        }
        return rows, nil
    })
    if err != nil {
        return nil, err
    }
    return Response{
        rows: rows,
    }, nil
}
```

因为请求的哈希在业务上一般表示相同的请求，所以上述代码使用它作为请求的键。当然，我们也可以选择其他的字段作为 golang/sync/singleflight.Group.Do 方法的第一个参数减少重复的请求。

golang/sync/singleflight.Group 结构体由一个互斥锁 sync.Mutex 和一个映射表组成，每一个 golang/sync/singleflight.call 结构体都保存了当前调用对应的信息：

```
type Group struct {
	mu sync.Mutex
	m  map[string]*call
}

type call struct {
	wg sync.WaitGroup

	val interface{}
	err error

	dups  int
	chans []chan<- Result
}
```

golang/sync/singleflight.call 结构体中的 val 和 err 字段都只会在执行传入的函数时赋值一次并在 sync.WaitGroup.Wait 返回时被读取；dups 和 chans 两个字段分别存储了抑制的请求数量以及用于同步结果的 Channel。

golang/sync/singleflight.Group 提供了两个用于抑制相同请求的方法：

* golang/sync/singleflight.Group.Do — 同步等待的方法；
* golang/sync/singleflight.Group.DoChan — 返回 Channel 异步等待的方法；

每次调用 golang/sync/singleflight.Group.Do 方法时都会获取互斥锁，随后判断是否已经存在键对应的 golang/sync/singleflight.call：

* 当不存在对应的 golang/sync/singleflight.call 时：
    1. 初始化一个新的 golang/sync/singleflight.call 指针；
    2. 增加 sync.WaitGroup 持有的计数器；
    3. 将 golang/sync/singleflight.call 指针添加到映射表；
    4. 释放持有的互斥锁；
    5. 阻塞地调用 golang/sync/singleflight.Group.doCall 方法等待结果的返回；
* 当存在对应的 golang/sync/singleflight.call 时；
    1. 增加 dups 计数器，它表示当前重复的调用次数；
    2. 释放持有的互斥锁；
    3. 通过 sync.WaitGroup.Wait 等待请求的返回；

```
func (g *Group) Do(key string, fn func() (interface{}, error)) (v interface{}, err error, shared bool) {
	g.mu.Lock()
	if g.m == nil {
		g.m = make(map[string]*call)
	}
	if c, ok := g.m[key]; ok {
		c.dups++
		g.mu.Unlock()
		c.wg.Wait()
		return c.val, c.err, true
	}
	c := new(call)
	c.wg.Add(1)
	g.m[key] = c
	g.mu.Unlock()

	g.doCall(c, key, fn)
	return c.val, c.err, c.dups > 0
}
```

因为 val 和 err 两个字段都只会在 golang/sync/singleflight.Group.doCall 方法中赋值，所以当 golang/sync/singleflight.Group.doCall 和 sync.WaitGroup.Wait 返回时，函数调用的结果和错误都会返回给 golang/sync/singleflight.Group.Do 的调用者。

```
func (g *Group) doCall(c *call, key string, fn func() (interface{}, error)) {
	c.val, c.err = fn()
	c.wg.Done()

	g.mu.Lock()
	delete(g.m, key)
	for _, ch := range c.chans {
		ch <- Result{c.val, c.err, c.dups > 0}
	}
	g.mu.Unlock()
}
```

1. 运行传入的函数 fn，该函数的返回值会赋值给 c.val 和 c.err；
2. 调用 sync.WaitGroup.Done 方法通知所有等待结果的 Goroutine — 当前函数已经执行完成，可以从 call 结构体中取出返回值并返回了；
3. 获取持有的互斥锁并通过管道将信息同步给使用 golang/sync/singleflight.Group.DoChan 方法的 Goroutine；

```
func (g *Group) DoChan(key string, fn func() (interface{}, error)) <-chan Result {
	ch := make(chan Result, 1)
	g.mu.Lock()
	if g.m == nil {
		g.m = make(map[string]*call)
	}
	if c, ok := g.m[key]; ok {
		c.dups++
		c.chans = append(c.chans, ch)
		g.mu.Unlock()
		return ch
	}
	c := &call{chans: []chan<- Result{ch}}
	c.wg.Add(1)
	g.m[key] = c
	g.mu.Unlock()

	go g.doCall(c, key, fn)

	return ch
}
```

golang/sync/singleflight.Group.Do 和 golang/sync/singleflight.Group.DoChan 分别提供了同步和异步的调用方式，这让我们使用起来也更加灵活。

-------
# 原文
https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-sync-primitives/#singleflight