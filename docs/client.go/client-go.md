# client-go

如果你要对k8s进行二次开发，那么就不得不接触client-go，那么了解并熟悉client-go就是非常有必要的了。 因为client-go是一个调用kubernetes集群资源对象API的客户端，即通过client-go实现对kubernetes集群中资源对象（包括deployment、service、ingress、replicaSet、pod、namespace、node等）的增删改查等操作。大部分对kubernetes进行前置API封装的二次开发都通过client-go这个第三方包来实现。

源码位于：

https://github.com/kubernetes/client-go

![client-go目录格式](https://github.com/wenchajun/docs/blob/master/docs/images/client-go%E7%9B%AE%E5%BD%95%E6%A0%BC%E5%BC%8F.png)

- The `kubernetes` package contains the clientset to access Kubernetes API.//连接kubernetesAPI作用
- The `discovery` package is used to discover APIs supported by a Kubernetes API server.//通过APIserver发现APIs
- The `dynamic` package contains a dynamic client that can perform generic operations on arbitrary Kubernetes API objects.//包含一个动态客户端，它可以在任意Kubernetes API对象上执行通用操作。
- The `plugin/pkg/client/auth` packages contain optional authentication plugins for obtaining credentials from external sources.//包含可选的身份验证插件，用于从外部来源获取凭证。 
- The `transport` package is used to set up auth and start a connection.//用于设置身份验证和启动连接
- The `tools/cache` package is useful for writing controllers.//对于写控制器很有用

Client-go提供了四种客户端，简单描述如下

| 客户端名称      | 源码目录              | 简单描述                                                     |
| --------------- | --------------------- | ------------------------------------------------------------ |
| RESTClient      | client-go/rest/       | 基础客户端，对HTTP Request封装                               |
| ClientSet       | client-go/kubernetes/ | 在RESTClient基础上封装了对Resource和Version，也就是说我们使用ClientSet的话是必须要知道Resource和Version， 例如AppsV1().Deployments或者CoreV1.Pods，缺点是不能访问CRD自定义资源 |
| DynamicClient   | client-go/dynamic/    | 包含一组动态的客户端，可以对任意的K8S API对象执行通用操作，包括CRD自定义资源 |
| DiscoveryClient | client-go/discovery/  | 在上述我们试过ClientSet是必须要知道Resource和Version, 但人是记不住的(例如我)，这个DiscoveryClient是提供一个发现客户端，发现API Server支持的资源组，资源版本和资源信息 |

![编写自定义控制器所依赖的组件](https://github.com/wenchajun/docs/blob/master/docs/images/%E7%BC%96%E5%86%99%E8%87%AA%E5%AE%9A%E4%B9%89%E6%8E%A7%E5%88%B6%E5%99%A8%E6%89%80%E4%BE%9D%E8%B5%96%E7%9A%84%E7%BB%84%E4%BB%B6.png)

在上图中，浅蓝色与client-go相关。可以将其划分为7个部分

- **Reflector**：Reflector 向 apiserver watch 特定类型的资源，拿到变更通知后将其丢到 DeltaFIFO 队列中；
- **Informer**： Informer 从 DeltaFIFO 中 pop 相应对象，然后通过 Indexer 将对象和索引丢到本地 cache 中，再触发相应的事件处理函数（Resource Event Handlers）运行；
- **Indexer**： Indexer 主要提供一个对象根据一定条件的检索能力，典型的实现是通过 namespace/name 来构造 key ，通过 Thread Safe Store 来存储对象；
- **Workqueue**：Workqueue 一般使用的是延时队列实现，在 Resource Event Handlers 中会完成将对象的 key 放入 workqueue 的过程，然后我们在自己的逻辑代码里从 workqueue 中消费这些 key；
- **ClientSet**：Clientset 提供的是资源的 CURD 能力，和 apiserver 交互；
- **Resource Event Handlers**：我们在 Resource Event Handlers 中一般是添加一些简单的过滤功能，判断哪些对象需要加到 workqueue 中进一步处理；对于需要加到 workqueue 中的对象，就提取其 key，然后入队；
- **Worker**：Worker 指的是我们自己的业务代码处理过程，在这里可以直接接收到 workqueue 里的任务，可以通过 Indexer 从本地缓存检索对象，通过 Clientset 实现对象的增删改查逻辑。

参考 [https://www.danielhu.cn](https://www.danielhu.cn/)
