# OpenFunction Cli介绍

### OpenFunction

[OpenFunction](https://github.com/OpenFunction/OpenFunction.git) 是一个云原生、开源的 FaaS（函数即服务）框架，旨在让开发人员专注于他们的开发意图，而不必关心底层运行环境和基础设施。用户只需提交一段代码，就可以生成事件驱动的、动态伸缩的 Serverless 工作负载。

#### 部署OpenFunction

OpenFunction 项目引用了很多第三方的项目，如 Knative、Tekton、ShipWright、Dapr、KEDA 等，手动安装部署较为繁琐，于是推荐使用 [Prerequisites 文档](https://github.com/OpenFunction/OpenFunction#prerequisites) 中的方法，一键部署 OpenFunction 的依赖组件。但是这种安装方式功能较单一，且随着OpenFunction和kubernetes的升级迭代，OpenFunction对于各个第三方项目的依赖版本也不一致，这就导致我们迫切需要一种新的安装方式，于是OpenFunction也就诞生了。

​                                                        表一 OpenFunction第三方组件依赖的kubernetes版本

| Components             | Kubernetes 1.17 | Kubernetes 1.18 | Kubernetes 1.19 | Kubernetes 1.20+ |
| ---------------------- | --------------- | --------------- | --------------- | ---------------- |
| Knative Serving        | 0.21.1          | 0.23.3          | 0.25.2          | 1.0.1            |
| Kourier                | 0.21.0          | 0.23.0          | 0.25.0          | 1.0.1            |
| Serving Default Domain | 0.21.0          | 0.23.0          | 0.25.0          | 1.0.1            |
| Dapr                   | 1.5.1           | 1.5.1           | 1.5.1           | 1.5.1            |
| Keda                   | 2.4.0           | 2.4.0           | 2.4.0           | 2.4.0            |
| Shipwright             | 0.6.1           | 0.6.1           | 0.6.1           | 0.6.1            |
| Tekton Pipelines       | 0.23.0          | 0.26.0          | 0.29.0          | 0.30.0           |
| Cert Manager           | 1.5.4           | 1.5.4           | 1.5.4           | 1.5.4            |
| Ingress Nginx          | na              | na              | 1.1.0           | 1.1.0            |

#### OpenFunction Cli

OpenFunction Cli 支持一键部署，一键卸载且提供Demo演示的功能。用户可以通过设置相应的参数自定义的选择各项组件，同时可以选择特定的版本，使安装更为灵活，在此基础之上，安装进程也提供了实时展示，使得界面更为美观。现在，你可以访问[ofn release](https://github.com/OpenFunction/cli/releases/)选择`ofn` v0.5.0版本部署OpenFunction到您的集群。OpenFunction支持三个子命令：install、uninstall以及demo，下面我们依次介绍。

##### [install](https://github.com/OpenFunction/cli/blob/main/docs/install.md)

`ofn install`将帮助你安装OpenFunction及其依赖，同时提供多种参数以供用户选择。

| 参数           | 功能                                                  |
| -------------- | ----------------------------------------------------- |
| --all          | 用于安装OpenFunction及其所有依赖。                    |
| --async        | 用于安装OpenFunction的异步运行时（Dapr & Keda）。     |
| --cert-manager | 用于安装Cert Manager。                                |
| --dapr         | 用于安装Dapr。                                        |
| --dry-run      | 用于提示当前命令所要安装的组件及其版本。              |
| --ingress      | 用于安装Ingress Nginx。                               |
| --keda         | 用于安装Keda。                                        |
| --knative      | 用于安装Knative Serving（以Kourier为默认网关）        |
| --region-cn    | 针对访问gcr.io或github.com受限的用户。                |
| --shipwright   | 用于安装ShipWright。                                  |
| --sync         | 用于安装OpenFunction Sync Runtime（待支持）。         |
| --upgrade      | 在安装时将组件升级到目标版本。                        |
| --verbose      | 显示粗略信息。                                        |
| --version      | 用于指定要安装的OpenFunction的版本。(默认为 "v0.4.0") |
| --timeout      | 设置超时时间。默认为5分钟。                           |

我们假设你已经将下载了`ofn cli`并将其放在`PATH`中的适当路径下。你可以使用 `ofn install --all` 来完成一个简单的部署。默认情况下，该命令将为你安装OpenFunction的v0.4.0版本，而对于已经存在的组件，它将跳过安装过程（你可以使用--upgrade命令来覆盖这些组件）。

```shell
# ofn install --all --upgrade
Start installing OpenFunction and its dependencies.
Here are the components and corresponding versions to be installed:
+------------------+---------+
| COMPONENT        | VERSION |
+------------------+---------+
| Kourier          | 1.0.1   |
| Keda             | 2.4.0   |
| Tekton Pipelines | 0.30.0  |
| OpenFunction     | 0.4.0   |
| Dapr             | 1.5.1   |
| CertManager      | 1.1.0   |
| Shipwright       | 0.6.1   |
| Knative Serving  | 1.0.1   |
| DefaultDomain    | 1.0.1   |
+------------------+---------+
You have used the `--upgrade` parameter, which means that the installation process will overwrite the components that already exist.
Make sure you know what happens when you do this.
Enter 'y' to continue and 'n' to abort:
-> y
🔄  -> INGRESS <- Installing Ingress...
🔄  -> KNATIVE <- Installing Knative Serving...
🔄  -> DAPR <- Installing Dapr...
🔄  -> DAPR <- Downloading Dapr Cli binary...
🔄  -> KEDA <- Installing Keda...
🔄  -> CERTMANAGER <- Installing Cert Manager...
🔄  -> SHIPWRIGHT <- Installing Shipwright...
🔄  -> INGRESS <- Checking if Ingress is ready...
🔄  -> KEDA <- Checking if Keda is ready...
🔄  -> CERTMANAGER <- Checking if Cert Manager is ready...
🔄  -> SHIPWRIGHT <- Checking if Shipwright is ready...
🔄  -> KNATIVE <- Installing Kourier as Knative's gateway...
🔄  -> KNATIVE <- Configuring Knative Serving's DNS...
🔄  -> KNATIVE <- Checking if Knative Serving is ready...
✅  -> CERTMANAGER <- Done!
🔄  -> DAPR <- Initializing Dapr with Kubernetes mode...
✅  -> SHIPWRIGHT <- Done!
✅  -> KNATIVE <- Done!
✅  -> INGRESS <- Done!
✅  -> DAPR <- Done!
✅  -> KEDA <- Done!
🔄  -> OPENFUNCTION <- Installing OpenFunction...
🔄  -> OPENFUNCTION <- Checking if OpenFunction is ready...
✅  -> OPENFUNCTION <- Done!
🚀 Completed in 2m3.638035129s.
```

##### [uninstall](https://github.com/OpenFunction/cli/blob/main/docs/uninstall.md)

`ofn uninstall`将帮助你卸载OpenFunction及其依赖，同时你可以根据相应的参数选择卸载指定的组件。

| 参数           | 功能                                                  |
| -------------- | ----------------------------------------------------- |
| --all          | 用于卸载OpenFunction及其所有依赖。                    |
| --async        | 用于卸载OpenFunction的异步运行时（Dapr & Keda）。     |
| --cert-manager | 用于卸载Cert Manager。                                |
| --dapr         | 用于卸载Dapr。                                        |
| --ingress      | 用于卸载Ingress Nginx。                               |
| --keda         | 用于卸载KEDA。                                        |
| --knative      | 用于卸载Knative Serving（以Kourier为默认网关）        |
| --region-cn    | 针对访问gcr.io或github.com受限的用户。                |
| --shipwright   | 用于安卸载hipWright。                                 |
| --sync         | 用于卸载OpenFunction同步运行时（待支持）。            |
| --verbose      | 显示粗略信息。                                        |
| --version      | 用于指定要卸载的OpenFunction的版本。(默认为 "v0.4.0") |
| --timeout      | 设置超时时间。默认为5分钟。                           |

你可以使用 `ofn uninstall --all `来轻松卸载 OpenFunction 及其依赖项（如果不加参数则表示只卸载 OpenFunction，不卸载其他组件）。

```shell
~# ofn uninstall --all
Start uninstalling OpenFunction and its dependencies.
The following components already exist:
+------------------+---------+
| COMPONENT        | VERSION |
+------------------+---------+
| Cert Manager     | v1.5.4  |
| Ingress Nginx    | 1.1.0   |
| Tekton Pipelines | v0.28.1 |
| Shipwright       | 0.6.0   |
| OpenFunction     | v0.4.0  |
| Dapr             | 1.4.3   |
| Keda             | 2.4.0   |
| Knative Serving  | 0.26.0  |
+------------------+---------+
You can see the list of components to be uninstalled and the list of components already exist in the cluster.
Make sure you know what happens when you do this.
Enter 'y' to continue and 'n' to abort:
-> y
🔄  -> OPENFUNCTION <- Uninstalling OpenFunction...
🔄  -> KNATIVE <- Uninstalling Knative Serving...
🔄  -> DAPR <- Uninstalling Dapr with Kubernetes mode...
🔄  -> KEDA <- Uninstalling Keda...
🔄  -> SHIPWRIGHT <- Uninstalling Tekton Pipeline & Shipwright...
🔄  -> INGRESS <- Uninstalling Ingress...
🔄  -> CERTMANAGER <- Uninstalling Cert Manager...
✅  -> OPENFUNCTION <- Done!
✅  -> DAPR <- Done!
🔄  -> KNATIVE <- Uninstalling Kourier...
✅  -> KEDA <- Done!
✅  -> CERTMANAGER <- Done!
✅  -> KNATIVE <- Done!
✅  -> INGRESS <- Done!
✅  -> SHIPWRIGHT <- Done!
🚀 Completed in 1m21.683329262s.
```

[demo](https://github.com/OpenFunction/cli/blob/main/docs/demo.md)

`ofn demo`将帮助你创建一个KIND集群，并且安装OpenFunction及其所有依赖并运行一个sample函数。

| 参数         | 功能                                                         |
| ------------ | ------------------------------------------------------------ |
| --region-cn  | 针对访问gcr.io或github.com受限的用户。                       |
| --auto-prune | 自动清理当前的KIND集群，如果设置为false，将保留当前的KIND集群 |
| --verbose    | 显示粗略信息。                                               |
| --timeout    | 设置超时时间。默认为10分钟。                                 |

你可以使用`ofn demo `运行一个KIND集群并安装最新版本的OpenFunction并运行一个示例,该KIND集群将在运行结束后自动删除。

```shell
# ./ofn demo
Launching OpenFunction demo...
The following components will be installed for this demo:
+--------------+---------+
| COMPONENT    | VERSION |
+--------------+---------+
| OpenFunction | v0.4.0  |
+--------------+---------+
A Kind cluster will be created and the OpenFunction Demo will be launched in it...
Enter 'y' to continue and 'n' to abort:
-> y
🔄  -> KIND <- Installing Kind...
🔄  -> KIND <- Downloading Kind binary...
🔄  -> KIND <- Creating cluster...
✅  -> KIND <- Done!
Start installing OpenFunction and its dependencies.
Here are the components and corresponding versions to be installed:
+------------------+---------+
| COMPONENT        | VERSION |
+------------------+---------+
| DefaultDomain    | 1.0.1   |
| Tekton Pipelines | 0.30.0  |
| OpenFunction     | 0.4.0   |
| Kourier          | 1.0.1   |
| Keda             | 2.4.0   |
| Knative Serving  | 1.0.1   |
| Dapr             | 1.5.1   |
| Shipwright       | 0.6.1   |
| CertManager      | 1.5.4   |
+------------------+---------+
🔄  -> CERTMANAGER <- Installing Cert Manager...
🔄  -> KEDA <- Installing Keda...
🔄  -> DAPR <- Installing Dapr...
🔄  -> DAPR <- Downloading Dapr Cli binary...
🔄  -> SHIPWRIGHT <- Installing Tekton Pipelines...
🔄  -> KNATIVE <- Installing Knative Serving...
🔄  -> SHIPWRIGHT <- Checking if Tekton Pipelines is ready...
🔄  -> KEDA <- Checking if Keda is ready...
🔄  -> DAPR <- Initializing Dapr with Kubernetes mode...
🔄  -> CERTMANAGER <- Checking if Cert Manager is ready...
🔄  -> KNATIVE <- Checking if Knative Serving is ready...
✅  -> DAPR <- Done!
✅  -> CERTMANAGER <- Done!
🔄  -> KNATIVE <- Configuring Knative Serving's DNS...
🔄  -> KNATIVE <- Installing Kourier as Knative's gateway...
🔄  -> KNATIVE <- Checking if Kourier is ready...
✅  -> KEDA <- Done!
🔄  -> SHIPWRIGHT <- Installing Shipwright...
🔄  -> SHIPWRIGHT <- Checking if Shipwright is ready...
✅  -> KNATIVE <- Done!
✅  -> SHIPWRIGHT <- Done!
🔄  -> OPENFUNCTION <- Installing OpenFunction...
🔄  -> OPENFUNCTION <- Checking if OpenFunction is ready...
✅  -> OPENFUNCTION <- Done!
🔄  -> DEMO <- Run OpenFunctionDemo...
Now we have configured the appropriate parameters for you, You can use this address to access related functions :
 http://serving-smwbh-ksvc-dn2vx.default.172.18.0.2.sslip.io

We now use the curl command to access the address. The following information was returned:
Hello, World!

✅  -> DEMO <- Done!
 Completed in 7m56.980904754s.
```

