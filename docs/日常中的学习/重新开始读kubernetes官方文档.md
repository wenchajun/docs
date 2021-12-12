11.25

## Kubernetes 可以为您做些什么?

通过现代的 Web 服务，用户希望应用程序能够 24/7 全天候使用，开发人员希望每天可以多次发布部署新版本的应用程序。 容器化可以帮助软件包达成这些目标，使应用程序能够以简单快速的方式发布和更新，而无需停机。Kubernetes 帮助您确保这些容器化的应用程序在您想要的时间和地点运行，并帮助应用程序找到它们需要的资源和工具。Kubernetes 是一个可用于生产的开源平台，根据 Google 容器集群方面积累的经验，以及来自社区的最佳实践而设计。

https://kubernetes.io/zh/docs/concepts/overview/components/

#### 从kubernetes组件开始

当你部署完 Kubernetes, 即拥有了一个完整的集群。

一个 Kubernetes 集群由一组被称作节点的机器组成。这些节点上运行 Kubernetes 所管理的容器化应用。集群具有至少一个工作节点。

工作节点托管作为应用负载的组件的 Pod 。控制平面管理集群中的工作节点和 Pod 。 为集群提供故障转移和高可用性，这些控制平面一般跨多主机运行，集群跨多个节点运行。

本文档概述了交付正常运行的 Kubernetes 集群所需的各种组件。

这张图表展示了包含所有相互关联组件的 Kubernetes 集群。

![components-of-kubernetes](F:\docs\docs\images\components-of-kubernetes.svg)

## 控制平面组件（Control Plane Components） 

控制平面的组件对集群做出全局决策(比如调度)，以及检测和响应集群事件（例如，当不满足部署的 `replicas` 字段时，启动新的 [pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/)）。

控制平面组件可以在集群中的任何节点上运行。 然而，为了简单起见，设置脚本通常会在同一个计算机上启动所有控制平面组件， 并且不会在此计算机上运行用户容器。 请参阅[使用 kubeadm 构建高可用性集群](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/high-availability/) 中关于多 VM 控制平面设置的示例。

### kube-apiserver

API 服务器是 Kubernetes [控制面](https://kubernetes.io/zh/docs/reference/glossary/?all=true#term-control-plane)的组件， 该组件公开了 Kubernetes API。 API 服务器是 Kubernetes 控制面的前端。

Kubernetes API 服务器的主要实现是 [kube-apiserver](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kube-apiserver/)。 kube-apiserver 设计上考虑了**水平伸缩**，也就是说，它可通过部署多个实例进行伸缩。 你可以运行 kube-apiserver 的多个实例，并在这些实例之间平衡流量。

流量高就扩散为多个pod

### etcd

etcd 是兼具一致性和高可用性的键值数据库，可以作为保存 Kubernetes 所有集群数据的后台数据库。

您的 Kubernetes 集群的 etcd 数据库通常需要有个备份计划。

要了解 etcd 更深层次的信息，请参考 [etcd 文档](https://etcd.io/docs/)。

### kube-scheduler

控制平面组件，负责监视新创建的、未指定运行[节点（node）](https://kubernetes.io/zh/docs/concepts/architecture/nodes/)的 [Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/)，选择节点让 Pod 在上面运行。

调度决策考虑的因素包括单个 Pod 和 Pod 集合的资源需求、硬件/软件/策略约束、亲和性和反亲和性规范、数据位置、工作负载间的干扰和最后时限。

### kube-controller-manager

运行[控制器](https://kubernetes.io/zh/docs/concepts/architecture/controller/)进程的控制平面组件。

从逻辑上讲，每个[控制器](https://kubernetes.io/zh/docs/concepts/architecture/controller/)都是一个单独的进程， 但是为了降低复杂性，它们都被编译到同一个可执行文件，并在一个进程中运行。

这些控制器包括:

- 节点控制器（Node Controller）: 负责在节点出现故障时进行通知和响应
- 任务控制器（Job controller）: 监测代表一次性任务的 Job 对象，然后创建 Pods 来运行这些任务直至完成
- 端点控制器（Endpoints Controller）: 填充端点(Endpoints)对象(即加入 Service 与 Pod)
- 服务帐户和令牌控制器（Service Account & Token Controllers）: 为新的命名空间创建默认帐户和 API 访问令牌

### cloud-controller-manager

云控制器管理器是指嵌入特定云的控制逻辑的 [控制平面](https://kubernetes.io/zh/docs/reference/glossary/?all=true#term-control-plane)组件。 云控制器管理器使得你可以将你的集群连接到云提供商的 API 之上， 并将与该云平台交互的组件同与你的集群交互的组件分离开来。

`cloud-controller-manager` 仅运行特定于云平台的控制回路。 如果你在自己的环境中运行 Kubernetes，或者在本地计算机中运行学习环境， 所部署的环境中不需要云控制器管理器。

与 `kube-controller-manager` 类似，`cloud-controller-manager` 将若干逻辑上独立的 控制回路组合到同一个可执行文件中，供你以同一进程的方式运行。 你可以对其执行水平扩容（运行不止一个副本）以提升性能或者增强容错能力。

下面的控制器都包含对云平台驱动的依赖：

- 节点控制器（Node Controller）: 用于在节点终止响应后检查云提供商以确定节点是否已被删除
- 路由控制器（Route Controller）: 用于在底层云基础架构中设置路由
- 服务控制器（Service Controller）: 用于创建、更新和删除云提供商负载均衡器

这是由云供应商提供的吧

## Node 组件 

节点组件在每个节点上运行，维护运行的 Pod 并提供 Kubernetes 运行环境。

### kubelet

一个在集群中每个[节点（node）](https://kubernetes.io/zh/docs/concepts/architecture/nodes/)上运行的代理。 它保证[容器（containers）](https://kubernetes.io/zh/docs/concepts/overview/what-is-kubernetes/#why-containers)都 运行在 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/) 中。

kubelet 接收一组通过各类机制提供给它的 PodSpecs，确保这些 PodSpecs 中描述的容器处于运行状态且健康。 kubelet 不会管理不是由 Kubernetes 创建的容器。

### kube-proxy

[kube-proxy](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kube-proxy/) 是集群中每个节点上运行的网络代理， 实现 Kubernetes [服务（Service）](https://kubernetes.io/zh/docs/concepts/services-networking/service/) 概念的一部分。

kube-proxy 维护节点上的网络规则。这些网络规则允许从集群内部或外部的网络会话与 Pod 进行网络通信。

如果操作系统提供了数据包过滤层并可用的话，kube-proxy 会通过它来实现网络规则。否则， kube-proxy 仅转发流量本身。

### 容器运行时（Container Runtime） 

容器运行环境是负责运行容器的软件。

Kubernetes 支持多个容器运行环境: [Docker](https://kubernetes.io/zh/docs/reference/kubectl/docker-cli-to-kubectl/)、 [containerd](https://containerd.io/docs/)、[CRI-O](https://cri-o.io/#what-is-cri-o) 以及任何实现 [Kubernetes CRI (容器运行环境接口)](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-node/container-runtime-interface.md)。

## 插件（Addons） 

插件使用 Kubernetes 资源（[DaemonSet](https://kubernetes.io/zh/docs/concepts/workloads/controllers/daemonset/)、 [Deployment](https://kubernetes.io/zh/docs/concepts/workloads/controllers/deployment/)等）实现集群功能。 因为这些插件提供集群级别的功能，插件中命名空间域的资源属于 `kube-system` 命名空间。

下面描述众多插件中的几种。有关可用插件的完整列表，请参见 [插件（Addons）](https://kubernetes.io/zh/docs/concepts/cluster-administration/addons/)。

### DNS 

尽管其他插件都并非严格意义上的必需组件，但几乎所有 Kubernetes 集群都应该 有[集群 DNS](https://kubernetes.io/zh/docs/concepts/services-networking/dns-pod-service/)， 因为很多示例都需要 DNS 服务。

集群 DNS 是一个 DNS 服务器，和环境中的其他 DNS 服务器一起工作，它为 Kubernetes 服务提供 DNS 记录。

Kubernetes 启动的容器自动将此 DNS 服务器包含在其 DNS 搜索列表中。

### Web 界面（仪表盘）

[Dashboard](https://kubernetes.io/zh/docs/tasks/access-application-cluster/web-ui-dashboard/) 是 Kubernetes 集群的通用的、基于 Web 的用户界面。 它使用户可以管理集群中运行的应用程序以及集群本身并进行故障排除。

### 容器资源监控

[容器资源监控](https://kubernetes.io/zh/docs/tasks/debug-application-cluster/resource-usage-monitoring/) 将关于容器的一些常见的时间序列度量值保存到一个集中的数据库中，并提供用于浏览这些数据的界面。

### 集群层面日志[ ](https://kubernetes.io/zh/docs/concepts/overview/components/#集群层面日志)

[集群层面日志](https://kubernetes.io/zh/docs/concepts/cluster-administration/logging/) 机制负责将容器的日志数据 保存到一个集中的日志存储中，该存储能够提供搜索和浏览接口。

12.6日

https://kubernetes.io/zh/docs/concepts/overview/kubernetes-api/

# Kubernetes API

Kubernetes [控制面](https://kubernetes.io/zh/docs/reference/glossary/?all=true#term-control-plane) 的核心是 [API 服务器](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kube-apiserver/)。 API 服务器负责提供 HTTP API，以供用户、集群中的不同部分和集群外部组件相互通信。

Kubernetes API 使你可以查询和操纵 Kubernetes API 中对象（例如：Pod、Namespace、ConfigMap 和 Event）的状态。

大部分操作都可以通过 [kubectl](https://kubernetes.io/zh/docs/reference/kubectl/overview/) 命令行接口或 类似 [kubeadm](https://kubernetes.io/zh/docs/reference/setup-tools/kubeadm/) 这类命令行工具来执行， 这些工具在背后也是调用 API。不过，你也可以使用 REST 调用来访问这些 API。

如果你正在编写程序来访问 Kubernetes API，可以考虑使用 [客户端库](https://kubernetes.io/zh/docs/reference/using-api/client-libraries/)之一。

> 比如client-go之类

## OpenAPI 规范 

完整的 API 细节是用 [OpenAPI](https://www.openapis.org/) 来表述的。

Kubernetes API 服务器通过 `/openapi/v2` 末端提供 OpenAPI 规范。 你可以按照下表所给的请求头部，指定响应的格式：

| 头部               | 可选值                                                       | 说明                     |
| ------------------ | ------------------------------------------------------------ | ------------------------ |
| `Accept-Encoding`  | `gzip`                                                       | *不指定此头部也是可以的* |
| `Accept`           | `application/com.github.proto-openapi.spec.v2@v1.0+protobuf` | *主要用于集群内部*       |
| `application/json` | *默认值*                                                     |                          |
| `*`                | *提供*`application/json`                                     |                          |

Kubernetes 为 API 实现了一种基于 Protobuf 的序列化格式，主要用于集群内部通信。 关于此格式的详细信息，可参考 [Kubernetes Protobuf 序列化](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/api-machinery/protobuf.md) 设计提案。每种模式对应的接口描述语言（IDL）位于定义 API 对象的 Go 包中。

## API 变更 

任何成功的系统都要随着新的使用案例的出现和现有案例的变化来成长和变化。 为此，Kubernetes 的功能特性设计考虑了让 Kubernetes API 能够持续变更和成长的因素。 Kubernetes 项目的目标是 *不要* 引发现有客户端的兼容性问题，并在一定的时期内 维持这种兼容性，以便其他项目有机会作出适应性变更。

一般而言，新的 API 资源和新的资源字段可以被频繁地添加进来。 删除资源或者字段则要遵从 [API 废弃策略](https://kubernetes.io/zh/docs/reference/using-api/deprecation-policy/)。

关于什么是兼容性的变更、如何变更 API 等详细信息，可参考 [API 变更](https://git.k8s.io/community/contributors/devel/sig-architecture/api_changes.md#readme)。

## API 组和版本 

为了简化删除字段或者重构资源表示等工作，Kubernetes 支持多个 API 版本， 每一个版本都在不同 API 路径下，例如 `/api/v1` 或 `/apis/rbac.authorization.k8s.io/v1alpha1`。

版本化是在 API 级别而不是在资源或字段级别进行的，目的是为了确保 API 为系统资源和行为提供清晰、一致的视图，并能够控制对已废止的和/或实验性 API 的访问。

为了便于演化和扩展其 API，Kubernetes 实现了 可被[启用或禁用](https://kubernetes.io/zh/docs/reference/using-api/#enabling-or-disabling)的 [API 组](https://kubernetes.io/zh/docs/reference/using-api/#api-groups)。

API 资源之间靠 API 组、资源类型、名字空间（对于名字空间作用域的资源而言）和 名字来相互区分。API 服务器可能通过多个 API 版本来向外提供相同的下层数据， 并透明地完成不同 API 版本之间的转换。所有这些不同的版本实际上都是同一资源 的（不同）表现形式。例如，假定同一资源有 `v1` 和 `v1beta1` 版本， 使用 `v1beta1` 创建的对象则可以使用 `v1beta1` 或者 `v1` 版本来读取、更改 或者删除。

关于 API 版本级别的详细定义，请参阅 [API 版本参考](https://kubernetes.io/zh/docs/reference/using-api/#api-versioning)。

## API 扩展 

有两种途径来扩展 Kubernetes API：

1. 你可以使用[自定义资源](https://kubernetes.io/zh/docs/concepts/extend-kubernetes/api-extension/custom-resources/) 来以声明式方式定义 API 服务器如何提供你所选择的资源 API。
2. 你也可以选择实现自己的 [聚合层](https://kubernetes.io/zh/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/) 来扩展 Kubernetes API。



#### protobuf介绍：

https://colobu.com/2019/10/03/protobuf-ultimate-tutorial-in-go/

https://blog.csdn.net/carson_ho/article/details/70568606

##### 序列化

序列化(serialization、marshalling)的过程是指将数据结构或者对象的状态转换成可以存储(比如文件、内存)或者传输的格式(比如网络)。反向操作就是反序列化(deserialization、unmarshalling)的过程。

1987年曾经的Sun Microsystems发布了XDR。

二十世纪九十年代后期，XML开始流行，它是一种人类易读的基于文本的编码方式，易于阅读和理解，但是失去了紧凑的基于字节流的编码的优势。

JSON是一种更轻量级的基于文本的编码方式，经常用在client/server端的通讯中。

YAML类似JSON，新的特性更强大，更适合人类阅读，也更紧凑。

还有苹果系统的property list。

除了上面这些和Protobuf，还有许许多多的序列化格式，比如Thrift、Avro、BSON、CBOR、MessagePack, 还有很多非跨语言的编码格式。项目[gosercomp](https://github.com/smallnest/gosercomp)对比了各种go的序列化库，包括序列化和反序列的性能，以及序列化后的数据大小。总体来说Protobuf序列化和反序列的性能都是比较高的，编码后的数据大小也不错。

Protobuf支持很多语言，比如C++、C#、Dart、Go、Java、Python、Rust等，同时也是跨平台的，所以得到了广泛的应用。

Protobuf包含序列化格式的定义、各种语言的库以及一个IDL编译器。正常情况下你需要定义proto文件，然后使用IDL编译器编译成你需要的语言。

protocol buffers 被寄予一下 2 个特点：

- 可以很容易地引入新的字段，并且不需要检查数据的中间服务器可以简单地解析并传递数据，而无需了解所有字段。
- 数据格式更加具有自我描述性，可以用各种语言来处理(C++, Java 等各种语言)

这个版本的 protocol buffers 仍需要自己手写解析的代码。

不过随着系统慢慢发展，演进，protocol buffers 目前具有了更多的特性：

- 自动生成的序列化和反序列化代码避免了手动解析的需要。（官方提供自动生成代码工具，各个语言平台的基本都有）
- 除了用于 RPC（远程过程调用）请求之外，人们开始将 protocol buffers 用作持久存储数据的便捷自描述格式（例如，在Bigtable中）。
- 服务器的 RPC 接口可以先声明为协议的一部分，然后用 protocol compiler 生成基类，用户可以使用服务器接口的实际实现来覆盖它们。

protocol buffers 现在是 Google 用于数据的通用语言。在撰写本文时，谷歌代码树中定义了 48162 种不同的消息类型，包括 12183 个 .proto 文件。它们既用于 RPC 系统，也用于在各种存储系统中持久存储数据。

小结：

**protocol buffers 诞生之初是为了解决服务器端新旧协议(高低版本)兼容性问题，名字也很体贴，“协议缓冲区”。只不过后期慢慢发展成用于传输数据**。

12.7

https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/

# 理解 Kubernetes 对象

本页说明了 Kubernetes 对象在 Kubernetes API 中是如何表示的，以及如何在 `.yaml` 格式的文件中表示。

## 理解 Kubernetes 对象

在 Kubernetes 系统中，*Kubernetes 对象* 是持久化的实体。 Kubernetes 使用这些实体去表示整个集群的状态。特别地，它们描述了如下信息：

- 哪些容器化应用在运行（以及在哪些节点上）
- 可以被应用使用的资源
- 关于应用运行时表现的策略，比如重启策略、升级策略，以及容错策略

Kubernetes 对象是 “目标性记录” —— 一旦创建对象，Kubernetes 系统将持续工作以确保对象存在。 通过创建对象，本质上是在告知 Kubernetes 系统，所需要的集群工作负载看起来是什么样子的， 这就是 Kubernetes 集群的 **期望状态（Desired State）**。

操作 Kubernetes 对象 —— 无论是创建、修改，或者删除 —— 需要使用 [Kubernetes API](https://kubernetes.io/zh/docs/concepts/overview/kubernetes-api)。 比如，当使用 `kubectl` 命令行接口时，CLI 会执行必要的 Kubernetes API 调用， 也可以在程序中使用 [客户端库](https://kubernetes.io/zh/docs/reference/using-api/client-libraries/)直接调用 Kubernetes API。

### 对象规约（Spec）与状态（Status） 

几乎每个 Kubernetes 对象包含两个嵌套的对象字段，它们负责管理对象的配置： 对象 *`spec`（规约）* 和 对象 *`status`（状态）* 。 对于具有 `spec` 的对象，你必须在创建对象时设置其内容，描述你希望对象所具有的特征： *期望状态（Desired State）* 。

`status` 描述了对象的 *当前状态（Current State）*，它是由 Kubernetes 系统和组件 设置并更新的。在任何时刻，Kubernetes [控制平面](https://kubernetes.io/zh/docs/reference/glossary/?all=true#term-control-plane) 都一直积极地管理着对象的实际状态，以使之与期望状态相匹配。

例如，Kubernetes 中的 Deployment 对象能够表示运行在集群中的应用。 当创建 Deployment 时，可能需要设置 Deployment 的 `spec`，以指定该应用需要有 3 个副本运行。 Kubernetes 系统读取 Deployment 规约，并启动我们所期望的应用的 3 个实例 —— 更新状态以与规约相匹配。 如果这些实例中有的失败了（一种状态变更），Kubernetes 系统通过执行修正操作 来响应规约和状态间的不一致 —— 在这里意味着它会启动一个新的实例来替换。

关于对象 spec、status 和 metadata 的更多信息，可参阅 [Kubernetes API 约定](https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md)。

> Note: 这里client-go list-and-watch也是这样



### 描述 Kubernetes 对象

创建 Kubernetes 对象时，必须提供对象的规约，用来描述该对象的期望状态， 以及关于对象的一些基本信息（例如名称）。 当使用 Kubernetes API 创建对象时（或者直接创建，或者基于`kubectl`）， API 请求必须在请求体中包含 JSON 格式的信息。 **大多数情况下，需要在 .yaml 文件中为 `kubectl` 提供这些信息**。 `kubectl` 在发起 API 请求时，将这些信息转换成 JSON 格式。

这里有一个 `.yaml` 示例文件，展示了 Kubernetes Deployment 的必需字段和对象规约：

[`application/deployment.yaml` ](https://raw.githubusercontent.com/kubernetes/website/main/content/zh/examples/application/deployment.yaml)![Copy application/deployment.yaml to clipboard](https://d33wubrfki0l68.cloudfront.net/0901162ab78eb4ff2e9e5dc8b17c3824befc91a6/44ccd/images/copycode.svg)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2 # tells deployment to run 2 pods matching the template
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

使用类似于上面的 `.yaml` 文件来创建 Deployment的一种方式是使用 `kubectl` 命令行接口（CLI）中的 [`kubectl apply`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#apply) 命令， 将 `.yaml` 文件作为参数。下面是一个示例：

```shell
kubectl apply -f https://k8s.io/examples/application/deployment.yaml --record
```

输出类似如下这样：

```
deployment.apps/nginx-deployment created
```

### 必需字段 

在想要创建的 Kubernetes 对象对应的 `.yaml` 文件中，需要配置如下的字段：

- `apiVersion` - 创建该对象所使用的 Kubernetes API 的版本
- `kind` - 想要创建的对象的类别
- `metadata` - 帮助唯一性标识对象的一些数据，包括一个 `name` 字符串、UID 和可选的 `namespace`
- `spec` - 你所期望的该对象的状态

对象 `spec` 的精确格式对每个 Kubernetes 对象来说是不同的，包含了特定于该对象的嵌套字段。 [Kubernetes API 参考](https://kubernetes.io/docs/reference/kubernetes-api/) 能够帮助我们找到任何我们想创建的对象的规约格式。

例如，Pod 参考文档详细说明了 API 中 Pod 的 [`spec` 字段](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#PodSpec)， Deployment 的参考文档则详细说明了 Deployment 的 [`spec` 字段](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/deployment-v1/#DeploymentSpec)。 在这些 API 参考页面中，你将看到提到的 PodSpec 和 DeploymentSpec。 这些名字是 Kubernetes 用来实现其 API 的 Golang 代码的实现细节。

https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/object-management/

# Kubernetes 对象管理

`kubectl` 命令行工具支持多种不同的方式来创建和管理 Kubernetes 对象。 本文档概述了不同的方法。 阅读 [Kubectl book](https://kubectl.docs.kubernetes.io/) 来了解 kubectl 管理对象的详细信息。

## 管理技巧

**警告：**

应该只使用一种技术来管理 Kubernetes 对象。混合和匹配技术作用在同一对象上将导致未定义行为。

| 管理技术       | 作用于   | 建议的环境 | 支持的写者 | 学习难度 |
| -------------- | -------- | ---------- | ---------- | -------- |
| 指令式命令     | 活跃对象 | 开发项目   | 1+         | 最低     |
| 指令式对象配置 | 单个文件 | 生产项目   | 1          | 中等     |
| 声明式对象配置 | 文件目录 | 生产项目   | 1+         | 最高     |

## 指令式命令

使用指令式命令时，用户可以在集群中的活动对象上进行操作。用户将操作传给 `kubectl` 命令作为参数或标志。

这是开始或者在集群中运行一次性任务的推荐方法。因为这个技术直接在活跃对象 上操作，所以它不提供以前配置的历史记录。

### 例子

通过创建 Deployment 对象来运行 nginx 容器的实例：

```sh
kubectl create deployment nginx --image nginx
```

### 权衡

与对象配置相比的优点：

- 命令简单，易学且易于记忆。
- 命令仅需一步即可对集群进行更改。

与对象配置相比的缺点：

- 命令不与变更审查流程集成。
- 命令不提供与更改关联的审核跟踪。
- 除了实时内容外，命令不提供记录源。
- 命令不提供用于创建新对象的模板。

## 指令式对象配置

在指令式对象配置中，kubectl 命令指定操作（创建，替换等），可选标志和 至少一个文件名。指定的文件必须包含 YAML 或 JSON 格式的对象的完整定义。

有关对象定义的详细信息，请查看 [API 参考](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.23/)。

**警告：**

`replace` 指令式命令将现有规范替换为新提供的规范，并放弃对配置文件中 缺少的对象的所有更改。此方法不应与对象规约被独立于配置文件进行更新的 资源类型一起使用。比如类型为 `LoadBalancer` 的服务，它的 `externalIPs` 字段就是独立于集群配置进行更新。

### 例子

创建配置文件中定义的对象：

```sh
kubectl create -f nginx.yaml
```

删除两个配置文件中定义的对象：

```sh
kubectl delete -f nginx.yaml -f redis.yaml
```

通过覆盖活动配置来更新配置文件中定义的对象：

```sh
kubectl replace -f nginx.yaml
```

### 权衡

与指令式命令相比的优点：

- 对象配置可以存储在源控制系统中，比如 Git。
- 对象配置可以与流程集成，例如在推送和审计之前检查更新。
- 对象配置提供了用于创建新对象的模板。

与指令式命令相比的缺点：

- 对象配置需要对对象架构有基本的了解。
- 对象配置需要额外的步骤来编写 YAML 文件。

与声明式对象配置相比的优点：

- 指令式对象配置行为更加简单易懂。
- 从 Kubernetes 1.5 版本开始，指令对象配置更加成熟。

与声明式对象配置相比的缺点：

- 指令式对象配置更适合文件，而非目录。
- 对活动对象的更新必须反映在配置文件中，否则会在下一次替换时丢失。

## 声明式对象配置

使用声明式对象配置时，用户对本地存储的对象配置文件进行操作，但是用户 未定义要对该文件执行的操作。 `kubectl` 会自动检测每个文件的创建、更新和删除操作。 这使得配置可以在目录上工作，根据目录中配置文件对不同的对象执行不同的操作。

**说明：**

声明式对象配置保留其他编写者所做的修改，即使这些更改并未合并到对象配置文件中。 可以通过使用 `patch` API 操作仅写入观察到的差异，而不是使用 `replace` API 操作来替换整个对象配置来实现。

### 例子

处理 `configs` 目录中的所有对象配置文件，创建并更新活跃对象。 可以首先使用 `diff` 子命令查看将要进行的更改，然后在进行应用：

```sh
kubectl diff -f configs/
kubectl apply -f configs/
```

递归处理目录：

```sh
kubectl diff -R -f configs/
kubectl apply -R -f configs/
```

### 权衡

与指令式对象配置相比的优点：

- 对活动对象所做的更改即使未合并到配置文件中，也会被保留下来。
- 声明性对象配置更好地支持对目录进行操作并自动检测每个文件的操作类型（创建，修补，删除）。

与指令式对象配置相比的缺点：

- 声明式对象配置难于调试并且出现异常时结果难以理解。
- 使用 diff 产生的部分更新会创建复杂的合并和补丁操作。

# 对象名称和 IDs

集群中的每一个对象都有一个[*名称*](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/names/#names) 来标识在同类资源中的唯一性。

每个 Kubernetes 对象也有一个[*UID*](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/names/#uids) 来标识在整个集群中的唯一性。

比如，在同一个[名字空间](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/namespaces/) 中有一个名为 `myapp-1234` 的 Pod, 但是可以命名一个 Pod 和一个 Deployment 同为 `myapp-1234`.

对于用户提供的非唯一性的属性，Kubernetes 提供了 [标签（Labels）](https://kubernetes.io/zh/docs/concepts/working-with-objects/labels)和 [注解（Annotation）](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/annotations/)机制。

> 也就是不同kind是可以同名的

## 名称 

客户端提供的字符串，引用资源 url 中的对象，如`/api/v1/pods/some name`。

某一时刻，只能有一个给定类型的对象具有给定的名称。但是，如果删除该对象，则可以创建同名的新对象。

**说明：**

当对象所代表的是一个物理实体（例如代表一台物理主机的 Node）时， 如果在 Node 对象未被删除并重建的条件下，重新创建了同名的物理主机， 则 Kubernetes 会将新的主机看作是老的主机，这可能会带来某种不一致性。

以下是比较常见的四种资源命名约束。

### DNS 子域名 

很多资源类型需要可以用作 DNS 子域名的名称。 DNS 子域名的定义可参见 [RFC 1123](https://tools.ietf.org/html/rfc1123)。 这一要求意味着名称必须满足如下规则：

- 不能超过253个字符
- 只能包含小写字母、数字，以及'-' 和 '.'
- 须以字母数字开头
- 须以字母数字结尾

### RFC 1123 标签名 

某些资源类型需要其名称遵循 [RFC 1123](https://tools.ietf.org/html/rfc1123) 所定义的 DNS 标签标准。也就是命名必须满足如下规则：

- 最多 63 个字符
- 只能包含小写字母、数字，以及 '-'
- 须以字母数字开头
- 须以字母数字结尾

### RFC 1035 标签名 

某些资源类型需要其名称遵循 [RFC 1035](https://tools.ietf.org/html/rfc1035) 所定义的 DNS 标签标准。也就是命名必须满足如下规则：

- 最多 63 个字符
- 只能包含小写字母、数字，以及 '-'
- 须以字母开头
- 须以字母数字结尾

### 路径分段名称 

某些资源类型要求名称能被安全地用作路径中的片段。 换句话说，其名称不能是 `.`、`..`，也不可以包含 `/` 或 `%` 这些字符。

下面是一个名为`nginx-demo`的 Pod 的配置清单：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-demo
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
```

**说明：** 某些资源类型可能具有额外的命名约束。

## UIDs

Kubernetes 系统生成的字符串，唯一标识对象。

在 Kubernetes 集群的整个生命周期中创建的每个对象都有一个不同的 uid，它旨在区分类似实体的历史事件。

Kubernetes UIDs 是全局唯一标识符（也叫 UUIDs）。 UUIDs 是标准化的，见 ISO/IEC 9834-8 和 ITU-T X.667.

# 名字空间

Kubernetes 支持多个虚拟集群，它们底层依赖于同一个物理集群。 这些虚拟集群被称为名字空间。 在一些文档里名字空间也称为命名空间。

## 何时使用多个名字空间

名字空间适用于存在很多跨多个团队或项目的用户的场景。对于只有几到几十个用户的集群，根本不需要创建或考虑名字空间。当需要名称空间提供的功能时，请开始使用它们。

名字空间为名称提供了一个范围。资源的名称需要在名字空间内是唯一的，但不能跨名字空间。 名字空间不能相互嵌套，每个 Kubernetes 资源只能在一个名字空间中。

名字空间是在多个用户之间划分集群资源的一种方法（通过[资源配额](https://kubernetes.io/zh/docs/concepts/policy/resource-quotas/)）。

不必使用多个名字空间来分隔仅仅轻微不同的资源，例如同一软件的不同版本： 应该使用[标签](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/labels/) 来区分同一名字空间中的不同资源。

## 使用名字空间

名字空间的创建和删除在[名字空间的管理指南文档](https://kubernetes.io/zh/docs/tasks/administer-cluster/namespaces/)描述。

**说明：** 避免使用前缀 `kube-` 创建名字空间，因为它是为 Kubernetes 系统名字空间保留的。

### 查看名字空间

你可以使用以下命令列出集群中现存的名字空间：

```shell
kubectl get namespace
NAME          STATUS    AGE
default       Active    1d
kube-node-lease   Active   1d
kube-system   Active    1d
kube-public   Active    1d
```

Kubernetes 会创建四个初始名字空间：

- `default` 没有指明使用其它名字空间的对象所使用的默认名字空间
- `kube-system` Kubernetes 系统创建对象所使用的名字空间
- `kube-public` 这个名字空间是自动创建的，所有用户（包括未经过身份验证的用户）都可以读取它。 这个名字空间主要用于集群使用，以防某些资源在整个集群中应该是可见和可读的。 这个名字空间的公共方面只是一种约定，而不是要求。
- `kube-node-lease` 此名字空间用于与各个节点相关的 [租约（Lease）](https://kubernetes.io/docs/reference/kubernetes-api/cluster-resources/lease-v1/)对象。 节点租期允许 kubelet 发送[心跳](https://kubernetes.io/zh/docs/concepts/architecture/nodes/#heartbeats)，由此控制面能够检测到节点故障。

### 为请求设置名字空间

要为当前请求设置名字空间，请使用 `--namespace` 参数。

例如：

```shell
kubectl run nginx --image=nginx --namespace=<名字空间名称>
kubectl get pods --namespace=<名字空间名称>
```

### 设置名字空间偏好

你可以永久保存名字空间，以用于对应上下文中所有后续 kubectl 命令。

```shell
kubectl config set-context --current --namespace=<名字空间名称>
# 验证之
kubectl config view | grep namespace:
```

## 名字空间和 DNS

当你创建一个[服务](https://kubernetes.io/zh/docs/concepts/services-networking/service/) 时， Kubernetes 会创建一个相应的 [DNS 条目](https://kubernetes.io/zh/docs/concepts/services-networking/dns-pod-service/)。

该条目的形式是 `<服务名称>.<名字空间名称>.svc.cluster.local`，这意味着如果容器只使用 `<服务名称>`，它将被解析到本地名字空间的服务。这对于跨多个名字空间（如开发、分级和生产） 使用相同的配置非常有用。如果你希望跨名字空间访问，则需要使用完全限定域名（FQDN）。

## 并非所有对象都在名字空间中

大多数 kubernetes 资源（例如 Pod、Service、副本控制器等）都位于某些名字空间中。 但是名字空间资源本身并不在名字空间中。而且底层资源，例如 [节点](https://kubernetes.io/zh/docs/concepts/architecture/nodes/) 和持久化卷不属于任何名字空间。

查看哪些 Kubernetes 资源在名字空间中，哪些不在名字空间中：

```shell
# 位于名字空间中的资源
kubectl api-resources --namespaced=true

# 不在名字空间中的资源
kubectl api-resources --namespaced=false
```

## 自动打标签 

**FEATURE STATE:** `Kubernetes 1.21 [beta]`

Kubernetes 控制面会为所有名字空间设置一个不可变更的 [标签](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/labels/) `kubernetes.io/metadata.name`，只要 `NamespaceDefaultLabelName` 这一 [特性门控](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/feature-gates/) 被启用。标签的值是名字空间的名称。

# 标签和选择算符

*标签（Labels）* 是附加到 Kubernetes 对象（比如 Pods）上的键值对。 标签旨在用于指定对用户有意义且相关的对象的标识属性，但不直接对核心系统有语义含义。 标签可以用于组织和选择对象的子集。标签可以在创建时附加到对象，随后可以随时添加和修改。 每个对象都可以定义一组键/值标签。每个键对于给定对象必须是唯一的。

```json
"metadata": {
  "labels": {
    "key1" : "value1",
    "key2" : "value2"
  }
}
```

标签能够支持高效的查询和监听操作，对于用户界面和命令行是很理想的。 应使用[注解](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/annotations/) 记录非识别信息。

>标签旨在用于指定对用户有意义且相关的对象的标识属性

## 动机

标签使用户能够以松散耦合的方式将他们自己的组织结构映射到系统对象，而无需客户端存储这些映射。

服务部署和批处理流水线通常是多维实体（例如，多个分区或部署、多个发行序列、多个层，每层多个微服务）。 管理通常需要交叉操作，这打破了严格的层次表示的封装，特别是由基础设施而不是用户确定的严格的层次结构。

示例标签：

- `"release" : "stable"`, `"release" : "canary"`
- `"environment" : "dev"`, `"environment" : "qa"`, `"environment" : "production"`
- `"tier" : "frontend"`, `"tier" : "backend"`, `"tier" : "cache"`
- `"partition" : "customerA"`, `"partition" : "customerB"`
- `"track" : "daily"`, `"track" : "weekly"`

有一些[常用标签](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/common-labels/)的例子; 你可以任意制定自己的约定。 请记住，标签的 Key 对于给定对象必须是唯一的。

## 语法和字符集

*标签* 是键值对。有效的标签键有两个段：可选的前缀和名称，用斜杠（`/`）分隔。 名称段是必需的，必须小于等于 63 个字符，以字母数字字符（`[a-z0-9A-Z]`）开头和结尾， 带有破折号（`-`），下划线（`_`），点（ `.`）和之间的字母数字。 前缀是可选的。如果指定，前缀必须是 DNS 子域：由点（`.`）分隔的一系列 DNS 标签，总共不超过 253 个字符， 后跟斜杠（`/`）。

如果省略前缀，则假定标签键对用户是私有的。 向最终用户对象添加标签的自动系统组件（例如 `kube-scheduler`、`kube-controller-manager`、 `kube-apiserver`、`kubectl` 或其他第三方自动化工具）必须指定前缀。

`kubernetes.io/` 和 `k8s.io/` 前缀是为 Kubernetes 核心组件[保留的](https://kubernetes.io/zh/docs/reference/labels-annotations-taints/)。

有效标签值：

- 必须为 63 个字符或更少（可以为空）
- 除非标签值为空，必须以字母数字字符（`[a-z0-9A-Z]`）开头和结尾
- 包含破折号（`-`）、下划线（`_`）、点（`.`）和字母或数字。

> 注意字符数要小于63

## 标签选择算符 

与[名称和 UID](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/names/) 不同， 标签不支持唯一性。通常，我们希望许多对象携带相同的标签。

通过 *标签选择算符*，客户端/用户可以识别一组对象。标签选择算符是 Kubernetes 中的核心分组原语。

API 目前支持两种类型的选择算符：*基于等值的* 和 *基于集合的*。 标签选择算符可以由逗号分隔的多个 *需求* 组成。 在多个需求的情况下，必须满足所有要求，因此逗号分隔符充当逻辑 *与*（`&&`）运算符。

空标签选择算符或者未指定的选择算符的语义取决于上下文， 支持使用选择算符的 API 类别应该将算符的合法性和含义用文档记录下来。

**说明：** 对于某些 API 类别（例如 ReplicaSet）而言，两个实例的标签选择算符不得在命名空间内重叠， 否则它们的控制器将互相冲突，无法确定应该存在的副本个数。

**注意：** 对于基于等值的和基于集合的条件而言，不存在逻辑或（`||`）操作符。 你要确保你的过滤语句按合适的方式组织。

### *基于等值的* 需求

*基于等值* 或 *基于不等值* 的需求允许按标签键和值进行过滤。 匹配对象必须满足所有指定的标签约束，尽管它们也可能具有其他标签。 可接受的运算符有`=`、`==` 和 `!=` 三种。 前两个表示 *相等*（并且只是同义词），而后者表示 *不相等*。例如：

```
environment = production
tier != frontend
```

前者选择所有资源，其键名等于 `environment`，值等于 `production`。 后者选择所有资源，其键名等于 `tier`，值不同于 `frontend`，所有资源都没有带有 `tier` 键的标签。 可以使用逗号运算符来过滤 `production` 环境中的非 `frontend` 层资源：`environment=production,tier!=frontend`。

基于等值的标签要求的一种使用场景是 Pod 要指定节点选择标准。 例如，下面的示例 Pod 选择带有标签 "`accelerator=nvidia-tesla-p100`"。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cuda-test
spec:
  containers:
    - name: cuda-test
      image: "k8s.gcr.io/cuda-vector-add:v0.1"
      resources:
        limits:
          nvidia.com/gpu: 1
  nodeSelector:
    accelerator: nvidia-tesla-p100
```

> nodeselect

### *基于集合* 的需求

*基于集合* 的标签需求允许你通过一组值来过滤键。 支持三种操作符：`in`、`notin` 和 `exists` (只可以用在键标识符上)。例如：

```
environment in (production, qa)
tier notin (frontend, backend)
partition
!partition
```

- 第一个示例选择了所有键等于 `environment` 并且值等于 `production` 或者 `qa` 的资源。
- 第二个示例选择了所有键等于 `tier` 并且值不等于 `frontend` 或者 `backend` 的资源，以及所有没有 `tier` 键标签的资源。
- 第三个示例选择了所有包含了有 `partition` 标签的资源；没有校验它的值。
- 第四个示例选择了所有没有 `partition` 标签的资源；没有校验它的值。 类似地，逗号分隔符充当 *与* 运算符。因此，使用 `partition` 键（无论为何值）和 `environment` 不同于 `qa` 来过滤资源可以使用 `partition, environment notin（qa)` 来实现。

*基于集合* 的标签选择算符是相等标签选择算符的一般形式，因为 `environment=production` 等同于 `environment in（production）`；`!=` 和 `notin` 也是类似的。

*基于集合* 的要求可以与基于 *相等* 的要求混合使用。例如：`partition in (customerA, customerB),environment!=qa`。

## API

### LIST 和 WATCH 过滤

LIST 和 WATCH 操作可以使用查询参数指定标签选择算符过滤一组对象。 两种需求都是允许的。（这里显示的是它们出现在 URL 查询字符串中）

- *基于等值* 的需求: `?labelSelector=environment%3Dproduction,tier%3Dfrontend`
- *基于集合* 的需求: `?labelSelector=environment+in+%28production%2Cqa%29%2Ctier+in+%28frontend%29`

两种标签选择算符都可以通过 REST 客户端用于 list 或者 watch 资源。 例如，使用 `kubectl` 定位 `apiserver`，可以使用 *基于等值* 的标签选择算符可以这么写：

```shell
kubectl get pods -l environment=production,tier=frontend
```

或者使用 *基于集合的* 需求：

```shell
kubectl get pods -l 'environment in (production),tier in (frontend)'
```

正如刚才提到的，*基于集合* 的需求更具有表达力。例如，它们可以实现值的 *或* 操作：

```shell
kubectl get pods -l 'environment in (production, qa)'
```

或者通过 *exists* 运算符限制不匹配：

```shell
kubectl get pods -l 'environment,environment notin (frontend)'
```

### 在 API 对象中设置引用

一些 Kubernetes 对象，例如 [`services`](https://kubernetes.io/zh/docs/concepts/services-networking/service/) 和 [`replicationcontrollers`](https://kubernetes.io/zh/docs/concepts/workloads/controllers/replicationcontroller/) ， 也使用了标签选择算符去指定了其他资源的集合，例如 [pods](https://kubernetes.io/zh/docs/concepts/workloads/pods/)。

#### Service 和 ReplicationController

一个 `Service` 指向的一组 Pods 是由标签选择算符定义的。同样，一个 `ReplicationController` 应该管理的 pods 的数量也是由标签选择算符定义的。

两个对象的标签选择算符都是在 `json` 或者 `yaml` 文件中使用映射定义的，并且只支持 *基于等值* 需求的选择算符：

```json
"selector": {
    "component" : "redis",
}
```

或者

```yaml
selector:
    component: redis
```

这个选择算符(分别在 `json` 或者 `yaml` 格式中) 等价于 `component=redis` 或 `component in (redis)` 。

#### 支持基于集合需求的资源

比较新的资源，例如 [`Job`](https://kubernetes.io/zh/docs/concepts/workloads/controllers/job/)、 [`Deployment`](https://kubernetes.io/zh/docs/concepts/workloads/controllers/deployment/)、 [`Replica Set`](https://kubernetes.io/zh/docs/concepts/workloads/controllers/replicaset/) 和 [`DaemonSet`](https://kubernetes.io/zh/docs/concepts/workloads/controllers/daemonset/) ， 也支持 *基于集合的* 需求。

```yaml
selector:
  matchLabels:
    component: redis
  matchExpressions:
    - {key: tier, operator: In, values: [cache]}
    - {key: environment, operator: NotIn, values: [dev]}
```

`matchLabels` 是由 `{key,value}` 对组成的映射。 `matchLabels` 映射中的单个 `{key,value }` 等同于 `matchExpressions` 的元素， 其 `key` 字段为 "key"，`operator` 为 "In"，而 `values` 数组仅包含 "value"。 `matchExpressions` 是 Pod 选择算符需求的列表。 有效的运算符包括 `In`、`NotIn`、`Exists` 和 `DoesNotExist`。 在 `In` 和 `NotIn` 的情况下，设置的值必须是非空的。 来自 `matchLabels` 和 `matchExpressions` 的所有要求都按逻辑与的关系组合到一起 -- 它们必须都满足才能匹配。

#### 选择节点集

通过标签进行选择的一个用例是确定节点集，方便 Pod 调度。 有关更多信息，请参阅[选择节点](https://kubernetes.io/zh/docs/concepts/scheduling-eviction/assign-pod-node/)文档。

# 注解

你可以使用 Kubernetes 注解为对象附加任意的非标识的元数据。客户端程序（例如工具和库）能够获取这些元数据信息。

## 为对象附加元数据

你可以使用标签或注解将元数据附加到 Kubernetes 对象。 标签可以用来选择对象和查找满足某些条件的对象集合。 相反，注解不用于标识和选择对象。 注解中的元数据，可以很小，也可以很大，可以是结构化的，也可以是非结构化的，能够包含标签不允许的字符。

> 注意：注解不用于标识和选择对象。

注解和标签一样，是键/值对:

```json
"metadata": {
  "annotations": {
    "key1" : "value1",
    "key2" : "value2"
  }
}
```

**说明：**

Map 中的键和值必须是字符串。 换句话说，你不能使用数字、布尔值、列表或其他类型的键或值。

以下是一些例子，用来说明哪些信息可以使用注解来记录:

- 由声明性配置所管理的字段。 将这些字段附加为注解，能够将它们与客户端或服务端设置的默认值、 自动生成的字段以及通过自动调整大小或自动伸缩系统设置的字段区分开来。
- 构建、发布或镜像信息（如时间戳、发布 ID、Git 分支、PR 数量、镜像哈希、仓库地址）。
- 指向日志记录、监控、分析或审计仓库的指针。

- 可用于调试目的的客户端库或工具信息：例如，名称、版本和构建信息。
- 用户或者工具/系统的来源信息，例如来自其他生态系统组件的相关对象的 URL。
- 轻量级上线工具的元数据信息：例如，配置或检查点。
- 负责人员的电话或呼机号码，或指定在何处可以找到该信息的目录条目，如团队网站。
- 从用户到最终运行的指令，以修改行为或使用非标准功能。

你可以将这类信息存储在外部数据库或目录中而不使用注解， 但这样做就使得开发人员很难生成用于部署、管理、自检的客户端共享库和工具。

## 语法和字符集

*注解（Annotations）* 存储的形式是键/值对。有效的注解键分为两部分： 可选的前缀和名称，以斜杠（`/`）分隔。 名称段是必需项，并且必须在63个字符以内，以字母数字字符（`[a-z0-9A-Z]`）开头和结尾， 并允许使用破折号（`-`），下划线（`_`），点（`.`）和字母数字。 前缀是可选的。如果指定，则前缀必须是DNS子域：一系列由点（`.`）分隔的DNS标签， 总计不超过253个字符，后跟斜杠（`/`）。 如果省略前缀，则假定注解键对用户是私有的。 由系统组件添加的注解 （例如，`kube-scheduler`，`kube-controller-manager`，`kube-apiserver`，`kubectl` 或其他第三方组件），必须为终端用户添加注解前缀。

`kubernetes.io/` 和 `k8s.io/` 前缀是为Kubernetes核心组件保留的。

例如，下面是一个 Pod 的配置文件，其注解中包含 `imageregistry: https://hub.docker.com/`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: annotations-demo
  annotations:
    imageregistry: "https://hub.docker.com/"
spec:
  containers:
  - name: nginx
    image: nginx:1.7.9
    ports:
    - containerPort: 80
```

# Finalizers

Finalizer 是带有命名空间的键，告诉 Kubernetes 等到特定的条件被满足后， 再完全删除被标记为删除的资源。 Finalizer 提醒[控制器](https://kubernetes.io/zh/docs/concepts/architecture/controller/)清理被删除的对象拥有的资源。

当你告诉 Kubernetes 删除一个指定了 Finalizer 的对象时， Kubernetes API 会将该对象标记为删除，使其进入只读状态。 此时控制平面或其他组件会采取 Finalizer 所定义的行动， 而目标对象仍然处于终止中（Terminating）的状态。 这些行动完成后，控制器会删除目标对象相关的 Finalizer。 当 `metadata.finalizers` 字段为空时，Kubernetes 认为删除已完成。

你可以使用 Finalizer 控制资源的[垃圾收集](https://kubernetes.io/zh/docs/concepts/workloads/controllers/garbage-collection/)。 例如，你可以定义一个 Finalizer，在删除目标资源前清理相关资源或基础设施。

你可以通过使用 Finalizers 提醒[控制器](https://kubernetes.io/zh/docs/concepts/architecture/controller/) 在删除目标资源前执行特定的清理任务， 来控制资源的[垃圾收集](https://kubernetes.io/zh/docs/concepts/workloads/controllers/garbage-collection/)。

Finalizers 通常不指定要执行的代码。 相反，它们通常是特定资源上的键的列表，类似于注解。 Kubernetes 自动指定了一些 Finalizers，但你也可以指定你自己的。

## Finalizers 如何工作 

当你使用清单文件创建资源时，你可以在 `metadata.finalizers` 字段指定 Finalizers。 当你试图删除该资源时，管理该资源的控制器会注意到 `finalizers` 字段中的值， 并进行以下操作：

- 修改对象，将你开始执行删除的时间添加到 `metadata.deletionTimestamp` 字段。
- 将该对象标记为只读，直到其 `metadata.finalizers` 字段为空。

然后，控制器试图满足资源的 Finalizers 的条件。 每当一个 Finalizer 的条件被满足时，控制器就会从资源的 `finalizers` 字段中删除该键。 当该字段为空时，垃圾收集继续进行。 你也可以使用 Finalizers 来阻止删除未被管理的资源。

一个常见的 Finalizer 的例子是 `kubernetes.io/pv-protection`， 它用来防止意外删除 `PersistentVolume` 对象。 当一个 `PersistentVolume` 对象被 Pod 使用时， Kubernetes 会添加 `pv-protection` Finalizer。 如果你试图删除 `PersistentVolume`，它将进入 `Terminating` 状态， 但是控制器因为该 Finalizer 存在而无法删除该资源。 当 Pod 停止使用 `PersistentVolume` 时， Kubernetes 清除 `pv-protection` Finalizer，控制器就会删除该卷。

## 属主引用、标签和 Finalizers

与[标签](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/labels/)类似， [属主引用](https://kubernetes.io/zh/concepts/overview/working-with-objects/owners-dependents/) 描述了 Kubernetes 中对象之间的关系，但它们作用不同。 当一个[控制器](https://kubernetes.io/zh/docs/concepts/architecture/controller/) 管理类似于 Pod 的对象时，它使用标签来跟踪相关对象组的变化。 例如，当 [Job](https://kubernetes.io/zh/docs/concepts/workloads/controllers/job/) 创建一个或多个 Pod 时， Job 控制器会给这些 Pod 应用上标签，并跟踪集群中的具有相同标签的 Pod 的变化。

Job 控制器还为这些 Pod 添加了*属主引用*，指向创建 Pod 的 Job。 如果你在这些 Pod 运行的时候删除了 Job， Kubernetes 会使用属主引用（而不是标签）来确定集群中哪些 Pod 需要清理。

当 Kubernetes 识别到要删除的资源上的属主引用时，它也会处理 Finalizers。

在某些情况下，Finalizers 会阻止依赖对象的删除， 这可能导致目标属主对象，保持在只读状态的时间比预期的长，且没有被完全删除。 在这些情况下，你应该检查目标属主和附属对象上的 Finalizers 和属主引用，来排查原因。

**说明：**

在对象卡在删除状态的情况下，尽量避免手动移除 Finalizers，以允许继续删除操作。 Finalizers 通常因为特殊原因被添加到资源上，所以强行删除它们会导致集群出现问题。

https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/field-selectors/
