# WorkQueue 源码分析

client-go的`util/workqueue`包里主要有三个队列，分别是普通队列（Queue）、延时队列（DelayingQueue）、限速队列（RateLimitingQueue），后一个队列以前一个队列的实现为基础，层层添加功能。

## Queue

`Queue`提供支持以下特性：
1. 公平：按元素的添加顺序进行处理。（队列的先入先出特性）
2. 吝啬：单个item不会被并发处理多次，如果一个item被多次添加才可以被处理，那么它只会被处理一次。
3. 多个消费者和生产者。特别是，允许在处理项目时将其重新排队。
4. 关闭通知。

### 结构体和接口

接口：
```go
type Interface interface {
	Add(item interface{}) //添加元素
	Len() int //元素个数
	Get() (item interface{}, shutdown bool) //获取一个元素，第二个返回值与channel类似，标记队列是否关闭了。
	Done(item interface{}) //标记一个元素已经处理完毕
	ShutDown() //关闭队列
	ShutDownWithDrain() //队列将忽略添加到其中的所有新项目，并等待Worker对队列中所有现有元素调用Done（否则该方法将一直阻塞）
	ShuttingDown() bool //是否正在关闭
}
```

该接口的实现：
```go
type Type struct {
	queue []t // 定义元素的处理顺序，里面所有元素都应该在 dirty set 中有，而不能出现在 processing set 中

	dirty set // 标记所有需要被处理的元素

	processing set // 当前正在被处理的元素，当处理完后需要检查该元素是否在 dirty set 中，如果有则添加到 queue 里

	cond *sync.Cond //条件锁

	shuttingDown bool // 是否正在关闭
	drain        bool

	metrics queueMetrics

	unfinishedWorkUpdatePeriod time.Duration
	clock                      clock.WithTicker
}
```

这个 Queue 的工作逻辑大致是这样，里面的三个属性 queue、dirty、processing 都保存 items，但是含义有所不同：

- `queue`： 这是一个 []t 类型，也就是一个切片，因为其有序，所以这里当作一个列表来存储 item 的处理顺序。
- `dirty`： 这是一个 set 类型，也就是一个集合，这个集合存储的是所有需要处理的 item，这些 item 也会保存在 queue 中，但是 set 里是无序的，set 的特性是里面元素具有唯一性。
- `processing`：这也是一个 set，存放的是当前正在处理的 item，也就是说这个 item 来自 queue 出队的元素，同时这个元素会被从 dirty 中删除。

## set 类型和 Queue 接口的集合核心方法的实现

### set
上面提到的 dirty 和 processing 字段都是 set 类型，set 相关定义如下：

```go
type empty struct{}
type t interface{}
type set map[t]empty

func (s set) has(item t) bool {
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

可以看到 set 是一个空接口到空结构体的 map，也就是实现了一个集合的功能，集合元素是 interface{} 类型，也就是可以存储任意类型。而 map 的 value 是 struct{} 类型，也就是空。这里利用 map 的 key 唯一的特性实现了一个集合类型，附带四个方法 has()、insert()、delete()、len() 来实现集合相关操作。

### Queue的实现方法（Queue的具体实现是名为`Type`的结构体）

**Add**方法：
```go
func (q *Type) Add(item interface{}) {
	q.cond.L.Lock()
	defer q.cond.L.Unlock()
	if q.shuttingDown {// 如果 queue 正在被关闭，则直接返回，不入队
		return
	}
	if q.dirty.has(item) {// 如果 dirty set 中已经有了该 item，则直接返回
		return
	}

	q.metrics.add(item)

	q.dirty.insert(item)// 添加到 dirty set 中
	if q.processing.has(item) { // 如果正在被处理，则返回
		return
	}

	q.queue = append(q.queue, item) // 如果没有正在处理，则加到 q.queue 中
	q.cond.Signal() // 通知 getter 有新 item 到来
}
```

**Get**方法：
该方法官方的说明是：该方法将会被阻塞知道有可被处理的元素；如果`shutdown`为真，则调用者应该结束他们的`goroutine`；在元素被处理完成后，必需对对应元素调用`Done`方法。

```go
func (q *Type) Get() (item interface{}, shutdown bool) {
	q.cond.L.Lock()
	defer q.cond.L.Unlock()
	for len(q.queue) == 0 && !q.shuttingDown { //如果没有可被处理的元素，且队列没有被关闭，则阻塞等待
		q.cond.Wait()
	}
	if len(q.queue) == 0 {// 这时候如果 q.queue 长度还是 0，说明 q.shuttingDown 为 true，所以直接返回
		// We must be shutting down.
		return nil, true
	}

	item = q.queue[0]
	//底层数组仍然存在并引用该对象，因此该对象不会被垃圾回收。
	q.queue[0] = nil
	q.queue = q.queue[1:] //更新队列

	q.metrics.get(item)

	q.processing.insert(item) // 刚才获取到的 q.queue 第一个元素放到 processing set 中
	q.dirty.delete(item) // dirty set 中删除该元素

	return item, false // 返回 item
```

**Done**方法：
```go
func (q *Type) Done(item interface{}) {
	q.cond.L.Lock()
	defer q.cond.L.Unlock()

	q.metrics.done(item)

	q.processing.delete(item) // processing set 中删除该 item
	if q.dirty.has(item) { // 如果 dirty 中还有，说明还需要再次处理，放到 q.queue 中
		q.queue = append(q.queue, item)
		q.cond.Signal() // 通知 getter 有新的 item
	} else if q.processing.len() == 0 {
		q.cond.Signal()
	}
}
```

## DelayingQueue

### 接口和结构体

接口定义：
```go
type DelayingInterface interface {
	Interface
	// AddAfter adds an item to the workqueue after the indicated duration has passed
	AddAfter(item interface{}, duration time.Duration)
}
```

相比 Queue 这里只是多了一个 AddAfter(item interface{}, duration time.Duration) 方法，望文生义，也就是延时添加 item。

结构体定义：
```go
type delayingType struct {
	Interface // 用来嵌套普通 Queue

	// clock tracks time for delayed firing
	clock clock.Clock // 计时器

	// stopCh lets us signal a shutdown to the waiting loop
	stopCh chan struct{}
	// stopOnce guarantees we only signal shutdown a single time
	stopOnce sync.Once // 用来确保 ShutDown() 方法只执行一次

	// heartbeat ensures we wait no more than maxWait before firing
	heartbeat clock.Ticker // 默认10s的心跳，后面用在一个大循环里，避免没有新元素时一直阻塞

	// waitingForAddCh is a buffered channel that feeds waitingForAdd
	waitingForAddCh chan *waitFor // 传递 waitFor 的 channel，默认大小 1000

	// metrics counts the number of retries
	metrics retryMetrics
}
```

对于延时队列，我们关注的入口方法肯定就是新增的 AddAfter() 了，看这个方法的具体的逻辑前我们先看下上面 delayingType 中涉及到的 waitFor 类型。

### waitFor
结构体定义：
```go
type waitFor struct {
	data    t // 准备添加到队列中的数据
	readyAt time.Time // 应该被加入队列的时间
	// index in the priority queue (heap)
	index int // 在 heap 中的索引
}
```

然后可以注意到有这样一行代码：
```go
type waitForPriorityQueue []*waitFor
```
这里定义了一个 waitFor 的优先级队列，用最小堆的方式来实现，这个类型实现了 heap.Interface 接口，我们具体看下源码：
```go
// Push adds an item to the queue. Push should not be called directly; instead,
// use `heap.Push`.
// 添加一个 item 到队列中
func (pq *waitForPriorityQueue) Push(x interface{}) {
	n := len(*pq)
	item := x.(*waitFor)
	item.index = n
	*pq = append(*pq, item) // 添加到队列的尾部
}

// Pop removes an item from the queue. Pop should not be called directly;
// instead, use `heap.Pop`.
// 从队列尾巴移除一个 item
func (pq *waitForPriorityQueue) Pop() interface{} {
	n := len(*pq)
	item := (*pq)[n-1]
	item.index = -1
	*pq = (*pq)[0:(n - 1)]
	return item
}

// Peek returns the item at the beginning of the queue, without removing the
// item or otherwise mutating the queue. It is safe to call directly.
// 获取队列第一个 item，但不会从队列中删除元素
func (pq waitForPriorityQueue) Peek() interface{} {
	return pq[0]
}
```

### NewDelayingQueue

接着看一下 DelayingQueue 相关的几个 New 函数，理解了这里的逻辑，才能继续往后面分析 AddAfter() 方法。

```go
// NewNamedDelayingQueue constructs a new named workqueue with delayed queuing ability.
// Deprecated: Use NewDelayingQueueWithConfig instead.
func NewNamedDelayingQueue(name string) DelayingInterface {// 这里可以传递一个名字作为队列名
	return NewDelayingQueueWithConfig(DelayingQueueConfig{Name: name})
}

// NewDelayingQueueWithCustomClock constructs a new named workqueue
// with ability to inject real or fake clock for testing purposes.
// Deprecated: Use NewDelayingQueueWithConfig instead.
// 上面一个函数只是调用当前函数，附带一个名字，这里加了一个指定 clock 的能力
func NewDelayingQueueWithCustomClock(clock clock.WithTicker, name string) DelayingInterface {
	return NewDelayingQueueWithConfig(DelayingQueueConfig{
		Name:  name,
		Clock: clock,
	})
}

// NewDelayingQueue constructs a new workqueue with delayed queuing ability.
// NewDelayingQueue does not emit metrics. For use with a MetricsProvider, please use
// NewDelayingQueueWithConfig instead and specify a name.
func NewDelayingQueue() DelayingInterface {
	return NewDelayingQueueWithConfig(DelayingQueueConfig{})
}

// NewDelayingQueueWithConfig constructs a new workqueue with options to
// customize different properties.
func NewDelayingQueueWithConfig(config DelayingQueueConfig) DelayingInterface {
	if config.Clock == nil {
		config.Clock = clock.RealClock{}
	}

	if config.Queue == nil {
		config.Queue = NewWithConfig(QueueConfig{
			Name:            config.Name,
			MetricsProvider: config.MetricsProvider,
			Clock:           config.Clock,
		})
	}

	return newDelayingQueue(config.Clock, config.Queue, config.Name, config.MetricsProvider)
}

func newDelayingQueue(clock clock.WithTicker, q Interface, name string, provider MetricsProvider) *delayingType {
	ret := &delayingType{
		Interface:       q,
		clock:           clock,
		heartbeat:       clock.NewTicker(maxWait), // 10s 一次心跳
		stopCh:          make(chan struct{}),
		waitingForAddCh: make(chan *waitFor, 1000),
		metrics:         newRetryMetrics(name, provider),
	}

	go ret.waitingLoop() // 留意这里的函数调用
	return ret
}

// DelayingQueueConfig specifies optional configurations to customize a DelayingInterface.
type DelayingQueueConfig struct {
	// Name for the queue. If unnamed, the metrics will not be registered.
	Name string

	// MetricsProvider optionally allows specifying a metrics provider to use for the queue
	// instead of the global provider.
	MetricsProvider MetricsProvider

	// Clock optionally allows injecting a real or fake clock for testing purposes.
	Clock clock.WithTicker

	// Queue optionally allows injecting custom queue Interface instead of the default one.
	Queue Interface
}
```

### waitingLoop方法

```go
// waitingLoop runs until the workqueue is shutdown and keeps a check on the list of items to be added.
func (q *delayingType) waitingLoop() {
	defer utilruntime.HandleCrash()

	// Make a placeholder channel to use when there are no items in our list
	never := make(<-chan time.Time) // 队列里没有 item 时实现等待用的

	// Make a timer that expires when the item at the head of the waiting queue is ready
	var nextReadyAtTimer clock.Timer

	waitingForQueue := &waitForPriorityQueue{} // 构造一个优先级队列
	heap.Init(waitingForQueue) // 这一行其实是多余的，功能上没有啥作用，不过在可读性上有点帮助。

	waitingEntryByData := map[t]*waitFor{} // 这个 map 用来处理重复添加逻辑的，下面会讲到

	for {// 无限循环
		if q.Interface.ShuttingDown() {// 这个地方 Interface 从语法上来看可有可无，不过放在这里能够强调是调用了内部 Queue 的 ShuttingDown() 方法
			return
		}

		now := q.clock.Now()

		// Add ready entries
		for waitingForQueue.Len() > 0 {// 队列里有 item 就开始循环
			entry := waitingForQueue.Peek().(*waitFor) // 获取第一个 item -- 这是一个优先队列（小根堆），因此第一个item的等待时间总是最小的。
			if entry.readyAt.After(now) {// 时间还没到，先不处理
				break
			}

			entry = heap.Pop(waitingForQueue).(*waitFor) // 时间到了，pop 出第一个元素；注意 waitingForQueue.Pop() 是最后一个 item，heap.Pop() 是第一个元素
			q.Add(entry.data) // 将数据加到延时队列里
			// map 里删除已经加到延时队列的 item
			delete(waitingEntryByData, entry.data)
		}

		// Set up a wait for the first item's readyAt (if one exists)
		// 如果队列中有 item，就用第一个 item 的等待时间初始化计时器，如果为空则一直等待
		nextReadyAt := never
		if waitingForQueue.Len() > 0 {
			if nextReadyAtTimer != nil {
				nextReadyAtTimer.Stop()
			}
			entry := waitingForQueue.Peek().(*waitFor)
			nextReadyAtTimer = q.clock.NewTimer(entry.readyAt.Sub(now))
			nextReadyAt = nextReadyAtTimer.C()
		}

		select {
		case <-q.stopCh:
			return

		case <-q.heartbeat.C(): // 心跳时间是 10s，到了就继续下一轮循环
			// continue the loop, which will add ready items

		case <-nextReadyAt: // 第一个 item 的等到时间到了，继续下一轮循环
			// continue the loop, which will add ready items

		case waitEntry := <-q.waitingForAddCh: // waitingForAddCh 收到新的 item
			if waitEntry.readyAt.After(q.clock.Now()) {// 如果时间没到，就加到优先级队列里，如果时间到了，就直接加到延时队列里
				insert(waitingForQueue, waitingEntryByData, waitEntry)
			} else {
				q.Add(waitEntry.data)
			}

      // 下面的逻辑就是将 waitingForAddCh 中的数据处理完
			drained := false
			for !drained {
				select {
				case waitEntry := <-q.waitingForAddCh:
					if waitEntry.readyAt.After(q.clock.Now()) {
						insert(waitingForQueue, waitingEntryByData, waitEntry)
					} else {
						q.Add(waitEntry.data)
					}
				default:
					drained = true
				}
			}
		}
	}
}
```
这个方法还有一个 insert() 调用，我们再来看一下这个插入逻辑：
```go
// insert adds the entry to the priority queue, or updates the readyAt if it already exists in the queue
func insert(q *waitForPriorityQueue, knownEntries map[t]*waitFor, entry *waitFor) {
	// if the entry already exists, update the time only if it would cause the item to be queued sooner
	// 这里的主要逻辑是看一个 entry 是否存在，如果已经存在，新的 entry 的 ready 时间更短，就更新时间
	existing, exists := knownEntries[entry.data]
	if exists {
		if existing.readyAt.After(entry.readyAt) {
			existing.readyAt = entry.readyAt // 如果存在就只更新时间
			heap.Fix(q, existing.index) //重新排序--维护小根堆
		}

		return
	}

  // 如果不存在就丢到 q 里，同时在 map 里记录一下，用于查重
	heap.Push(q, entry)
	knownEntries[entry.data] = entry
}
```

### AddAfter方法
这个方法的作用是在指定的延时到达之后，在 work queue 中添加一个元素，源码如下：
```go
// AddAfter adds the given item to the work queue after the given delay
func (q *delayingType) AddAfter(item interface{}, duration time.Duration) {
	// don't add if we're already shutting down
	if q.ShuttingDown() {// 已经在关闭中就直接返回
		return
	}

	q.metrics.retry()

	// immediately add things with no delay
	if duration <= 0 { // 如果时间到了，就直接添加
		q.Add(item)
		return
	}

	select {
	case <-q.stopCh:
		// unblock if ShutDown() is called
	case q.waitingForAddCh <- &waitFor{data: item, readyAt: q.clock.Now().Add(duration)}:      // 构造 waitFor{}，丢到 waitingForAddCh
	}
}
```

## RateLimitingQueue

### 接口和结构体
接口：
```go
// RateLimitingInterface is an interface that rate limits items being added to the queue.
type RateLimitingInterface interface {
	DelayingInterface // 延时队列里内嵌了普通队列，限速队列里则内嵌了延时队列

	// AddRateLimited adds an item to the workqueue after the rate limiter says it's ok
	AddRateLimited(item interface{}) // 以限速方式往队列里加入一个元素

	// Forget indicates that an item is finished being retried.  Doesn't matter whether it's for perm failing
	// or for success, we'll stop the rate limiter from tracking it.  This only clears the `rateLimiter`, you
	// still have to call `Done` on the queue.
	Forget(item interface{}) // 标识一个元素结束重试

	// NumRequeues returns back how many times the item was requeued
	NumRequeues(item interface{}) int // 标识这个元素被处理里多少次了
}
```

然后看下两个 New 函数

```go
// NewRateLimitingQueue constructs a new workqueue with rateLimited queuing ability
// Remember to call Forget!  If you don't, you may end up tracking failures forever.
// NewRateLimitingQueue does not emit metrics. For use with a MetricsProvider, please use
// NewRateLimitingQueueWithConfig instead and specify a name.
func NewRateLimitingQueue(rateLimiter RateLimiter) RateLimitingInterface {
	return NewRateLimitingQueueWithConfig(rateLimiter, RateLimitingQueueConfig{})
}

// NewNamedRateLimitingQueue constructs a new named workqueue with rateLimited queuing ability.
// Deprecated: Use NewRateLimitingQueueWithConfig instead.
func NewNamedRateLimitingQueue(rateLimiter RateLimiter, name string) RateLimitingInterface {
	return NewRateLimitingQueueWithConfig(rateLimiter, RateLimitingQueueConfig{
		Name: name,
	})
}

// NewRateLimitingQueueWithConfig constructs a new workqueue with rateLimited queuing ability
// with options to customize different properties.
// Remember to call Forget!  If you don't, you may end up tracking failures forever.
func NewRateLimitingQueueWithConfig(rateLimiter RateLimiter, config RateLimitingQueueConfig) RateLimitingInterface {
	if config.Clock == nil {
		config.Clock = clock.RealClock{}
	}

	if config.DelayingQueue == nil {
		config.DelayingQueue = NewDelayingQueueWithConfig(DelayingQueueConfig{
			Name:            config.Name,
			MetricsProvider: config.MetricsProvider,
			Clock:           config.Clock,
		})
	}

	return &rateLimitingType{
		DelayingInterface: config.DelayingQueue,
		rateLimiter:       rateLimiter,
	}
}
```
这里的区别就是里面的延时队列有没有指定的名字。注意到这里有一个 RateLimiter 类型，后面要详细讲，另外 rateLimitingType 就是上面接口的具体实现类型了。

## RateLimiter

RateLimiter 表示一个限速器，我们看下限速器是什么意思。先看接口定义：

```go
type RateLimiter interface {
	// When gets an item and gets to decide how long that item should wait
	When(item interface{}) time.Duration // 返回一个 item 需要等待的时长
	// Forget indicates that an item is finished being retried.  Doesn't matter whether it's for failing
	// or for success, we'll stop tracking it
	Forget(item interface{}) // 标识一个元素结束重试
	// NumRequeues returns back how many failures the item has had
	NumRequeues(item interface{}) int  // 标识这个元素被处理里多少次了
}
```

这个接口有五个实现，分别叫做：

1. `BucketRateLimiter`
2. `ItemExponentialFailureRateLimiter`
3. `ItemFastSlowRateLimiter`
4. `MaxOfRateLimiter`
5. `WithMaxWaitRateLimiter`

### BucketRateLimiter

这个限速器可说的不多，用了 golang 标准库的 `golang.org/x/time/rate.Limiter` 实现。`BucketRateLimiter` 实例化的时候比如传递一个 `rate.NewLimiter(rate.Limit(10), 100)` 进去，表示令牌桶里最多有 100 个令牌，每秒发放 10 个令牌。

```go
// BucketRateLimiter adapts a standard bucket to the workqueue ratelimiter API
type BucketRateLimiter struct {
	*rate.Limiter
}

func (r *BucketRateLimiter) When(item interface{}) time.Duration {
	return r.Limiter.Reserve().Delay() // 过多久后给当前 item 发放一个令牌
}

func (r *BucketRateLimiter) NumRequeues(item interface{}) int {
	return 0
}

func (r *BucketRateLimiter) Forget(item interface{}) {
}
```

### ItemExponentialFailureRateLimiter

Exponential 是指数的意思，从这个限速器的名字大概能猜到是失败次数越多，限速越长而且是指数级增长的一种限速器。

结构体定义如下，属性含义基本可以望文生义

```go
// ItemExponentialFailureRateLimiter does a simple baseDelay*2^<num-failures> limit
// dealing with max failures and expiration are up to the caller
type ItemExponentialFailureRateLimiter struct {
	failuresLock sync.Mutex
	failures     map[interface{}]int

	baseDelay time.Duration
	maxDelay  time.Duration
}
```
主要逻辑是 When() 函数是如何实现的


```go
func (r *ItemExponentialFailureRateLimiter) When(item interface{}) time.Duration {
	r.failuresLock.Lock()
	defer r.failuresLock.Unlock()

	exp := r.failures[item]
	r.failures[item] = r.failures[item] + 1 // 失败次数加一

  // 每调用一次，exp 也就加了1，对应到这里时 2^n 指数爆炸
	// The backoff is capped such that 'calculated' value never overflows.
	backoff := float64(r.baseDelay.Nanoseconds()) * math.Pow(2, float64(exp))
	if backoff > math.MaxInt64 {
		return r.maxDelay
	}

	calculated := time.Duration(backoff)
	if calculated > r.maxDelay { // 如果超过最大延时，则返回最大延时
		return r.maxDelay
	}

	return calculated
}
```
另外两个函数太简单了：

```go
func (r *ItemExponentialFailureRateLimiter) NumRequeues(item interface{}) int {
	r.failuresLock.Lock()
	defer r.failuresLock.Unlock()

	return r.failures[item]
}

func (r *ItemExponentialFailureRateLimiter) Forget(item interface{}) {
	r.failuresLock.Lock()
	defer r.failuresLock.Unlock()

	delete(r.failures, item)
}
```

### ItemFastSlowRateLimiter
快慢限速器，也就是先快后慢，定义一个阈值，超过了就慢慢重试。先看类型定义：

```go
// ItemFastSlowRateLimiter does a quick retry for a certain number of attempts, then a slow retry after that
type ItemFastSlowRateLimiter struct {
	failuresLock sync.Mutex
	failures     map[interface{}]int

	maxFastAttempts int
	fastDelay       time.Duration
	slowDelay       time.Duration
}
```
同样继续来看具体的方法实现

```go
func (r *ItemFastSlowRateLimiter) When(item interface{}) time.Duration {
	r.failuresLock.Lock()
	defer r.failuresLock.Unlock()

	r.failures[item] = r.failures[item] + 1 // 标识重试次数 + 1

	if r.failures[item] <= r.maxFastAttempts { // 如果快重试次数没有用完，则返回 fastDelay
		return r.fastDelay
	}

	return r.slowDelay // 反之返回 slowDelay
}

func (r *ItemFastSlowRateLimiter) NumRequeues(item interface{}) int {
	r.failuresLock.Lock()
	defer r.failuresLock.Unlock()

	return r.failures[item]
}

func (r *ItemFastSlowRateLimiter) Forget(item interface{}) {
	r.failuresLock.Lock()
	defer r.failuresLock.Unlock()

	delete(r.failures, item)
}
```

### MaxOfRateLimiter

这个限速器看着有点乐呵人，内部放多个限速器，然后返回限速最狠的一个延时：

```go
// MaxOfRateLimiter calls every RateLimiter and returns the worst case response
// When used with a token bucket limiter, the burst could be apparently exceeded in cases where particular items
// were separately delayed a longer time.
type MaxOfRateLimiter struct {
	limiters []RateLimiter
}
```

### WithMaxWaitRateLimiter

这个限速器也很简单，就是在其他限速器上包装一个最大延迟的属性，如果到了最大延时，则直接返回：

```go
// WithMaxWaitRateLimiter have maxDelay which avoids waiting too long
type WithMaxWaitRateLimiter struct {
	limiter  RateLimiter
	maxDelay time.Duration
}

func (w WithMaxWaitRateLimiter) When(item interface{}) time.Duration {
	delay := w.limiter.When(item)
	if delay > w.maxDelay {
		return w.maxDelay
	}

	return delay
}

func (w WithMaxWaitRateLimiter) Forget(item interface{}) {
	w.limiter.Forget(item)
}

func (w WithMaxWaitRateLimiter) NumRequeues(item interface{}) int {
	return w.limiter.NumRequeues(item)
}
```

## 限速队列的实现

看完了上面的限速器的概念，限速队列的实现就很简单了：

```go
// rateLimitingType wraps an Interface and provides rateLimited re-enquing
type rateLimitingType struct {
	DelayingInterface

	rateLimiter RateLimiter
}

// AddRateLimited AddAfter's the item based on the time when the rate limiter says it's ok
func (q *rateLimitingType) AddRateLimited(item interface{}) {
	q.DelayingInterface.AddAfter(item, q.rateLimiter.When(item))
}

func (q *rateLimitingType) NumRequeues(item interface{}) int {
	return q.rateLimiter.NumRequeues(item)
}

func (q *rateLimitingType) Forget(item interface{}) {
	q.rateLimiter.Forget(item)
}
```

在自定义控制器开发场景下，我们用到的 workqueue 其实是用的这里的延时队列实现，一个延时队列也就是实现了 item 延时入队效果，内部是一个“优先级队列”，用了“最小堆”（有序完全二叉树），从而我们在 requeueAfter 中指定一个调谐过程 1 分钟后重试，实现原理也就明白了。
