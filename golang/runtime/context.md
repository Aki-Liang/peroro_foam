# Context

Context 是Go语言中用来设置截止日期，同步信号，传递请求相关值的结构体。

该接口定义了四个需要实现的方法

* Deadline - 返回Context被取消的时间，也就是完成工作的截止日期
* Done - 返回一个Channel，这个Channel会在当前工作完成或Context被取消后关闭，多次调用Done方法会返回同一个Channel
* Err - 返回Context结束的原因，只会在Done方法对应的Channel关闭时返回非空的值
    1. Context被取消，返回Canceled错误
    2. Context超时，返回DeadlineExceeded错误
* Value - 从Context中获取键对应的值，对同一个上下文来说，多次调用Value并传入相同的Key会返回相同的结果

```
type Context interface {
	Deadline() (deadline time.Time, ok bool)
	Done() <-chan struct{}
	Err() error
	Value(key interface{}) interface{}
}
```

## 设计原理

在 Goroutine 构成的树形结构中对信号进行同步以减少计算资源的浪费是 context.Context 的最大作用

我们可能会创建多个Goroutine来处理一次请求，Context的作用是在不同的Goroutine之间同步请求特定数据，取消信号以及处理请求的截止日期

每一个Context都会从最顶层的Goroutine一层一层传递到最下层，Context可以在上层Goroutine出现错误时将信号及时同步给下层，减少额外资源的消耗。

## 默认上下文

`context.Background`、`context.TODO`这两个方法都会返回预先初始化好的私有变量background和todo，它们会在同一个Go程序中被复用

```
func Background() Context {
	return background
}

func TODO() Context {
	return todo
}
```

这两个私有变量都通过new（emtyCtx）语句初始化，指向私有结构体context.emptyCtx

```
type emptyCtx int

func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
	return
}

func (*emptyCtx) Done() <-chan struct{} {
	return nil
}

func (*emptyCtx) Err() error {
	return nil
}

func (*emptyCtx) Value(key interface{}) interface{} {
	return nil
}
```

background和todo互为别名，只是在语义上稍有不同
* `context.Background`是上下文的默认值，所有其他的上下文都应该从它衍生出来
* `context.TODO`应该仅在不确定使用哪种上下文时使用

通常情况下如果当前函数没有上下文作为入参，使用`context.Background`作为起始上下文向下传递

## 取消信号

`context.WithCancel`函数能够从Context中衍生出一个新的子上下文并返回用于取消该上下文的函数。一旦执行返回的取消函数，当前上下文及其子上下文都会被取消，所有的Goroutine都会同步收到这一取消信号

```
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	c := newCancelCtx(parent)
	propagateCancel(parent, &c)
	return &c, func() { c.cancel(true, Canceled) }
}
```

* `context.newCancelCtx`将传入的上下文包装成私有结构体`context.cancelCtx`
* `context.propagateCancel`会构建父子上下文之间的关联，当父上下文被取消时，子上下文也会被取消

```
func propagateCancel(parent Context, child canceler) {
	done := parent.Done()
	if done == nil {
		return // 父上下文不会触发取消信号
	}
	select {
	case <-done:
		child.cancel(false, parent.Err()) // 父上下文已经被取消
		return
	default:
	}

	if p, ok := parentCancelCtx(parent); ok {
		p.mu.Lock()
		if p.err != nil {
			child.cancel(false, p.err)
		} else {
			p.children[child] = struct{}{}
		}
		p.mu.Unlock()
	} else {
		go func() {
			select {
			case <-parent.Done():
				child.cancel(false, parent.Err())
			case <-child.Done():
			}
		}()
	}
}
```
* 当parent.Done() == nil,parent不会触发取消事件时，直接返回
* 当child的继承链包含可以取消的上下文时，判断parent是否已经触发了取消信号
    * 如果已经取消，child立刻被取消
    * 如果没有取消，child会被加入parent的children列表中，当代parent释放取消信号
* 当父上下文是开发者自定的类型，实现了Context接口并在Done方法中返回了飞空管道时
    * 运行一个新的Goroutine同时监听parent.Done()和child.Done()两个Channel
    * 在parent.Done()关闭时调用child.cancel()取消上下文

`context.propagateCancel`的作用是在parent和child之间同步取消和结束的信号，保证在parent被取消时child也会受到对应的信号，避免出现状态不一致的情况

`context.cancelCtx`结构体中最重要的方法是`context.cancelCtx.cancel`该方法会关闭上下文中的Channel并向所有的子上下文同步取消信号

```
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
	c.mu.Lock()
	if c.err != nil {
		c.mu.Unlock()
		return
	}
	c.err = err
	if c.done == nil {
		c.done = closedchan
	} else {
		close(c.done)
	}
	for child := range c.children {
		child.cancel(false, err)
	}
	c.children = nil
	c.mu.Unlock()

	if removeFromParent {
		removeChild(c.Context, c)
	}
}

```

`context`包中的另外两个函数`context.WithDeadline`和`context.WithTimeout`也都能创建可以被取消的计时器上下文`context.timerCtx`

```
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
	return WithDeadline(parent, time.Now().Add(timeout))
}

func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
	if cur, ok := parent.Deadline(); ok && cur.Before(d) {
		return WithCancel(parent)
	}
	c := &timerCtx{
		cancelCtx: newCancelCtx(parent),
		deadline:  d,
	}
	propagateCancel(parent, c)
	dur := time.Until(d)
	if dur <= 0 {
		c.cancel(true, DeadlineExceeded) // 已经过了截止日期
		return c, func() { c.cancel(false, Canceled) }
	}
	c.mu.Lock()
	defer c.mu.Unlock()
	if c.err == nil {
		c.timer = time.AfterFunc(dur, func() {
			c.cancel(true, DeadlineExceeded)
		})
	}
	return c, func() { c.cancel(true, Canceled) }
}
```
`context.WithDeadline`在创建`context.timerCtx`的过程中判断了上下文的截止日期与当前日期，通过`time.AfterFunc`创建定时器，当时间超过了截止日期后调用`context.timerCtx.cancel`同步取消信号

```
type timerCtx struct {
	cancelCtx
	timer *time.Timer // Under cancelCtx.mu.

	deadline time.Time
}

func (c *timerCtx) Deadline() (deadline time.Time, ok bool) {
	return c.deadline, true
}

func (c *timerCtx) cancel(removeFromParent bool, err error) {
	c.cancelCtx.cancel(false, err)
	if removeFromParent {
		removeChild(c.cancelCtx.Context, c)
	}
	c.mu.Lock()
	if c.timer != nil {
		c.timer.Stop()
		c.timer = nil
	}
	c.mu.Unlock()
}
```
`context.timerCtx`嵌入`context.cancelCtx`结构体集成相关变量和方法，通过持有`time.Timer`和`deadline`来实现定时取消功能

`context.timerCtx.cancel`方法除了调用`context.cancelCtx.cancel`之外还需要停止定时器来减少资源浪费

## 传值方法

`context.WithValue`能从父上下文中创建一个子上下文，传值的子上下文使用`context.valueCtx`类型

```
func WithValue(parent Context, key, val interface{}) Context {
	if key == nil {
		panic("nil key")
	}
	if !reflectlite.TypeOf(key).Comparable() {
		panic("key is not comparable")
	}
	return &valueCtx{parent, key, val}
}

type valueCtx struct {
	Context
	key, val interface{}
}

func (c *valueCtx) Value(key interface{}) interface{} {
	if c.key == key {
		return c.val
	}
	return c.Context.Value(key)
}
```

`context.valueCtx`结构体只会响应`context.valueCtx.Value`方法

如果`context.valueCtx`中存储的键值对与`context.valueCtx.Value`方法中传入的参数不匹配，就会从父上下文中查找该键对应的值，知道某个父上下文中返回nil或者查找对应的值。

----
# 原文

https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-context/