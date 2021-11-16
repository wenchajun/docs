那么，我们首先从main.go开始看

##### [Every journey needs a start, every program needs a main](https://book.kubebuilder.io/cronjob-tutorial/empty-main.html#every-journey-needs-a-start-every-program-needs-a-main)

先看导入的包

```go

package main

import (
    "flag"
    "fmt"
    "os"

    // Import all Kubernetes client auth plugins (e.g. Azure, GCP, OIDC, etc.)
    // to ensure that exec-entrypoint and run can make use of them.
    _ "k8s.io/client-go/plugin/pkg/client/auth"

    "k8s.io/apimachinery/pkg/runtime"
    utilruntime "k8s.io/apimachinery/pkg/util/runtime"
    clientgoscheme "k8s.io/client-go/kubernetes/scheme"
    _ "k8s.io/client-go/plugin/pkg/client/auth/gcp"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/cache"
    "sigs.k8s.io/controller-runtime/pkg/healthz"
    "sigs.k8s.io/controller-runtime/pkg/log/zap"
    // +kubebuilder:scaffold:imports
)
```

每一组controller都需要一个Scheme, Scheme提供了types和它们对应的Go类型之间的映射。在编写API定义时，我们将更多地讨论类型，所以请记住这一点

```go
var (
    scheme   = runtime.NewScheme()
    setupLog = ctrl.Log.WithName("setup")
)

func init() {
    utilruntime.Must(clientgoscheme.AddToScheme(scheme))
//注意在这里记得add相应的scheme
    //+kubebuilder:scaffold:scheme
}
```

此时，我们的主要功能相当简单：

- 我们为metrics设置了一些基本的指标（flag）。

- 我们实例化了一个manager，它跟踪我们所有controller的运行情况，并为API服务器设置共享缓存和客户端(注意，我们需要告诉manager关于我们的Scheme信息)。

- 我们运行我们的manager，而manager又运行我们所有的controller以及webhooks。manager被设置为运行，直到它收到一个优雅的停止信号。这样，当我们在Kubernetes上运行时，我们就可以优雅地终止pod。

虽然我们还没有任何东西可以运行，但请记住`+kubebuilder:scaffold:builder`注释的位置——事情很快就会变得有趣起来。

```go
func main() {
    var metricsAddr string
    var enableLeaderElection bool
    var probeAddr string
    flag.StringVar(&metricsAddr, "metrics-bind-address", ":8080", "The address the metric endpoint binds to.")
    flag.StringVar(&probeAddr, "health-probe-bind-address", ":8081", "The address the probe endpoint binds to.")
    flag.BoolVar(&enableLeaderElection, "leader-elect", false,
        "Enable leader election for controller manager. "+
            "Enabling this will ensure there is only one active controller manager.")
    opts := zap.Options{
        Development: true,
    }
    opts.BindFlags(flag.CommandLine)
    flag.Parse()

    ctrl.SetLogger(zap.New(zap.UseFlagOptions(&opts)))

    mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
        Scheme:                 scheme,
        MetricsBindAddress:     metricsAddr,
        Port:                   9443,
        HealthProbeBindAddress: probeAddr,
        LeaderElection:         enableLeaderElection,
        LeaderElectionID:       "80807133.tutorial.kubebuilder.io",
    })
    if err != nil {
        setupLog.Error(err, "unable to start manager")
        os.Exit(1)
    }
```

注意，Manager可以通过以下方式限制所有controller监视资源的namespace（Note that the Manager can restrict the namespace that all controllers will watch for resources by）:

```go
    mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
        Scheme:                 scheme,
        Namespace:              namespace,
        MetricsBindAddress:     metricsAddr,
        Port:                   9443,
        HealthProbeBindAddress: probeAddr,
        LeaderElection:         enableLeaderElection,
        LeaderElectionID:       "80807133.tutorial.kubebuilder.io",
    })

```

上面的示例将把项目的范围更改为单个名称空间。在这个场景中，还建议通过将默认的ClusterRole和ClusterRoleBinding分别替换为Role和RoleBinding，将提供的授权限制到这个名称空间。有关更多信息，请参阅kubernetes关于使用[RBAC授权](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)的文档。

同样，也可以使用MultiNamespacedCacheBuilder来监视一组特定的名称空间:

```go
   var namespaces []string // List of Namespaces

    mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
        Scheme:                 scheme,
        NewCache:               cache.MultiNamespacedCacheBuilder(namespaces),
        MetricsBindAddress:     fmt.Sprintf("%s:%d", metricsHost, metricsPort),
        Port:                   9443,
        HealthProbeBindAddress: probeAddr,
        LeaderElection:         enableLeaderElection,
        LeaderElectionID:       "80807133.tutorial.kubebuilder.io",
    })
```

For further information see [MultiNamespacedCacheBuilder](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/cache?tab=doc#MultiNamespacedCacheBuilder)

```go
  // +kubebuilder:scaffold:builder

    if err := mgr.AddHealthzCheck("healthz", healthz.Ping); err != nil {
        setupLog.Error(err, "unable to set up health check")
        os.Exit(1)
    }
    if err := mgr.AddReadyzCheck("readyz", healthz.Ping); err != nil {
        setupLog.Error(err, "unable to set up ready check")
        os.Exit(1)
    }

    setupLog.Info("starting manager")
    if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil {
        setupLog.Error(err, "problem running manager")
        os.Exit(1)
    }
}
```

这样，我们就可以搭建我们的API了!

#### 1.3[Groups and Versions and Kinds](https://book.kubebuilder.io/cronjob-tutorial/gvks.html#groups-and-versions-and-kinds-oh-my)

实际上，在开始我们的API之前，我们应该讨论一些术语。

当我们谈论Kubernetes中的api时，我们经常使用4个术语: *groups*, *versions*, *kinds*, and *resources*。

#### [Groups and Versions](https://book.kubebuilder.io/cronjob-tutorial/gvks.html#groups-and-versions)

Kubernetes中的*API Group*只是相关功能的集合。每个group都有一个或多个 *versions*，顾名思义，这些 *versions*允许我们随时间改变API的工作方式。

#### [Kinds and Resources](https://book.kubebuilder.io/cronjob-tutorial/gvks.html#kinds-and-resources)

每一个API group-version包含一个或者多个API种类，我们叫做*Kinds*。虽然Kind可以在不同versions之间更改表单，但每个表单必须能够以某种方式存储其他表单的所有数据(我们可以将数据存储在字段或注释中)。这意味着使用较旧的API版本不会导致新数据丢失或损坏。有关更多信息，请参阅[Kubernetes API指南](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md)。

你也会偶尔听到*resources*。在API中，一个resource只是Kind的简单使用。通常，Kinds和resources之间是一对一的映射。举个例子，`Pod`的resource对应于`Pod`的Kind。然而，有时同一Kind可以返回多种resource。举个例子，`Scale`类型的Kind返回所有scale的子资源，比如`deployments/scale` 或者 `replicasets/scale`。这就是Kubernetes中的HorizontalPodAutoscaler可以与不同资源交互的原因。然而，如果使用crd，每个Kind将与单个资源相对应。

#### [So, how does that correspond to Go?](https://book.kubebuilder.io/cronjob-tutorial/gvks.html#so-how-does-that-correspond-to-go)

当我们引用特定组版本中的一种类型时，我们将其称为*GroupVersionKind*，或简称GVK。对于resource和GVR也是如此。我们很快就会看到，每个GVK对应于包中给定的根Go类型（指each GVK corresponds to a given root Go type in a package）。

现在我们已经有了明确的术语，我们可以创建我们的API了!

#### [So, how can we create our API?](https://book.kubebuilder.io/cronjob-tutorial/gvks.html#so-how-can-we-create-our-api)

在下一节中，我们将使用kubebuilder create API命令创建我们自己的API。这个命令的目标是为我们的Kind(s)创建自定义资源(CR)和自定义资源定义(CRD)。详见 [Extend the Kubernetes API with CustomResourceDefinitions](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)。

#### [But, why create APIs at all?](https://book.kubebuilder.io/cronjob-tutorial/gvks.html#but-why-create-apis-at-all)

新的API是我们向kubernetes传递我们自定义对象的方式。Go的结构体被用于生成CRD，其中包括我们的数据的schema（计划？）以及跟踪数据，比如我们的新类型被称为什么。然后，我们可以创建自定义对象的实例，这些实例将由controller管理。

我们的api和resource代表我们在集群上的解决方案。基本上，crd是我们定制对象的定义，而CRs是它的实例。

#### [Ah, do you have an example?](https://book.kubebuilder.io/cronjob-tutorial/gvks.html#ah-do-you-have-an-example)

让我们考虑一个经典场景，其目标是让应用程序及其数据库在Kubernetes平台上运行。然后，一个CRD可以代表App，另一个CRD可以代表DB。通过使用一个CRD来描述应用程序，而另一个CRD用于数据库，我们不会损害封装、单一责任原则和内聚等概念。破坏这些概念可能会导致意想不到的副作用，比如难以扩展、重用或维护，这只是这些副作用中的一小部分。

通过这种方式，我们可以创建App CRD，它的operator负责App的部署以及创建访问它的服务等等。类似地，我们可以创建一个CRD来运行DB，并部署一个controller来管理DB实例。

#### [Err, but what’s that Scheme thing?](https://book.kubebuilder.io/cronjob-tutorial/gvks.html#err-but-whats-that-scheme-thing)

我们前面看到的`Scheme`只是一种跟踪Go类型与给定GVK来对应的方法。举个栗子，假设我们标记`“tutorial.kubebuilder.io/api/v1”.CronJob{}`类型在`batch.tutorial.kubebuilder.io/v1` API组中(这里就表示有Kind `CronJob`)。

然后，我们稍后构造一个新的`&CronJob{}`，给出一些由API服务器返回的JSON

```json
{
    "kind": "CronJob",
    "apiVersion": "batch.tutorial.kubebuilder.io/v1",
    ...
}
```

或者当我们在更新中提交一个`&CronJob{}`时，能正确地查找group-version

#### 1.4[Adding a new API](https://book.kubebuilder.io/cronjob-tutorial/new-api.html#adding-a-new-api)

要构建出一个新的类(你注意到了上一章，对吧?)和相应的controller，我们可以使用kubebuilder创建api:

```
kubebuilder create api --group batch --version v1 --kind CronJob
```

按 `y` f “Create Resource” 以及 “Create Controller”.

当我们第一次为每个组版本调用这个命令时，它将为新的组版本创建一个目录。

> note 支持老版本
>
> 这里我不怎么想翻译，主要是因为kubebuilderv2.x与v3.x支持的api版本变了，v3.x支持v1。你要调整一下配置
>
> The default CustomResourceDefinition manifests created alongside your Go API types use API version `v1`. If your project intends to support Kubernetes cluster versions older than v1.16, you must set `--crd-version v1beta1` and remove `preserveUnknownFields=false` from the `CRD_OPTIONS` Makefile variable. See the [CustomResourceDefinition generation reference](https://book.kubebuilder.io/reference/generating-crd.html#supporting-older-cluster-versions) for details.

在本例中，`api/v1/`目录已经被创建，与`batch.tutorial.kubebuilder.io/v1`相对应（你还记得我们最初的`--domain`这个设置吗）。

为了`CornJob`Kind，`api/v1/cronjob_types.go`文件已经被添加。每当我们执行不同Kind的命令时，他会添加对于的新文件。

让我们看看文件里都有什么，然后我们可以继续填充它。

我们从足够简单的开始:我们导入`meta/v1` API组，它通常不是公开的，包含了Kubernetes Kinds中的共同的元数据（metadata）。

```GO
package v1

import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

```

接下来为我们的Kind中的Spec与Status定义类型。Kubernetes通过reconciling(协调)所期望的状态(Spec)与实际集群状态(其他对象的`Status`)和外部状态，然后记录它观察到的状态(Status)来实现功能。//其实就是list and watch吧

然后, 每一个 *functional* object 包含 spec 和 status.但是对于一些类型，比如`configmap`，不遵守这个约束，因为他们没有编码所需的状态，但是大多数类型都是这样的。

```go

// EDIT THIS FILE!  THIS IS SCAFFOLDING FOR YOU TO OWN!
// NOTE: json tags are required.  Any new fields you add must have json tags for the fields to be serialized.

// CronJobSpec defines the desired state of CronJob
type CronJobSpec struct {
    // INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
    // Important: Run "make" to regenerate code after modifying this file
}

// CronJobStatus defines the observed state of CronJob
type CronJobStatus struct {
    // INSERT ADDITIONAL STATUS FIELD - define observed state of cluster
    // Important: Run "make" to regenerate code after modifying this file
}

```

接下来，定义与实际类型对应的类型，`CronJob` 和`CronJobList`.`CronJob` 是我们的根类型, 并且描述 `CronJob` kind.与所有的kubernetes对象一样，它包含`TypeMeta`（描述API version以及Kind），并且包含了`ObjectMeta`，里面保存了name、namespace、label等信息。

