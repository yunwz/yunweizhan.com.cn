# Reflector 源码分析

## 概述

Reflector 的任务就是向 apiserver watch 特定类型的资源，拿到变更通知后将其丢到 DeltaFIFO 队列中。

另外在ListerWatcher中分析过 ListerWatcher 是如何从 apiserver 中 list-watch 资源的，今天我们继续来看 Reflector 的实现。

## 入口 - Reflector.Run()
Reflector 的启动入口是 Run() 方法：
client-go/tools/cache/relector.go:

```go
// Run repeatedly uses the reflector's ListAndWatch to fetch all the
// objects and subsequent deltas.
// Run will exit when stopCh is closed.
func (r *Reflector) Run(stopCh <-chan struct{}) {
	klog.V(3).Infof("Starting reflector %s (%s) from %s", r.typeDescription, r.resyncPeriod, r.name)
	wait.BackoffUntil(func() {
		if err := r.ListAndWatch(stopCh); err != nil {
			r.watchErrorHandler(r, err)
		}
	}, r.backoffManager, true, stopCh)
	klog.V(3).Infof("Stopping reflector %s (%s) from %s", r.typeDescription, r.resyncPeriod, r.name)
}
```
这里有一些健壮性机制，用于处理 apiserver 短暂失联的场景。我们直接来看主要逻辑先，也就是 Reflector.ListAndWatch() 方法的内容。

## 核心 - Reflector.ListAndWatch()
Reflector.ListAndWatch() 方法有将近 200 行，是 Reflector 的核心逻辑之一。ListAndWatch() 方法做的事情是先 list 特定资源的所有对象，然后获取其资源版本，接着使用这个资源版本来开始 watch 流程。watch 到新版本资源然后将其加入 DeltaFIFO 的动作是在 watchHandler() 方法中具体实现的，后面一节会单独分析。在此之前 list 到的最新 items 会通过 syncWith() 方法添加一个 Sync 类型的 DeltaType 到 DeltaFIFO 中，所以 list 操作本身也会触发后面的调谐逻辑运行。具体来看:

client-go/tools/cache/relector.go:
```go
// ListAndWatch first lists all items and get the resource version at the moment of call,
// and then use the resource version to watch.
// It returns error if ListAndWatch didn't even try to initialize watch.
func (r *Reflector) ListAndWatch(stopCh <-chan struct{}) error {
	klog.V(3).Infof("Listing and watching %v from %s", r.typeDescription, r.name)
	var err error
	var w watch.Interface
	useWatchList := ptr.Deref(r.UseWatchList, false) //是否使用流式传输从ApiServer获取数据
	fallbackToList := !useWatchList

	if useWatchList { //打开流以从服务器获取数据
		w, err = r.watchList(stopCh)
		if w == nil && err == nil {
			// stopCh was closed
			return nil
		}
		if err != nil {
			klog.Warningf("The watchlist request ended with an error, falling back to the standard LIST/WATCH semantics because making progress is better than deadlocking, err = %v", err)
			fallbackToList = true //回退到使用旧方式获取数据
			// ensure that we won't accidentally pass some garbage down the watch.
			w = nil
		}
	}

	if fallbackToList { //建立一个 LIST 请求，以块的形式获取数据
		err = r.list(stopCh)
		if err != nil {
			return err
		}
	}

	klog.V(2).Infof("Caches populated for %v from %s", r.typeDescription, r.name)

	resyncerrc := make(chan error, 1)
	cancelCh := make(chan struct{})
	defer close(cancelCh)
	go r.startResync(stopCh, cancelCh, resyncerrc) //开启重新同步携程
	return r.watch(w, stopCh, resyncerrc) //调用watch
}
```
如果useWatchList为true，则调用watchList方法（以流的方式与ApiServer通信），否则调用list方法。

接下来看看`watchList`和`list`方法。

### watchList()
watchList以流的方式与ApiServer通信,具体的交互逻辑可以参考[官方github的提案](https://github.com/kubernetes/enhancements/tree/master/keps/sig-api-machinery/3157-watch-list#proposal)
client-go/tools/cache/relector.go:

```go

```

### list()

client-go/tools/cache/reflector.go:
```go
func (r *Reflector) list(stopCh <-chan struct{}) error {
	var resourceVersion string
	// 当 r.lastSyncResourceVersion 为 "" 时这里为 "0"，当使用 r.lastSyncResourceVersion 失败时这里为 ""
	// 区别是 "" 会直接请求到 etcd，获取一个最新的版本，而 "0" 访问的是 cache
	options := metav1.ListOptions{ResourceVersion: r.relistResourceVersion()}

  // trace 是用于记录操作耗时的，这里的逻辑是超过 10s 的步骤打印出来
	initTrace := trace.New("Reflector ListAndWatch", trace.Field{Key: "name", Value: r.name})
	defer initTrace.LogIfLong(10 * time.Second)
	var list runtime.Object
	var paginatedResult bool
	var err error
	listCh := make(chan struct{}, 1)
	panicCh := make(chan interface{}, 1)
	go func() {// 内嵌一个函数，这里会直接调用
		defer func() {
			if r := recover(); r != nil {
				panicCh <- r // 收集这个 goroutine 的 panic 信息
			}
		}()
		// 开始尝试收集 list 的 chunks，我们在 《Kubernetes List-Watch 机制原理与实现 - chunked》中介绍过相关逻辑
		pager := pager.New(pager.SimplePageFunc(func(opts metav1.ListOptions) (runtime.Object, error) {
			return r.listerWatcher.List(opts)
		}))
		switch {
		case r.WatchListPageSize != 0:
			pager.PageSize = r.WatchListPageSize
		case r.paginatedResult:
			// We got a paginated result initially. Assume this resource and server honor
			// paging requests (i.e. watch cache is probably disabled) and leave the default
			// pager size set.
		case options.ResourceVersion != "" && options.ResourceVersion != "0":
			// User didn't explicitly request pagination.
			//
			// With ResourceVersion != "", we have a possibility to list from watch cache,
			// but we do that (for ResourceVersion != "0") only if Limit is unset.
			// To avoid thundering herd on etcd (e.g. on master upgrades), we explicitly
			// switch off pagination to force listing from watch cache (if enabled).
			// With the existing semantic of RV (result is at least as fresh as provided RV),
			// this is correct and doesn't lead to going back in time.
			//
			// We also don't turn off pagination for ResourceVersion="0", since watch cache
			// is ignoring Limit in that case anyway, and if watch cache is not enabled
			// we don't introduce regression.
			pager.PageSize = 0
		}

		list, paginatedResult, err = pager.ListWithAlloc(context.Background(), options)
		if isExpiredError(err) || isTooLargeResourceVersionError(err) {
			r.setIsLastSyncResourceVersionUnavailable(true)
			// Retry immediately if the resource version used to list is unavailable.
			// The pager already falls back to full list if paginated list calls fail due to an "Expired" error on
			// continuation pages, but the pager might not be enabled, the full list might fail because the
			// resource version it is listing at is expired or the cache may not yet be synced to the provided
			// resource version. So we need to fallback to resourceVersion="" in all to recover and ensure
			// the reflector makes forward progress.
			list, paginatedResult, err = pager.ListWithAlloc(context.Background(), metav1.ListOptions{ResourceVersion: r.relistResourceVersion()})
		}
		close(listCh)
	}()
	select {
	case <-stopCh:
		return nil
	case r := <-panicCh:
		panic(r)
	case <-listCh:
	}
	initTrace.Step("Objects listed", trace.Field{Key: "error", Value: err})
	if err != nil {
		klog.Warningf("%s: failed to list %v: %v", r.name, r.typeDescription, err)
		return fmt.Errorf("failed to list %v: %w", r.typeDescription, err)
	}

	// We check if the list was paginated and if so set the paginatedResult based on that.
	// However, we want to do that only for the initial list (which is the only case
	// when we set ResourceVersion="0"). The reasoning behind it is that later, in some
	// situations we may force listing directly from etcd (by setting ResourceVersion="")
	// which will return paginated result, even if watch cache is enabled. However, in
	// that case, we still want to prefer sending requests to watch cache if possible.
	//
	// Paginated result returned for request with ResourceVersion="0" mean that watch
	// cache is disabled and there are a lot of objects of a given type. In such case,
	// there is no need to prefer listing from watch cache.
	if options.ResourceVersion == "0" && paginatedResult {
		r.paginatedResult = true
	}

  // list 成功
	r.setIsLastSyncResourceVersionUnavailable(false) // list was successful
	listMetaInterface, err := meta.ListAccessor(list)
	if err != nil {
		return fmt.Errorf("unable to understand list result %#v: %v", list, err)
	}
	resourceVersion = listMetaInterface.GetResourceVersion()
	initTrace.Step("Resource version extracted")
	items, err := meta.ExtractListWithAlloc(list)
	if err != nil {
		return fmt.Errorf("unable to understand list result %#v (%v)", list, err)
	}
	initTrace.Step("Objects extracted")
	// 将 list 到的 items 添加到 store 里，这里是 store 也就是 DeltaFIFO，也就是添加一个 Sync DeltaType 这里的 resourveVersion 并没有用到
	if err := r.syncWith(items, resourceVersion); err != nil {
		return fmt.Errorf("unable to sync list result: %v", err)
	}
	initTrace.Step("SyncWith done")
	r.setLastSyncResourceVersion(resourceVersion)
	initTrace.Step("Resource version updated")
	return nil
}
```

### startResync()
client-go/tools/cache/reflector.go:
```go
func (r *Reflector) startResync(stopCh <-chan struct{}, cancelCh <-chan struct{}, resyncerrc chan error) {
	resyncCh, cleanup := r.resyncChan()
	defer func() {
		cleanup() // Call the last one written into cleanup
	}()
	for {
		select {
		case <-resyncCh:
		case <-stopCh:
			return
		case <-cancelCh:
			return
		}
		if r.ShouldResync == nil || r.ShouldResync() {
			klog.V(4).Infof("%s: forcing resync", r.name)
			if err := r.store.Resync(); err != nil {
				resyncerrc <- err
				return
			}
		}
		cleanup()
		resyncCh, cleanup = r.resyncChan()
	}
}
```

### watch()

client-go/tools/cache/reflector.go:
```go
func (r *Reflector) watch(w watch.Interface, stopCh <-chan struct{}, resyncerrc chan error) error {
	var err error
	retry := NewRetryWithDeadline(r.MaxInternalErrorRetryDuration, time.Minute, apierrors.IsInternalError, r.clock)

	for {
		// give the stopCh a chance to stop the loop, even in case of continue statements further down on errors
		select {
		case <-stopCh:
			// we can only end up here when the stopCh
			// was closed after a successful watchlist or list request
			if w != nil {
				w.Stop()
			}
			return nil
		default:
		}

		// start the clock before sending the request, since some proxies won't flush headers until after the first watch event is sent
		start := r.clock.Now()

		if w == nil {
		  // 超时时间是 5-10分钟
			timeoutSeconds := int64(minWatchTimeout.Seconds() * (rand.Float64() + 1.0))
			options := metav1.ListOptions{
				ResourceVersion: r.LastSyncResourceVersion(),
				// We want to avoid situations of hanging watchers. Stop any watchers that do not
				// receive any events within the timeout window.
				// 如果超时没有接收到任何 Event，这时候需要停止 watch，避免一直挂着
				TimeoutSeconds: &timeoutSeconds,
				// To reduce load on kube-apiserver on watch restarts, you may enable watch bookmarks.
				// Reflector doesn't assume bookmarks are returned at all (if the server do not support
				// watch bookmarks, it will ignore this field).
				// 用于降低 apiserver 压力，bookmark 类型响应的对象主要只有 RV 信息
				AllowWatchBookmarks: true,
			}

      // 调用 watch, w是watch.Interface类型
			w, err = r.listerWatcher.Watch(options)
			if err != nil {
			  // 这时候直接 re-list 已经没有用了，apiserver 暂时拒绝服务
				if canRetry := isWatchErrorRetriable(err); canRetry {
					klog.V(4).Infof("%s: watch of %v returned %v - backing off", r.name, r.typeDescription, err)
					select {
					case <-stopCh:
						return nil
					case <-r.backoffManager.Backoff().C():
						continue
					}
				}
				return err
			}
		}

     // 核心逻辑之一，后面单独会讲到
		err = watchHandler(start, w, r.store, r.expectedType, r.expectedGVK, r.name, r.typeDescription, r.setLastSyncResourceVersion, nil, r.clock, resyncerrc, stopCh)
		// Ensure that watch will not be reused across iterations.
		// 需要关闭 watch.Interface，因为新一轮的调用会传递新的 watch.Interface 进来
		w.Stop()
		w = nil
		retry.After(err)
		if err != nil {
			if err != errorStopRequested {
				switch {
				case isExpiredError(err):
					// Don't set LastSyncResourceVersionUnavailable - LIST call with ResourceVersion=RV already
					// has a semantic that it returns data at least as fresh as provided RV.
					// So first try to LIST with setting RV to resource version of last observed object.
					klog.V(4).Infof("%s: watch of %v closed with: %v", r.name, r.typeDescription, err)
				case apierrors.IsTooManyRequests(err):
					klog.V(2).Infof("%s: watch of %v returned 429 - backing off", r.name, r.typeDescription)
					select {
					case <-stopCh:
						return nil
					case <-r.backoffManager.Backoff().C():
						continue
					}
				case apierrors.IsInternalError(err) && retry.ShouldRetry():
					klog.V(2).Infof("%s: retrying watch of %v internal error: %v", r.name, r.typeDescription, err)
					continue
				default:
					klog.Warningf("%s: watch of %v ended with: %v", r.name, r.typeDescription, err)
				}
			}
			return nil
		}
	}
}
```

### watchHandler

在 watchHandler() 方法中完成了将 watch 到的 Event 根据其 EventType 分别调用 DeltaFIFO 的 Add()/Update/Delete() 等方法完成对象追加到 DeltaFIFO 队列的过程。
watchHandler() 方法的调用在一个 for 循环中，所以一次 watchHandler() 工作流程完成后，函数退出，新一轮的调用会传递进来新的 watch.Interface 和 resourceVersion 等，我们具体来看。

client-go/tools/cache/relector.go
```go
// watchHandler watches w and sets setLastSyncResourceVersion
func watchHandler(start time.Time,
	w watch.Interface,
	store Store,
	expectedType reflect.Type,
	expectedGVK *schema.GroupVersionKind,
	name string,
	expectedTypeName string,
	setLastSyncResourceVersion func(string),
	exitOnInitialEventsEndBookmark *bool,
	clock clock.Clock,
	errc chan error,
	stopCh <-chan struct{},
) error {
	eventCount := 0
	if exitOnInitialEventsEndBookmark != nil {
		// set it to false just in case somebody
		// made it positive
		*exitOnInitialEventsEndBookmark = false
	}

loop:
	for {
		select {
		case <-stopCh:
			return errorStopRequested
		case err := <-errc:
			return err
		// 接收 event
		case event, ok := <-w.ResultChan():
			if !ok {
				break loop
			}
			// 如果是 "ERROR"
			if event.Type == watch.Error {
				return apierrors.FromObject(event.Object)
			}
			// 创建 Reflector 的时候会指定一个 expectedType
			if expectedType != nil {
			  // 类型不匹配
				if e, a := expectedType, reflect.TypeOf(event.Object); e != a {
					utilruntime.HandleError(fmt.Errorf("%s: expected type %v, but watch event object had type %v", name, e, a))
					continue
				}
			}
			// 没有对应 Golang 结构体的对象可以通过这种方式来指定期望类型
			if expectedGVK != nil {
				if e, a := *expectedGVK, event.Object.GetObjectKind().GroupVersionKind(); e != a {
					utilruntime.HandleError(fmt.Errorf("%s: expected gvk %v, but watch event object had gvk %v", name, e, a))
					continue
				}
			}
			meta, err := meta.Accessor(event.Object)
			if err != nil {
				utilruntime.HandleError(fmt.Errorf("%s: unable to understand watch event %#v", name, event))
				continue
			}
			// 新的 ResourceVersion
			resourceVersion := meta.GetResourceVersion()
			switch event.Type {
			// 调用 DeltaFIFO 的 Add/Update/Delete 等方法完成不同类型 Event 等处理
			case watch.Added:
				err := store.Add(event.Object)
				if err != nil {
					utilruntime.HandleError(fmt.Errorf("%s: unable to add watch event object (%#v) to store: %v", name, event.Object, err))
				}
			case watch.Modified:
				err := store.Update(event.Object)
				if err != nil {
					utilruntime.HandleError(fmt.Errorf("%s: unable to update watch event object (%#v) to store: %v", name, event.Object, err))
				}
			case watch.Deleted:
				// TODO: Will any consumers need access to the "last known
				// state", which is passed in event.Object? If so, may need
				// to change this.
				err := store.Delete(event.Object)
				if err != nil {
					utilruntime.HandleError(fmt.Errorf("%s: unable to delete watch event object (%#v) from store: %v", name, event.Object, err))
				}
			case watch.Bookmark:
				// A `Bookmark` means watch has synced here, just update the resourceVersion
				if meta.GetAnnotations()["k8s.io/initial-events-end"] == "true" {
					if exitOnInitialEventsEndBookmark != nil {
						*exitOnInitialEventsEndBookmark = true
					}
				}
			default:
				utilruntime.HandleError(fmt.Errorf("%s: unable to understand watch event %#v", name, event))
			}
			// 更新 resourceVersion
			setLastSyncResourceVersion(resourceVersion)
			if rvu, ok := store.(ResourceVersionUpdater); ok {
				rvu.UpdateResourceVersion(resourceVersion)
			}
			eventCount++
			if exitOnInitialEventsEndBookmark != nil && *exitOnInitialEventsEndBookmark {
				watchDuration := clock.Since(start)
				klog.V(4).Infof("exiting %v Watch because received the bookmark that marks the end of initial events stream, total %v items received in %v", name, eventCount, watchDuration)
				return nil
			}
		}
	}

  // 耗时
	watchDuration := clock.Since(start)
	// 1s 就结束了，而且没有收到 event，属于异常情况
	if watchDuration < 1*time.Second && eventCount == 0 {
		return fmt.Errorf("very short watch: %s: Unexpected watch close - watch lasted less than a second and no items received", name)
	}
	klog.V(4).Infof("%s: Watch close - %v total %v items received", name, expectedTypeName, eventCount)
	return nil
}
```

## NewReflector()

继续来看下 Reflector 的初始化。NewReflector() 的参数里有一个 ListerWatcher 类型的 lw，还有有一个 expectedType 和 store，
lw 就是我们在《Kubernetes client-go 源码分析 - ListWatcher》中介绍的那个 ListerWatcher，expectedType指定期望关注的类型，
而 store 是一个 DeltaFIFO，我们在《Kubernetes client-go 源码分析 - DeltaFIFO》中也有详细的介绍过。加在一起大致可以预想到 Reflector 通过 ListWatcher 提供的能力去 list-watch apiserver，然后将 Event 加到 DeltaFIFO 中。

client-go/tools/cache/reflector.go:
```go
// NewReflector creates a new Reflector with its name defaulted to the closest source_file.go:line in the call stack
// that is outside this package. See NewReflectorWithOptions for further information.
func NewReflector(lw ListerWatcher, expectedType interface{}, store Store, resyncPeriod time.Duration) *Reflector {
	return NewReflectorWithOptions(lw, expectedType, store, ReflectorOptions{ResyncPeriod: resyncPeriod})
}

func NewReflectorWithOptions(lw ListerWatcher, expectedType interface{}, store Store, options ReflectorOptions) *Reflector {
	reflectorClock := options.Clock
	if reflectorClock == nil {
		reflectorClock = clock.RealClock{}
	}
	r := &Reflector{
		name:            options.Name,
		resyncPeriod:    options.ResyncPeriod,
		typeDescription: options.TypeDescription,
		listerWatcher:   lw,
		store:           store,
		// We used to make the call every 1sec (1 QPS), the goal here is to achieve ~98% traffic reduction when
		// API server is not healthy. With these parameters, backoff will stop at [30,60) sec interval which is
		// 0.22 QPS. If we don't backoff for 2min, assume API server is healthy and we reset the backoff.
		backoffManager:    wait.NewExponentialBackoffManager(800*time.Millisecond, 30*time.Second, 2*time.Minute, 2.0, 1.0, reflectorClock),
		clock:             reflectorClock,
		watchErrorHandler: WatchErrorHandler(DefaultWatchErrorHandler),
		expectedType:      reflect.TypeOf(expectedType),
	}

	if r.name == "" {
		r.name = naming.GetNameFromCallsite(internalPackages...)
	}

	if r.typeDescription == "" {
		r.typeDescription = getTypeDescriptionFromObject(expectedType)
	}

	if r.expectedGVK == nil {
		r.expectedGVK = getExpectedGVKFromObject(expectedType)
	}

	// don't overwrite UseWatchList if already set
	// because the higher layers (e.g. storage/cacher) disabled it on purpose
	if r.UseWatchList == nil {
		if s := os.Getenv("ENABLE_CLIENT_GO_WATCH_LIST_ALPHA"); len(s) > 0 {
			r.UseWatchList = ptr.To(true)
		}
	}

	return r
}
```

## 小结

如文章开头的图中所示，Reflector 的职责很清晰，要做的事情是保持 DeltaFIFO 中的 items 持续更新，具体实现是通过 ListWatcher 提供的 list-watch 能力来 list 指定类型的资源，这时候会产生一系列 Sync 事件，然后通过 list 到的 ResourceVersion 来开启 watch 过程，而 watch 到新的事件后，会和前面提到的 Sync 事件一样，都通过 DeltaFIFO 提供的方法构造相应的 DeltaType 添加到 DeltaFIFO 中。当然前面提到的更新也并不是直接修改 DeltaFIFO 中已经存在的 items，而是添加一个新的 DeltaType 到队列中。另外 DeltaFIFO 中添加新 DeltaType 的时候也会有一定的去重机制，我们以前在 ListWatcher 和 DeltaFIFO 中分别介绍过这两个组件的工作逻辑，有了这个基础后再看 Reflector 的工作流就相对轻松很多了。这里还有一个细节就是 watch 过程不是一劳永逸的，watch 到新的 event 后，会拿着对象的新 ResourceVersion 重新开启一轮新的 watch 过程。当然这里的 watch 调用也有超时机制，一系列的健壮性措施，所以我们脱离 Reflector(Informer) 直接使用 list-watch 还是很难手撕一套健壮的代码出来。

