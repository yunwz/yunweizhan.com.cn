== 模型

image:images/struct.svg[领域模型]

== 逻辑分析

.pkg/main/main.go
```go
func main() {
	cp, err := opensergo.NewControlPlane()
	if err != nil {
		log.Fatal(err)
	}
	err = cp.Start()
	if err != nil {
		log.Fatal(err)
	}
}
```

.controll_plane.go
```go
func NewControlPlane() (*ControlPlane, error) {
	cp := &ControlPlane{}

	operator, err := controller.NewKubernetesOperator(cp.sendMessage)
	if err != nil {
		return nil, err
	}

	cp.server = transport.NewServer(uint32(10246), []model.SubscribeRequestHandler{cp.handleSubscribeRequest})
	cp.operator = operator

	hostname, herr := os.Hostname()
	if herr != nil {
		// TODO: log here
		hostname = "unknown-host"
	}
	cp.protoDesc = &trpb.ControlPlaneDesc{Identifier: "osg-" + hostname}

	return cp, nil
}
```

从上面代码可以知道，控制面主要包含operator和server。前者用于与k8s交互，后者则用于下发配置给数据面。

=== ControllPlane的Start方法

```go
func (c *ControlPlane) Start() error {
	// Run the Kubernetes operator
	err := c.operator.Run()
	if err != nil {
		return err
	}
	// Run the transport server
	err = c.server.Run()
	if err != nil {
		return err
	}

	return nil
}
```

Start方法主要调用了operator和server的Run方法，接下来分别分析。

=== KubernetesOperator及其Run方法

.KubernetesOperator定义
```go
type KubernetesOperator struct {
	crdManager  ctrl.Manager
	controllers map[string]*CRDWatcher
	ctx         context.Context
	ctxCancel   context.CancelFunc
	started     atomic.Value

	sendDataHandler model.DataEntirePushHandler

	controllerMux sync.RWMutex
}
```

.Run方法
```go
func (k *KubernetesOperator) Run() error {

	// +kubebuilder:scaffold:builder
	go util.RunWithRecover(func() {
		setupLog.Info("Starting OpenSergo operator")
		if err := k.crdManager.Start(k.ctx); err != nil {
			setupLog.Error(err, "problem running OpenSergo operator")
		}
		setupLog.Info("OpenSergo operator will be closed")
	})
	return nil
}
```

可以看到operator的主要逻辑就是调用crdManager的Start方法。这里的crdManager实际类型是 `sigs.k8s.io/controller-runtime/pkg/manager/internal.go#controllerManager`

=== Server及Run方法

```go
type Server struct {
	transportServer *TransportServer
	grpcServer      *grpc.Server

	connectionManager *ConnectionManager

	port    uint32
	started *atomic.Bool
}

func (s *Server) Run() error {
	if s.started.CAS(false, true) {
		listener, err := net.Listen("tcp", fmt.Sprintf(":%d", s.port))
		if err != nil {
			return err
		}

		trpb.RegisterOpenSergoUniversalTransportServiceServer(s.grpcServer, s.transportServer)
		err = s.grpcServer.Serve(listener)
		if err != nil {
			return err
		}
	}
	return nil
}
```

逻辑：

. 将started状态设置为 `true`;
. 创建TcpListener，监听端口*10246*;
. 将transportServer注册到GrpcServer;
. 在tcplistener上启动grpcServer;

=== `KubernetesOperator` 逻辑

==== AddWatcher方法

该方法用于向 k8s 添加 controller ，用于监控具体类型的 CRD 资源。

PS：目前没看到该方法在哪里使用。

.AddWatcher 定义
```go
func (k *KubernetesOperator) AddWatcher(target model.SubscribeTarget) error {
	k.controllerMux.Lock()
	defer k.controllerMux.Unlock()

	var err error

  //如果指定类型的 CRD 资源已存在（已经在监控中），则直接将需要订阅的App加入到订阅列表中。
	existingWatcher, exists := k.controllers[target.Kind]
	if exists && !existingWatcher.HasSubscribed(target) {
    //将需要订阅的目标App加入到对应的CRDWatcher的订阅列表中
		// TODO: think more about here
		err = existingWatcher.AddSubscribeTarget(target)
		if err != nil {
			return err
		}
	} else {
    //查找该类型的 CRD 资源，找到说明支持该类型的订阅，否则不支持--返回error。
		crdMetadata, crdSupports := GetCrdMetadata(target.Kind)
		if !crdSupports {
			return errors.New("CRD not supported: " + target.Kind)
		}
    //创建对应 CRD 资源的CRDWatcher
		crdWatcher := NewCRDWatcher(k.crdManager, target.Kind, crdMetadata.Generator(), k.sendDataHandler)
		err = crdWatcher.AddSubscribeTarget(target)
		if err != nil {
			return err
		}

    //将新的CRDWatcher加入到crdManager中。
		crdRunnable, err := ctrl.NewControllerManagedBy(k.crdManager).For(crdMetadata.Generator()()).Build(crdWatcher)
		if err != nil {
			return err
		}
    //上面Build方法内部已经执行过`crdManager.Add`方法了，感觉这一步冗余了。
		err = k.crdManager.Add(crdRunnable)
		if err != nil {
			return err
		}
		//_ = crdRunnable.Start(k.ctx)
		k.controllers[target.Kind] = crdWatcher

	}
	setupLog.Info("OpenSergo CRD watcher has been added successfully")
	return nil
}
```

=== CRDWatcher

`CRDWatcher` 实现了k8s的controller(实现了 `reconcile.Reconciler` 接口)，在 `KubernetesOperator` 中，每个`CRDWatcher` 负责监控一种资源。

其定义如下:
.CRDWatcher struct
```go
type CRDWatcher struct {
	kind model.SubscribeKind

	client.Client
	logger logr.Logger
	scheme *runtime.Scheme

	// crdCache represents associated local cache for current kind of CRD.
	crdCache *CRDCache

	// subscribedList consists of all subscribed target of current kind of CRD.
	subscribedList       map[model.SubscribeTarget]bool
	subscribedNamespaces map[string]bool
	subscribedApps       map[string]bool

	crdGenerator    func() client.Object
	sendDataHandler model.DataEntirePushHandler

	updateMux sync.RWMutex
}
```

`client.Client`: k8s `controller-runtime` 中的接口，用于和kube-apiserver交互；
`crdCache`: 用于缓存数据；
`subscribedList`: 当前有哪些App被订阅；
`sendDataHandler`: 发送数据给数据面的方法，实际引用为 `ControlPlane.sendMessage`

==== Reconcile方法

CRDWatcher 是 k8s的controller实现，因此当接收到CRD时，会调用该方法进行处理。接下来我们看看该方法的逻辑：

.Reconcile 方法
```go
func (r *CRDWatcher) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
  // 如果当前CRD 资源所属的命名空间没有订阅（同时也说明该app没有被订阅），则不做任何处理直接返回
	if !r.HasAnySubscribedOfNamespace(req.Namespace) {
		// Ignore unmatched namespace
		return ctrl.Result{Requeue: false, RequeueAfter: 0}, nil
	}
	log := r.logger.WithValues("crdNamespace", req.Namespace, "crdName", req.Name, "kind", r.kind)

	// 从informer中获取CRD资源
	crd := r.crdGenerator()
	if err := r.Get(ctx, req.NamespacedName, crd); err != nil {
		k8sApiErr, ok := err.(*k8sApiError.StatusError)
		if !ok {
			log.Error(err, "Failed to get OpenSergo CRD")
			return ctrl.Result{
				Requeue:      false,
				RequeueAfter: 0,
			}, nil
		}
		if k8sApiErr.Status().Code != http.StatusNotFound {
			log.Error(err, "Failed to get OpenSergo CRD")
			return ctrl.Result{
				Requeue:      false,
				RequeueAfter: 0,
			}, nil
		}

		// 设置会nil，后边会执行缓存删除逻辑
		crd = nil
	}

	app := ""
	if crd != nil {
		// TODO: bugs here: we need to check for namespace-app group, not only for app.
		// 		 And we may also need to check for namespace change of a CRD.
		var hasAppLabel bool
		app, hasAppLabel = crd.GetLabels()["app"]
		appSubscribed := r.HasAnySubscribedOfApp(app)
    //如果当前CRD的Labels中不存在`app`标签，或者没有被订阅
		if !hasAppLabel || !appSubscribed {
      //Case1：prevContains为true，则需要将crdCache中的缓存删掉
			if _, prevContains := r.crdCache.GetByNamespacedName(req.NamespacedName); prevContains {
				log.Info("OpenSergo CRD will be deleted because app label has been changed", "newApp", app)
				crd = nil
			} else {
        //没有匹配到 `app` 标签，该crd之前也没有缓存过，直接返回，不需要其他处理
				// Ignore unmatched app label
				return ctrl.Result{
					Requeue:      false,
					RequeueAfter: 0,
				}, nil
			}
    //Case2：此时Labels中存在 `app` 标签，且被数据面订阅
		} else {
			log.Info("OpenSergo CRD received", "crd", crd)
		}

    //对于Case1，其效果等效于删除缓存；Case2则是将crd缓存到crdCache中。
		r.crdCache.SetByNamespaceApp(model.NamespacedApp{
			Namespace: req.Namespace,
			App:       app,
		}, crd)
		r.crdCache.SetByNamespacedName(req.NamespacedName, crd)

	} else {
    //删除之后，是否应该通知之前订阅该规则的数据面？
		app, _ = r.crdCache.GetAppByNamespacedName(req.NamespacedName)
		r.crdCache.DeleteByNamespaceApp(model.NamespacedApp{Namespace: req.Namespace, App: app}, req.Name)
		r.crdCache.DeleteByNamespacedName(req.NamespacedName)
		log.Info("OpenSergo CRD will be deleted")
	}

	nsa := model.NamespacedApp{
		Namespace: req.Namespace,
		App:       app,
	}
  //获取当前app的所有规则（可能存在多个CRD资源都是同一个app的情况）
	// TODO: Now we can do something for the crd object!
	rules, version := r.GetRules(nsa)
	status := &trpb.Status{
		Code:    int32(200),
		Message: "Get and send rule success",
		Details: nil,
	}
  //将所有规则和版本号封装成对象，之后将其对送到订阅该资源的数据面
	dataWithVersion := &trpb.DataWithVersion{Data: rules, Version: version}
  //实际是 `ControllPlane.sendMessage` 方法，用于向控制面发送更新后的规则
	err := r.sendDataHandler(req.Namespace, app, r.kind, dataWithVersion, status, "")
	if err != nil {
		log.Error(err, "Failed to send rules", "kind", r.kind)
	}
	return ctrl.Result{}, nil
}
```

以上就是opensergo控制面接收到k8s CRD时的处理逻辑了。

=== ControlPlane的sendMessage/sendMessageToStream方法

.ControlPlane#sendMessage方法
```go
func (c *ControlPlane) sendMessage(namespace, app, kind string, dataWithVersion *trpb.DataWithVersion, status *trpb.Status, respId string) error {
  //获取所有订阅该类型应用的所有连接
	connections, exists := c.server.ConnectionManager().Get(namespace, app, kind)
	if !exists || connections == nil {
		return errors.New("There is no connection for this kind")
	}
  //发送数据
	for _, connection := range connections {
		if connection == nil || !connection.IsValid() {
			// TODO: log.Debug
			continue
		}
    //通过grpc流发送数据包
		err := c.sendMessageToStream(connection.Stream(), namespace, app, kind, dataWithVersion, status, respId)
		if err != nil {
      //此处直接return会导致后续的数据面无法接收到数据
			// TODO: should not short-break here. Handle partial failure here.
			return err
		}
	}
	return nil
}

func (c *ControlPlane) sendMessageToStream(stream model.OpenSergoTransportStream, namespace, app, kind string, dataWithVersion *trpb.DataWithVersion, status *trpb.Status, respId string) error {
	if stream == nil {
		return nil
	}
  //发送数据
	return stream.SendMsg(&trpb.SubscribeResponse{
		Status:          status,
		Ack:             "",
		Namespace:       namespace,
		App:             app,
		Kind:            kind,
		DataWithVersion: dataWithVersion,
		ControlPlane:    c.protoDesc,
		ResponseId:      respId,
	})
}
```

以上便是下发数据给数据面的逻辑了，整体还是很清晰的。

=== gRPC连接的处理逻辑

控制面使用gRPC协议与数据面通信，本篇文章主要接收控制面，因此本节只分析gRPC服务端的处理逻辑

==== TransportServer结构体及其SubscribeConfig方法

.TransportServer结构体
```go
type TransportServer struct {
	trpb.OpenSergoUniversalTransportServiceServer

	connectionManager *ConnectionManager

	subscribeHandlers []model.SubscribeRequestHandler
}
```

 从上面的定义可知TransportServer组合了OpenSergoUniversalTransportServiceServer，并实现其 `SubscribeConfig` 方法。

`OpenSergoUniversalTransportServiceServer` 是grpc的service定义, 如下：

[source, protobuf]
```
service OpenSergoUniversalTransportService {
  rpc SubscribeConfig(stream SubscribeRequest) returns (stream SubscribeResponse);
}
```

.SubscribeConfig方法
```go
func (s *TransportServer) SubscribeConfig(stream trpb.OpenSergoUniversalTransportService_SubscribeConfigServer) error {
	var clientIdentifier model.ClientIdentifier
  //注意这里是死循环
	for {
    //从grpc流中获取数据
		recvData, err := stream.Recv()

    //如果是EOF错误则移除连接并结束循环
		if err == io.EOF {
			// Stream EOF
			_ = s.connectionManager.RemoveByIdentifier(clientIdentifier)
			return nil
		}
    //其他错误则结束循环并返回错误
		if err != nil {
			//remove stream
			_ = s.connectionManager.RemoveByIdentifier(clientIdentifier)
			return err
		}

    //ACK标志客户端接收到服务端发送的数据且已成功处理数据，无需其他处理
		if recvData.ResponseAck == ACKFlag {
			// This indicates the received data is a response of push-success.
			continue
    //NACK标志客户端无法处理接收到的数据，目前只是打印日志，无其他处理逻辑
		} else if recvData.ResponseAck == NACKFlag {
			// This indicates the received data is a response of push-failure.
			if recvData.Status.Code == CheckFormatError {
				// TODO: handle here (cannot retry)
				log.Println("Client response CheckFormatError")
			} else {
				// TODO: record error here and do something
				log.Printf("Client response NACK, code=%d\n", recvData.Status.Code)
			}
    //如果响应标志为空，则说明是请求，需要处理
		} else {
			// This indicates the received data is a SubscribeRequest.
			if clientIdentifier == "" && recvData.Identifier != "" {
				clientIdentifier = model.ClientIdentifier(recvData.Identifier)
			}
      //此处缺失recvData.Identifier为空的处理逻辑，问题不大--只有客户端第一次发送请求时携带recvData.Identifier即可

      //校验请求是否有效
			if !util.IsValidReq(recvData) {
				status := &trpb.Status{
					Code:    ReqFormatError,
					Message: "Request is invalid",
					Details: nil,
				}
				_ = stream.Send(&trpb.SubscribeResponse{
					Status:     status,
					Ack:        NACKFlag,
					ResponseId: recvData.RequestId,
				})
				continue
			}

      //调用subscribeHandlers处理请求，目前只有ControlPlane的handleSubscribeRequest方法
			for _, handler := range s.subscribeHandlers {
				err = handler(clientIdentifier, recvData, stream)
				if err != nil {
					// TODO: handle error
					log.Printf("Failed to handle SubscribeRequest, err=%s\n", err.Error())
				}
			}
		}

	}
}
```

==== ControlPlane的handleSubscribeRequest方法

该放用用于将客户端连接及其订阅类型加入到对应的CRDWather中。

```go
func (c *ControlPlane) handleSubscribeRequest(clientIdentifier model.ClientIdentifier, request *trpb.SubscribeRequest, stream model.OpenSergoTransportStream) error {
  //根据订阅的类型将其加入到对应的CRDWatcher中
	for _, kind := range request.Target.Kinds {
		crdWatcher, err := c.operator.RegisterWatcher(model.SubscribeTarget{
			NamespacedApp: model.NamespacedApp{Namespace: request.Target.Namespace, App: request.Target.App},
			Kind:          kind,
		})
		if err != nil {
			status := &trpb.Status{
				Code:    transport.RegisterWatcherError,
				Message: "Register watcher error",
				Details: nil,
			}
			err = c.sendMessageToStream(stream, request.Target.Namespace, request.Target.App, kind, nil, status, request.RequestId)
			if err != nil {
				// TODO: log here
				log.Printf("sendMessageToStream failed, err=%s\n", err.Error())
			}
      //如果订阅失败则跳过
			continue
		}
		_ = c.server.ConnectionManager().Add(request.Target.Namespace, request.Target.App, kind, transport.NewConnection(clientIdentifier, stream))
		// watcher缓存不空就发送
		rules, version := crdWatcher.GetRules(model.NamespacedApp{
			Namespace: request.Target.Namespace,
			App:       request.Target.App,
		})
		if len(rules) > 0 {
			status := &trpb.Status{
				Code:    transport.Success,
				Message: "Get and send rule success",
				Details: nil,
			}
			dataWithVersion := &trpb.DataWithVersion{
				Data:    rules,
				Version: version,
			}
			err = c.sendMessageToStream(stream, request.Target.Namespace, request.Target.App, kind, dataWithVersion, status, request.RequestId)
			if err != nil {
				// TODO: log here
				log.Printf("sendMessageToStream failed, err=%s\n", err.Error())
			}
		}
	}
	return nil
}
```
