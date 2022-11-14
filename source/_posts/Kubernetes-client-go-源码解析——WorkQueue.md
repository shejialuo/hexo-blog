---
title: Kubernetes client-go 源码解析——WorkQueue
tags:
  - 技术
categories:
  - Kubernetes源码解析
date: 2022-11-14 19:30:30
---

WorkQueue称为工作队列，支持如下的特性：

+ 有序：按照添加顺序处理元素
+ 去重：相同元素在同一时间不会被重复处理，例如一个元素在处理之前被添加了多次，它只会被处理一次。
+ 并发性：多生产者与多消费者。
+ 标记机制：支持标记功能，标记一个元素是否被处理，也允许元素在处理时重新排队。
+ 通知机制：`ShutDown`方法通过信号量通知队列不再接收新的元素，并通过`metric goroutine`退出。
+ 延迟：支持延迟队列，延迟一段时间后再将元素存入队列。
+ 限速：支持限速队列，元素存入队列时进行速率限制。限制一个元素被重新排队的次数。
+ Metric：支持metric监控指标，可用于Prometheus监控。

WorkQueue支持3种队列，并提供了3种接口，不同队列实现可应对不同的使用场景：

+ `Interface`：FIFO队列接口，支持去重机制。
+ `DelayingInterface`：延迟队列接口，基于`Interface`封装，延迟一段时间后再将元素存入队列。
+ `RateLimitingInterface`：限速队列接口，基于`DelayingInterface`封装，支持元素存入队列时进行速率限制。

## FIFO队列

此部分代码实现位于`util/workqueue/queue.go`中。首先，client-go定义了一个`Type`类型作为FIFO队列的数据结构实现。

```go
type Type struct {
  queue []t
  dirty set
  processing set
  cond *sync.Cond
  shuttingDown bool
  drain bool
  metrics queueMetrics
  unfinishedWorkUpdatePeriod time.Duration
  clock clock.WithTicker
}
```

一些重要字段的含义如下：

+ `queue`：包含按照顺序需要处理的事件。类型`t`只是一个别名`type t interface{}`。其本质的目的在于用于保证元素的有序。
+ `dirty`：包含所有需要处理的事件，按照这个定义所有在`queue`的事件必然在`dirty`中。类型`set`也是一个别名 `type set map[t]empty`，`empty`也是一个别名，`type empty struct{}`。既保证了去重，还能保证在处理一个元素之前哪怕其被添加了多次，但也只会被处理一次。
+ `processing`：包含正在处理的事件。`dirty`也可能包含某些`processing`中的事件。
+ `cond`：条件变量。
+ `shuttingDown`：是否关闭队列。
+ `drain`：关闭队列时是否立即消耗完仍然存在的元素。
+ `metrics`：指标接口，定义在`util/workqueue/metrics.go`。
+ `unfinishedWorkUpdatePeriod`：每次时钟更新的时间，用于初始化时钟。
+ `clock`：时钟。

然后，client-go定义了用于FIFO队列的接口`Interface`。

```go
type Interface interface {
  Add(item interface{})
  Len() int
  Get() (item interface{}, shutdown bool)
  Done(item interface{})
  ShutDown()
  ShutDownWithDrain()
  ShuttingDown() bool
}
```

+ `Add`：给队列添加元素。
+ `Len`：返回当前队列的长度。
+ `Get`：获取队列头部的一个元素。
+ `Done`：标记队列中该元素已被处理。
+ `ShutDown`：关闭队列。
+ `ShutDownWithDrain`：关闭队列并立即处理完在`processing`中的元素。
+ `ShuttingDown`：查询队列是否正在关闭。

为了更好地理解FIFO队列，我们首先对两个场景进行描述。一个场景是无并发环境，一个是存在并发的环境。

场景一：通过`Add`方法往FIFO队列中分别插入1、2、3三个元素，此时队列中的`queue`和`dirty`字段分别存有1、2、3元素，`processing`字段为空。元素1被放入`processing`字段，表示该元素正在被处理。最后，当处理完1元素时，通过`Done`方法标记该元素已经处理完成，此时队列中的`processing`字段中的1元素会被删除。

![FIFO无并发场景下存储过程](https://s2.loli.net/2022/07/19/s5NhxylkIJX2CUm.png)

场景二：在并发场景下，假设goroutine A通过`Get`方法获取1元素，1元素此时被添加到`processing`字段中，同一时间，goroutine B通过`Add`方法插入另一个元素，由于`processing`已经有元素1，故不会存入`queue`中而是存入`dirty`中。当元素1处理完成后，则将1元素追加到`queue`字段中的尾部。

![FIFO并发场景下存储过程](https://s2.loli.net/2022/07/19/ypdvTi47Um1Vwfn.png)

### 队列初始化及其生命周期

`queue.go`首先定义了一个辅助函数`updateUnfinishedWorkLoop`用于对整个队列的生命周期进行管理。

```go
func (q *Type) updateUnfinishedWorkLoop() {
  t := q.clock.NewTicker(q.unfinishedWorkUpdatePeriod)
  defer t.Stop()
  for range t.C() {
    if !func() bool {
      q.cond.L.Lock()
      defer q.cond.L.Unlock()
      if !q.shuttingDown {
        q.metrics.updateUnfinishedWork()
        return true
      }
      return false
    }() {
      return
    }
  }
}
```

该方法利用`for range t.C()`，每隔`q.unfinishedWorkUpdatePeriod`秒，都会判断队列是否被关闭，如果队列没有被关闭，则返回`true`，继续执行循环。如果队列被关闭，则直接返回`false`，然后退出循环，结束函数执行。

然后是对其进行初始化。首先定义了一个基本的`New`函数。然后层层抽象。

```go
func New() *Type {
  return NewNamed("")
}

func NewNamed(name string) *Type {
  rc := clock.RealClock{}
  return newQueue(
    rc,
    globalMetricsFactory.newQueueMetrics(name, rc),
    defaultUnfinishedWorkUpdatePeriod,
  )
}
```

其中，`defaultUnfinishedWorkUpdatePeriod`只是一个常量。

```go
const defaultUnfinishedWorkUpdatePeriod = 500 * time.Millisecond
```

然后就是最关键的就是`newQueue`函数。

```go
func newQueue(c clock.WithTicker, metrics queueMetrics,updatePeriod time.Duration) *Type {
  t := &Type{
    clock: c,
    dirty: set{},
    processing: set{},
    cond: sync.NewCond(&sync.Mutex{}),
    metrics: metrics,
    unfinishedWorkUpdatePeriod: updatePeriod,
  }

  if _, ok := metrics.(noMetrics); !ok {
    go t.updateUnfinishedWorkLoop()
  }
}
```

可以看见，当定义了一个FIFO队列后，`updateUnfinishedWorkLoop`将会一直运行然后每隔一定的时间检测队列的`shuttingDown`的值。可见，利用该值作为信号来传递。

### set方法定义

为了更加方便地对`set`，即`map[t]empty`类型进行操作。`queue.go`定义了一系列方法简化，把基本的`map`方法封装了一下。

```go
func (s set) has(item t) bool{
  _, exists := s[item]
  return exists
}

func (s set) insert(item t) {
  s[item] = empty{}
}

func (s set) delete(item t) {
  delete(s, item)
}

func (s set) len() int {
  return len(s)
}
```

### Type方法定义

首先是实现`Interface`的`Add`方法。

```go
func (q *Type) Add(item interface{}) {
  q.cond.L.Lock()
  defer q.cond.L.Unlock()
  if q.shuttingDown {
    return
  }

  q.metrics.add(item)

  q.dirty.insert(item)
  if q.processing.has(item) {
    return
  }

  q.queue = append(q.queue, item)
  q.cond.Signal()
}
```

为了保持互斥首先需要上锁，然后判断队列是否已经被关闭。如果没有被关闭，首先添加到`dirty`中，然后判断`item`是否位于`processing`中，如果没有添加到`queue`中。然后释放信号用于同步。这是一个典型的生产者与消费者的问题。生产者是`Add`方法，消费者是`Get`方法。

同样实现`Interface`的`Len`方法。很简单，加个锁即可。

```go
func (q *Type) Len() int {
  q.cond.L.Lock()
  defer q.cond.L.Unlock()
  return len(q.queue)
}
```

然后实现`Interface`的`Get`方法。

```go
func (q *Type) Get() (item interface{}, shutdown bool) {
  q.cond.L.Lock()
  defer q.cond.L.Unlock()
  for len(q.queue) == 0 && !q.shuttingDown {
    q.cond.Wait()
  }
  if len(q.queue) == 0 {
    return nil, true
  }
  item = q.queue[0]
  q.queue[0] = nil
  q.queue = q.queue[1:]

  q.metrics.get(item)

  q.processing.insert(item)
  q.dirty.delete(item)

  return item, false
}
```

显然，`Get`方法作为一个消费者当队列为空时，应该等待故调用了`q.cond.Wait`。然后取出第一个元素，置`queue[0]`为空以便进行垃圾回收，插入到`processing`中，然后从`dirty`中删除。

`Done`方法用于告知队列该元素已被处理，如果该元素存在`dirty`中，我们需要将其重新加入到`queue`中来处理。

```go
func (q *Type) Done(item interface{}) {
  q.cond.L.Lock()
  defer q.cond.L.Unlock()

  q.metrics.done(item)

  q.processing.delete(item)
  if q.dirty.has(item) {
    q.queue = append(q.queue, item)
    q.cond.Signal()
  } else if q.processing.len() == 0 {
    q.cond.Signal()
  }
}
```

`ShutDown`方法让队列忽略所有需要新增的item然后立即告知worker goroutine退出。

```go
func (q* Type) ShutDown() {
  q.setDrain(false)
  q.shutdown()
}
```

定义了两个辅助函数`setDrain`和`shutdown`进行抽象。

```go
func (q* Type) setDrain(shouldDrain bool) {
  q.cond.L.Lock()
  defer q.cond.L.Unlock()
  q.drain = shouldDrain
}

func (q* Type) shutdown() {
  q.cond.L.Lock()
  defer q.cond.L.Unlock()
  q.shuttingDown = true
  q.cond.Broadcast()
}
```

注意`q.cond.Broadcast`的使用，尽管关闭了队列，但我们仍然可以使用`Get`方法去获取元素。可能此时`q.queue`有元素或者没有，但我们必须使用避免阻塞了其他函数。

然后`queue.go`定义了`ShutDownWithDrain`关闭队列并立即处理`q.queue`中的元素。

```go
func (q *Type) ShutDownWithDrain() {
  q.setDrain(true)
  q.shutdown()
  for q.isProcessing() && q.shouldDrain() {
    q.waitForProcessing()
  }
}

func (q *Type) isProcessing() bool {
  q.cond.L.Lock()
  defer q.cond.L.Unlock()
  return q.processing.len() != 0
}

func (q *Type) shouldDrain() bool {
  q.cond.L.Lock()
  defer q.cond.L.Unlock()
  return q.drain
}

func (q *Type) waitForProcessing() {
  q.cond.L.Lock()
  defer q.cond.L.Unlock()

  if q.processing.len() == 0 {
    return
  }
  q.cond.Wait()
}
```

### 同步机制总结

在FIFO队列中，存在许多同步的东西，总结如下：

+ 基本的生产者-消费者模型。`Add`方法需要调用`q.cond.Signal`通知等待的`Get`方法。`Done`方法可能也会添加新的item。故如果新增了新的item，也需要调用`q.cond.Signal`通知等待的`Get`方法。
+ `Done`方法在`q.processing.len() == 0`时也会调用`q.cond.Signal`实现同步，这是因为`ShutDownWithDrain`方法会调用`waitForProcessing`然后再调用`q.cond.Wait`阻塞自身。显然对于`drain`来说，开发者必须对每个方法item调用`Done`方法，以便在`q.processing`的长度为0时，唤醒`ShutDownWithDrain`方法。

### 指标

指标定义在`metrics.go`中，可以看见我们在处理item的时候，也会对其的指标进行操作，由于该部分不是核心代码，此处不进行分析。

## 延迟队列

此部分代码定义在`util/workqueue/delaying_queue.go`中。延迟队列基于FIFO队列接口封装，在原有的功能上增加了`AddAfter`方法。

```go
type DelayingInterface interface {
  Interface
  AddAfter(item interface{}, duration time.Duration)
}
```

我们首先看延迟队列的`AddAfter`方法是如何实现的：

```go
func (q *delayingType) AddAfter(item interface{}, duration time.Duration) {
  if q.ShuttingDown() {
    return
  }

  q.metrics.retry()

  if duration <= 0 {
    q.Add(item)
    return
  }

  select {
    case <-q.stopCh:
    case q.waitingForAddCh <- &waitFor{data: item, readyAt: q.clock.Now().Add(duration)}
  }
}
```

很明显，这个函数就做了一个极其简单的事情，当`duration`小于0时，直接加入队列中。

我们首先需要看`delayingType`这个类型：

```go
type delayingType struct {
  Interface
  clock           clock.Clock
  stopCh          chan struct{}
  stopOnce        sync.Once
  heartbeat       clock.Ticker
  waitingForAddCh chan* waitFor
  metrics         retryMetrics
}
```

一些重要的字段含义如下：

+ `stopCh`：利用channel作信号量。
+ `stopOnce`：只能Stop一次。
+ `waitingForAddCh`：用来存储待加入的`item`。

其中`waitFor`的类型定义如下：

```go
type waitFor struct {
  data    t
  readyAt time.Time
  index   int
}
```

代码维护了一个优先队列`waitForPriorityQueue`，其本质的思路在于实现`heap`的接口，内容比较简单，此处忽略细节。

`newDelayingQueue`函数创建了一个新的延迟队列，然后启用`waitingLoop` goroutine。

```go
func newDelayingQueue(clock clock.WithTicker, q Interface, name string) *delayingType {
  ret := &delayingType{
    Interface:       q,
    clock:           clock,
    heartbeat:       clock.NewTicker(maxWait),
    stopCh:          make(chan struct{}),
    waitingForAddCh: make(chan *waitFor, 1000),
    metrics:         newRetryMetrics(name),
  }

  go ret.waitingLoop()
  return ret
}
```

`waitingLoop`是延迟队列实现的核心所在。其本质的思路仍然是实现同步，此处忽略细节。

## 限速队列

限速队列在延迟队列接口的基础上增加了`AddRateLimited`, `Forget`, `NumRequeues`方法：

```go
type RateLimitingInterface interface {
  DelayingInterface

  AddRateLimited(item interface{})

  Forget(item interface{})

  NumRequeues(item interface{}) int
}
```

然而这三个方法都是调用`RateLimiter`的接口方法：

```go

type RateLimiter interface {
  When(item interface{}) time.Duration

  Forget(item interface{})

  NumRequeues(item interface{}) int
}
```

+ `When`：获取指定元素应该等待的时间。
+ `Forget`：释放指定元素。
+ `NumRequeues`：返回元素的失败数。

WorkQueue提供了4种限速算法：

+ 令牌桶算法：下节单独介绍
+ 排队指数算法：将相同元素的排队数作为指数，排队数增大，速率限制呈指数级增长。
+ 计数器算法：限制一段时间内允许通过的元素数量。
+ 混合模式

## 令牌桶

client-go使用[令牌桶](https://en.wikipedia.org/wiki/Token_bucket)作为限流算法。

### Limiter数据结构定义

Go语言标准库提供了令牌桶算法的实现。首先在`rate.go`中定义了`Limiter`。

```go
type Limiter struct {
  mu     sync.Mutex
  limit  Limit
  burst  int
  tokens float64
  last time.Time
  lastEvent time.Time
}
```

其中`Limit`仅仅只是一个类型Wrapper：`type Limit float64`。字段的含义如下：

+ `mu`：互斥锁
+ `limit`：每秒下发令牌的个数
+ `burst`：桶的最大令牌数量
+ `tokens`：当前令牌数量
+ `last`：最后一次`tokens`字段更新时间
+ `lastEvent`：最近一次限流事件发生的时间

目前`limit`的定义为每秒下发令牌的个数，故`rate.go`定义了`Every`函数将事件之间的最小时间间隔转换为`Limit`。当`limit = Inf`时，`burst`可以被忽略，允许任何事件通过，因为下发令牌的个数是无限的。同时，`limit`也可以为0，代表不允许任何事件通过。

```go
func Every(interval time.Duration) Limit {
  if interval <= 0 {
    return Inf
  }
  return 1 / Limit(interval.Seconds())
}
```

`rate.go`定义了一些基本的getter和setter方法。

```go
func (lim *Limiter) Limit() Limit {
  lim.mu.Lock()
  defer lim.mu.Unlock()
  return lim.Limit
}

func (lim *Limiter) Burst() int {
  lim.mu.Lock()
  defer lim.mu.Unlock()
  return lim.burst
}

func NewLimiter(r Limit, b int) *Limiter {
  return &Limiter{
    limit: r,
    burst: b
  }
}
```

### 辅助函数

为了更加好的抽象，`rate.go`定义了一系列的辅助函数。根据令牌桶的概念我们可以知道，随着时间的变化，令牌桶中的令牌数量会增加。故为了实现时间间隔和令牌桶的令牌数量相互的转化，`rate.go`定义了`tokensFromDuration`和`durationFromTokens`。

```go
// 得到一个时间段会产生多少个令牌
func (limit Limit) tokensFromDuration(d time.Duration) float64 {
  if limit <= 0 {
    return 0
  }
  return d.Seconds() * float64(limit)
}

// 目前令牌桶中的令牌代表了多少时间段
func (limit Limit) durationFromTokens(tokens float64) {
  if limit <= 0 {
    return InfDuration
  }
  seconds := tokens / float64(limit)
  return time.Duration(float64(time.Second) * seconds)
}
```

随着时间的变化，需要对令牌桶中的令牌也就是`token`进行更新，故定义了`advance`函数。

```go
func (lim *Limiter) advance(now time.Time) (newNow time.Time, newLast time.Time, newTokens float64) {
  last := lim.last
  if now.Before(last) {
    last = now
  }

  elapsed := now.Sub(last)
  delta := lim.limit.tokensFromDuration(elapsed)
  tokens := lim.tokens + delta
  if burst := float64(lim.burst); tokens > burst {
    tokens = burst
  }
  return now, last, tokens
}
```

### 方法

Limiter还包含一些setter方法，介绍了辅助函数后，对于这些setter方法就比较容易理解。

```go
func (lim *Limiter) SetLimit(newLimit Limit) {
  lim.SetLimitAt(time.Now(), newLimit)
}

func (lim *Limiter) SetLimitAt(now time.Time, newLimit Limit) {
  lim.mu.Lock()
  defer lim.mu.Unlock()

  now, _, tokens := lim.advance(now)

  lim.last = now
  lim.tokens = tokens
  lim.limit = newLimit
}

func (lim *Limiter) SetBurst(newBurst int) {
  lim.SetBurstAt(time.Now(), newBurst)
}

// SetBurstAt sets a new burst size for the limiter.
func (lim *Limiter) SetBurstAt(now time.Time, newBurst int) {
  lim.mu.Lock()
  defer lim.mu.Unlock()

  now, _, tokens := lim.advance(now)

  lim.last = now
  lim.tokens = tokens
  lim.burst = newBurst
}
```

Limiter主要有三个方法：`Allow`, `Reserve`和`Wait`。这三个方法在被调用时，都会消耗掉一个令牌。这三个方法分别被`AllowN`，`ReserveN`以及`WaitN`抽象。

```go
func (lim *Limiter) Allow() bool {
  return lim.AllowN(time.Now(), 1)
}

func (lim *Limiter) Reserve() *Reservation {
  return lim.ReserveN(time.Now(), 1)
}

func (lim *Limiter) Wait(ctx context.Context) (err error) {
  return lim.WaitN(ctx, 1)
}
```

首先，`rate.go`定义了`Reservation`数据结构，包含了已经被限流器所允许的事件的信息。

```go
type Reservation struct {
  ok        bool
  lim       *Limiter
  tokens    int
  timeToAct time.Time
  limit Limit
}
```

字段的含义如下：

+ `ok`：表示事件能否发生
+ `lim`: 属于哪个Limiter
+ `tokens`：表示该事件需要消耗的令牌数量
+ `timeToAct`：执行的时间
+ `limit`：在`Reserve`操作的时候定义

我们首先看函数`AllowN`。

```go
func (lim *Limiter) AllowN(now time.Time, n int) bool {
  return lim.reserveN(now, n, 0).ok
}
```

再看函数`ReserveN`

```go
func (lim *Limiter) ReserveN(now time.Time, n int) *Reservation {
  r := lim.reserveN(now, n, InfDuration)
  return &r
}
```

可以看出`AllowN`和`ReserveN`都是通过`reserveN`进行抽象的。首先`reserveN`处理特殊情况，即`limit = Inf`（允许任何事件通过）和`limit = 0`（不允许任何事件通过），虽然不允许任何事件通过，但是本身令牌桶初始化時有`burst`个令牌数，故还是可以允许通过`burst`个令牌。

再处理完特殊情况后，首先通过`advance`计算出现在时刻的令牌桶中的令牌数量的个数，减去该事件所消耗的令牌个数。当令牌数小于0证明该事件需要等待，故通过`durationFromTokens`计算需要等待的时间。

其次，判断事件能否发生。事件能发生需要满足两个条件，一是事件发生消耗的令牌数量不能超过令牌桶最大的令牌数量，二是等待时间不能超过参数`maxFutureReserve`的值。

后面的操作就是更新字段。

```go
func (lim *Limiter) reserveN(now time.Time, n int, maxFutureReserve time.Duration) Reservation {
  lim.mu.Lock()
  defer lim.mu.Unlock()

  if lim.limit = Inf {
    return Reservation{
      ok:        true,
      lim:       lim,
      tokens:    n,
      timeToAct: now,
    }
  } else if lim.limit == 0 {
    var ok bool
    if lim.burst >= n {
      ok = true
      lim.burst -= n
    }
    return Reservation{
      ok:        true,
      lim:       lim,
      tokens:    lim.burst,
      timeToAct: now,
    }
  }

  now, last, tokens := lim.advance(now)

  tokens -= float64(n)

  var waitDuration time.Duration
  if tokens < 0 {
    waitDuration = lim.limit.durationFromTokens(-tokens)
  }

  ok := n <= lim.burst && waitDuration <= maxFutureReserve

  r := Reservation{
    ok:    ok,
    lim:   lim,
    limit: lim.limit,
  }
  if ok {
    r.tokens = n
    r.timeToAct = now.Add(waitDuration)
  }

  if ok {
    lim.last = now
    lim.tokens = tokens
    lim.lastEvent = r.timeToAct
  } else {
    lim.last = last
  }

  return r
}
```

可以看出`reserveN`作为一个核心的函数，无非就是查询令牌桶中的令牌数量足不足以支持一个消耗`n`个令牌的任务，为了维持这个任务的状态必须定义一个数据结构来维持。

在讲`WaitN`函数之前，我们先看看`DelayFrom`函数，这个函数很简单，对于已经ok的任务，得到其延迟发生的时间。

```go
func (r *Reservation) DelayFrom(now time.Time) time.Duration {
  if !r.ok {
    return InfDuration
  }
  delay := r.timeToAct.Sub(now)
  if delay < 0 {
    return 0
  }
  return delay
}
```

现在我们可以去看`WaitN`函数，很显然对于一个任务来说，其可以通过调用`WaitN`来实现限流。

+ 当所需的令牌数目大于令牌桶所能包含的最大令牌，直接返回error。
+ 如果在调用时，任务已经结束了，直接返回error。
+ 计算`waitLimit`其值为任务结束的时间和现在的时间的差值，然后使用`reserveN`得到任务的状态。
+ 然后使用`DelayFrom`计算需要延迟的时间，如果有必要延迟的话，通过一个定时器来延时。如果定时器完成了，就继续。如果定时器结束之前，`Context`被取消了，返回错误。

```go
func (lim *Limiter) WaitN(ctx context.Context, n int) (err error) {
  lim.mu.Lock()
  burst := lim.burst
  limit := lim.limit
  lim.mu.Unlock()

  if n > burst && limit != Inf {
    return fmt.Errorf("rate: Wait(n=%d) exceeds limiter's burst %d", n, burst)
  }
  select {
  case <-ctx.Done():
    return ctx.Err()
  default:
  }
  now := time.Now()
  waitLimit := InfDuration
  if deadline, ok := ctx.Deadline(); ok {
    waitLimit = deadline.Sub(now)
  }
  r := lim.reserveN(now, n, waitLimit)
  if !r.ok {
    return fmt.Errorf("rate: Wait(n=%d) would exceed context deadline", n)
  }
  delay := r.DelayFrom(now)
  if delay == 0 {
    return nil
  }
  t := time.NewTimer(delay)
  defer t.Stop()
  select {
  case <-t.C:
    return nil
  case <-ctx.Done():
    r.Cancel()
    return ctx.Err()
  }
}
```

注意`r.Cancel()`的使用，既然我们已经给了令牌给一个任务而这个任务并没有实际的执行，我们应该还给令牌桶相应的数目。由于此时已经介绍了大部分的函数，此处忽略其细节。

### 小结

令牌桶的实现与时间有很大的关系，看似需要每隔1s就需要更新令牌桶中的令牌数目，实则上是完全没有必要的。因为可以从未来借。当每一次调用主要方法时，都会通过现在的时间减去上一次令牌桶数目更新的时间来更新令牌桶中的令牌数目，令牌桶中的令牌数目是负的也根本无所谓，很棒的设计。

### Client-go封装

Client-go在`util/workqueue/default_rate_limiters.go`中定义了`BucketRateLimiter`用于封装标准库中的`Limiter`。

```go
type BucketRateLimiter struct {
  *rate.Limiter
}

func (r *BucketRateLimiter) When(item interface{}) time.Duration {
  return r.Limiter.Reserve().Delay()
}

func (r *BucketRateLimiter) NumRequeues(item interface{}) int {
  return 0
}

func (r *BucketRateLimiter) Forget(item interface{}) {}
```
