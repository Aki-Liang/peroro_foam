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