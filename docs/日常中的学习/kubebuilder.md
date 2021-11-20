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

`CronJobList` 简单的包含了多个 `CronJob`s. 它是批量操作中使用的Kind , 像 LIST。

一般来说，我们从不修改其中任何一个——所有的修改都在Spec或Status中进行。

注释`+kubebuilder:object:root`称为标记。我们将在后面看到更多，它告诉controller-tools（我们的代码以及YAML自动生成器）信息，controller-tool会自动生成。这个特殊的参数告诉`obeject`生成器这个类型代表一个Kind。然后，这个`object`生成器为我们 [runtime.Object](https://pkg.go.dev/k8s.io/apimachinery/pkg/runtime?tab=doc#Object) 的生成实现，这是所有代表Kinds的类型必须实现的标准接口。（这个就像长难句Then, the `object` generator generates an implementation of the [runtime.Object](https://pkg.go.dev/k8s.io/apimachinery/pkg/runtime?tab=doc#Object) interface for us, which is the standard interface that all types representing Kinds must implement.）

```go
//+kubebuilder:object:root=true
//+kubebuilder:subresource:status

// CronJob is the Schema for the cronjobs API
type CronJob struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   CronJobSpec   `json:"spec,omitempty"`
    Status CronJobStatus `json:"status,omitempty"`
}

//+kubebuilder:object:root=true

// CronJobList contains a list of CronJob
type CronJobList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`
    Items           []CronJob `json:"items"`
}


```

最后，我们将Go类型添加到API组。这允许我们将这个API组中的类型添加到任何Scheme中。

```go
func init() {
    SchemeBuilder.Register(&CronJob{}, &CronJobList{})
}
```

现在我们已经看到了基本结构，让我们来填充它!

#### 1.5[Designing an API](https://book.kubebuilder.io/cronjob-tutorial/api-design.html#designing-an-api)

在Kubernetes中，我们有一些设计api的规则。也就是说，所有序列化的字段必须是驼峰式的，因此我们使用JSON结构标记来指定这一点。我们还可以使用`omitempty`结构标记来标记一个字段在为空时应该从序列化字段中删除。

字段可以使用大多数基本类型。数字除外:为了API的兼容性，我们接受三种形式的数字:对于整数有`int32` 以及`int64`，对于`resource.Quantity`允许设置小数。

> Hold up, what's a Quantity?
>
> Quantities是十进制数字的一种特殊表示法，它具有显式固定的表示形式，使它们更易于跨机器移植。你可能已经在kubernetes中的requests与limits中见过了。他们在概念上类似于浮点数：they have a significant, base, and exponent.它们使用整数和后缀来形成序列化和人类可读的格式，就像我们描述计算机存储一样。
>
> 例如，2m在十进制中表示0.002。2Ki表示十进制2048,2K表示十进制2000。如果我们想要指定分数，我们可以切换到一个后缀，比如:2.5是2500m
>
> There are two supported bases: 10 and 2 (called decimal and binary, respectively). Decimal base is indicated with “normal” SI suffixes (e.g. `M` and `K`), while Binary base is specified in “mebi” notation (e.g. `Mi` and `Ki`). Think [megabytes vs mebibytes](https://en.wikipedia.org/wiki/Binary_prefix).

我们还使用了另一种特殊类型:`metav1.Time`。这个字段等价于 `time.Time`，除了它有一个固定的、可移植的序列化格式。

解决了这些问题之后，让我们看看CronJob对象是什么样子的!

```go
//Apache License
package v1
//Imports
import (
    batchv1beta1 "k8s.io/api/batch/v1beta1"
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// EDIT THIS FILE!  THIS IS SCAFFOLDING FOR YOU TO OWN!
// NOTE: json tags are required.  Any new fields you add must have json tags for the fields to be serialized.

```

首先，让我们看看我们的spec，正如我们之前所讨论的，spec持有期望的状态，所以controller的任何“输入”都在这里。

从根本上说，CronJob需要以下几点:

- A schedule (the *cron* in CronJob)
- A template for the Job to run (the *job* in CronJob)

我们还需要一些额外的功能，这将让我们的用户使用更轻松:

- A deadline for starting jobs (if we miss this deadline, we’ll just wait till the next scheduled time)停止时间
- What to do if multiple jobs would run at once (do we wait? stop the old one? run both?)多个任务
- A way to pause the running of a CronJob, in case something’s wrong with it暂停方面
- Limits on old job history对于老的job历史的限制

记住，因为我们从不读自己的status，我们需要另外的方法来跟踪job是否运行。我们至少可以用一个old job来做这个。

我们将使用几个标记(// +comment)来指定额外的元数据。当生成我们的CRD清单时，这些将被controller-tool使用。稍后我们将看到，controller-tool也将使用GoDoc来形成字段的描述。

```go

// CronJobSpec defines the desired state of CronJob
type CronJobSpec struct {
    //+kubebuilder:validation:MinLength=0

    // The schedule in Cron format, see https://en.wikipedia.org/wiki/Cron.
    Schedule string `json:"schedule"`

    //+kubebuilder:validation:Minimum=0

    // Optional deadline in seconds for starting the job if it misses scheduled
    // time for any reason.  Missed jobs executions will be counted as failed ones.
    // +optional
    StartingDeadlineSeconds *int64 `json:"startingDeadlineSeconds,omitempty"`

    // Specifies how to treat concurrent executions of a Job.
    // Valid values are:
    // - "Allow" (default): allows CronJobs to run concurrently;
    // - "Forbid": forbids concurrent runs, skipping next run if previous run hasn't finished yet;
    // - "Replace": cancels currently running job and replaces it with a new one
    // +optional
    ConcurrencyPolicy ConcurrencyPolicy `json:"concurrencyPolicy,omitempty"`

    // This flag tells the controller to suspend subsequent executions, it does
    // not apply to already started executions.  Defaults to false.
    // +optional
    Suspend *bool `json:"suspend,omitempty"`

    // Specifies the job that will be created when executing a CronJob.
    JobTemplate batchv1beta1.JobTemplateSpec `json:"jobTemplate"`

    //+kubebuilder:validation:Minimum=0

    // The number of successful finished jobs to retain.
    // This is a pointer to distinguish between explicit zero and not specified.
    // +optional
    SuccessfulJobsHistoryLimit *int32 `json:"successfulJobsHistoryLimit,omitempty"`

    //+kubebuilder:validation:Minimum=0

    // The number of failed finished jobs to retain.
    // This is a pointer to distinguish between explicit zero and not specified.
    // +optional
    FailedJobsHistoryLimit *int32 `json:"failedJobsHistoryLimit,omitempty"`
}
```

我们定义一个自定义类型来保存并发策略。它实际上只是一个字符串，但类型提供了额外的文档，并允许我们将验证附加到类型上，而不是字段上，使验证更容易重用。

```go

// ConcurrencyPolicy describes how the job will be handled.
// Only one of the following concurrent policies may be specified.
// If none of the following policies is specified, the default one
// is AllowConcurrent.
// +kubebuilder:validation:Enum=Allow;Forbid;Replace
type ConcurrencyPolicy string

const (
    // AllowConcurrent allows CronJobs to run concurrently.
    AllowConcurrent ConcurrencyPolicy = "Allow"

    // ForbidConcurrent forbids concurrent runs, skipping next run if previous
    // hasn't finished yet.
    ForbidConcurrent ConcurrencyPolicy = "Forbid"

    // ReplaceConcurrent cancels currently running job and replaces it with a new one.
    ReplaceConcurrent ConcurrencyPolicy = "Replace"
)

```

接下来，让我们设计status，它保存观察到的状态。它包含我们希望用户或其他controller能够轻松获得的任何信息。

我们将保留一个积极运行jobs的列表，以及最后一次成功运行job的记录。注意，就像上面提到的，我们使用`metav1.Time`而不是`time.Time`

```go

// CronJobStatus defines the observed state of CronJob
type CronJobStatus struct {
    // INSERT ADDITIONAL STATUS FIELD - define observed state of cluster
    // Important: Run "make" to regenerate code after modifying this file

    // A list of pointers to currently running jobs.
    // +optional
    Active []corev1.ObjectReference `json:"active,omitempty"`

    // Information when was the last time the job was successfully scheduled.
    // +optional
    LastScheduleTime *metav1.Time `json:"lastScheduleTime,omitempty"`
}
```

最后，我们有我们已经讨论过的其余样板。如前所述，除了subresource status标记外，我们不需要更改它，这样我们的行为就像内置的kubernetes类型。

```go
//+kubebuilder:object:root=true
//+kubebuilder:subresource:status

// CronJob is the Schema for the cronjobs API
type CronJob struct {
//Root Object Definitions
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   CronJobSpec   `json:"spec,omitempty"`
    Status CronJobStatus `json:"status,omitempty"`
}

//+kubebuilder:object:root=true

// CronJobList contains a list of CronJob
type CronJobList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`
    Items           []CronJob `json:"items"`
}

func init() {
    SchemeBuilder.Register(&CronJob{}, &CronJobList{})
}
```

现在我们有了一个API，我们需要编写一个controller来实际实现这个功能

#### 1.5.1[A Brief Aside: What’s the rest of this stuff?](https://book.kubebuilder.io/cronjob-tutorial/other-api-files.html#a-brief-aside-whats-the-rest-of-this-stuff)

旁白，剩下的东西是什么

如果您查看了api/v1/目录中的其他文件，您可能会注意到除了cronjob_types之外还有两个文件`groupversion_info.go` 和`zz_generated.deepcopy.go`.

这两个文件都不需要编辑(前一个保持不变，后一个是自动生成的)，但是知道其中的内容是很有用的。

`groupversion_info.go`包含了关于group-version中共同的元数据。

首先，我们有一些包级别的标记，它们表示这个包中有Kubernetes对象，这个包代表的是group`batch.tutorial.kubebuilder.io`。`object`生成器利用前者，而后者由CRD生成器用于为它从这个包中创建的CRD生成正确的元数据。

```go
// Package v1 contains API Schema definitions for the batch v1 API group
//+kubebuilder:object:generate=true
//+groupName=batch.tutorial.kubebuilder.io
package v1

import (
    "k8s.io/apimachinery/pkg/runtime/schema"
    "sigs.k8s.io/controller-runtime/pkg/scheme"
)
```

然后，我们有了帮助我们设置Scheme的常用变量。因为我们需要在controller中使用这个包中的所有类型，所以使用一个方便的方法将所有类型添加到其他Scheme中是很有帮助的(也是约定的)。SchemeBuilder使我们很容易做到这一点。

```go
var (
    // GroupVersion is group version used to register these objects
    GroupVersion = schema.GroupVersion{Group: "batch.tutorial.kubebuilder.io", Version: "v1"}

    // SchemeBuilder is used to add go types to the GroupVersionKind scheme
    SchemeBuilder = &scheme.Builder{GroupVersion: GroupVersion}

    // AddToScheme adds the types in this group-version to the given scheme.
    AddToScheme = SchemeBuilder.AddToScheme
)
```

#### [`zz_generated.deepcopy.go`](https://book.kubebuilder.io/cronjob-tutorial/other-api-files.html#zz_generateddeepcopygo)

`zz_generated.deepcopy.go` contains the autogenerated implementation of the aforementioned `runtime.Object` interface, which marks all of our root types as representing Kinds.

The core of the `runtime.Object` interface is a deep-copy method, `DeepCopyObject`.

The `object` generator in controller-tools also generates two other handy methods for each root type and all its sub-types: `DeepCopy` and `DeepCopyInto`.

简言之，就是自动生成的。

#### [What’s in a controller?](https://book.kubebuilder.io/cronjob-tutorial/controller-overview.html#whats-in-a-controller)

Controllers是Kubernetes以及任何operator中的核心。

Controllers的工作是确保任何给定对象的实际状态(包括集群状态和潜在的外部状态，如Kubelet的运行容器或云提供商的负载平衡器)与对象中的期望状态相匹配。每个控制器专注于一个root类型，但可能与其他类型交互。

我们把这个过程叫做 *reconciling*。

在Controllers运行时中，为特定类型实现协调的逻辑称为[*Reconciler*](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/reconcile?tab=doc)。Reconciler获取对象的名称，并返回是否需要再次尝试(例如，在出现错误或定期控制器的情况下，如HorizontalPodAutoscaler)。

首先，我们从一些标准导入开始。和以前一样，我们需要核心控制器运行时库、client包和API类型包。

```go

package controllers

import (
    "context"

    "k8s.io/apimachinery/pkg/runtime"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/log"

    batchv1 "tutorial.kubebuilder.io/project/api/v1"
)
```

接下来，kubebuilder为我们搭建了一个基本的reconciler结构。几乎每个reconciler都需要记录日志，并且需要能够获取对象，所以这些都是一开始时自动添加的。

```go

// CronJobReconciler reconciles a CronJob object
type CronJobReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}
```

大多数controllers最终会运行在集群上，因此它们需要RBAC权限，我们使用控制器工具RBAC标记来指定权限。这些是运行所需的最低权限。随着我们添加更多的功能，我们需要重新访问这些功能。

```go
// +kubebuilder:rbac:groups=batch.tutorial.kubebuilder.io,resources=cronjobs,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=batch.tutorial.kubebuilder.io,resources=cronjobs/status,verbs=get;update;patch
```

实际上，Reconcile对单个命名对象执行协调。我们的Request只有一个名称，但是我们可以使用客户端从缓存中获取该对象。

我们返回一个空的结果并且没有错误，这向控制器运行时表明我们已经成功地协调了这个对象，并且在发生一些更改之前不需要再次尝试。

大多数controllers 需要 一个logging handle 以及 a context, 所以我们在这里设置它。

上下文用于允许取消请求，以及可能进行的跟踪等操作。它是所有客户端方法的第一个参数。Background上下文只是一个基本的上下文，没有任何额外的数据或时间限制。

logging handle让我们记录日志。控制器运行时通过一个名为logr的库使用结构化日志记录。我们很快就会看到，日志是通过将键值对附加到静态消息来工作的。我们可以在调和方法的顶部预先分配一些对，以便将这些键值对附加到这个调和器中的所有日志线上。

```go
func (r *CronJobReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    _ = log.FromContext(ctx)

    // your logic here

    return ctrl.Result{}, nil
}

```

最后，我们将这个协调器添加到管理器中，以便在管理器启动时启动协调器。

现在，我们只注意到这个协调器对CronJobs起作用。稍后，我们将使用它来标记我们关心的相关对象。

```go

func (r *CronJobReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&batchv1.CronJob{}).
        Complete(r)
}

```

现在我们已经看到了协调器的基本结构，让我们来填写CronJobs的逻辑。

#### [Implementing a controller](https://book.kubebuilder.io/cronjob-tutorial/controller-implementation.html#implementing-a-controller)

我们的 CronJob controller的具体逻辑是这样的:

1. Load the named CronJob加载命名的CronJob
2. List all active jobs, and update the status列出所有运行的jobs，更新status
3. Clean up old jobs according to the history limits通过历史限制清扫老的jobs
4. Check if we’re suspended (and don’t do anything else if we are)检查我们是否被暂停(如果被暂停，不要做其他任何事情)
5. Get the next scheduled run获得下一次计划运行
6. Run a new job if it’s on schedule, not past the deadline, and not blocked by our concurrency policy如果新作业按计划运行，且未超过截止日期，且未被并发策略阻塞，则运行它
7. Requeue when we either see a running job (done automatically) or it’s time for the next scheduled run.当我们看到一个正在运行的作业(自动完成)或者是下一次计划运行的时间时，就会调用Requeue。

