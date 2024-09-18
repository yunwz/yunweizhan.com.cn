# ListerWatcher 源码分析

## 概述

ListerWatcher 是 Reflector 的一个主要能力提供者，今天我们具体看下 ListerWatcher 是如何实现 List() 和 Watch() 过程的。这里我们只跟到 RESTClient 到调用层，不深入 RESTClient 本身的实现；后面有机会再单独结合 apiserver、etcd 等整体串在一起讲 k8s 里的 list-watch 机制底层原理。

## `ListerWatcher` 接口定义

```go
type Lister interface {
	// List should return a list type object; the Items field will be extracted, and the
	// ResourceVersion field will be used to start the watch in the right place.
	List(options metav1.ListOptions) (runtime.Object, error)
}

// Watcher is any object that knows how to start a watch on a resource.
type Watcher interface {
	// Watch should begin a watch at the specified version.
	Watch(options metav1.ListOptions) (watch.Interface, error)
}

// ListerWatcher is any object that knows how to perform an initial list and start a watch on a resource.
type ListerWatcher interface {
	Lister
	Watcher
}
```

## `ListWatch` 对象的创建
ListWatch 结构体定义及对应的新建实例函数如下：

client-go/tools/cache/listwatch.go:
```go
type ListWatch struct {
	ListFunc  ListFunc
	WatchFunc WatchFunc
	// DisableChunking requests no chunking for this list watcher.
	DisableChunking bool
}

// NewListWatchFromClient creates a new ListWatch from the specified client, resource, namespace and field selector.
// 这里 Getter 类型的 c 对应一个 RESTClient
func NewListWatchFromClient(c Getter, resource string, namespace string, fieldSelector fields.Selector) *ListWatch {
	optionsModifier := func(options *metav1.ListOptions) {
		options.FieldSelector = fieldSelector.String()
	}
	// 调用下面这个 NewFilteredListWatchFromClient() 函数
	return NewFilteredListWatchFromClient(c, resource, namespace, optionsModifier)
}
```
主要逻辑在下面，list 和 watch 能力都是通过 RESTClient 提供：

```go
// NewFilteredListWatchFromClient creates a new ListWatch from the specified client, resource, namespace, and option modifier.
// Option modifier is a function takes a ListOptions and modifies the consumed ListOptions. Provide customized modifier function
// to apply modification to ListOptions with a field selector, a label selector, or any other desired options.
func NewFilteredListWatchFromClient(c Getter, resource string, namespace string, optionsModifier func(options *metav1.ListOptions)) *ListWatch {
  // list 某个 namespace 下的某个 resource
	listFunc := func(options metav1.ListOptions) (runtime.Object, error) {
		optionsModifier(&options)
		return c.Get(). // RESTClient.Get() 返回 *request.Request
			Namespace(namespace).
			Resource(resource).
			VersionedParams(&options, metav1.ParameterCodec).
			Do(context.TODO()).
			Get()
	}

	// watch 某个 namespace 下的某个 resource
	watchFunc := func(options metav1.ListOptions) (watch.Interface, error) {
		options.Watch = true
		optionsModifier(&options)
		return c.Get().
			Namespace(namespace).
			Resource(resource).
			VersionedParams(&options, metav1.ParameterCodec).
			Watch(context.TODO())
	}
	return &ListWatch{ListFunc: listFunc, WatchFunc: watchFunc}
}
```

## Getter
上面有一个 Getter 接口，看下定义：

client-go/tools/cache/listwatch.go:
```go
// Getter interface knows how to access Get method from RESTClient.
type Getter interface {
	Get() *restclient.Request
}
```
这里需要一个能够获得 *restclient.Request 的方式

我们实际使用的时候，会用 rest.Interface 接口类型的实例，这是一个相对底层的工具，封装的是 Kubernetes REST apis 相应动作：

client-go/rest/client.go:
```go
// Interface captures the set of operations for generically interacting with Kubernetes REST apis.
type Interface interface {
	GetRateLimiter() flowcontrol.RateLimiter
	Verb(verb string) *Request
	Post() *Request
	Put() *Request
	Patch(pt types.PatchType) *Request
	Get() *Request
	Delete() *Request
	APIVersion() schema.GroupVersion
}
```
对应实现是：

client-go/rest/client.go:
```go
// RESTClient imposes common Kubernetes API conventions on a set of resource paths.
// The baseURL is expected to point to an HTTP or HTTPS path that is the parent
// of one or more resources.  The server should return a decodable API resource
// object, or an api.Status object which contains information about the reason for
// any failure.
//
// Most consumers should use client.New() to get a Kubernetes API client.
type RESTClient struct {
	// base is the root URL for all invocations of the client
	base *url.URL
	// versionedAPIPath is a path segment connecting the base URL to the resource root
	versionedAPIPath string

	// content describes how a RESTClient encodes and decodes responses.
	content ClientContentConfig

	// creates BackoffManager that is passed to requests.
	createBackoffMgr func() BackoffManager

	// rateLimiter is shared among all requests created by this client unless specifically
	// overridden.
	rateLimiter flowcontrol.RateLimiter

	// warningHandler is shared among all requests created by this client.
	// If not set, defaultWarningHandler is used.
	warningHandler WarningHandler

	// Set specific behavior of the client.  If not set http.DefaultClient will be used.
	Client *http.Client
}
```
Getter 接口的 Get() 方法返回的是一个 *restclient.Request 类型，Request 的用法我们直接看 ListWatch 的 New 函数里已经看到是怎么玩的了。

至于这里的 RESTClient 和我们代码里常用的 Clientset 的关系，这里先简单举个例子介绍一下：我们在用 clientset 去 Get 一个指定名字的 DaemonSet 的时候，调用过程类似这样：

```go
r.AppsV1().DaemonSets("default").Get(ctx, "test-ds", getOpt)
```
这里的 Get 其实就是利用了 RESTClient 提供的能力，方法实现对应如下：

```go
// Get takes name of the daemonSet, and returns the corresponding daemonSet object, and an error if there is any.
func (c *daemonSets) Get(ctx context.Context, name string, options metav1.GetOptions) (result *v1.DaemonSet, err error) {
	result = &v1.DaemonSet{}
	err = c.client.Get(). //c.client就是RESTClient
		Namespace(c.ns).
		Resource("daemonsets").
		Name(name).
		VersionedParams(&options, scheme.ParameterCodec).
		Do(ctx).
		Into(result)
	return
}
```

## ListWatch
上面 NewFilteredListWatchFromClient() 函数里实现了 ListFunc 和 WatchFunc 属性的初始化，我们接着看下 ListWatch 结构体定义：

client-go/tools/cache/listwatch.go:
```go
type ListWatch struct {
	ListFunc  ListFunc
	WatchFunc WatchFunc
	// DisableChunking requests no chunking for this list watcher.
	DisableChunking bool
}
```
实现的接口叫做 ListerWatcher

```go
// ListerWatcher is any object that knows how to perform an initial list and start a watch on a resource.
type ListerWatcher interface {
	Lister
	Watcher
}
```

这里的 Lister 是
```go
type Lister interface {
   // List 的返回值应该是一个 list 类型对象，也就是里面有 Items 字段，里面的 ResourceVersion 可以用来 watch
   List(options metav1.ListOptions) (runtime.Object, error)
}
```

这里的 Watcher 是
```go
type Watcher interface {
	// Watch should begin a watch at the specified version.
	// 从指定的资源版本开始 watch
	Watch(options metav1.ListOptions) (watch.Interface, error)
}
```

### List() & Watch()
最后 ListWatch 对象的 List() 和 Watch() 的实现就没有太多新内容了：
client-go/tools/cache/listwatch.go:
```go
func (lw *ListWatch) List(options metav1.ListOptions) (runtime.Object, error) {
   // ListWatch 在 Reflector 中使用，在 Reflector 中已经有了分页逻辑，所以这里不能再添加分页相关代码
   return lw.ListFunc(options)
}

func (lw *ListWatch) Watch(options metav1.ListOptions) (watch.Interface, error) {
   return lw.WatchFunc(options)
}
```
