# Informer 源码分析

Informer 这个词的出镜率很高，我们在很多文章里都可以看到 Informer 的身影，但是我们在源码里真的去找一个叫做 Informer 的对象，却又发现找不到一个单纯的 Informer，但是有很多结构体或者接口里包含了 Informer 这个词……

和 Reflector、Workqueue 等组件不同，Informer 相对来说更加模糊，让人初读源码时感觉迷惑。今天我们一起来揭开 Informer 的面纱，看下到底什么是 Informer。

在《Kubernetes client-go 源码分析 - 开篇》中我们提到过 Informer 从 DeltaFIFO 中 pop 相应对象，然后通过 Indexer 将对象和索引丢到本地 cache 中，再触发相应的事件处理函数（Resource Event Handlers）运行。Informer 在整个自定义控制器工作流程中的位置如下图所示，今天我们具体分析下 Informer 的源码实现。

![client-go
](../assets/images/client-go-workflow.png)

## Controller

Informer 通过一个 controller 对象来定义，本身很简单，长这样：

clietn-go/tools/cache/controller.go:
```go
// `*controller` implements Controller
type controller struct {
	config         Config
	reflector      *Reflector
	reflectorMutex sync.RWMutex
	clock          clock.Clock
}
```

这里有我们熟悉的 Reflector，可以猜到 Informer 启动的时候会去运行 Reflector，
从而通过 Reflector 实现 list-watch apiserver，更新“事件”到 DeltaFIFO 中用于进一步处理。
Config 对象等会再看，我们继续看下 controller 对应的接口：

```go
// Controller is a low-level controller that is parameterized by a
// Config and used in sharedIndexInformer.
type Controller interface {
	// Run does two things.  One is to construct and run a Reflector
	// to pump objects/notifications from the Config's ListerWatcher
	// to the Config's Queue and possibly invoke the occasional Resync
	// on that Queue.  The other is to repeatedly Pop from the Queue
	// and process with the Config's ProcessFunc.  Both of these
	// continue until `stopCh` is closed.
	Run(stopCh <-chan struct{})

	// HasSynced delegates to the Config's Queue
	HasSynced() bool

	// LastSyncResourceVersion delegates to the Reflector when there
	// is one, otherwise returns the empty string
	LastSyncResourceVersion() string
}
```
这里的核心明显是 Run(stopCh <-chan struct{}) 方法，Run 负责两件事情：

1. 构造 Reflector 利用 ListerWatcher 的能力将对象事件更新到 DeltaFIFO；
2. 从 DeltaFIFO 中 Pop 对象然后调用 ProcessFunc 来处理；

### Controller 的初始化
Controller 的 New 方法很简单：

client-go/tools/cache/controller.go:
```go
func New(c *Config) Controller {
	ctlr := &controller{
		config: *c,
		clock:  &clock.RealClock{},
	}
	return ctlr
}
```
这里没有太多的逻辑，主要是传递了一个 Config 进来，可以猜到核心逻辑是 Config 从何而来以及后面如何使用。
我们先向上跟一下 Config 从哪里来，New() 的调用有几个地方，我们不去看 newInformer() 分支的代码，
因为实际开发中主要是使用 SharedIndexInformer，两个入口初始化 Controller 的逻辑类似，
我们直接跟更实用的一个分支，看 `func (s *sharedIndexInformer) Run(stopCh <-chan struct{})` 方法中如何调用的 `New()`：

client-go/tools/cache/shared_informer.go:
```go
func (s *sharedIndexInformer) Run(stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()

  //确保Run方法只会被执行一次
	if s.HasStarted() {
		klog.Warningf("The sharedIndexInformer has started, run more than once is not allowed")
		return
	}

  //调用匿名函数
	func() {
		s.startedLock.Lock()
		defer s.startedLock.Unlock()

    //创建DeltaFIFO
		fifo := NewDeltaFIFOWithOptions(DeltaFIFOOptions{
			KnownObjects:          s.indexer,
			EmitDeltaTypeReplaced: true,
			Transformer:           s.transform,
		})

		cfg := &Config{
			Queue:             fifo,
			ListerWatcher:     s.listerWatcher,
			ObjectType:        s.objectType,
			ObjectDescription: s.objectDescription,
			FullResyncPeriod:  s.resyncCheckPeriod,
			RetryOnError:      false,
			ShouldResync:      s.processor.shouldResync,

			Process:           s.HandleDeltas,
			WatchErrorHandler: s.watchErrorHandler,
		}

		s.controller = New(cfg)
		s.controller.(*controller).clock = s.clock
		s.started = true
	}()

	// Separate stop channel because Processor should be stopped strictly after controller
	processorStopCh := make(chan struct{})
	var wg wait.Group
	defer wg.Wait()              // Wait for Processor to stop
	defer close(processorStopCh) // Tell Processor to stop
	wg.StartWithChannel(processorStopCh, s.cacheMutationDetector.Run)
	wg.StartWithChannel(processorStopCh, s.processor.run)

	defer func() {
		s.startedLock.Lock()
		defer s.startedLock.Unlock()
		s.stopped = true // Don't want any new listeners
	}()
	s.controller.Run(stopCh)
}
```
后面会分析 SharedIndexInformer，所以这里先不纠结 SharedIndexInformer 的细节，
我们从这里可以看到 SharedIndexInformer 的 Run() 过程里会构造一个 Config，
然后创建 Controller，最后调用 Controller 的 Run() 方法。
另外这里也可以看到我们前面系列文章里分析过的 DeltaFIFO、ListerWatcher 等，
这里还有一个比较重要的是 Process:s.HandleDeltas, 这一行，Process 属性的类型是 ProcessFunc，
这里可以看到具体的 ProcessFunc 是 HandleDeltas 方法。

### Controller 的启动
上面提到 Controller 的初始化本身没有太多的逻辑，主要是构造了一个 Config 对象传递进来，所以 Controller 启动的时候肯定会有这个 Config 的使用逻辑，我们具体来看：

client-go/tools/cache/controller.go:
```go
// Run begins processing items, and will continue until a value is sent down stopCh or it is closed.
// It's an error to call Run more than once.
// Run blocks; call via go.
func (c *controller) Run(stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()
	go func() {
		<-stopCh
		c.config.Queue.Close()
	}()

	// 利用 Config 里的配置构造 Reflector
	r := NewReflectorWithOptions(
		c.config.ListerWatcher,
		c.config.ObjectType,
		c.config.Queue,
		ReflectorOptions{
			ResyncPeriod:    c.config.FullResyncPeriod,
			TypeDescription: c.config.ObjectDescription,
			Clock:           c.clock,
		},
	)
	r.ShouldResync = c.config.ShouldResync
	r.WatchListPageSize = c.config.WatchListPageSize
	if c.config.WatchErrorHandler != nil {
		r.watchErrorHandler = c.config.WatchErrorHandler
	}

	c.reflectorMutex.Lock()
	c.reflector = r
	c.reflectorMutex.Unlock()

	var wg wait.Group

  // 启动 Reflector
	wg.StartWithChannel(stopCh, r.Run)

	wait.Until(c.processLoop, time.Second, stopCh)
	wg.Wait()
}
```
这里的逻辑很简单，构造 Reflector 后运行起来，然后执行 c.processLoop，所以很明显，Controller 的业务逻辑肯定隐藏在 processLoop 方法里，我们继续来看。

### processLoop
client-go/tools/cache/controller.go:
```go
func (c *controller) processLoop() {
	for {
		obj, err := c.config.Queue.Pop(PopProcessFunc(c.config.Process))
		if err != nil {
			if err == ErrFIFOClosed {
				return
			}
			if c.config.RetryOnError {
				// This is the safe way to re-enqueue.
				c.config.Queue.AddIfNotPresent(obj)
			}
		}
	}
}
```
这里的逻辑是从 DeltaFIFO 中 Pop 出一个对象丢给 PopProcessFunc 处理，如果失败了就 re-enqueue 到 DeltaFIFO 中。
我们前面提到过这里的 PopProcessFunc 实现是 HandleDeltas() 方法，所以这里的主要逻辑就转到了 HandleDeltas() 是如何实现的了。

### HandleDeltas
这里我们先回顾下 DeltaFIFO 的存储结构，看下这个图：

![Delta FIFO
](../assets/images/DeltaFIFO.png)
然后再看源码，这里的逻辑主要是遍历一个 Deltas 里的所有 Delta，然后根据 Delta 的类型来决定如何操作 Indexer，也就是更新本地 cache，同时分发相应的通知。

client-go/tools/cache/shared_informer.go:
```go
func (s *sharedIndexInformer) HandleDeltas(obj interface{}, isInInitialList bool) error {
	s.blockDeltas.Lock()
	defer s.blockDeltas.Unlock()

	if deltas, ok := obj.(Deltas); ok {
	 //处理逻辑
		return processDeltas(s, s.indexer, deltas, isInInitialList)
	}
	return errors.New("object given as Process argument is not Deltas")
}

func processDeltas(
	// Object which receives event notifications from the given deltas
	handler ResourceEventHandler,
	clientState Store,
	deltas Deltas,
	isInInitialList bool,
) error {
	// from oldest to newest
	// 对于每个 Deltas 来说，里面存了很多的 Delta，也就是对应不同 Type 的多个 Object，这里的遍历会从旧往新走
	for _, d := range deltas {
		obj := d.Object

		switch d.Type {
		// 除了 Deleted 外所有情况
		case Sync, Replaced, Added, Updated:
		  // 通过 indexer 从 cache 里查询当前 Object，如果存在
			if old, exists, err := clientState.Get(obj); err == nil && exists {
			  // 更新 indexer 里的对象
				if err := clientState.Update(obj); err != nil {
					return err
				}
				handler.OnUpdate(old, obj)
			} else {// 如果本地 cache 里没有这个 Object，则添加
				if err := clientState.Add(obj); err != nil {
					return err
				}
				handler.OnAdd(obj, isInInitialList)
			}
		// 如果是删除操作，则从 indexer 里删除这个 Object
		case Deleted:
			if err := clientState.Delete(obj); err != nil {
				return err
			}
			handler.OnDelete(obj)
		}
	}
	return nil
}
```
对于增删改等情况，会调用`handler`等相应方法，从前一节的调用可以知道handler实际上就是`sharedIndexInformer`，因此继续看sharedIndexInformer的几个方法：

client-go/tools/cache/shared_informer.go:
```go
func (s *sharedIndexInformer) OnAdd(obj interface{}, isInInitialList bool) {
	// Invocation of this function is locked under s.blockDeltas, so it is
	// save to distribute the notification
	s.cacheMutationDetector.AddObject(obj)
	// 分发一个新增通知
	s.processor.distribute(addNotification{newObj: obj, isInInitialList: isInInitialList}, false)
}

// Conforms to ResourceEventHandler
func (s *sharedIndexInformer) OnUpdate(old, new interface{}) {
	isSync := false

	// If is a Sync event, isSync should be true
	// If is a Replaced event, isSync is true if resource version is unchanged.
	// If RV is unchanged: this is a Sync/Replaced event, so isSync is true

	if accessor, err := meta.Accessor(new); err == nil {
		if oldAccessor, err := meta.Accessor(old); err == nil {
			// Events that didn't change resourceVersion are treated as resync events
			// and only propagated to listeners that requested resync
			isSync = accessor.GetResourceVersion() == oldAccessor.GetResourceVersion()
		}
	}

	// Invocation of this function is locked under s.blockDeltas, so it is
	// save to distribute the notification
	s.cacheMutationDetector.AddObject(new)
	// 分发一个更新通知
	s.processor.distribute(updateNotification{oldObj: old, newObj: new}, isSync)
}

// Conforms to ResourceEventHandler
func (s *sharedIndexInformer) OnDelete(old interface{}) {
	// Invocation of this function is locked under s.blockDeltas, so it is
	// save to distribute the notification
	//分发一个删除通知
	s.processor.distribute(deleteNotification{oldObj: old}, false)
}
```
这里涉及到一个知识点：s.processor.distribute(addNotification{newObj: d.Object}, false) 中 processor 是什么？如何分发通知的？谁来接收通知？

## SharedIndexInformer
我们在 Operator 开发中，如果不使用 controller-runtime 库，也就是不通过 Kubebuilder 等工具来生成脚手架时，
经常会用到 SharedInformerFactory，比如典型的 sample-controller 中的 main() 函数：
sample-controller/main.go:
```go
func main() {
	klog.InitFlags(nil)
	flag.Parse()

	stopCh := signals.SetupSignalHandler()

	cfg, err := clientcmd.BuildConfigFromFlags(masterURL, kubeconfig)
	if err != nil {
		klog.Fatalf("Error building kubeconfig: %s", err.Error())
	}

	kubeClient, err := kubernetes.NewForConfig(cfg)
	if err != nil {
		klog.Fatalf("Error building kubernetes clientset: %s", err.Error())
	}

	exampleClient, err := clientset.NewForConfig(cfg)
	if err != nil {
		klog.Fatalf("Error building example clientset: %s", err.Error())
	}

	kubeInformerFactory := kubeinformers.NewSharedInformerFactory(kubeClient, time.Second*30)
	exampleInformerFactory := informers.NewSharedInformerFactory(exampleClient, time.Second*30)

	controller := NewController(kubeClient, exampleClient,
		kubeInformerFactory.Apps().V1().Deployments(),
		exampleInformerFactory.Samplecontroller().V1alpha1().Foos())

	kubeInformerFactory.Start(stopCh)
	exampleInformerFactory.Start(stopCh)

	if err = controller.Run(2, stopCh); err != nil {
		klog.Fatalf("Error running controller: %s", err.Error())
	}
}
```
这里可以看到我们依赖于 kubeInformerFactory.Apps().V1().Deployments() 提供一个 Informer，这里的 Deployments() 方法返回的是一个 DeploymentInformer 类型，DeploymentInformer 是什么呢？如下

client-go/informers/apps/v1/deployment.go:
```go
type DeploymentInformer interface {
	Informer() cache.SharedIndexInformer
	Lister() v1.DeploymentLister
}
```
可以看到所谓的 DeploymentInformer 由 “Informer” 和 “Lister” 组成，也就是说我们编码时用到的 Informer 本质就是一个 SharedIndexInformer

client-go/tools/cache/shared_informer.go:
```go
type SharedIndexInformer interface {
   SharedInformer
   AddIndexers(indexers Indexers) error
   GetIndexer() Indexer
}
```
这里的 Indexer 就很熟悉了，SharedInformer 又是啥呢？

client-go/tools/cache/shared_informer.go:
```go
type SharedInformer interface {
   // 可以添加自定义的 ResourceEventHandler
   AddEventHandler(handler ResourceEventHandler)
   // 附带 resync 间隔配置，设置为 0 表示不关心 resync
   AddEventHandlerWithResyncPeriod(handler ResourceEventHandler, resyncPeriod time.Duration)
   // 这里的 Store 指的是 Indexer
   GetStore() Store
   // 过时了，没有用
   GetController() Controller
   // 通过 Run 来启动
   Run(stopCh <-chan struct{})
   // 这里和 resync 逻辑没有关系，表示 Indexer 至少更新过一次全量的对象
   HasSynced() bool
   // 最后一次拿到的 RV
   LastSyncResourceVersion() string
   // 用于每次 ListAndWatch 连接断开时回调，主要就是日志记录的作用
   SetWatchErrorHandler(handler WatchErrorHandler) error
}
```

### sharedIndexerInformer
接下来就该看下 SharedIndexInformer 接口的实现了，sharedIndexerInformer 定义如下：

client-go/tools/cache/shared_informer.go:
```go
type sharedIndexInformer struct {
	indexer    Indexer
	controller Controller

	processor             *sharedProcessor
	cacheMutationDetector MutationDetector

	listerWatcher ListerWatcher

	// objectType is an example object of the type this informer is expected to handle. If set, an event
	// with an object with a mismatching type is dropped instead of being delivered to listeners.
	// 表示当前 Informer 期望关注的类型，主要是 GVK 信息
	objectType runtime.Object

	// objectDescription is the description of this informer's objects. This typically defaults to
	objectDescription string

	// resyncCheckPeriod is how often we want the reflector's resync timer to fire so it can call
	// shouldResync to check if any of our listeners need a resync.
	// reflector 的 resync 计时器计时间隔，通知所有的 listener 执行 resync
	resyncCheckPeriod time.Duration
	// defaultEventHandlerResyncPeriod is the default resync period for any handlers added via
	// AddEventHandler (i.e. they don't specify one and just want to use the shared informer's default
	// value).
	defaultEventHandlerResyncPeriod time.Duration
	// clock allows for testability
	clock clock.Clock

	started, stopped bool
	startedLock      sync.Mutex

	// blockDeltas gives a way to stop all event distribution so that a late event handler
	// can safely join the shared informer.
	blockDeltas sync.Mutex

	// Called whenever the ListAndWatch drops the connection with an error.
	watchErrorHandler WatchErrorHandler

	transform TransformFunc
}
```
这里的 Indexer、Controller、ListerWatcher 等都是我们熟悉的组件，sharedProcessor 我们在前面遇到了，需要重点关注一下。

### sharedProcessor
sharedProcessor 中维护了 processorListener 集合，然后分发通知对象到这些 listeners，先看下结构定义：

client-go/tools/cache/shared_informer.go:
```go
type sharedProcessor struct {
	listenersStarted bool
	listenersLock    sync.RWMutex
	// Map from listeners to whether or not they are currently syncing
	listeners map[*processorListener]bool
	clock     clock.Clock
	wg        wait.Group
}
```
马上就会有一个疑问了，processorListener 是什么？

#### processorListener
```go
type processorListener struct {
   nextCh chan interface{}
   addCh  chan interface{}
   // 核心属性
   handler ResourceEventHandler
   pendingNotifications buffer.RingGrowing
   requestedResyncPeriod time.Duration
   resyncPeriod time.Duration
   nextResync time.Time
   resyncLock sync.Mutex
}
```
可以看到 processorListener 里有一个 ResourceEventHandler，这是我们认识的组件。processorListener 有三个主要方法：
1. add(notification interface{})
2. pop()
3. run()

*run()*
```go
func (p *sharedProcessor) run(stopCh <-chan struct{}) {
	func() {
		p.listenersLock.RLock()
		defer p.listenersLock.RUnlock()
		for listener := range p.listeners {
		  //执行run方法
			p.wg.Start(listener.run)
			//执行pop方法
			p.wg.Start(listener.pop)
		}
		p.listenersStarted = true
	}()
	<-stopCh

	p.listenersLock.Lock()
	defer p.listenersLock.Unlock()
	for listener := range p.listeners {
		close(listener.addCh) // Tell .pop() to stop. .pop() will tell .run() to stop
	}

	// Wipe out list of listeners since they are now closed
	// (processorListener cannot be re-used)
	p.listeners = nil

	// Reset to false since no listeners are running
	p.listenersStarted = false

	p.wg.Wait() // Wait for all .pop() and .run() to stop
}

func (p *processorListener) run() {
	// this call blocks until the channel is closed.  When a panic happens during the notification
	// we will catch it, **the offending item will be skipped!**, and after a short delay (one second)
	// the next notification will be attempted.  This is usually better than the alternative of never
	// delivering again.
	stopCh := make(chan struct{})
	wait.Until(func() {
		for next := range p.nextCh {
			switch notification := next.(type) {
			case updateNotification:
				p.handler.OnUpdate(notification.oldObj, notification.newObj)
			case addNotification:
				p.handler.OnAdd(notification.newObj, notification.isInInitialList)
				if notification.isInInitialList {
					p.syncTracker.Finished()
				}
			case deleteNotification:
				p.handler.OnDelete(notification.oldObj)
			default:
				utilruntime.HandleError(fmt.Errorf("unrecognized notification: %T", next))
			}
		}
		// the only way to get here is if the p.nextCh is empty and closed
		close(stopCh)
	}, 1*time.Second, stopCh)
}
```
这里的逻辑很清晰，从 nextCh 里拿通知，然后根据其类型去调用 ResourceEventHandler 相应的 OnAdd/OnUpdate/OnDelete 方法。

*add() 和 pop()*
```go
func (p *processorListener) add(notification interface{}) {
	if a, ok := notification.(addNotification); ok && a.isInInitialList {
		p.syncTracker.Start()
	}
	// 将通知放到 addCh 中，所以下面 pop() 方法里先执行到的 case 是第二个
	p.addCh <- notification
}

func (p *processorListener) pop() {
	defer utilruntime.HandleCrash()
	defer close(p.nextCh) // Tell .run() to stop

	var nextCh chan<- interface{}
	var notification interface{}
	for {
		select {
		// 下面获取到的通知，添加到 nextCh 里，供 run() 方法中消费
		case nextCh <- notification:
			// Notification dispatched
			var ok bool
			// 从 pendingNotifications 里消费通知，生产者在下面 case 里
			notification, ok = p.pendingNotifications.ReadOne()
			if !ok { // Nothing to pop
				nextCh = nil // Disable this select case
			}
		case notificationToAdd, ok := <-p.addCh: // 逻辑从这里开始，从 addCh 里提取通知
			if !ok {
				return
			}
			if notification == nil { // No notification to pop (and pendingNotifications is empty)
				// Optimize the case - skip adding to pendingNotifications
				notification = notificationToAdd
				nextCh = p.nextCh
			} else { // There is already a notification waiting to be dispatched
			// 新添加的通知丢到 pendingNotifications
				p.pendingNotifications.WriteOne(notificationToAdd)
			}
		}
	}
}
```
也就是说 processorListener 提供了一定的缓冲机制来接收 notification，然后去消费这些 notification 调用 ResourceEventHandler 相关方法。
然后接着继续看 sharedProcessor 的几个主要方法。

#### sharedProcessor.addListener()
addListener 会直接调用 listener 的 run() 和 pop() 方法，这两个方法的逻辑我们上面已经分析过

```go
func (p *sharedProcessor) addListener(listener *processorListener) ResourceEventHandlerRegistration {
	p.listenersLock.Lock()
	defer p.listenersLock.Unlock()

	if p.listeners == nil {
		p.listeners = make(map[*processorListener]bool)
	}

	p.listeners[listener] = true

	if p.listenersStarted {
		p.wg.Start(listener.run)
		p.wg.Start(listener.pop)
	}

	return listener
}
```

#### sharedProcessor.distribute()
distribute 的逻辑就是调用 sharedProcessor 内部维护的所有 listener 的 add() 方法

client-go/tools/cache/shared_informer.go:
```go
func (p *sharedProcessor) distribute(obj interface{}, sync bool) {
	p.listenersLock.RLock()
	defer p.listenersLock.RUnlock()

	for listener, isSyncing := range p.listeners {
		switch {
		case !sync:
			// non-sync messages are delivered to every listener
			listener.add(obj)
		case isSyncing:
			// sync messages are delivered to every syncing listener
			listener.add(obj)
		default:
			// skipping a sync obj for a non-syncing listener
		}
	}
}
```

#### sharedProcessor.run()
run() 的逻辑和前面的 addListener() 类似，也就是调用 listener 的 run() 和 pop() 方法

```go
func (p *sharedProcessor) run(stopCh <-chan struct{}) {
	func() {
		p.listenersLock.RLock()
		defer p.listenersLock.RUnlock()
		for listener := range p.listeners {
			p.wg.Start(listener.run)
			p.wg.Start(listener.pop)
		}
		p.listenersStarted = true
	}()
	<-stopCh

	p.listenersLock.Lock()
	defer p.listenersLock.Unlock()
	for listener := range p.listeners {
		close(listener.addCh) // Tell .pop() to stop. .pop() will tell .run() to stop
	}

	// Wipe out list of listeners since they are now closed
	// (processorListener cannot be re-used)
	p.listeners = nil

	// Reset to false since no listeners are running
	p.listenersStarted = false

	p.wg.Wait() // Wait for all .pop() and .run() to stop
}
```
到这里基本就知道 sharedProcessor 的能力了，继续往下看。

#### sharedIndexInformer.Run()
继续来看 sharedIndexInformer 的 Run() 方法，这里面已经几乎没有陌生的内容了。

```go
func (s *sharedIndexInformer) Run(stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()

	if s.HasStarted() {
		klog.Warningf("The sharedIndexInformer has started, run more than once is not allowed")
		return
	}

	func() {
		s.startedLock.Lock()
		defer s.startedLock.Unlock()

    // DeltaFIFO 就很熟悉了
		fifo := NewDeltaFIFOWithOptions(DeltaFIFOOptions{
			KnownObjects:          s.indexer,
			EmitDeltaTypeReplaced: true,
			Transformer:           s.transform,
		})

    // Config 的逻辑也在上面遇到过了
		cfg := &Config{
			Queue:             fifo,
			ListerWatcher:     s.listerWatcher,
			ObjectType:        s.objectType,
			ObjectDescription: s.objectDescription,
			FullResyncPeriod:  s.resyncCheckPeriod,
			RetryOnError:      false,
			ShouldResync:      s.processor.shouldResync,

			Process:           s.HandleDeltas,
			WatchErrorHandler: s.watchErrorHandler,
		}

    // 前文分析过这里的 New() 函数逻辑了
		s.controller = New(cfg)
		s.controller.(*controller).clock = s.clock
		s.started = true
	}()

	// Separate stop channel because Processor should be stopped strictly after controller
	processorStopCh := make(chan struct{})
	var wg wait.Group
	defer wg.Wait()              // Wait for Processor to stop
	defer close(processorStopCh) // Tell Processor to stop
	wg.StartWithChannel(processorStopCh, s.cacheMutationDetector.Run)
	// processor 的 run 方法
	wg.StartWithChannel(processorStopCh, s.processor.run)

	defer func() {
		s.startedLock.Lock()
		defer s.startedLock.Unlock()
		s.stopped = true // Don't want any new listeners
	}()
	// controller 的 Run()
	s.controller.Run(stopCh)
}
```
到这里也就基本知道了 sharedIndexInformer 的逻辑了，再往上层走就剩下一个 SharedInformerFactory 了，继续看吧～


## SharedInformerFactory
我们前面提到过 SharedInformerFactory，现在具体来看一下 SharedInformerFactory 是怎么实现的。先看接口定义：

client-go/informers/factory.go:
```go
type SharedInformerFactory interface {
	internalinterfaces.SharedInformerFactory

	// Start initializes all requested informers. They are handled in goroutines
	// which run until the stop channel gets closed.
	Start(stopCh <-chan struct{})

	// Shutdown marks a factory as shutting down. At that point no new
	// informers can be started anymore and Start will return without
	// doing anything.
	//
	// In addition, Shutdown blocks until all goroutines have terminated. For that
	// to happen, the close channel(s) that they were started with must be closed,
	// either before Shutdown gets called or while it is waiting.
	//
	// Shutdown may be called multiple times, even concurrently. All such calls will
	// block until all goroutines have terminated.
	Shutdown()

	// WaitForCacheSync blocks until all started informers' caches were synced
	// or the stop channel gets closed.
	WaitForCacheSync(stopCh <-chan struct{}) map[reflect.Type]bool

	// ForResource gives generic access to a shared informer of the matching type.
	ForResource(resource schema.GroupVersionResource) (GenericInformer, error)

	// InformerFor returns the SharedIndexInformer for obj using an internal
	// client.
	InformerFor(obj runtime.Object, newFunc internalinterfaces.NewInformerFunc) cache.SharedIndexInformer

	Admissionregistration() admissionregistration.Interface
	Internal() apiserverinternal.Interface
	Apps() apps.Interface
	Autoscaling() autoscaling.Interface
	Batch() batch.Interface
	Certificates() certificates.Interface
	Coordination() coordination.Interface
	Core() core.Interface
	Discovery() discovery.Interface
	Events() events.Interface
	Extensions() extensions.Interface
	Flowcontrol() flowcontrol.Interface
	Networking() networking.Interface
	Node() node.Interface
	Policy() policy.Interface
	Rbac() rbac.Interface
	Resource() resource.Interface
	Scheduling() scheduling.Interface
	Storage() storage.Interface
	Storagemigration() storagemigration.Interface
}
```
这里涉及到几个点：

1. internalinterfaces.SharedInformerFactory
   这也是一个接口，比较简短：
   ```go
   type SharedInformerFactory interface {
   	  Start(stopCh <-chan struct{})
   	  InformerFor(obj runtime.Object, newFunc NewInformerFunc) cache.SharedIndexInformer
   }
   ```
   可以看到熟悉的 SharedIndexInformer

2. ForResource(resource schema.GroupVersionResource) (GenericInformer, error)
   这里接收一个 GVR，返回了一个 GenericInformer，看下什么是 GenericInformer：
   ```go
   type GenericInformer interface {
   	 Informer() cache.SharedIndexInformer
   	 Lister() cache.GenericLister
   }
   ```
   也很简短。

3. Apps() apps.Interface 等
   后面一堆方法是类似的，我们以 Apps() 为例来看下怎么回事。这里的 Interface 定义如下：
   client-go/informers/apps/interface.go:
   ```go
   type Interface interface {
   	 // V1 provides access to shared informers for resources in V1.
   	 V1() v1.Interface
   	 // V1beta1 provides access to shared informers for resources in V1beta1.
   	 V1beta1() v1beta1.Interface
   	 // V1beta2 provides access to shared informers for resources in V1beta2.
   	 V1beta2() v1beta2.Interface
   }
   ```
   显然应该继续看下 v1.Interface 是个啥。
   client-go/informers/apps/v1/interface.go:
   ```go
   type Interface interface {
  	   // ControllerRevisions returns a ControllerRevisionInformer.
  	   ControllerRevisions() ControllerRevisionInformer
  	   // DaemonSets returns a DaemonSetInformer.
  	   DaemonSets() DaemonSetInformer
  	   // Deployments returns a DeploymentInformer.
  	   Deployments() DeploymentInformer
  	   // ReplicaSets returns a ReplicaSetInformer.
  	   ReplicaSets() ReplicaSetInformer
  	   // StatefulSets returns a StatefulSetInformer.
  	   StatefulSets() StatefulSetInformer
   }
   ```
  到这里已经有看着很眼熟的 Deployments() DeploymentInformer 之类的代码了，DeploymentInformer 我们刚才看过内部结构，长这样：
  ```go
  type DeploymentInformer interface {
  	Informer() cache.SharedIndexInformer
  	Lister() v1.DeploymentLister
  }
  ```
  到这里也就不难理解 SharedInformerFactory 的作用了，它提供了所有 API group-version 的资源对应的 SharedIndexInformer，也就不难理解开头我们引用的 sample-controller 中的这行代码：
  ```go
  kubeInformerFactory.Apps().V1().Deployments()
  ```
  通过其可以拿到一个 Deployment 资源对应的 SharedIndexInformer。

### NewSharedInformerFactory
继续看下 SharedInformerFactory 是如何创建的

client-go/informers/factory.go:
```go
func NewSharedInformerFactory(client kubernetes.Interface, defaultResync time.Duration) SharedInformerFactory {
	return NewSharedInformerFactoryWithOptions(client, defaultResync)
}

func NewFilteredSharedInformerFactory(client kubernetes.Interface, defaultResync time.Duration, namespace string, tweakListOptions internalinterfaces.TweakListOptionsFunc) SharedInformerFactory {
	return NewSharedInformerFactoryWithOptions(client, defaultResync, WithNamespace(namespace), WithTweakListOptions(tweakListOptions))
}

// NewSharedInformerFactoryWithOptions constructs a new instance of a SharedInformerFactory with additional options.
func NewSharedInformerFactoryWithOptions(client kubernetes.Interface, defaultResync time.Duration, options ...SharedInformerOption) SharedInformerFactory {
	factory := &sharedInformerFactory{
		client:           client,
		namespace:        v1.NamespaceAll,// 空字符串 ""
		defaultResync:    defaultResync,
		informers:        make(map[reflect.Type]cache.SharedIndexInformer),// 可以存放不同类型的 SharedIndexInformer
		startedInformers: make(map[reflect.Type]bool),
		customResync:     make(map[reflect.Type]time.Duration),
	}

	// Apply all options
	for _, opt := range options {
		factory = opt(factory)
	}

	return factory
}
```
可以看到参数非常简单，主要是需要一个 Clientset，毕竟 ListerWatcher 的能力本质还是 client 提供的。

接着是如何启动

### sharedInformerFactory.Start()
client-go/informers/factory.go:
```go
func (f *sharedInformerFactory) Start(stopCh <-chan struct{}) {
	f.lock.Lock()
	defer f.lock.Unlock()

	if f.shuttingDown {
		return
	}

	for informerType, informer := range f.informers {
	  // 同类型只会调用一次，Run() 的逻辑我们前面介绍过了
		if !f.startedInformers[informerType] {
			f.wg.Add(1)
			// We need a new variable in each loop iteration,
			// otherwise the goroutine would use the loop variable
			// and that keeps changing.
			informer := informer
			go func() {
				defer f.wg.Done()
				informer.Run(stopCh)
			}()
			f.startedInformers[informerType] = true
		}
	}
}
```

## 小结
今天我们从一个基础 Informer - Controller 开始介绍，先分析了 Controller 的能力，也就是其通过构造 Reflector 并启动从而能够获取指定类型资源的“更新”事件，然后通过事件构造 Delta 放到 DeltaFIFO 中，进而在 processLoop 中从 DeltaFIFO 里 pop Deltas 来处理，一方面将对象通过 Indexer 同步到本地 cache，也就是一个 ThreadSafeStore，一方面调用 ProcessFunc 来处理这些 Delta。

然后 SharedIndexInformer 提供了构造 Controller 的能力，通过 HandleDeltas() 方法实现上面提到的 ProcessFunc，同时还引入了 sharedProcessor 在 HandleDeltas() 中用于事件通知的处理。sharedProcessor 分发事件通知的时候，接收方是内部继续抽象出来的 processorListener，在 processorListener 中完成了 ResourceEventHandler 具体回调函数的调用。

最后 SharedInformerFactory 又进一步封装了提供所有 api 资源对应的 SharedIndexInformer 的能力。也就是说一个 SharedIndexInformer 可以处理一种类型的资源，比如 Deployment 或者 Pod 等，而通过 SharedInformerFactory 可以轻松构造任意已知类型的 SharedIndexInformer。另外这里用到了 Clientset 提供的访问所有 api 资源的能力，通过其也就能够完整实现整套 Informer 逻辑了。




