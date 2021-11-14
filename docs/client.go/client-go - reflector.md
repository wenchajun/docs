# client-go

在说之前先引入上次讲的图：

![编写自定义控制器所依赖的组件](https://github.com/wenchajun/docs/blob/master/docs/images/%E7%BC%96%E5%86%99%E8%87%AA%E5%AE%9A%E4%B9%89%E6%8E%A7%E5%88%B6%E5%99%A8%E6%89%80%E4%BE%9D%E8%B5%96%E7%9A%84%E7%BB%84%E4%BB%B6.png)

在client-gp中我们提到过 **Reflector** 的任务就是向 apiserver watch 特定类型的资源，拿到变更通知后将其丢到 DeltaFIFO 队列中。另外前面已经在 client-go中分析过 **ListWatcher** 是如何从 apiserver 中 list-watch 资源的，今天我们继续来看 **Reflector** 的实现。

## 入口 - Reflector.Run()

**Reflector** 的启动入口是 `Run()` 方法：

- **client-go/tools/cache/reflector.go:218**

```go
func (r *Reflector) Run(stopCh <-chan struct{}) {
   klog.V(3).Infof("Starting reflector %s (%s) from %s", r.expectedTypeName, r.resyncPeriod, r.name)
   wait.BackoffUntil(func() {
      if err := r.ListAndWatch(stopCh); err != nil {
         r.watchErrorHandler(r, err)
      }
   }, r.backoffManager, true, stopCh)
   klog.V(3).Infof("Stopping reflector %s (%s) from %s", r.expectedTypeName, r.resyncPeriod, r.name)
}

```

这里有一些健壮性机制，用于处理 apiserver 短暂失联的场景。我们直接来看主要逻辑先，也就是 `Reflector.ListAndWatch()` 方法的内容。

## 核心 - Reflector.ListAndWatch()

`Reflector.ListAndWatch()` 方法有将近 200 行，是 Reflector 的核心逻辑之一。ListAndWatch() 方法做的事情是先 list 特定资源的所有对象，然后获取其资源版本，接着使用这个资源版本来开始 watch 流程。watch 到新版本资源然后将其加入 DeltaFIFO 的动作是在 watchHandler() 方法中具体实现的，后面一节会单独分析。在此之前 list 到的最新 items 会通过 syncWith() 方法添加一个 Sync 类型的 DeltaType 到 DeltaFIFO 中，所以 list 操作本身也会触发后面的调谐逻辑运行。具体来看：

- **client-go/tools/cache/reflector.go:254**

```go

func (r *Reflector) ListAndWatch(stopCh <-chan struct{}) error {
   klog.V(3).Infof("Listing and watching %v from %s", r.expectedTypeName, r.name)
   var resourceVersion string

   // 当 r.lastSyncResourceVersion 为 "" 时这里为 "0"，当使用 r.lastSyncResourceVersion 失败时这里为 ""
   // 区别是 "" 会直接请求到 etcd，获取一个最新的版本，而 "0" 访问的是 cache
   options := metav1.ListOptions{ResourceVersion: r.relistResourceVersion()}

   if err := func() error {
      // trace 是用于记录操作耗时的，这里的逻辑是超过 10s 的步骤打印出来
      initTrace := trace.New("Reflector ListAndWatch", trace.Field{"name", r.name})
      defer initTrace.LogIfLong(10 * time.Second)
      var list runtime.Object
      var paginatedResult bool
      var err error
      listCh := make(chan struct{}, 1)
      panicCh := make(chan interface{}, 1)
      go func() { // 内嵌一个函数，这里会直接调用
         defer func() {
            if r := recover(); r != nil { // 收集这个 goroutine panic 的时候将奔溃信息
               panicCh <- r
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
         case options.ResourceVersion != "" && options.ResourceVersion != "0":
            pager.PageSize = 0
         }

         list, paginatedResult, err = pager.List(context.Background(), options)
         if isExpiredError(err) || isTooLargeResourceVersionError(err) {
            // 设置这个属性后，下一次 list 会从 etcd 里取
            r.setIsLastSyncResourceVersionUnavailable(true)
            list, paginatedResult, err = pager.List(context.Background(), metav1.ListOptions{ResourceVersion: r.relistResourceVersion()})
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
      if err != nil {
         return fmt.Errorf("failed to list %v: %v", r.expectedTypeName, err)
      }

      if options.ResourceVersion == "0" && paginatedResult {
         r.paginatedResult = true
      }

      // list 成功
      r.setIsLastSyncResourceVersionUnavailable(false)
      initTrace.Step("Objects listed")
      listMetaInterface, err := meta.ListAccessor(list)
      if err != nil {
         return fmt.Errorf("unable to understand list result %#v: %v", list, err)
      }
      resourceVersion = listMetaInterface.GetResourceVersion()
      initTrace.Step("Resource version extracted")
      items, err := meta.ExtractList(list)
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
   }(); err != nil {
      return err
   }

   resyncerrc := make(chan error, 1)
   cancelCh := make(chan struct{})
   defer close(cancelCh)
   go func() {
      resyncCh, cleanup := r.resyncChan()
      defer func() {
         cleanup()
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
   }()

   for {
      select {
      case <-stopCh:
         return nil
      default:
      }
      // 超时时间是 5-10分钟
      timeoutSeconds := int64(minWatchTimeout.Seconds() * (rand.Float64() + 1.0))
      options = metav1.ListOptions{
         ResourceVersion: resourceVersion,
         // 如果超时没有接收到任何 Event，这时候需要停止 watch，避免一直挂着
         TimeoutSeconds: &timeoutSeconds,
         // 用于降低 apiserver 压力，bookmark 类型响应的对象主要只有 RV 信息
         AllowWatchBookmarks: true,
      }

      start := r.clock.Now()
      // 调用 watch
      w, err := r.listerWatcher.Watch(options)
      if err != nil {
         // 这时候直接 re-list 已经没有用了，apiserver 暂时拒绝服务
         if utilnet.IsConnectionRefused(err) || apierrors.IsTooManyRequests(err) {
            <-r.initConnBackoffManager.Backoff().C()
            continue
         }
         return err
      }
      // 核心逻辑之一，后面单独会讲到
      if err := r.watchHandler(start, w, &resourceVersion, resyncerrc, stopCh); err != nil {
         if err != errorStopRequested {
            switch {
            case isExpiredError(err):
               klog.V(4).Infof("%s: watch of %v closed with: %v", r.name, r.expectedTypeName, err)
            case apierrors.IsTooManyRequests(err):
               klog.V(2).Infof("%s: watch of %v returned 429 - backing off", r.name, r.expectedTypeName)
               <-r.initConnBackoffManager.Backoff().C()
               continue
            default:
               klog.Warningf("%s: watch of %v ended with: %v", r.name, r.expectedTypeName, err)
            }
         }
         return nil
      }
   }
}

```

### Reflector.watchHandler()

在 `watchHandler()` 方法中完成了将 **watch** 到的 **Event** 根据其 **EventType** 分别调用 **DeltaFIFO** 的 `Add()/Update/Delete()` 等方法完成对象追加到 **DeltaFIFO** 队列的过程。watchHandler() 方法的调用在一个 for 循环中，所以一次 watchHandler() 工作流程完成后，函数退出，新一轮的调用会传递进来新的 `watch.Interface` 和 `resourceVersion` 等，我们具体来看。

- **client-go/tools/cache/reflector.go:459**

```go
func (r *Reflector) watchHandler(start time.Time, w watch.Interface, resourceVersion *string, errc chan error, stopCh <-chan struct{}) error {
   eventCount := 0

   // 当前函数返回时需要关闭 watch.Interface，因为新一轮的调用会传递新的 watch.Interface 进来
   defer w.Stop()

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
         if r.expectedType != nil {
            // 类型不匹配
            if e, a := r.expectedType, reflect.TypeOf(event.Object); e != a {
               utilruntime.HandleError(fmt.Errorf("%s: expected type %v, but watch event object had type %v", r.name, e, a))
               continue
            }
         }
         // 没有对应 Golang 结构体的对象可以通过这种方式来指定期望类型
         if r.expectedGVK != nil {
            if e, a := *r.expectedGVK, event.Object.GetObjectKind().GroupVersionKind(); e != a {
               utilruntime.HandleError(fmt.Errorf("%s: expected gvk %v, but watch event object had gvk %v", r.name, e, a))
               continue
            }
         }
         meta, err := meta.Accessor(event.Object)
         if err != nil {
            utilruntime.HandleError(fmt.Errorf("%s: unable to understand watch event %#v", r.name, event))
            continue
         }
         // 新的 ResourceVersion
         newResourceVersion := meta.GetResourceVersion()
         switch event.Type {
         // 调用 DeltaFIFO 的 Add/Update/Delete 等方法完成不同类型 Event 等处理，我们在《Kubernetes client-go 源码分析 - DeltaFIFO》详细介绍过 DeltaFIFO 对应的 Add/Update/Delete 是如何实现的
         case watch.Added:
            err := r.store.Add(event.Object)
            if err != nil {
               utilruntime.HandleError(fmt.Errorf("%s: unable to add watch event object (%#v) to store: %v", r.name, event.Object, err))
            }
         case watch.Modified:
            err := r.store.Update(event.Object)
            if err != nil {
               utilruntime.HandleError(fmt.Errorf("%s: unable to update watch event object (%#v) to store: %v", r.name, event.Object, err))
            }
         case watch.Deleted:
            err := r.store.Delete(event.Object)
            if err != nil {
               utilruntime.HandleError(fmt.Errorf("%s: unable to delete watch event object (%#v) from store: %v", r.name, event.Object, err))
            }
         case watch.Bookmark:
         default:
            utilruntime.HandleError(fmt.Errorf("%s: unable to understand watch event %#v", r.name, event))
         }
         // 更新 resourceVersion
         *resourceVersion = newResourceVersion
         r.setLastSyncResourceVersion(newResourceVersion)
         if rvu, ok := r.store.(ResourceVersionUpdater); ok {
            rvu.UpdateResourceVersion(newResourceVersion)
         }
         eventCount++
      }
   }
   // 耗时
   watchDuration := r.clock.Since(start)
   // 1s 就结束了，而且没有收到 event，属于异常情况
   if watchDuration < 1*time.Second && eventCount == 0 {
      return fmt.Errorf("very short watch: %s: Unexpected watch close - watch lasted less than a second and no items received", r.name)
   }
   klog.V(4).Infof("%s: Watch close - %v total %v items received", r.name, r.expectedTypeName, eventCount)
   return nil
}



```

## NewReflector()

继续来看下 Reflector 的初始化。NewReflector() 的参数里有一个 ListerWatcher 类型的 lw，还有有一个 expectedType 和 store，lw 就是我们在[《Kubernetes client-go 源码分析 - ListWatcher》](https://www.danielhu.cn/post/k8s/client-go-listwatcher/)中介绍的那个 ListerWatcher，expectedType指定期望关注的类型，而 store 是一个 DeltaFIFO，我们在[《Kubernetes client-go 源码分析 - DeltaFIFO》](https://www.danielhu.cn/post/k8s/client-go-deltafifo/)中也有详细的介绍过。加在一起大致可以预想到 Reflector 通过 ListWatcher 提供的能力去 list-watch apiserver，然后将 Event 加到 DeltaFIFO 中。

- **client-go/tools/cache/reflector.go:166**

```go

func NewReflector(lw ListerWatcher, expectedType interface{}, store Store, resyncPeriod time.Duration) *Reflector {
   // 直接调用下面的 NewNamedReflector
   return NewNamedReflector(naming.GetNameFromCallsite(internalPackages...), lw, expectedType, store, resyncPeriod)
}

func NewNamedReflector(name string, lw ListerWatcher, expectedType interface{}, store Store, resyncPeriod time.Duration) *Reflector {
   realClock := &clock.RealClock{}
   r := &Reflector{
      name:          name,
      listerWatcher: lw,
      store:         store,
      // 重试机制，这里可以有效降低 apiserver 的负载，也就是重试间隔会越来越长
      backoffManager:         wait.NewExponentialBackoffManager(800*time.Millisecond, 30*time.Second, 2*time.Minute, 2.0, 1.0, realClock),
      initConnBackoffManager: wait.NewExponentialBackoffManager(800*time.Millisecond, 30*time.Second, 2*time.Minute, 2.0, 1.0, realClock),
      resyncPeriod:           resyncPeriod,
      clock:                  realClock,
      watchErrorHandler:      WatchErrorHandler(DefaultWatchErrorHandler),
   }
   r.setExpectedType(expectedType)
   return r
}
```

## 小结

如文章开头的图中所示，Reflector 的职责很清晰，要做的事情是保持 DeltaFIFO 中的 items 持续更新，具体实现是通过 ListWatcher 提供的 list-watch 能力来 list 指定类型的资源，这时候会产生一系列 Sync 事件，然后通过 list 到的 ResourceVersion 来开启 watch 过程，而 watch 到新的事件后，会和前面提到的 Sync 事件一样，都通过 DeltaFIFO 提供的方法构造相应的 DeltaType 添加到 DeltaFIFO 中。当然前面提到的更新也并不是直接修改 DeltaFIFO 中已经存在的 items，而是添加一个新的 DeltaType 到队列中。另外 DeltaFIFO 中添加新 DeltaType 的时候也会有一定的去重机制，我们以前在 ListWatcher 和 DeltaFIFO 中分别介绍过这两个组件的工作逻辑，有了这个基础后再看 Reflector 的工作流就相对轻松很多了。这里还有一个细节就是 watch 过程不是一劳永逸的，watch 到新的 event 后，会拿着对象的新 ResourceVersion 重新开启一轮新的 watch 过程。当然这里的 watch 调用也有超时机制，一系列的健壮性措施，所以我们脱离 Reflector(Informer) 直接使用 list-watch 还是很难手撕一套健壮的代码出来

参考 [https://www.danielhu.cn](https://www.danielhu.cn/)

