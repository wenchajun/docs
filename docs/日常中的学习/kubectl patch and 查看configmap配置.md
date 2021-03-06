### kubectl patch

使用（patch）补丁修改、更新资源的字段。

支持JSON和YAML格式。

### 语法

```
$ patch (-f FILENAME | TYPE NAME) -p PATCH
```

在 patch 命令中，将 `type` 设置为 `merge`：

```shell
kubectl patch deployment patch-demo --type merge --patch "$(cat patch-file-2.yaml)"
```



#### 查看configmap配置

cd /var/lib/kubelet/pods

找volume configmap等目录

kubectl get pod xx -o jsonpath='{.metadata.uid}'

kubectl get pod -o wide





以编程的方式访问

https://stackoverflow.com/questions/53283347/how-to-get-current-namespace-of-an-in-cluster-go-kubernetes-client

Add this environment variable in your deployment config.

```yaml
 - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
```

This is using the [kubernetes downward api](https://kubernetes.io/docs/tasks/inject-data-application/downward-api-volume-expose-pod-information/#capabilities-of-the-downward-api)

### 从 Pod 中访问 API 

当你从 Pod 中访问 API 时，定位和验证 apiserver 会有些许不同。

在 Pod 中定位 apiserver 的推荐方式是通过 `kubernetes.default.svc` 这个 DNS 名称，该名称将会解析为服务 IP，然后服务 IP 将会路由到 apiserver。

向 apiserver 进行身份验证的推荐方法是使用 [服务帐户](https://kubernetes.io/zh/docs/tasks/configure-pod-container/configure-service-account/) 凭据。 通过 kube-system，Pod 与服务帐户相关联，并且该服务帐户的凭证（token） 被放置在该 Pod 中每个容器的文件系统中，位于 `/var/run/secrets/kubernetes.io/serviceaccount/token`。

如果可用，则将证书放入每个容器的文件系统中的 `/var/run/secrets/kubernetes.io/serviceaccount/ca.crt`， 并且应该用于验证 apiserver 的服务证书。

最后，名字空间作用域的 API 操作所使用的 default 名字空间将被放置在 每个容器的 `/var/run/secrets/kubernetes.io/serviceaccount/namespace` 文件中。

在 Pod 中，建议连接 API 的方法是：

- 在 Pod 的边车容器中运行 `kubectl proxy`，或者以后台进程的形式运行。 这将把 Kubernetes API 代理到当前 Pod 的 localhost 接口， 所以 Pod 中的所有容器中的进程都能访问它。
- 使用 Go 客户端库，并使用 `rest.InClusterConfig()` 和 `kubernetes.NewForConfig()` 函数创建一个客户端。 他们处理 apiserver 的定位和身份验证。 [示例](https://git.k8s.io/client-go/examples/in-cluster-client-configuration/main.go)

在每种情况下，Pod 的凭证都是为了与 apiserver 安全地通信。

> b, err := ioutil.ReadFile("/var/run/secrets/kubernetes.io/serviceaccount/token")  
>
> if err != nil {    panic(err)  }
