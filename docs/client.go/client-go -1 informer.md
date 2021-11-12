# client-go之informer

在说之前先引入上次讲的图：

![编写自定义控制器所依赖的组件](https://github.com/wenchajun/docs/blob/master/docs/images/%E7%BC%96%E5%86%99%E8%87%AA%E5%AE%9A%E4%B9%89%E6%8E%A7%E5%88%B6%E5%99%A8%E6%89%80%E4%BE%9D%E8%B5%96%E7%9A%84%E7%BB%84%E4%BB%B6.png)

从上面讲的图中我们本来应该讲一讲第一个，也就是**Reflector** ，从这里可以看出**Reflector** 的任务就是向 apiserver watch 特定类型的资源，拿到变更通知后将其丢到 DeltaFIFO 队列中。但是在这里我想先说一说**informer**，因为**Informer** 这个词的出镜率很高，而且我们在很多文章里都可以看到 **Informer** 的身影，但是我们在源码里真的去找一个叫做 **Informer** 的对象，却又发现找不到一个单纯的 **Informer**，但是有很多结构体或者接口里包含了 **Informer** 这个词……和 **Reflector**、**Workqueue** 等组件不同，**Informer** 相对来说更加模糊，让人初读源码时感觉迷惑。所以如果直接说**Reflector**，会显得很突兀。

**Informer** 从 **DeltaFIFO** 中 pop 相应对象，然后通过 **Indexer** 将对象和索引丢到本地 **cache** 中，再触发相应的事件处理函数（Resource Event Handlers）运行。

#### 在这里我们有一个疑问，就是为什么有*informer*?

其实我们可以通过Clientset来获取所有的原生资源对象，并对其进行CRUD操作，并且我们还可以进行 Watch 操作，可以监听资源对象的增、删、改、查操作，这样我们就可以根据自己的业务逻辑去处理这些数据了。

Watch 通过一个 event 接口监听对象的所有变化（添加、删除、更新）：

```
// staging/src/k8s.io/apimachinery/pkg/watch/watch.go
// Interface 可以被任何知道如何 watch 和通知变化的对象实现
type Interface interface {
  // Stops watching. Will close the channel returned by ResultChan(). Releases
  // any resources used by the watch.
  Stop()

  // Returns a chan which will receive all the events. If an error occurs
  // or Stop() is called, this channel will be closed, in which case the
  // watch should be completely cleaned up.
  ResultChan() <-chan Event
}
```

watch 接口的 ResultChan 方法会返回如下几种事件：

```

// staging/src/k8s.io/apimachinery/pkg/watch/watch.go
// EventType 定义可能的事件类型
type EventType string

const (
  Added    EventType = "ADDED"
  Modified EventType = "MODIFIED"
  Deleted  EventType = "DELETED"
  Bookmark EventType = "BOOKMARK"
  Error    EventType = "ERROR"

  DefaultChanSize int32 = 100
)

// Event represents a single event to a watched resource.
// +k8s:deepcopy-gen=true
type Event struct {
  Type EventType

  // Object is:
  //  * If Type is Added or Modified: the new state of the object.
  //  * If Type is Deleted: the state of the object immediately before deletion.
  //  * If Type is Bookmark: the object (instance of a type being watched) where
  //    only ResourceVersion field is set. On successful restart of watch from a
  //    bookmark resourceVersion, client is guaranteed to not get repeat event
  //    nor miss any events.
  //  * If Type is Error: *api.Status is recommended; other types may make sense
  //    depending on context.
  Object runtime.Object
}
```

这个接口虽然我们可以直接去使用，但是实际上并不建议这样使用，因为往往由于集群中的资源较多，我们需要自己在客户端去维护一套缓存，而这个维护成本也是非常大的，为此 client-go 也提供了自己的实现机制，那就是 `Informers`。Informers 是这个事件接口和带索引查找功能的内存缓存的组合，这样也是目前最常用的用法。Informers 第一次被调用的时候会首先在客户端调用 List 来获取全量的对象集合，然后通过 Watch 来获取增量的对象更新缓存。

#### 运行原理

一个控制器每次需要获取对象的时候都要访问 APIServer，这会给系统带来很高的负载，Informers 的内存缓存就是来解决这个问题的，此外 Informers 还可以几乎实时的监控对象的变化，而不需要轮询请求，这样就可以保证客户端的缓存数据和服务端的数据一致，就可以大大降低 APIServer 的压力了。

![informer](https://github.com/wenchajun/docs/blob/master/docs/images/informer.png)

如上图展示了 Informer 的基本处理流程：

- 以 events 事件的方式从 APIServer 获取数据

- 提供一个类似客户端的 Lister 接口，从内存缓存中 get 和 list 对象

- 为添加、删除、更新注册事件处理程序

此外 Informers 也有错误处理方式，当长期运行的 watch 连接中断时，它们会尝试使用另一个 watch 请求来恢复连接，在不丢失任何事件的情况下恢复事件流。如果中断的时间较长，而且 APIServer 丢失了事件（etcd 在新的 watch 请求成功之前从数据库中清除了这些事件），那么 Informers 就会重新 List 全量数据。

而且在重新 List 全量操作的时候还可以配置一个重新同步的周期参数，用于协调内存缓存数据和业务逻辑的数据一致性，每次过了该周期后，注册的事件处理程序就将被所有的对象调用，通常这个周期参数以分为单位，比如10分钟或者30分钟。

Informers 的这些高级特性以及超强的鲁棒性，都足以让我们不去直接使用客户端的 Watch() 方法来处理自己的业务逻辑，而且在 Kubernetes 中也有很多地方都有使用到 Informers。但是在使用 Informers 的时候，通常每个 GroupVersionResource（GVR）只实例化一个 Informers，但是有时候我们在一个应用中往往有使用多种资源对象的需求，这个时候为了方便共享 Informers，我们可以通过使用共享 Informer 工厂来实例化一个 Informer。

共享 Informer 工厂允许我们在应用中为同一个资源共享 Informer，也就是说不同的控制器循环可以使用相同的 watch 连接到后台的 APIServer，例如，kube-controller-manager 中的控制器数据量就非常多，但是对于每个资源（比如 Pod），在这个进程中只有一个 Informer。

#### 示例

首先我们创建一个 Clientset 对象（-->clientset就是上次说的四种clientset），然后使用 Clientset 来创建一个共享的 Informer 工厂，Informer 是通过 `informer-gen` 这个代码生成器工具自动生成的，位于 `k8s.io/client-go/informers` 中。

这里我们来创建一个用于获取 Deployment 的共享 Informer，代码如下所示：

```Go
package main

import (
  "flag"
  "fmt"
  "path/filepath"
  "time"

  v1 "k8s.io/api/apps/v1"
  "k8s.io/apimachinery/pkg/labels"
  "k8s.io/client-go/informers"
  "k8s.io/client-go/kubernetes"
  "k8s.io/client-go/rest"
  "k8s.io/client-go/tools/cache"
  "k8s.io/client-go/tools/clientcmd"
  "k8s.io/client-go/util/homedir"
)

func main() {
  var err error
  var config *rest.Config

  var kubeconfig *string

  if home := homedir.HomeDir(); home != "" {
    kubeconfig = flag.String("kubeconfig", filepath.Join(home, ".kube", "config"), "[可选] kubeconfig 绝对路径")
  } else {
    kubeconfig = flag.String("kubeconfig", "", "kubeconfig 绝对路径")
  }
  // 初始化 rest.Config 对象
  if config, err = rest.InClusterConfig(); err != nil {
    if config, err = clientcmd.BuildConfigFromFlags("", *kubeconfig); err != nil {
      panic(err.Error())
    }
  }
  // 创建 Clientset 对象
  clientset, err := kubernetes.NewForConfig(config)
  if err != nil {
    panic(err.Error())
  }

  // 初始化 informer factory（为了测试方便这里设置每30s重新 List 一次）
  informerFactory := informers.NewSharedInformerFactory(clientset, time.Second*30)
  // 对 Deployment 监听
  deployInformer := informerFactory.Apps().V1().Deployments()
  // 创建 Informer（相当于注册到工厂中去，这样下面启动的时候就会去 List & Watch 对应的资源）
  informer := deployInformer.Informer()
  // 创建 Lister
  deployLister := deployInformer.Lister()
  // 注册事件处理程序
  informer.AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc:    onAdd,
    UpdateFunc: onUpdate,
    DeleteFunc: onDelete,
  })

  stopper := make(chan struct{})
  defer close(stopper)

  // 启动 informer，List & Watch
  informerFactory.Start(stopper)
  // 等待所有启动的 Informer 的缓存被同步
  informerFactory.WaitForCacheSync(stopper)

  // 从本地缓存中获取 default 中的所有 deployment 列表
  deployments, err := deployLister.Deployments("default").List(labels.Everything())
  if err != nil {
    panic(err)
  }
  for idx, deploy := range deployments {
    fmt.Printf("%d -> %s\n", idx+1, deploy.Name)
  }
  <-stopper
}

func onAdd(obj interface{}) {
  deploy := obj.(*v1.Deployment)
  fmt.Println("add a deployment:", deploy.Name)
}

func onUpdate(old, new interface{}) {
  oldDeploy := old.(*v1.Deployment)
  newDeploy := new.(*v1.Deployment)
  fmt.Println("update deployment:", oldDeploy.Name, newDeploy.Name)
}

func onDelete(obj interface{}) {
  deploy := obj.(*v1.Deployment)
  fmt.Println("delete a deployment:", deploy.Name)
}




```

上面的代码运行可以获得 default 命名空间之下的所有 Deployment 信息以及整个集群的 Deployment 数据：

```

$ go run main.go
add a deployment: dingtalk-hook
add a deployment: spin-orca
add a deployment: argocd-server
add a deployment: istio-egressgateway
add a deployment: vault-agent-injector
add a deployment: rook-ceph-osd-0
add a deployment: rook-ceph-osd-2
add a deployment: code-server
......
1 -> nginx
2 -> helloworld-v1
3 -> productpage-v1
4 -> details-v1
......


```

这是因为我们首先通过 Informer 注册了事件处理程序，这样当我们启动 Informer 的时候首先会将集群的全量 Deployment 数据同步到本地的缓存中，会触发 `AddFunc` 这个回调函数，然后我们又在下面使用 `Lister()` 来获取 default 命名空间下面的所有 Deployment 数据，这个时候的数据是从本地的缓存中获取的，所以就看到了上面的结果，由于我们还配置了每30s重新全量 List 一次，所以正常每30s我们也可以看到所有的 Deployment 数据出现在 `UpdateFunc` 回调函数下面，我们也可以尝试去删除一个 Deployment，同样也会出现对应的 `DeleteFunc` 下面的事件。

Informers 是 client-go 中非常重要的概念，接下来我们将仔细分析 Informers 的实现原理，由于 Informers 实现非常复杂，我们将按照 Informers 的几个核心知识点分别进行讲解。

## Controller

**Informer** 通过一个 **controller** 对象来定义，本身很简单，长这样：

- **client-go/tools/cache/controller.go:89**

```go

type controller struct {
   config         Config
   reflector      *Reflector
   reflectorMutex sync.RWMutex
   clock          clock.Clock
}

```

这里有我们熟悉的 **Reflector**(将在下一节讲到)，可以猜到 Informer 启动的时候会去运行 Reflector，从而通过 Reflector 实现 list-watch apiserver，更新“事件”到 DeltaFIFO 中用于进一步处理。Config 对象等会再看，我们继续看下 controller 对应的接口：

```go
type Controller interface {
   Run(stopCh <-chan struct{})
   HasSynced() bool
   LastSyncResourceVersion() string
}
```

这里的核心明显是 `Run(stopCh <-chan struct{})` 方法，Run 负责两件事情：

1. 构造 Reflector 利用 ListerWatcher 的能力将对象事件更新到 DeltaFIFO；
2. 从 DeltaFIFO 中 Pop 对象然后调用 ProcessFunc 来处理；

### Controller 的初始化

Controller 的 New 方法很简单：

- **client-go/tools/cache/controller.go:116**

```
func New(c *Config) Controller {
   ctlr := &controller{
      config: *c,
      clock:  &clock.RealClock{},
   }
   return ctlr
}
```

这里没有太多的逻辑，主要是传递了一个 Config 进来，可以猜到核心逻辑是 Config 从何而来以及后面如何使用。我们先向上跟一下 Config 从哪里来，New() 的调用有几个地方，我们不去看 `newInformer()` 分支的代码，因为实际开发中主要是使用 **SharedIndexInformer**(这里的原因可以看上面，共享informer)，两个入口初始化 Controller 的逻辑类似，我们直接跟更实用的一个分支，看 `func (s *sharedIndexInformer) Run(stopCh <-chan struct{})` 方法中如何调用的 `New()`：

```go
func (s *sharedIndexInformer) Run(stopCh <-chan struct{}) {
	// ……
	fifo := NewDeltaFIFOWithOptions(DeltaFIFOOptions{
		KnownObjects:          s.indexer,
		EmitDeltaTypeReplaced: true,
	})

	cfg := &Config{
		Queue:            fifo,
		ListerWatcher:    s.listerWatcher,
		ObjectType:       s.objectType,
		FullResyncPeriod: s.resyncCheckPeriod,
		RetryOnError:     false,
		ShouldResync:     s.processor.shouldResync,

		Process:           s.HandleDeltas,//Process 属性的类型是 ProcessFunc，这里可以看到具体的 ProcessFunc 是 HandleDeltas 方法
		WatchErrorHandler: s.watchErrorHandler,
	}

	func() {
		s.startedLock.Lock()
		defer s.startedLock.Unlock()

		s.controller = New(cfg)//构造controller
		s.controller.(*controller).clock = s.clock
		s.started = true
	}()
  // ……
	s.controller.Run(stopCh)//run方法
}


```

上面只保留了主要代码，我们后面会分析 SharedIndexInformer，所以这里先不纠结 SharedIndexInformer 的细节，我们从这里可以看到 SharedIndexInformer 的 Run() 过程里会构造一个 Config，然后创建 Controller，最后调用 Controller 的 Run() 方法。另外这里也可以看到 DeltaFIFO、ListerWatcher 等，这里还有一个比较重要的是 `Process:s.HandleDeltas,` 这一行，Process 属性的类型是 ProcessFunc，这里可以看到具体的 ProcessFunc 是 HandleDeltas 方法。

### Controller 的启动

上面提到 Controller 的初始化本身没有太多的逻辑，主要是构造了一个 Config 对象传递进来，所以 Controller 启动的时候肯定会有这个 Config 的使用逻辑，我们具体来看：

- **client-go/tools/cache/controller.go:127**

```go
func (c *controller) Run(stopCh <-chan struct{}) {
   defer utilruntime.HandleCrash()
   go func() {
      <-stopCh
      c.config.Queue.Close()
   }()
   // 利用 Config 里的配置构造 Reflector
   r := NewReflector(
      c.config.ListerWatcher,
      c.config.ObjectType,
      c.config.Queue,
      c.config.FullResyncPeriod,
   )
   r.ShouldResync = c.config.ShouldResync
   r.WatchListPageSize = c.config.WatchListPageSize
   r.clock = c.clock
   if c.config.WatchErrorHandler != nil {
      r.watchErrorHandler = c.config.WatchErrorHandler
   }

   c.reflectorMutex.Lock()
   c.reflector = r
   c.reflectorMutex.Unlock()

   var wg wait.Group
   // 启动 Reflector
   wg.StartWithChannel(stopCh, r.Run)
   // 执行 Controller 的 processLoop
   wait.Until(c.processLoop, time.Second, stopCh)
   wg.Wait()
}
```

这里的逻辑很简单，构造 Reflector 后运行起来，然后执行 `c.processLoop`，所以很明显，Controller 的业务逻辑肯定隐藏在 processLoop 方法里，我们继续来看。

### processLoop

- **client-go/tools/cache/controller.go:181**

```go
func (c *controller) processLoop() {
   for {
      obj, err := c.config.Queue.Pop(PopProcessFunc(c.config.Process))
      if err != nil {//失败了再加
         if err == ErrFIFOClosed {
            return
         }
         if c.config.RetryOnError {
            c.config.Queue.AddIfNotPresent(obj)
         }
      }
   }
}
```

这里的逻辑是从 DeltaFIFO 中 Pop 出一个对象丢给 PopProcessFunc 处理，如果失败了就 re-enqueue 到 DeltaFIFO 中。我们前面提到过这里的 PopProcessFunc 实现是 `HandleDeltas()` 方法（在contronller初始化中提到的），所以这里的主要逻辑就转到了 `HandleDeltas()` 是如何实现的了。

### HandleDeltas()

这里我们先回顾下 DeltaFIFO 的存储结构，看下这个图：

![DeltaFIFO结构体](F:\docs\docs\images\DeltaFIFO结构体.png)

然后再看源码，这里的逻辑主要是遍历一个 Deltas 里的所有 Delta，然后根据 Delta 的类型来决定如何操作 Indexer，也就是更新本地 cache，同时分发相应的通知。

**client-go/tools/cache/shared_informer.go:537**

```go
func (s *sharedIndexInformer) HandleDeltas(obj interface{}) error {
   s.blockDeltas.Lock()
   defer s.blockDeltas.Unlock()
   // 对于每个 Deltas 来说，里面存了很多的 Delta，也就是对应不同 Type 的多个 Object，这里的遍历会从旧往新走
   for _, d := range obj.(Deltas) {
      switch d.Type {
      // 除了 Deleted 外所有情况
      case Sync, Replaced, Added, Updated:
         // 记录变更，没有太多实际作用
         s.cacheMutationDetector.AddObject(d.Object)
         // 通过 indexer 从 cache 里查询当前 Object，如果存在
         if old, exists, err := s.indexer.Get(d.Object); err == nil && exists {
            // 更新 indexer 里的对象
            if err := s.indexer.Update(d.Object); err != nil {
               return err
            }

            isSync := false
            switch {
            case d.Type == Sync:
               isSync = true
            case d.Type == Replaced:
               if accessor, err := meta.Accessor(d.Object); err == nil {
                  if oldAccessor, err := meta.Accessor(old); err == nil {
                     isSync = accessor.GetResourceVersion() == oldAccessor.GetResourceVersion()
                  }
               }
            }
            // 分发一个更新通知
            s.processor.distribute(updateNotification{oldObj: old, newObj: d.Object}, isSync)
           // 如果本地 cache 里没有这个 Object，则添加
         } else {
            if err := s.indexer.Add(d.Object); err != nil {
               return err
            }
            // 分发一个新增通知
            s.processor.distribute(addNotification{newObj: d.Object}, false)
         }
        // 如果是删除操作，则从 indexer 里删除这个 Object，然后分发一个删除通知
      case Deleted:
         if err := s.indexer.Delete(d.Object); err != nil {
            return err
         }
         s.processor.distribute(deleteNotification{oldObj: d.Object}, false)
      }
   }
   return nil
}
```

这里涉及到一个知识点：`s.processor.distribute(addNotification{newObj: d.Object}, false)` 中 processor 是什么？如何分发通知的？谁来接收通知？

我们回到 ProcessFunc 的实现上，除了 **sharedIndexInformer** 的 `HandleDeltas()` 方法外，还有一个基础版本：

- **client-go/tools/cache/controller.go:446**

```go
		Process: func(obj interface{}) error {
			for _, d := range obj.(Deltas) {
				obj := d.Object
				if transformer != nil {
					var err error
					obj, err = transformer(obj)
					if err != nil {
						return err
					}
				}

				switch d.Type {
				case Sync, Replaced, Added, Updated:
					if old, exists, err := clientState.Get(obj); err == nil && exists {
						if err := clientState.Update(obj); err != nil {
							return err
						}
						h.OnUpdate(old, obj)
					} else {
						if err := clientState.Add(obj); err != nil {
							return err
						}
						h.OnAdd(obj)
					}
				case Deleted:
					if err := clientState.Delete(obj); err != nil {
						return err
					}
					h.OnDelete(obj)
				}
			}
			return nil
		},
```

这里可以看到逻辑简单很多，除了更新 cache 外，调用了 `h.OnAdd(obj)/h.OnUpdate(old, obj)/h.OnDelete(obj)` 等方法，这里的 h 是 ResourceEventHandler 类型的，也就是 Process 过程直接调用了 ResourceEventHandler 的相应方法，这样就已经逻辑闭环了，ResourceEventHandler 的这几个方法里做一些简单的过滤后，会将这些对象的 key 丢到 workqueue，进而就触发了自定义调谐函数的运行。

换言之，sharedIndexInformer 中实现的 ProcessFunc 是一个进阶版本，不满足于简单调用 ResourceEventHandler 对应方法来完成 Process 逻辑，所以到这里基础的 Informer 逻辑已经闭环了，我们后面继续来看 sharedIndexInformer 中又对 Informer 做了哪些“增强”

## SharedIndexInformer

我们在 Operator 开发中，如果不使用 controller-runtime 库，也就是不通过 Kubebuilder 等工具来生成脚手架时，经常会用到 SharedInformerFactory，比如典型的 sample-controller 中的 main() 函数：

- **sample-controller/main.go:40**

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
		kubeInformerFactory.Apps().V1().Deployments(),//提供informer，会返回DeploymentInformer 类型
		exampleInformerFactory.Samplecontroller().V1alpha1().Foos())

	kubeInformerFactory.Start(stopCh)
	exampleInformerFactory.Start(stopCh)

	if err = controller.Run(2, stopCh); err != nil {
		klog.Fatalf("Error running controller: %s", err.Error())
	}
}
```

这里可以看到我们依赖于 `kubeInformerFactory.Apps().V1().Deployments() `提供一个 Informer，这里的 `Deployments()` 方法返回的是一个 **DeploymentInformer** 类型，DeploymentInformer 是什么呢？如下

- **client-go/informers/apps/v1/deployment.go:37**

```
type DeploymentInformer interface {
	Informer() cache.SharedIndexInformer
	Lister() v1.DeploymentLister
}
```

可以看到所谓的 DeploymentInformer 由 “Informer” 和 “Lister” 组成，也就是说我们编码时用到的 Informer 本质就是一个 **SharedIndexInformer**

- **client-go/tools/cache/shared_informer.go:186**

```
type SharedIndexInformer interface {
   SharedInformer
   AddIndexers(indexers Indexers) error
   GetIndexer() Indexer
}
```

这里的 Indexer 就很熟悉了，即DeploymentInformer->cache.SharedIndexInformer-> SharedInformer.那么SharedInformer 又是啥呢？

- **client-go/tools/cache/shared_informer.go:133**

```

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

接下来就该看下 **SharedIndexInformer** 接口的实现了，**sharedIndexerInformer** 定义如下：

- **client-go/tools/cache/shared_informer.go:287**

```go
type sharedIndexInformer struct {
   indexer    Indexer
   controller Controller
   processor             *sharedProcessor
   cacheMutationDetector MutationDetector
   listerWatcher ListerWatcher
   // 表示当前 Informer 期望关注的类型，主要是 GVK 信息
   objectType runtime.Object
   // reflector 的 resync 计时器计时间隔，通知所有的 listener 执行 resync
   resyncCheckPeriod time.Duration
   defaultEventHandlerResyncPeriod time.Duration
   clock clock.Clock
   started, stopped bool
   startedLock      sync.Mutex
   blockDeltas sync.Mutex
   watchErrorHandler WatchErrorHandler
}
```

这里的 Indexer、Controller、ListerWatcher 等都是我们熟悉的组件，**sharedProcessor** 我们在前面遇到了，需要重点关注一下。

### sharedProcessor

**sharedProcessor** 中维护了 processorListener 集合，然后分发通知对象到这些 listeners，先看下结构定义：

- **client-go/tools/cache/shared_informer.go:588**

```go
type sharedProcessor struct {
   listenersStarted bool
   listenersLock    sync.RWMutex
   listeners        []*processorListener
   syncingListeners []*processorListener
   clock            clock.Clock
   wg               wait.Group
}
```

马上就会有一个疑问了，**processorListener** 是什么？

#### processorListener

- **client-go/tools/cache/shared_informer.go:690**

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

- `add(notification interface{})`
- `pop()`
- `run()`

一个个来看吧。

**run()**

- **client-go/tools/cache/shared_informer.go:775**

```go
func (p *processorListener) run() {
   stopCh := make(chan struct{})
   wait.Until(func() {
      for next := range p.nextCh {
         switch notification := next.(type) {
         case updateNotification:
            p.handler.OnUpdate(notification.oldObj, notification.newObj)
         case addNotification:
            p.handler.OnAdd(notification.newObj)
         case deleteNotification:
            p.handler.OnDelete(notification.oldObj)
         default:
            utilruntime.HandleError(fmt.Errorf("unrecognized notification: %T", next))
         }
      }
      close(stopCh)
   }, 1*time.Second, stopCh)
}
```

这里的逻辑很清晰，从 nextCh 里拿通知，然后根据其类型去调用 **ResourceEventHandler** 相应的 `OnAdd/OnUpdate/OnDelete` 方法。

**add() 和 pop()**

- **client-go/tools/cache/shared_informer.go:741**

```go
func (p *processorListener) add(notification interface{}) {
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
         var ok bool
         // 从 pendingNotifications 里消费通知，生产者在下面 case 里
         notification, ok = p.pendingNotifications.ReadOne()
         if !ok {
            nextCh = nil
         }
        // 逻辑从这里开始，从 addCh 里提取通知
      case notificationToAdd, ok := <-p.addCh:
         if !ok {
            return
         }
         if notification == nil { 
            notification = notificationToAdd
            nextCh = p.nextCh
         } else {
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

**addListener** 会直接调用 **listener** 的 `run()` 和 `pop()` 方法，这两个方法的逻辑我们上面已经分析过

- **client-go/tools/cache/shared_informer.go:597**

```go

func (p *sharedProcessor) addListener(listener *processorListener) {
   p.listenersLock.Lock()
   defer p.listenersLock.Unlock()

   p.addListenerLocked(listener)
   if p.listenersStarted {
      p.wg.Start(listener.run)
      p.wg.Start(listener.pop)
   }
}
```

#### sharedProcessor.distribute()

**distribute** 的逻辑就是调用 **sharedProcessor** 内部维护的所有 listner 的 `add()` 方法

- **client-go/tools/cache/shared_informer.go:613**

```go
func (p *sharedProcessor) distribute(obj interface{}, sync bool) {
   p.listenersLock.RLock()
   defer p.listenersLock.RUnlock()

   if sync {
      for _, listener := range p.syncingListeners {
         listener.add(obj)
      }
   } else {
      for _, listener := range p.listeners {
         listener.add(obj)
      }
   }
}

```

#### sharedProcessor.run()

`run()` 的逻辑和前面的 addListener() 类似，也就是调用 **listener** 的 `run()` 和 `pop()` 方法

- **client-go/tools/cache/shared_informer.go:628**

```go
func (p *sharedProcessor) run(stopCh <-chan struct{}) {
	func() {
		p.listenersLock.RLock()
		defer p.listenersLock.RUnlock()
		for _, listener := range p.listeners {
			p.wg.Start(listener.run)
			p.wg.Start(listener.pop)
		}
		p.listenersStarted = true
	}()
	<-stopCh
	p.listenersLock.RLock()
	defer p.listenersLock.RUnlock()
	for _, listener := range p.listeners {
		close(listener.addCh)
	}
	p.wg.Wait()
}

```

到这里基本就知道 **sharedProcessor** 的能力了，继续往下看。

### sharedIndexInformer.Run()

继续来看 **sharedIndexInformer** 的 `Run()` 方法，这里面已经几乎没有陌生的内容了。

- **client-go/tools/cache/shared_informer.go:368**

```go
func (s *sharedIndexInformer) Run(stopCh <-chan struct{}) {
   defer utilruntime.HandleCrash()

   if s.HasStarted() {
      klog.Warningf("The sharedIndexInformer has started, run more than once is not allowed")
      return
   }
   // DeltaFIFO 就很熟悉了
   fifo := NewDeltaFIFOWithOptions(DeltaFIFOOptions{
      KnownObjects:          s.indexer,
      EmitDeltaTypeReplaced: true,
   })
   // Config 的逻辑也在上面遇到过了
   cfg := &Config{
      Queue:            fifo,
      ListerWatcher:    s.listerWatcher,
      ObjectType:       s.objectType,
      FullResyncPeriod: s.resyncCheckPeriod,
      RetryOnError:     false,
      ShouldResync:     s.processor.shouldResync,

      Process:           s.HandleDeltas,
      WatchErrorHandler: s.watchErrorHandler,
   }

   func() {
      s.startedLock.Lock()
      defer s.startedLock.Unlock()
      // 前文分析过这里的 New() 函数逻辑了
      s.controller = New(cfg)
      s.controller.(*controller).clock = s.clock
      s.started = true
   }()

   processorStopCh := make(chan struct{})
   var wg wait.Group
   defer wg.Wait()              
   defer close(processorStopCh) 
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

到这里也就基本知道了 sharedIndexInformer 的逻辑了，再往上层走就剩下一个 **SharedInformerFactory** 了，继续看吧～

## SharedInformerFactory

我们前面提到过 SharedInformerFactory，现在具体来看一下 SharedInformerFactory 是怎么实现的。先看接口定义：

- **client-go/informers/factory.go:187**

```go

type SharedInformerFactory interface {
   internalinterfaces.SharedInformerFactory
   ForResource(resource schema.GroupVersionResource) (GenericInformer, error)
   WaitForCacheSync(stopCh <-chan struct{}) map[reflect.Type]bool

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
   Scheduling() scheduling.Interface
   Storage() storage.Interface
}
```

这里涉及到几个点：

1. **internalinterfaces.SharedInformerFactory**

这也是一个接口，比较简短：

```go
type SharedInformerFactory interface {
   Start(stopCh <-chan struct{})
   InformerFor(obj runtime.Object, newFunc NewInformerFunc) cache.SharedIndexInformer
}

```

可以看到熟悉的 **SharedIndexInformer**

1. **ForResource(resource schema.GroupVersionResource) (GenericInformer, error)**

这里接收一个 GVR，返回了一个 **GenericInformer**，看下什么是 GenericInformer：

```go
type GenericInformer interface {
   Informer() cache.SharedIndexInformer
   Lister() cache.GenericLister
}
```

也很简短。

1. **Apps() apps.Interface 等**

后面一堆方法是类似的，我们以 Apps() 为例来看下怎么回事。这里的 Interface 定义如下：

- **client-go/informers/apps/interface.go:29**

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

显然应该继续看下 **v1.Interface** 是个啥。

- **client-go/informers/apps/v1/interface.go:26**

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

到这里已经有看着很眼熟的 `Deployments() DeploymentInformer` 之类的代码了，DeploymentInformer 我们刚才看过内部结构，长这样：

```go
type DeploymentInformer interface {
   Informer() cache.SharedIndexInformer
   Lister() v1.DeploymentLister
}

```

到这里也就不难理解 **SharedInformerFactory** 的作用了，它提供了所有 API group-version 的资源对应的 SharedIndexInformer，也就不难理解开头我们引用的 sample-controller 中的这行代码：

```
kubeInformerFactory.Apps().V1().Deployments()
```

通过其可以拿到一个 Deployment 资源对应的 SharedIndexInformer。

### NewSharedInformerFactory

继续看下 SharedInformerFactory 是如何创建的

- **client-go/informers/factory.go:96**

```go
func NewSharedInformerFactory(client kubernetes.Interface, defaultResync time.Duration) SharedInformerFactory {
   return NewSharedInformerFactoryWithOptions(client, defaultResync)
}
```

可以看到参数非常简单，主要是需要一个 Clientset，毕竟 ListerWatcher 的能力本质还是 client 提供的。

- **client-go/informers/factory.go:109**

```go

func NewSharedInformerFactoryWithOptions(client kubernetes.Interface, defaultResync time.Duration, options ...SharedInformerOption) SharedInformerFactory {
   factory := &sharedInformerFactory{
      client:           client,
      namespace:        v1.NamespaceAll, // 空字符串 ""
      defaultResync:    defaultResync,
      informers:        make(map[reflect.Type]cache.SharedIndexInformer), // 可以存放不同类型的 SharedIndexInformer
      startedInformers: make(map[reflect.Type]bool),
      customResync:     make(map[reflect.Type]time.Duration),
   }

   for _, opt := range options {
      factory = opt(factory)
   }

   return factory
}
```

接着是如何启动

### sharedInformerFactory.Start()

- **client-go/informers/factory.go:128**

```go
func (f *sharedInformerFactory) Start(stopCh <-chan struct{}) {
   f.lock.Lock()
   defer f.lock.Unlock()

   for informerType, informer := range f.informers {
      // 同类型只会调用一次，Run() 的逻辑我们前面介绍过了
      if !f.startedInformers[informerType] {
         go informer.Run(stopCh)
         f.startedInformers[informerType] = true
      }
   }
}
```

## 小结

今天我们一个基础 Informer - **Controller** 开始介绍，先分析了 Controller 的能力，也就是其通过构造 Reflector 并启动从而能够获取指定类型资源的“更新”事件，然后通过事件构造 Delta 放到 DeltaFIFO 中，进而在 **processLoop** 中从 DeltaFIFO 里 pop Deltas 来处理，一方面将对象通过 **Indexer** 同步到本地 cache，也就是一个 **ThreadSafeStore**，一方面调用 **ProcessFunc** 来处理这些 Delta。

然后 **SharedIndexInformer** 提供了构造 Controller 的能力，通过 **HandleDeltas()** 方法实现上面提到的 ProcessFunc，同时还引入了 **sharedProcessor** 在 HandleDeltas() 中用于事件通知的处理。sharedProcessor 分发事件通知的时候，接收方是内部继续抽象出来的 **processorListener**，在 processorListener 中完成了 **ResourceEventHandler** 具体回调函数的调用。







参考 

https://mp.weixin.qq.com/s?__biz=MzU4MjQ0MTU4Ng==&mid=2247485580&idx=1&sn=7392dbadff9ab450d93c5dd0449dace5&chksm=fdb90791cace8e871b1dcdd00be21f16a23504f48634f4fb7d3c87a552b3f8ce2765862fb4e9&scene=21#wechat_redirect

[https://www.danielhu.cn](https://www.danielhu.cn/)
