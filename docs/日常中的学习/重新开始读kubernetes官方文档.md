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

# 字段选择器

*字段选择器（Field selectors*）允许你根据一个或多个资源字段的值 [筛选 Kubernetes 资源](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/kubernetes-objects)。 下面是一些使用字段选择器查询的例子：

- `metadata.name=my-service`
- `metadata.namespace!=default`
- `status.phase=Pending`

下面这个 `kubectl` 命令将筛选出 [`status.phase`](https://kubernetes.io/zh/docs/concepts/workloads/pods/pod-lifecycle/#pod-phase) 字段值为 `Running` 的所有 Pod：

```shell
kubectl get pods --field-selector status.phase=Running
```

**说明：**

字段选择器本质上是资源*过滤器（Filters）*。默认情况下，字段选择器/过滤器是未被应用的， 这意味着指定类型的所有资源都会被筛选出来。 这使得以下的两个 `kubectl` 查询是等价的：

```shell
kubectl get pods
kubectl get pods --field-selector ""
```

## 支持的字段 

不同的 Kubernetes 资源类型支持不同的字段选择器。 所有资源类型都支持 `metadata.name` 和 `metadata.namespace` 字段。 使用不被支持的字段选择器会产生错误。例如：

```shell
kubectl get ingress --field-selector foo.bar=baz
Error from server (BadRequest): Unable to find "ingresses" that match label selector "", field selector "foo.bar=baz": "foo.bar" is not a known field selector: only "metadata.name", "metadata.namespace"
```

## 支持的操作符 

你可在字段选择器中使用 `=`、`==`和 `!=` （`=` 和 `==` 的意义是相同的）操作符。 例如，下面这个 `kubectl` 命令将筛选所有不属于 `default` 命名空间的 Kubernetes 服务：

```shell
kubectl get services  --all-namespaces --field-selector metadata.namespace!=default
```

## 链式选择器 

同[标签](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/labels/)和其他选择器一样， 字段选择器可以通过使用逗号分隔的列表组成一个选择链。 下面这个 `kubectl` 命令将筛选 `status.phase` 字段不等于 `Running` 同时 `spec.restartPolicy` 字段等于 `Always` 的所有 Pod：

```shell
kubectl get pods --field-selector=status.phase!=Running,spec.restartPolicy=Always
```

## 多种资源类型 

你能够跨多种资源类型来使用字段选择器。 下面这个 `kubectl` 命令将筛选出所有不在 `default` 命名空间中的 StatefulSet 和 Service：

```shell
kubectl get statefulsets,services --all-namespaces --field-selector metadata.namespace!=default
```

# 属主与附属

在 Kubernetes 中，一些对象是其他对象的*属主（Owner）*。 例如，[ReplicaSet](https://kubernetes.io/zh/docs/concepts/workloads/controllers/replicaset/) 是一组 Pod 的属主。 具有属主的对象是属主的*附属（Dependent）* 。

属主关系不同于一些资源使用的[标签和选择算符](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/labels/)机制。 例如，有一个创建 `EndpointSlice` 对象的 Service， 该 Service 使用标签来让控制平面确定，哪些 `EndpointSlice` 对象属于该 Service。 除开标签，每个代表 Service 所管理的 `EndpointSlice` 都有一个属主引用。 属主引用避免 Kubernetes 的不同部分干扰到不受它们控制的对象。

## 对象规约中的属主引用 

附属对象有一个 `metadata.ownerReferences` 字段，用于引用其属主对象。 一个有效的属主引用，包含与附属对象同在一个命名空间下的对象名称和一个 UID。 Kubernetes 自动为一些对象的附属资源设置属主引用的值， 这些对象包含 ReplicaSet、DaemonSet、Deployment、Job、CronJob、ReplicationController 等。 你也可以通过改变这个字段的值，来手动配置这些关系。 然而，你通常不需要这么做，你可以让 Kubernetes 自动管理附属关系。

附属对象还有一个 `ownerReferences.blockOwnerDeletion` 字段，该字段使用布尔值， 用于控制特定的附属对象是否可以阻止垃圾收集删除其属主对象。 如果[控制器](https://kubernetes.io/zh/docs/concepts/architecture/controller/)（例如 Deployment 控制器） 设置了 `metadata.ownerReferences` 字段的值，Kubernetes 会自动设置 `blockOwnerDeletion` 的值为 `true`。 你也可以手动设置 `blockOwnerDeletion` 字段的值，以控制哪些附属对象会阻止垃圾收集。

**说明：**

根据设计，kubernetes 不允许跨名字空间指定属主。 名字空间范围的附属可以指定集群范围的或者名字空间范围的属主。 名字空间范围的属主**必须**和该附属处于相同的名字空间。 如果名字空间范围的属主和附属不在相同的名字空间，那么该属主引用就会被认为是缺失的， 并且当附属的所有属主引用都被确认不再存在之后，该附属就会被删除。

集群范围的附属只能指定集群范围的属主。 在 v1.20+ 版本，如果一个集群范围的附属指定了一个名字空间范围类型的属主， 那么该附属就会被认为是拥有一个不可解析的属主引用，并且它不能够被垃圾回收。

在 v1.20+ 版本，如果垃圾收集器检测到无效的跨名字空间的属主引用， 或者一个集群范围的附属指定了一个名字空间范围类型的属主， 那么它就会报告一个警告事件。该事件的原因是 `OwnerRefInvalidNamespace`， `involvedObject` 属性中包含无效的附属。 你可以运行 `kubectl get events -A --field-selector=reason=OwnerRefInvalidNamespace` 来获取该类型的事件。

## 属主关系与 Finalizer 

当你告诉 Kubernetes 删除一个资源，API 服务器允许管理控制器处理该资源的任何 [Finalizer 规则](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/finalizers/)。 [Finalizer](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/finalizers/) 防止意外删除你的集群所依赖的、用于正常运作的资源。 例如，如果你试图删除一个仍被 Pod 使用的 `PersistentVolume`，该资源不会被立即删除， 因为 `PersistentVolume` 有 `kubernetes.io/pv-protection` Finalizer。 相反，它将进入 `Terminating` 状态，直到 Kubernetes 清除这个 Finalizer， 而这种情况只会发生在 `PersistentVolume` 不再被挂载到 Pod 上时。

当你使用[前台或孤立级联删除](https://kubernetes.io/zh/docs/concepts/architecture/garbage-collection/#cascading-deletion)时， Kubernetes 也会向属主资源添加 Finalizer。 在前台删除中，会添加 `foreground` Finalizer，这样控制器必须在删除了拥有 `ownerReferences.blockOwnerDeletion=true` 的附属资源后，才能删除属主对象。 如果你指定了孤立删除策略，Kubernetes 会添加 `orphan` Finalizer， 这样控制器在删除属主对象后，会忽略附属资源。

# 推荐使用的标签

除了 kubectl 和 dashboard 之外，您可以使用其他工具来可视化和管理 Kubernetes 对象。一组通用的标签可以让多个工具之间相互操作，用所有工具都能理解的通用方式描述对象。

除了支持工具外，推荐的标签还以一种可以查询的方式描述了应用程序。

元数据围绕 *应用（application）* 的概念进行组织。Kubernetes 不是 平台即服务（PaaS），没有或强制执行正式的应用程序概念。 相反，应用程序是非正式的，并使用元数据进行描述。应用程序包含的定义是松散的。

**说明：**

这些是推荐的标签。它们使管理应用程序变得更容易但不是任何核心工具所必需的。

共享标签和注解都使用同一个前缀：`app.kubernetes.io`。没有前缀的标签是用户私有的。共享前缀可以确保共享标签不会干扰用户自定义的标签。

## 标签

为了充分利用这些标签，应该在每个资源对象上都使用它们。

| 键                             | 描述                                               | 示例                 | 类型   |
| ------------------------------ | -------------------------------------------------- | -------------------- | ------ |
| `app.kubernetes.io/name`       | 应用程序的名称                                     | `mysql`              | 字符串 |
| `app.kubernetes.io/instance`   | 用于唯一确定应用实例的名称                         | `mysql-abcxzy`       | 字符串 |
| `app.kubernetes.io/version`    | 应用程序的当前版本（例如，语义版本，修订版哈希等） | `5.7.21`             | 字符串 |
| `app.kubernetes.io/component`  | 架构中的组件                                       | `database`           | 字符串 |
| `app.kubernetes.io/part-of`    | 此级别的更高级别应用程序的名称                     | `wordpress`          | 字符串 |
| `app.kubernetes.io/managed-by` | 用于管理应用程序的工具                             | `helm`               | 字符串 |
| `app.kubernetes.io/created-by` | 创建该资源的控制器或者用户                         | `controller-manager` | 字符串 |

为说明这些标签的实际使用情况，请看下面的 StatefulSet 对象：

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app.kubernetes.io/name: mysql
    app.kubernetes.io/instance: mysql-abcxzy
    app.kubernetes.io/version: "5.7.21"
    app.kubernetes.io/component: database
    app.kubernetes.io/part-of: wordpress
    app.kubernetes.io/managed-by: helm
    app.kubernetes.io/created-by: controller-manager
```

## 应用和应用实例

应用可以在 Kubernetes 集群中安装一次或多次。在某些情况下，可以安装在同一命名空间中。例如，可以不止一次地为不同的站点安装不同的 WordPress。

应用的名称和实例的名称是分别记录的。例如，WordPress 应用的 `app.kubernetes.io/name` 为 `wordpress`，而其实例名称 `app.kubernetes.io/instance` 为 `wordpress-abcxzy`。 这使得应用和应用的实例均可被识别，应用的每个实例都必须具有唯一的名称。

## 示例

为了说明使用这些标签的不同方式，以下示例具有不同的复杂性。

### 一个简单的无状态服务

考虑使用 `Deployment` 和 `Service` 对象部署的简单无状态服务的情况。以下两个代码段表示如何以最简单的形式使用标签。

下面的 `Deployment` 用于监督运行应用本身的 pods。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/name: myservice
    app.kubernetes.io/instance: myservice-abcxzy
...
```

下面的 `Service` 用于暴露应用。

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/name: myservice
    app.kubernetes.io/instance: myservice-abcxzy
...
```

### 带有一个数据库的 Web 应用程序

考虑一个稍微复杂的应用：一个使用 Helm 安装的 Web 应用（WordPress），其中 使用了数据库（MySQL）。以下代码片段说明用于部署此应用程序的对象的开始。

以下 `Deployment` 的开头用于 WordPress：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/name: wordpress
    app.kubernetes.io/instance: wordpress-abcxzy
    app.kubernetes.io/version: "4.9.4"
    app.kubernetes.io/managed-by: helm
    app.kubernetes.io/component: server
    app.kubernetes.io/part-of: wordpress
...
```

这个 `Service` 用于暴露 WordPress：

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/name: wordpress
    app.kubernetes.io/instance: wordpress-abcxzy
    app.kubernetes.io/version: "4.9.4"
    app.kubernetes.io/managed-by: helm
    app.kubernetes.io/component: server
    app.kubernetes.io/part-of: wordpress
...
```

MySQL 作为一个 `StatefulSet` 暴露，包含它和它所属的较大应用程序的元数据：

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app.kubernetes.io/name: mysql
    app.kubernetes.io/instance: mysql-abcxzy
    app.kubernetes.io/version: "5.7.21"
    app.kubernetes.io/managed-by: helm
    app.kubernetes.io/component: database
    app.kubernetes.io/part-of: wordpress
...
```

`Service` 用于将 MySQL 作为 WordPress 的一部分暴露：

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/name: mysql
    app.kubernetes.io/instance: mysql-abcxzy
    app.kubernetes.io/version: "5.7.21"
    app.kubernetes.io/managed-by: helm
    app.kubernetes.io/component: database
    app.kubernetes.io/part-of: wordpress
...
```

使用 MySQL `StatefulSet` 和 `Service`，您会注意到有关 MySQL 和 Wordpress 的信息，包括更广泛的应用程序。

https://kubernetes.io/zh/docs/tasks/administer-cluster/migrating-from-dockershim/

# 节点

Kubernetes 通过将容器放入在节点（Node）上运行的 Pod 中来执行你的工作负载。 节点可以是一个虚拟机或者物理机器，取决于所在的集群配置。 每个节点包含运行 [Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/) 所需的服务； 这些节点由 [控制面](https://kubernetes.io/zh/docs/reference/glossary/?all=true#term-control-plane) 负责管理。

通常集群中会有若干个节点；而在一个学习用或者资源受限的环境中，你的集群中也可能 只有一个节点。

节点上的[组件](https://kubernetes.io/zh/docs/concepts/overview/components/#node-components)包括 [kubelet](https://kubernetes.io/docs/reference/generated/kubelet)、 [容器运行时](https://kubernetes.io/zh/docs/setup/production-environment/container-runtimes)以及 [kube-proxy](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kube-proxy/)。

## 管理 

向 [API 服务器](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kube-apiserver/)添加节点的方式主要有两种：

1. 节点上的 `kubelet` 向控制面执行自注册；
2. 你，或者别的什么人，手动添加一个 Node 对象。

在你创建了 Node 对象或者节点上的 `kubelet` 执行了自注册操作之后， 控制面会检查新的 Node 对象是否合法。例如，如果你使用下面的 JSON 对象来创建 Node 对象：

```json
{
  "kind": "Node",
  "apiVersion": "v1",
  "metadata": {
    "name": "10.240.79.157",
    "labels": {
      "name": "my-first-k8s-node"
    }
  }
}
```

Kubernetes 会在内部创建一个 Node 对象作为节点的表示。Kubernetes 检查 `kubelet` 向 API 服务器注册节点时使用的 `metadata.name` 字段是否匹配。 如果节点是健康的（即所有必要的服务都在运行中），则该节点可以用来运行 Pod。 否则，直到该节点变为健康之前，所有的集群活动都会忽略该节点。

**说明：** Kubernetes 会一直保存着非法节点对应的对象，并持续检查该节点是否已经 变得健康。 你，或者某个[控制器](https://kubernetes.io/zh/docs/concepts/architecture/controller/)必需显式地 删除该 Node 对象以停止健康检查操作。

Node 对象的名称必须是合法的 [DNS 子域名](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/names#dns-subdomain-names)。

### 节点自注册

当 kubelet 标志 `--register-node` 为 true（默认）时，它会尝试向 API 服务注册自己。 这是首选模式，被绝大多数发行版选用。

对于自注册模式，kubelet 使用下列参数启动：

- `--kubeconfig` - 用于向 API 服务器表明身份的凭据路径。
- `--cloud-provider` - 与某[云驱动](https://kubernetes.io/zh/docs/reference/glossary/?all=true#term-cloud-provider) 进行通信以读取与自身相关的元数据的方式。
- `--register-node` - 自动向 API 服务注册。
- `--register-with-taints` - 使用所给的污点列表（逗号分隔的 `<key>=<value>:<effect>`）注册节点。 当 `register-node` 为 false 时无效。
- `--node-ip` - 节点 IP 地址。
- `--node-labels` - 在集群中注册节点时要添加的 [标签](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/labels/)。 （参见 [NodeRestriction 准入控制插件](https://kubernetes.io/zh/docs/reference/access-authn-authz/admission-controllers/#noderestriction)所实施的标签限制）。
- `--node-status-update-frequency` - 指定 kubelet 向控制面发送状态的频率。

启用[节点授权模式](https://kubernetes.io/zh/docs/reference/access-authn-authz/node/)和 [NodeRestriction 准入插件](https://kubernetes.io/zh/docs/reference/access-authn-authz/admission-controllers/#noderestriction) 时，仅授权 `kubelet` 创建或修改其自己的节点资源。

### 手动节点管理

你可以使用 [kubectl](https://kubernetes.io/docs/user-guide/kubectl-overview/) 来创建和修改 Node 对象。

如果你希望手动创建节点对象时，请设置 kubelet 标志 `--register-node=false`。

你可以修改 Node 对象（忽略 `--register-node` 设置）。 例如，修改节点上的标签或标记其为不可调度。

你可以结合使用节点上的标签和 Pod 上的选择算符来控制调度。 例如，你可以限制某 Pod 只能在符合要求的节点子集上运行。

如果标记节点为不可调度（unschedulable），将阻止新 Pod 调度到该节点之上，但不会 影响任何已经在其上的 Pod。 这是重启节点或者执行其他维护操作之前的一个有用的准备步骤。

要标记一个节点为不可调度，执行以下命令：

```shell
kubectl cordon $NODENAME
```

更多细节参考[安全腾空节点](https://kubernetes.io/zh/docs/tasks/administer-cluster/safely-drain-node/)。

**说明：** 被 [DaemonSet](https://kubernetes.io/zh/docs/concepts/workloads/controllers/daemonset/) 控制器创建的 Pod 能够容忍节点的不可调度属性。 DaemonSet 通常提供节点本地的服务，即使节点上的负载应用已经被腾空，这些服务也仍需 运行在节点之上。

## 节点状态 

一个节点的状态包含以下信息:

- [地址](https://kubernetes.io/zh/docs/concepts/architecture/nodes/#addresses)
- [状况](https://kubernetes.io/zh/docs/concepts/architecture/nodes/#condition)
- [容量与可分配](https://kubernetes.io/zh/docs/concepts/architecture/nodes/#capacity)
- [信息](https://kubernetes.io/zh/docs/concepts/architecture/nodes/#info)

你可以使用 `kubectl` 来查看节点状态和其他细节信息：

```shell
kubectl describe node <节点名称>
```

下面对每个部分进行详细描述。

### 地址 

这些字段的用法取决于你的云服务商或者物理机配置。

- HostName：由节点的内核设置。可以通过 kubelet 的 `--hostname-override` 参数覆盖。
- ExternalIP：通常是节点的可外部路由（从集群外可访问）的 IP 地址。
- InternalIP：通常是节点的仅可在集群内部路由的 IP 地址。

### 状况

`conditions` 字段描述了所有 `Running` 节点的状态。状况的示例包括：

| 节点状况             | 描述                                                         |
| -------------------- | ------------------------------------------------------------ |
| `Ready`              | 如节点是健康的并已经准备好接收 Pod 则为 `True`；`False` 表示节点不健康而且不能接收 Pod；`Unknown` 表示节点控制器在最近 `node-monitor-grace-period` 期间（默认 40 秒）没有收到节点的消息 |
| `DiskPressure`       | `True` 表示节点存在磁盘空间压力，即磁盘可用量低, 否则为 `False` |
| `MemoryPressure`     | `True` 表示节点存在内存压力，即节点内存可用量低，否则为 `False` |
| `PIDPressure`        | `True` 表示节点存在进程压力，即节点上进程过多；否则为 `False` |
| `NetworkUnavailable` | `True` 表示节点网络配置不正确；否则为 `False`                |

**说明：** 如果使用命令行工具来打印已保护（Cordoned）节点的细节，其中的 Condition 字段可能 包括 `SchedulingDisabled`。`SchedulingDisabled` 不是 Kubernetes API 中定义的 Condition，被保护起来的节点在其规约中被标记为不可调度（Unschedulable）。

在 Kubernetes API 中，节点的状况表示节点资源中`.status` 的一部分。 例如，以下 JSON 结构描述了一个健康节点：

```json
"conditions": [
  {
    "type": "Ready",
    "status": "True",
    "reason": "KubeletReady",
    "message": "kubelet is posting ready status",
    "lastHeartbeatTime": "2019-06-05T18:38:35Z",
    "lastTransitionTime": "2019-06-05T11:41:27Z"
  }
]
```

如果 Ready 条件的 `status` 处于 `Unknown` 或者 `False` 状态的时间超过了 `pod-eviction-timeout` 值， （一个传递给 [kube-controller-manager](https://kubernetes.io/docs/reference/generated/kube-controller-manager/) 的参数）， [节点控制器](https://kubernetes.io/zh/docs/concepts/architecture/nodes/#node-controller) 会对节点上的所有 Pod 触发 [API-发起的驱逐](https://kubernetes.io/zh/docs/concepts/scheduling-eviction/pod-eviction/#api-eviction)。 默认的逐出超时时长为 **5 分钟**。 某些情况下，当节点不可达时，API 服务器不能和其上的 kubelet 通信。 删除 Pod 的决定不能传达给 kubelet，直到它重新建立和 API 服务器的连接为止。 与此同时，被计划删除的 Pod 可能会继续在游离的节点上运行。

节点控制器在确认 Pod 在集群中已经停止运行前，不会强制删除它们。 你可以看到这些可能在无法访问的节点上运行的 Pod 处于 `Terminating` 或者 `Unknown` 状态。 如果 kubernetes 不能基于下层基础设施推断出某节点是否已经永久离开了集群， 集群管理员可能需要手动删除该节点对象。 从 Kubernetes 删除节点对象将导致 API 服务器删除节点上所有运行的 Pod 对象并释放它们的名字。

节点生命周期控制器会自动创建代表状况的 [污点](https://kubernetes.io/zh/docs/concepts/scheduling-eviction/taint-and-toleration/)。 当调度器将 Pod 指派给某节点时，会考虑节点上的污点。 Pod 则可以通过容忍度（Toleration）表达所能容忍的污点。

### 容量与可分配

描述节点上的可用资源：CPU、内存和可以调度到节点上的 Pod 的个数上限。

`capacity` 块中的字段标示节点拥有的资源总量。 `allocatable` 块指示节点上可供普通 Pod 消耗的资源量。

可以在学习如何在节点上[预留计算资源](https://kubernetes.io/zh/docs/tasks/administer-cluster/reserve-compute-resources/#node-allocatable) 的时候了解有关容量和可分配资源的更多信息。

### 信息

描述节点的一般信息，如内核版本、Kubernetes 版本（`kubelet` 和 `kube-proxy` 版本）、 容器运行时详细信息，以及 节点使用的操作系统。 `kubelet` 从节点收集这些信息并将其发布到 Kubernetes API。

## 心跳 

Kubernetes 节点发送的心跳帮助你的集群确定每个节点的可用性，并在检测到故障时采取行动。

对于节点，有两种形式的心跳:

- 更新节点的 `.status`
- [Lease](https://kubernetes.io/docs/reference/kubernetes-api/cluster-resources/lease-v1/) 对象 在 `kube-node-lease` [命名空间](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/namespaces/)中。 每个节点都有一个关联的 Lease 对象。

与 Node 的 `.status` 更新相比，`Lease` 是一种轻量级资源。 使用 `Leases` 心跳在大型集群中可以减少这些更新对性能的影响。

kubelet 负责创建和更新节点的 `.status`，以及更新它们对应的 `Lease`。

- 当状态发生变化时，或者在配置的时间间隔内没有更新事件时，kubelet 会更新 `.status`。 `.status` 更新的默认间隔为 5 分钟（比不可达节点的 40 秒默认超时时间长很多）。
- `kubelet` 会每 10 秒（默认更新间隔时间）创建并更新其 `Lease` 对象。 `Lease` 更新独立于 `NodeStatus` 更新而发生。 如果 `Lease` 的更新操作失败，`kubelet` 会采用指数回退机制，从 200 毫秒开始 重试，最长重试间隔为 7 秒钟。

## 节点控制器 

节点[控制器](https://kubernetes.io/zh/docs/concepts/architecture/controller/)是 Kubernetes 控制面组件，管理节点的方方面面。

节点控制器在节点的生命周期中扮演多个角色。 第一个是当节点注册时为它分配一个 CIDR 区段（如果启用了 CIDR 分配）。

第二个是保持节点控制器内的节点列表与云服务商所提供的可用机器列表同步。 如果在云环境下运行，只要某节点不健康，节点控制器就会询问云服务是否节点的虚拟机仍可用。 如果不可用，节点控制器会将该节点从它的节点列表删除。

第三个是监控节点的健康状况。 节点控制器是负责：

- 在节点节点不可达的情况下，在 Node 的 `.status` 中更新 `NodeReady` 状况。 在这种情况下，节点控制器将 `NodeReady` 状况更新为 `ConditionUnknown` 。
- 如果节点仍然无法访问：对于不可达节点上的所有 Pod触发 [API-发起的逐出](https://kubernetes.io/zh/docs/concepts/scheduling-eviction/api-eviction/)。 默认情况下，节点控制器 在将节点标记为 `ConditionUnknown` 后等待 5 分钟 提交第一个驱逐请求。

节点控制器每隔 `--node-monitor-period` 秒检查每个节点的状态。

### 逐出速率限制 

大部分情况下，节点控制器把逐出速率限制在每秒 `--node-eviction-rate` 个（默认为 0.1）。 这表示它每 10 秒钟内至多从一个节点驱逐 Pod。

当一个可用区域（Availability Zone）中的节点变为不健康时，节点的驱逐行为将发生改变。 节点控制器会同时检查可用区域中不健康（NodeReady 状况为 `ConditionUnknown` 或 `ConditionFalse`） 的节点的百分比：

- 如果不健康节点的比例超过 `--unhealthy-zone-threshold` （默认为 0.55）， 驱逐速率将会降低。
- 如果集群较小（意即小于等于 `--large-cluster-size-threshold` 个节点 - 默认为 50），驱逐操作将会停止。
- 否则驱逐速率将降为每秒 `--secondary-node-eviction-rate` 个（默认为 0.01）。

在单个可用区域实施这些策略的原因是当一个可用区域可能从控制面脱离时其它可用区域 可能仍然保持连接。 如果你的集群没有跨越云服务商的多个可用区域，那（整个集群）就只有一个可用区域。

跨多个可用区域部署你的节点的一个关键原因是当某个可用区域整体出现故障时， 工作负载可以转移到健康的可用区域。 因此，如果一个可用区域中的所有节点都不健康时，节点控制器会以正常的速率 `--node-eviction-rate` 进行驱逐操作。 在所有的可用区域都不健康（也即集群中没有健康节点）的极端情况下， 节点控制器将假设控制面与节点间的连接出了某些问题，它将停止所有驱逐动作（如果故障后部分节点重新连接， 节点控制器会从剩下不健康或者不可达节点中驱逐 `pods`）。

节点控制器还负责驱逐运行在拥有 `NoExecute` 污点的节点上的 Pod， 除非这些 Pod 能够容忍此污点。 节点控制器还负责根据节点故障（例如节点不可访问或没有就绪）为其添加 [污点](https://kubernetes.io/zh/docs/concepts/scheduling-eviction/taint-and-toleration/)。 这意味着调度器不会将 Pod 调度到不健康的节点上。

### 资源容量跟踪 

Node 对象会跟踪节点上资源的容量（例如可用内存和 CPU 数量）。 通过[自注册](https://kubernetes.io/zh/docs/concepts/architecture/nodes/#self-registration-of-nodes)机制生成的 Node 对象会在注册期间报告自身容量。 如果你[手动](https://kubernetes.io/zh/docs/concepts/architecture/nodes/#manual-node-administration)添加了 Node，你就需要在添加节点时 手动设置节点容量。

Kubernetes [调度器](https://kubernetes.io/docs/reference/generated/kube-scheduler/)保证节点上 有足够的资源供其上的所有 Pod 使用。它会检查节点上所有容器的请求的总和不会超过节点的容量。 总的请求包括由 kubelet 启动的所有容器，但不包括由容器运行时直接启动的容器， 也不包括不受 `kubelet` 控制的其他进程。

**说明：** 如果要为非 Pod 进程显式保留资源。请参考 [为系统守护进程预留资源](https://kubernetes.io/zh/docs/tasks/administer-cluster/reserve-compute-resources/#system-reserved)。

## 节点拓扑 

**FEATURE STATE:** `Kubernetes v1.16 [alpha]`

如果启用了 `TopologyManager` [特性门控](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/feature-gates/)， `kubelet` 可以在作出资源分配决策时使用拓扑提示。 参考[控制节点上拓扑管理策略](https://kubernetes.io/zh/docs/tasks/administer-cluster/topology-manager/) 了解详细信息。

## 节点体面关闭

**FEATURE STATE:** `Kubernetes v1.21 [beta]`

kubelet 会尝试检测节点系统关闭事件并终止在节点上运行的 Pods。

在节点终止期间，kubelet 保证 Pod 遵从常规的 [Pod 终止流程](https://kubernetes.io/zh/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination)。

体面节点关闭特性依赖于 systemd，因为它要利用 [systemd 抑制器锁](https://www.freedesktop.org/wiki/Software/systemd/inhibit/) 在给定的期限内延迟节点关闭。

体面节点关闭特性受 `GracefulNodeShutdown` [特性门控](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/) 控制，在 1.21 版本中是默认启用的。

注意，默认情况下，下面描述的两个配置选项，`ShutdownGracePeriod` 和 `ShutdownGracePeriodCriticalPods` 都是被设置为 0 的，因此不会激活 体面节点关闭功能。 要激活此功能特性，这两个 kubelet 配置选项要适当配置，并设置为非零值。

在体面关闭节点过程中，kubelet 分两个阶段来终止 Pods：

1. 终止在节点上运行的常规 Pod。
2. 终止在节点上运行的[关键 Pod](https://kubernetes.io/zh/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/#marking-pod-as-critical)。

节点体面关闭的特性对应两个 [`KubeletConfiguration`](https://kubernetes.io/zh/docs/tasks/administer-cluster/kubelet-config-file/) 选项：

- ```
  ShutdownGracePeriod
  ```

  ：

  - 指定节点应延迟关闭的总持续时间。此时间是 Pod 体面终止的时间总和，不区分常规 Pod 还是 [关键 Pod](https://kubernetes.io/zh/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/#marking-pod-as-critical)。

- ```
  ShutdownGracePeriodCriticalPods
  ```

  ：

  - 在节点关闭期间指定用于终止 [关键 Pod](https://kubernetes.io/zh/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/#marking-pod-as-critical) 的持续时间。该值应小于 `ShutdownGracePeriod`。

例如，如果设置了 `ShutdownGracePeriod=30s` 和 `ShutdownGracePeriodCriticalPods=10s`， 则 kubelet 将延迟 30 秒关闭节点。 在关闭期间，将保留前 20（30 - 10）秒用于体面终止常规 Pod， 而保留最后 10 秒用于终止 [关键 Pod](https://kubernetes.io/zh/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/#marking-pod-as-critical)。

**说明：**

当 Pod 在正常节点关闭期间被驱逐时，它们会被标记为 `failed`。 运行 `kubectl get pods` 将被驱逐的 pod 的状态显示为 `Shutdown`。 并且 `kubectl describe pod` 表示 pod 因节点关闭而被驱逐：

```
Status:         Failed
Reason:         Shutdown
Message:        Node is shutting, evicting pods
```

`Failed` 的 pod 对象将被保留，直到被明确删除或 [由 GC 清理](https://kubernetes.io/zh/docs/concepts/workloads/pods/pod-lifecycle/#pod-garbage-collection)。 与突然的节点终止相比这是一种行为变化。

## 交换内存管理

**FEATURE STATE:** `Kubernetes v1.22 [alpha]`

在 Kubernetes 1.22 之前，节点不支持使用交换内存，并且 默认情况下，如果在节点上检测到交换内存配置，kubelet 将无法启动。 在 1.22 以后，可以在每个节点的基础上启用交换内存支持。

要在节点上启用交换内存，必须启用kubelet 的 `NodeSwap` 特性门控， 同时使用 `--fail-swap-on` 命令行参数或者将 `failSwapOn` [配置](https://kubernetes.io/zh/docs/reference/config-api/kubelet-config.v1beta1/#kubelet-config-k8s-io-v1beta1-KubeletConfiguration) 设置为false。

用户还可以选择配置 `memorySwap.swapBehavior` 以指定节点使用交换内存的方式。 例如:

```yaml
memorySwap:
  swapBehavior: LimitedSwap
```

已有的 `swapBehavior` 的配置选项有：

- `LimitedSwap`：Kubernetes 工作负载的交换内存会受限制。 不受 Kubernetes 管理的节点上的工作负载仍然可以交换。
- `UnlimitedSwap`：Kubernetes 工作负载可以使用尽可能多的交换内存 请求，一直到系统限制。

如果启用了特性门控但是未指定 `memorySwap` 的配置，默认情况下 kubelet 将使用 `LimitedSwap` 设置。

`LimitedSwap` 设置的行为还取决于节点运行的是 v1 还是 v2 的控制组（也就是 `cgroups`）：

- **cgroupsv1:** Kubernetes 工作负载可以使用内存和 交换，达到 pod 的内存限制（如果设置）。
- **cgroupsv2:** Kubernetes 工作负载不能使用交换内存。

如需更多信息以及协助测试和提供反馈，请 参见 [KEP-2400](https://github.com/kubernetes/enhancements/issues/2400) 及其 [设计方案](https://github.com/kubernetes/enhancements/blob/master/keps/sig-node/2400-node-swap/README.md)。

# 控制面到节点通信

本文列举控制面节点（确切说是 API 服务器）和 Kubernetes 集群之间的通信路径。 目的是为了让用户能够自定义他们的安装，以实现对网络配置的加固，使得集群能够在不可信的网络上 （或者在一个云服务商完全公开的 IP 上）运行。

## 节点到控制面

Kubernetes 采用的是中心辐射型（Hub-and-Spoke）API 模式。 所有从集群（或所运行的 Pods）发出的 API 调用都终止于 apiserver。 其它控制面组件都没有被设计为可暴露远程服务。 apiserver 被配置为在一个安全的 HTTPS 端口（通常为 443）上监听远程连接请求， 并启用一种或多种形式的客户端[身份认证](https://kubernetes.io/zh/docs/reference/access-authn-authz/authentication/)机制。 一种或多种客户端[鉴权机制](https://kubernetes.io/zh/docs/reference/access-authn-authz/authorization/)应该被启用， 特别是在允许使用[匿名请求](https://kubernetes.io/zh/docs/reference/access-authn-authz/authentication/#anonymous-requests) 或[服务账号令牌](https://kubernetes.io/zh/docs/reference/access-authn-authz/authentication/#service-account-tokens)的时候。

应该使用集群的公共根证书开通节点，这样它们就能够基于有效的客户端凭据安全地连接 apiserver。 一种好的方法是以客户端证书的形式将客户端凭据提供给 kubelet。 请查看 [kubelet TLS 启动引导](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kubelet-tls-bootstrapping/) 以了解如何自动提供 kubelet 客户端证书。

想要连接到 apiserver 的 Pod 可以使用服务账号安全地进行连接。 当 Pod 被实例化时，Kubernetes 自动把公共根证书和一个有效的持有者令牌注入到 Pod 里。 `kubernetes` 服务（位于 `default` 名字空间中）配置了一个虚拟 IP 地址，用于（通过 kube-proxy）转发 请求到 apiserver 的 HTTPS 末端。

控制面组件也通过安全端口与集群的 apiserver 通信。

这样，从集群节点和节点上运行的 Pod 到控制面的连接的缺省操作模式即是安全的， 能够在不可信的网络或公网上运行。

## 控制面到节点

从控制面（apiserver）到节点有两种主要的通信路径。 第一种是从 apiserver 到集群中每个节点上运行的 kubelet 进程。 第二种是从 apiserver 通过它的代理功能连接到任何节点、Pod 或者服务。

### API 服务器到 kubelet

从 apiserver 到 kubelet 的连接用于：

- 获取 Pod 日志
- 挂接（通过 kubectl）到运行中的 Pod
- 提供 kubelet 的端口转发功能。

这些连接终止于 kubelet 的 HTTPS 末端。 默认情况下，apiserver 不检查 kubelet 的服务证书。这使得此类连接容易受到中间人攻击， 在非受信网络或公开网络上运行也是 **不安全的**。

为了对这个连接进行认证，使用 `--kubelet-certificate-authority` 标志给 apiserver 提供一个根证书包，用于 kubelet 的服务证书。

如果无法实现这点，又要求避免在非受信网络或公共网络上进行连接，可在 apiserver 和 kubelet 之间使用 [SSH 隧道](https://kubernetes.io/zh/docs/concepts/architecture/control-plane-node-communication/#ssh-tunnels)。

最后，应该启用 [kubelet 用户认证和/或鉴权](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kubelet-authentication-authorization/) 来保护 kubelet API。

### apiserver 到节点、Pod 和服务

从 apiserver 到节点、Pod 或服务的连接默认为纯 HTTP 方式，因此既没有认证，也没有加密。 这些连接可通过给 API URL 中的节点、Pod 或服务名称添加前缀 `https:` 来运行在安全的 HTTPS 连接上。 不过这些连接既不会验证 HTTPS 末端提供的证书，也不会提供客户端证书。 因此，虽然连接是加密的，仍无法提供任何完整性保证。 这些连接 **目前还不能安全地** 在非受信网络或公共网络上运行。

### SSH 隧道

Kubernetes 支持使用 SSH 隧道来保护从控制面到节点的通信路径。在这种配置下，apiserver 建立一个到集群中各节点的 SSH 隧道（连接到在 22 端口监听的 SSH 服务） 并通过这个隧道传输所有到 kubelet、节点、Pod 或服务的请求。 这一隧道保证通信不会被暴露到集群节点所运行的网络之外。

SSH 隧道目前已被废弃。除非你了解个中细节，否则不应使用。 Konnectivity 服务是对此通信通道的替代品。

### Konnectivity 服务

**FEATURE STATE:** `Kubernetes v1.18 [beta]`

作为 SSH 隧道的替代方案，Konnectivity 服务提供 TCP 层的代理，以便支持从控制面到集群的通信。 Konnectivity 服务包含两个部分：Konnectivity 服务器和 Konnectivity 代理，分别运行在 控制面网络和节点网络中。Konnectivity 代理建立并维持到 Konnectivity 服务器的网络连接。 启用 Konnectivity 服务之后，所有控制面到节点的通信都通过这些连接传输。

请浏览 [Konnectivity 服务任务](https://kubernetes.io/zh/docs/tasks/extend-kubernetes/setup-konnectivity/) 在你的集群中配置 Konnectivity 服务。

# 控制器

在机器人技术和自动化领域，控制回路（Control Loop）是一个非终止回路，用于调节系统状态。

这是一个控制环的例子：房间里的温度自动调节器。

当你设置了温度，告诉了温度自动调节器你的*期望状态（Desired State）*。 房间的实际温度是*当前状态（Current State）*。 通过对设备的开关控制，温度自动调节器让其当前状态接近期望状态。

在 Kubernetes 中，控制器通过监控[集群](https://kubernetes.io/zh/docs/reference/glossary/?all=true#term-cluster) 的公共状态，并致力于将当前状态转变为期望的状态。

## 控制器模式

一个控制器至少追踪一种类型的 Kubernetes 资源。这些 [对象](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/kubernetes-objects/) 有一个代表期望状态的 `spec` 字段。 该资源的控制器负责确保其当前状态接近期望状态。

控制器可能会自行执行操作；在 Kubernetes 中更常见的是一个控制器会发送信息给 [API 服务器](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kube-apiserver/)，这会有副作用。 具体可参看后文的例子。

### 通过 API 服务器来控制

[Job](https://kubernetes.io/zh/docs/concepts/workloads/controllers/job/) 控制器是一个 Kubernetes 内置控制器的例子。 内置控制器通过和集群 API 服务器交互来管理状态。

Job 是一种 Kubernetes 资源，它运行一个或者多个 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/)， 来执行一个任务然后停止。 （一旦[被调度了](https://kubernetes.io/zh/docs/concepts/scheduling-eviction/)，对 `kubelet` 来说 Pod 对象就会变成了期望状态的一部分）。

在集群中，当 Job 控制器拿到新任务时，它会保证一组 Node 节点上的 `kubelet` 可以运行正确数量的 Pod 来完成工作。 Job 控制器不会自己运行任何的 Pod 或者容器。Job 控制器是通知 API 服务器来创建或者移除 Pod。 [控制面](https://kubernetes.io/zh/docs/reference/glossary/?all=true#term-control-plane)中的其它组件 根据新的消息作出反应（调度并运行新 Pod）并且最终完成工作。

创建新 Job 后，所期望的状态就是完成这个 Job。Job 控制器会让 Job 的当前状态不断接近期望状态：创建为 Job 要完成工作所需要的 Pod，使 Job 的状态接近完成。

控制器也会更新配置对象。例如：一旦 Job 的工作完成了，Job 控制器会更新 Job 对象的状态为 `Finished`。

（这有点像温度自动调节器关闭了一个灯，以此来告诉你房间的温度现在到你设定的值了）。

### 直接控制

相比 Job 控制器，有些控制器需要对集群外的一些东西进行修改。

例如，如果你使用一个控制回路来保证集群中有足够的 [节点](https://kubernetes.io/zh/docs/concepts/architecture/nodes/)，那么控制器就需要当前集群外的 一些服务在需要时创建新节点。

和外部状态交互的控制器从 API 服务器获取到它想要的状态，然后直接和外部系统进行通信 并使当前状态更接近期望状态。

（实际上有一个[控制器](https://github.com/kubernetes/autoscaler/) 可以水平地扩展集群中的节点。）

这里，很重要的一点是，控制器做出了一些变更以使得事物更接近你的期望状态， 之后将当前状态报告给集群的 API 服务器。 其他控制回路可以观测到所汇报的数据的这种变化并采取其各自的行动。

在温度计的例子中，如果房间很冷，那么某个控制器可能还会启动一个防冻加热器。 就 Kubernetes 集群而言，控制面间接地与 IP 地址管理工具、存储服务、云驱动 APIs 以及其他服务协作，通过[扩展 Kubernetes](https://kubernetes.io/zh/docs/concepts/extend-kubernetes/) 来实现这点。

## 期望状态与当前状态

Kubernetes 采用了系统的云原生视图，并且可以处理持续的变化。

在任务执行时，集群随时都可能被修改，并且控制回路会自动修复故障。 这意味着很可能集群永远不会达到稳定状态。

只要集群中的控制器在运行并且进行有效的修改，整体状态的稳定与否是无关紧要的。

## 设计

作为设计原则之一，Kubernetes 使用了很多控制器，每个控制器管理集群状态的一个特定方面。 最常见的一个特定的控制器使用一种类型的资源作为它的期望状态， 控制器管理控制另外一种类型的资源向它的期望状态演化。

使用简单的控制器而不是一组相互连接的单体控制回路是很有用的。 控制器会失败，所以 Kubernetes 的设计正是考虑到了这一点。

**说明：**

可以有多个控制器来创建或者更新相同类型的对象。 在后台，Kubernetes 控制器确保它们只关心与其控制资源相关联的资源。

例如，你可以创建 Deployment 和 Job；它们都可以创建 Pod。 Job 控制器不会删除 Deployment 所创建的 Pod，因为有信息 （[标签](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/labels/)）让控制器可以区分这些 Pod。

## 运行控制器的方式

Kubernetes 内置一组控制器，运行在 [kube-controller-manager](https://kubernetes.io/docs/reference/generated/kube-controller-manager/) 内。 这些内置的控制器提供了重要的核心功能。

Deployment 控制器和 Job 控制器是 Kubernetes 内置控制器的典型例子。 Kubernetes 允许你运行一个稳定的控制平面，这样即使某些内置控制器失败了， 控制平面的其他部分会接替它们的工作。

你会遇到某些控制器运行在控制面之外，用以扩展 Kubernetes。 或者，如果你愿意，你也可以自己编写新控制器。 你可以以一组 Pod 来运行你的控制器，或者运行在 Kubernetes 之外。 最合适的方案取决于控制器所要执行的功能是什么
