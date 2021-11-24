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

我们将从一些import开始。你会看到下面，我们将需要更多的导入比那些脚手架为我们。我们会在使用时逐一讨论。

```go
package controllers

import (
    "context"
    "fmt"
    "sort"
    "time"

    "github.com/robfig/cron"
    kbatch "k8s.io/api/batch/v1"
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
    ref "k8s.io/client-go/tools/reference"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/log"

    batchv1 "tutorial.kubebuilder.io/project/api/v1"
)
```

接下来，我们需要一个Clock，它将允许我们在测试中伪造计时。

```go

// CronJobReconciler reconciles a CronJob object
type CronJobReconciler struct {
    client.Client
    Scheme *runtime.Scheme
    Clock
}
//Clock
```



我们将模拟时钟，使它更容易在测试时跳过时间，“真正的”时钟只是调用`time.Now`

```go
type realClock struct{}

func (_ realClock) Now() time.Time { return time.Now() }

// clock knows how to get the current time.
// It can be used to fake out timing for testing.
type Clock interface {
    Now() time.Time
}
```

注意，我们还需要一些RBAC权限——因为我们现在正在创建和管理作业，所以需要这些权限，这意味着添加更多的标记

```go
//+kubebuilder:rbac:groups=batch.tutorial.kubebuilder.io,resources=cronjobs,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=batch.tutorial.kubebuilder.io,resources=cronjobs/status,verbs=get;update;patch
//+kubebuilder:rbac:groups=batch.tutorial.kubebuilder.io,resources=cronjobs/finalizers,verbs=update
//+kubebuilder:rbac:groups=batch,resources=jobs,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=batch,resources=jobs/status,verbs=get
```

现在，我们来看看controller的核心——reconciler逻辑。

```go
var (
    scheduledTimeAnnotation = "batch.tutorial.kubebuilder.io/scheduled-at"
)

func (r *CronJobReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := log.FromContext(ctx)

```

##### [1: Load the CronJob by name](https://book.kubebuilder.io/cronjob-tutorial/controller-implementation.html#1-load-the-cronjob-by-name)

我们将使用我们的client来获取CronJob。所有client方法都将使用context(允许取消)作为其第一个参数，将所讨论的对象作为其最后一个参数。Get有点特殊，因为它接受NamespacedName作为中间参数(正如我们将在下面看到的，大多数都没有中间参数)。

许多client方法最后也采用可变选项。

```go
 var cronJob batchv1.CronJob
    if err := r.Get(ctx, req.NamespacedName, &cronJob); err != nil {
        log.Error(err, "unable to fetch CronJob")
        // we'll ignore not-found errors, since they can't be fixed by an immediate
        // requeue (we'll need to wait for a new notification), and we can get them
        // on deleted requests.
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
```

##### [2: List all active jobs, and update the status](https://book.kubebuilder.io/cronjob-tutorial/controller-implementation.html#2-list-all-active-jobs-and-update-the-status)

为了完全更新我们的status，我们需要列出这个namespace中属于这个CronJob的所有子job。与Get类似，我们可以使用List方法来列出子job。注意，我们使用可变参数选项来设置名称空间和字段匹配(这实际上是我们在下面设置的索引查找)。

```go
   var childJobs kbatch.JobList
    if err := r.List(ctx, &childJobs, client.InNamespace(req.Namespace), client.MatchingFields{jobOwnerKey: req.Name}); err != nil {
        log.Error(err, "unable to list child Jobs")
        return ctrl.Result{}, err
    }

```

> 这个index是什么？
>
> reconciler获取拥有该状态下的cronjob的所有作业。随着cronjobs的增加，查找这些job的速度会变得相当缓慢，因为我们必须过滤所有这些工作。为了更高效地查找，这些作业将在控制器名称上进行本地索引。缓存的作业对象中增加了一个jobOwnerKey字段。这个键引用归属控制器并作为索引。在本文档的后面部分，我们将配置管理器来实际索引这个字段。

一旦我们拥有了所有jobs，我们将它们划分为active、successful和failed的作业，并跟踪最近的运行情况，以便在状态中记录它。请记住，状态应该能够从原来的状态中重新构建，所以从根对象的状态中读取通常不是一个好主意。相反，您应该在每次运行时重新构建它。这就是我们要做的。

我们可以使用状态条件检查一个作业是否“finished”，以及它是成功还是失败。我们将把这个逻辑放到一个helper中，以使我们的代码更清晰。

```go
    // find the active list of jobs
    var activeJobs []*kbatch.Job
    var successfulJobs []*kbatch.Job
    var failedJobs []*kbatch.Job
    var mostRecentTime *time.Time // find the last run so we can update the status

    for i, job := range childJobs.Items {
        _, finishedType := isJobFinished(&job)
        switch finishedType {
        case "": // ongoing
            activeJobs = append(activeJobs, &childJobs.Items[i])
        case kbatch.JobFailed:
            failedJobs = append(failedJobs, &childJobs.Items[i])
        case kbatch.JobComplete:
            successfulJobs = append(successfulJobs, &childJobs.Items[i])
        }

        // We'll store the launch time in an annotation, so we'll reconstitute that from
        // the active jobs themselves.
        scheduledTimeForJob, err := getScheduledTimeForJob(&job)
        if err != nil {
            log.Error(err, "unable to parse schedule time for child job", "job", &job)
            continue
        }
        if scheduledTimeForJob != nil {
            if mostRecentTime == nil {
                mostRecentTime = scheduledTimeForJob
            } else if mostRecentTime.Before(*scheduledTimeForJob) {
                mostRecentTime = scheduledTimeForJob
            }
        }
    }

    if mostRecentTime != nil {
        cronJob.Status.LastScheduleTime = &metav1.Time{Time: *mostRecentTime}
    } else {
        cronJob.Status.LastScheduleTime = nil
    }
    cronJob.Status.Active = nil
    for _, activeJob := range activeJobs {
        jobRef, err := ref.GetReference(r.Scheme, activeJob)
        if err != nil {
            log.Error(err, "unable to make reference to active job", "job", activeJob)
            continue
        }
        cronJob.Status.Active = append(cronJob.Status.Active, *jobRef)
    }
```

这里，我们将记录在稍高一些的日志级别上观察到的作业数量，以便进行调试。注意，我们没有使用格式字符串，而是使用固定的消息，并将键值对与额外的信息附加在一起。这使得过滤和查询日志行变得更加容易。

```go
 log.V(1).Info("job count", "active jobs", len(activeJobs), "successful jobs", len(successfulJobs), "failed jobs", len(failedJobs))
```

使用我们收集的日期，我们将更新我们的CRD的状态。和以前一样，我们利用clinet。为了具体地更新status子资源，我们将使用client的status部分和update方法。

status子资源忽略对spec的更改，因此它不太可能与任何其他更新冲突，并且可以拥有单独的权限。

```go
if err := r.Status().Update(ctx, &cronJob); err != nil {
        log.Error(err, "unable to update CronJob status")
        return ctrl.Result{}, err
    }
```

#### [3: Clean up old jobs according to the history limit](https://book.kubebuilder.io/cronjob-tutorial/controller-implementation.html#3-clean-up-old-jobs-according-to-the-history-limit)

首先，我们要努力清理旧的jobs，这样我们就不会留下太多的垃圾

```go
  // NB: deleting these is "best effort" -- if we fail on a particular one,
    // we won't requeue just to finish the deleting.
    if cronJob.Spec.FailedJobsHistoryLimit != nil {
        sort.Slice(failedJobs, func(i, j int) bool {
            if failedJobs[i].Status.StartTime == nil {
                return failedJobs[j].Status.StartTime != nil
            }
            return failedJobs[i].Status.StartTime.Before(failedJobs[j].Status.StartTime)
        })
        for i, job := range failedJobs {
            if int32(i) >= int32(len(failedJobs))-*cronJob.Spec.FailedJobsHistoryLimit {
                break
            }
            if err := r.Delete(ctx, job, client.PropagationPolicy(metav1.DeletePropagationBackground)); client.IgnoreNotFound(err) != nil {
                log.Error(err, "unable to delete old failed job", "job", job)
            } else {
                log.V(0).Info("deleted old failed job", "job", job)
            }
        }
    }

    if cronJob.Spec.SuccessfulJobsHistoryLimit != nil {
        sort.Slice(successfulJobs, func(i, j int) bool {
            if successfulJobs[i].Status.StartTime == nil {
                return successfulJobs[j].Status.StartTime != nil
            }
            return successfulJobs[i].Status.StartTime.Before(successfulJobs[j].Status.StartTime)
        })
        for i, job := range successfulJobs {
            if int32(i) >= int32(len(successfulJobs))-*cronJob.Spec.SuccessfulJobsHistoryLimit {
                break
            }
            if err := r.Delete(ctx, job, client.PropagationPolicy(metav1.DeletePropagationBackground)); (err) != nil {
                log.Error(err, "unable to delete old successful job", "job", job)
            } else {
                log.V(0).Info("deleted old successful job", "job", job)
            }
        }
    }

```

#### [4: Check if we’re suspended](https://book.kubebuilder.io/cronjob-tutorial/controller-implementation.html#4-check-if-were-suspended)

如果这个对象被挂起，我们就不想运行任何job，所以立刻停止。如果正在运行的job出现故障，并且我们希望暂停运行以调查或处理集群，而不删除对象，这将非常有用。

```go
if cronJob.Spec.Suspend != nil && *cronJob.Spec.Suspend {
        log.V(1).Info("cronjob suspended, skipping")
        return ctrl.Result{}, nil
    }
```

#### [5: Get the next scheduled run](https://book.kubebuilder.io/cronjob-tutorial/controller-implementation.html#5-get-the-next-scheduled-run)

如果我们没有暂停，我们将需要计算下一次预定的运行，以及我们是否有一个尚未处理的运行。

```go
// figure out the next times that we need to create
    // jobs at (or anything we missed).
    missedRun, nextRun, err := getNextSchedule(&cronJob, r.Now())
    if err != nil {
        log.Error(err, "unable to figure out CronJob schedule")
        // we don't really care about requeuing until we get an update that
        // fixes the schedule, so don't return an error
        return ctrl.Result{}, nil
    }
```

我们将准备最终的请求，以便在下一次job之前重新请求，然后确定是否确实需要运行。

```go
scheduledResult := ctrl.Result{RequeueAfter: nextRun.Sub(r.Now())} // save this so we can re-use it elsewhere
    log = log.WithValues("now", r.Now(), "next run", nextRun)
```

#### [6: Run a new job if it’s on schedule, not past the deadline, and not blocked by our concurrency policy](https://book.kubebuilder.io/cronjob-tutorial/controller-implementation.html#6-run-a-new-job-if-its-on-schedule-not-past-the-deadline-and-not-blocked-by-our-concurrency-policy)

如果我们错过了一次运行，而我们仍然在启动它的最后期限内，我们将需要运行一个job。

```go
  if missedRun.IsZero() {
        log.V(1).Info("no upcoming scheduled times, sleeping until next")
        return scheduledResult, nil
    }

    // make sure we're not too late to start the run
    log = log.WithValues("current run", missedRun)
    tooLate := false
    if cronJob.Spec.StartingDeadlineSeconds != nil {
        tooLate = missedRun.Add(time.Duration(*cronJob.Spec.StartingDeadlineSeconds) * time.Second).Before(r.Now())
    }
    if tooLate {
        log.V(1).Info("missed starting deadline for last run, sleeping till next")
        // TODO(directxman12): events
        return scheduledResult, nil
    }
```

如果我们确实必须运行一个job，我们将需要等待现有job完成，替换现有job，或者只是添加新的job。如果我们的信息由于缓存延迟而过时，当我们获得最新的信息时，我们将获得一个requeue。

```go
  // figure out how to run this job -- concurrency policy might forbid us from running
    // multiple at the same time...
    if cronJob.Spec.ConcurrencyPolicy == batchv1.ForbidConcurrent && len(activeJobs) > 0 {
        log.V(1).Info("concurrency policy blocks concurrent runs, skipping", "num active", len(activeJobs))
        return scheduledResult, nil
    }

    // ...or instruct us to replace existing ones...
    if cronJob.Spec.ConcurrencyPolicy == batchv1.ReplaceConcurrent {
        for _, activeJob := range activeJobs {
            // we don't care if the job was already deleted
            if err := r.Delete(ctx, activeJob, client.PropagationPolicy(metav1.DeletePropagationBackground)); client.IgnoreNotFound(err) != nil {
                log.Error(err, "unable to delete active job", "job", activeJob)
                return ctrl.Result{}, err
            }
        }
    }

```

一旦我们弄清楚如何处理现有的job，我们实际上就会创造出我们想要的job

```go
// actually make the job...
    job, err := constructJobForCronJob(&cronJob, missedRun)
    if err != nil {
        log.Error(err, "unable to construct job from template")
        // don't bother requeuing until we get a change to the spec
        return scheduledResult, nil
    }

    // ...and create it on the cluster
    if err := r.Create(ctx, job); err != nil {
        log.Error(err, "unable to create Job for CronJob", "job", job)
        return ctrl.Result{}, err
    }

    log.V(1).Info("created Job for CronJob run", "job", job)
```

#### [7: Requeue when we either see a running job or it’s time for the next scheduled run](https://book.kubebuilder.io/cronjob-tutorial/controller-implementation.html#7-requeue-when-we-either-see-a-running-job-or-its-time-for-the-next-scheduled-run)

最后，我们将返回上面执行的结果，这表示当需要进行下一次运行时，我们想要重新排队。这被视为最大期限——如果在这期间有其他事情发生变化，比如我们的工作开始或结束，被修改，等等，我们可能会更快地再次reconcile。

```go
   // we'll requeue once we see the running job, and update our status
    return scheduledResult, nil
}

```

#### [Setup](https://book.kubebuilder.io/cronjob-tutorial/controller-implementation.html#setup)

最后，我们将更新setup。为了让我们的reconciler程序能够快速地通过其所有者找到Jobs，我们需要一个index。我们声明一个索引键，稍后可以在client中将其用作伪字段名，然后描述如何从Job对象中提取索引值。索引器将自动为我们处理namespace，所以如果Job被CronJob所拥有，我们只需要提取所有者的name。

此外，我们将通知manager这个contronller拥有一些Job，这样当Job发生变化、被删除等时，它将自动调用底层CronJob上的Reconcile。

```go
var (
    jobOwnerKey = ".metadata.controller"
    apiGVStr    = batchv1.GroupVersion.String()
)

func (r *CronJobReconciler) SetupWithManager(mgr ctrl.Manager) error {
    // set up a real clock, since we're not in a test
    if r.Clock == nil {
        r.Clock = realClock{}
    }

    if err := mgr.GetFieldIndexer().IndexField(context.Background(), &kbatch.Job{}, jobOwnerKey, func(rawObj client.Object) []string {
        // grab the job object, extract the owner...
        job := rawObj.(*kbatch.Job)
        owner := metav1.GetControllerOf(job)
        if owner == nil {
            return nil
        }
        // ...make sure it's a CronJob...
        if owner.APIVersion != apiGVStr || owner.Kind != "CronJob" {
            return nil
        }

        // ...and if so, return it
        return []string{owner.Name}
    }); err != nil {
        return err
    }

    return ctrl.NewControllerManagedBy(mgr).
        For(&batchv1.CronJob{}).
        Owns(&kbatch.Job{}).
        Complete(r)
}

```

这很了不起，但现在我们有了一个工作控制器。让我们针对集群进行测试，然后，如果没有任何问题，就部署它!

### [You said something about main?](https://book.kubebuilder.io/cronjob-tutorial/main-revisited.html#you-said-something-about-main)

但首先，记得我们说过要回到`main.go`。再去一次吗?让我们看看有什么变化，以及我们需要添加什么。

```go
package main

import (
    "flag"
    "os"

    // Import all Kubernetes client auth plugins (e.g. Azure, GCP, OIDC, etc.)
    // to ensure that exec-entrypoint and run can make use of them.
    _ "k8s.io/client-go/plugin/pkg/client/auth"

    "k8s.io/apimachinery/pkg/runtime"
    utilruntime "k8s.io/apimachinery/pkg/util/runtime"
    clientgoscheme "k8s.io/client-go/kubernetes/scheme"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/healthz"
    "sigs.k8s.io/controller-runtime/pkg/log/zap"

    batchv1 "tutorial.kubebuilder.io/project/api/v1"
    "tutorial.kubebuilder.io/project/controllers"
    //+kubebuilder:scaffold:imports
)
```

要注意的第一个区别是kubebuilder将新的API组的包(batchv1)添加到我们的scheme中。这意味着我们可以在contronller中使用这些对象

如果我们要使用任何其他的CRD，我们将不得不以同样的方式添加它们的scheme。诸如Job之类的内置类型的方案由`clientgoscheme`添加。

```go
var (
    scheme   = runtime.NewScheme()
    setupLog = ctrl.Log.WithName("setup")
)

func init() {
    utilruntime.Must(clientgoscheme.AddToScheme(scheme))

    utilruntime.Must(batchv1.AddToScheme(scheme))
    //+kubebuilder:scaffold:scheme
}
```

另一个改变是kubebuilder添加了一个调用CronJob的controller的SetupWithManager方法的块。

```go
func main() {
//old stuff
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

if err = (&controllers.CronJobReconciler{
        Client: mgr.GetClient(),
        Scheme: mgr.GetScheme(),
    }).SetupWithManager(mgr); err != nil {
        setupLog.Error(err, "unable to create controller", "controller", "CronJob")
        os.Exit(1)
    }
```

我们还将为我们的类型设置webhooks，我们将在后面讨论。我们只需要将它们添加到manager中。因为我们可能想要单独运行webhook，或者在本地测试controller时不运行它们，所以我们将它们放在一个环境变量后面。

我们要确保在本地运行时设置`ENABLE_WEBHOOKS=false`。

```go
 if os.Getenv("ENABLE_WEBHOOKS") != "false" {
        if err = (&batchv1.CronJob{}).SetupWebhookWithManager(mgr); err != nil {
            setupLog.Error(err, "unable to create webhook", "webhook", "CronJob")
            os.Exit(1)
        }
    }
    //+kubebuilder:scaffold:builder

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

现在我们可以实现contronller了。

#### [Implementing defaulting/validating webhooks](https://book.kubebuilder.io/cronjob-tutorial/webhook-implementation.html#implementing-defaultingvalidating-webhooks)

如果你想为你的CRD实现准入webhook，你唯一需要做的就是实现`Defaulter`和(或)`Validator`接口。

kubebuilder 会帮助你处理剩下的事，比如：

1.创建webhook服务器。

2.确保服务器已经添加到controller中。

3.为webhook创建处理程序。

4.用服务器中的路径注册每个处理程序。

首先，让我们为CRD (CronJob)搭建webhook。我们需要使用`--default`和`--programming -validation`标志运行以下命令(因为我们的测试项目将使用默认和验证的webhook):

```go
kubebuilder create webhook --group batch --version v1 --kind CronJob --defaulting --programmatic-validation

```

这将scaffold这个webhook函数，并将您的webhook注册到main.go的controller中。

##### [Supporting older cluster versions](https://book.kubebuilder.io/cronjob-tutorial/webhook-implementation.html#supporting-older-cluster-versions)

> 与Go webhook实现一起创建的默认WebhookConfiguration清单使用API版本v1。如果您的项目打算支持比v1.16更早的Kubernetes集群版本，请设置——webhook-version v1beta1。更多信息请参见webhook参考。

```go
package v1

import (
    "github.com/robfig/cron"
    apierrors "k8s.io/apimachinery/pkg/api/errors"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/apimachinery/pkg/runtime/schema"
    validationutils "k8s.io/apimachinery/pkg/util/validation"
    "k8s.io/apimachinery/pkg/util/validation/field"
    ctrl "sigs.k8s.io/controller-runtime"
    logf "sigs.k8s.io/controller-runtime/pkg/log"
    "sigs.k8s.io/controller-runtime/pkg/webhook"
)

```

接下来，我们将为webhook设置一个日志记录器。

```go
var cronjoblog = logf.Log.WithName("cronjob-resource")
```

