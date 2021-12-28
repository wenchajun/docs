# Build at the Edge with OpenFaaS and GitHub Actions

https://www.openfaas.com/blog/edge-actions/

## Learn how GitHub Actions and OpenFaaS can be used for simple functions at the edge of your network

了解GitHub Actions和OpenFaaS如何在边缘节点中与函数实例相结合并发挥作用。

#### 为什么选择边缘节点？

如果你像我一样，进入Kubernetes的世界，那么你可能会认为这个平台是云计算和边缘计算的唯一真正选择。事实上，随着Darren Shepherd的K3s的出现，Kubernetes正在成为边缘的现实。我甚至为LinuxFoundation写了一个关于在边缘运行Kubernetes的课程。

那么，在边缘运行Kubernetes以外的东西是否合理？

如果你在生产中运行Kubernetes，那么你会意识到它的操作有多么困难。你不仅需要学习它的概念和API，而且如果你以任何方式扩展它，那么你将需要长期维护你所有的自定义资源（CR）的变化。作为几个针对Kubernetes的应用程序和运营商的作者，我必须把大部分时间用于维护和迁移。

我想告诉你如何用faasd项目在你的边缘节点（网络边缘）建立函数。faasd与你在Kubernetes世界中知道的OpenFaaS一样，但经过重新包装，在物联网设备上使用起来要简单得多。与K3s不同，它在空闲时几乎不消耗任何资源，它的API很稳定，升级就像更换二进制文件一样简单。

> K3s为Kubernetes做了什么，faasd为函数做了什么。

![devices](https://github.com/wenchajun/docs/blob/master/docs/images/devices.jpg)

> Pictured: Intel NUC, Bitscope Edge Rack and 6x RPi 4, Turing Pi, Compute Module Carrier with NVMe, PoE Cluster Blade.

> 边缘设备可以是在离用户较近的网络内运行的任何一种计算机。有许多工业和业余级别的选项可用于在私有云上运行工作负载。这些都可以连接到一个中央系统或真实来源（如GitHub）并由其管理。

那么，哪些合理的理由会让人部署faasd而不是Kubernetes呢？

- 如果你需要一些功能来扩展SaaS或响应webhooks
- 你想运行一些定时作业，并使用JavaScript、Go或Python等代码编写它们
- 你想把函数和一些东西打包，作为一个应用来运行它们
- 你需要在边缘或物联网设备上运行

在这篇文章中，我将向你展示使用faasd和GitHub Actions构建和运送函数到你的边缘节点是什么样子。

![faasd-conceptual](https://github.com/wenchajun/docs/blob/master/docs/images/faasd-conceptual.png)

> 概念图 边缘的OpenFaaS，由GitHub Actions管理

你可以用OpenFaaS运行什么？

OpenFaaS的功能是内置于容器中的，任何可以打包成容器镜像的东西都可以做成一个功能--无论是bash、PowerShell还是像Java、Go或Python这样支持HTTP服务器的语言。

GitHub Actions是一个可以用来构建OpenFaaS函数的多功能的平台。在将这些镜像部署到faasd之前，GitHub容器注册中心可以为其提供一个方便存储这些镜像的地方。

为什么选择边缘节点？

一个函数可以通过使用其URL的HTTP请求来调用--无论是同步还是异步。异步端点更适合运行时间较长的任务，或者当你的调用客户端需要在短时间内得到响应以防止重试时也可以选择异步函数。

函数也可以从一个定时调度规则或通过另一个支持的事件触发器（如Apache Kafka）来触发。

一旦你有一个安全的链接来为你的边缘环境提供入口，你也可以使用GitHub Action和HTTP / curl请求来调用函数。

#### 实验室

我将指导你为你的边缘计算环境建立一个实验室。我们将首先搭建一个带有操作系统的树莓派，安装faasd，然后建立一个GitHub Action，通过安全的inlets隧道建立、发布和部署一个函数。

#### 配置边缘设备

我使用Raspberry Pi（树莓派）作为边缘设备，然而你也可以使用64位PC、Nvidia Jetson Nano或你企业内部管理程序中的虚拟机。

你可以将faasd部署到Raspberry Pi Zero W（512MB）、Raspberry Pi 3（1GB）和Raspberry Pi 4（1-8GB）。我更喜欢Raspberry Pi 4，因为它的I/O速度更快，内存容量也比它的前辈们大。你需要多少内存取决于你的功能的数量和大小。2-4GB的型号是一个开始探索的好地方，可以给你一些成长的空间。

所有我提到的模型都支持32位和64位的操作系统。faasd支持这两种类型的arm架构，但你应该注意，你需要为你选择的一种架构交叉编译你的函数。我们会在GitHubActions部分找到怎么做的方法。我更喜欢32位的官方操作系统 "Raspberry Pi OS Lite"，它有最好的也最全面的支持，在我看来，它比所有其他可选项更有用。我也在Raspberry Pi 4上设置了Ubuntu 20.04，但我发现它开机后会消耗更多的内存，并导致额外的延迟。你的情况因为你的环境可能有所不同。

我倾向于使用32GB的10级SD卡，但也可以把它换成USB硬盘或固态硬盘，以增加可靠性。

Download the Buster variant of the OS:

```
curl -SL -o 2021-05-07-raspios-buster-armhf-lite.zip \
  https://downloads.raspberrypi.org/raspios_lite_armhf/images/raspios_lite_armhf-2021-05-28/2021-05-07-raspios-buster-armhf-lite.zip
```

然后使用你喜欢的工具，如Etcher.io，将镜像刻录到SD卡上。我倾向于在我的Linux电脑上使用`dd`。

接下来的一个重要步骤是建立一个`ssh`文件，这样我们就可以使用Raspberry Pi的headless。在SD卡的第一个分区的`/boot/`目录下创建一个`ssh`文件。

启动Raspberry Pi，并更改其主机名：

```
ssh pi@raspberrypi.local
sudo raspi-config
```

将主机名设为faasd，将GPU的内存分割设为16MB，然后重新启动。

#### 安装faasd

现在重新连接SSH并安装faasd:

```
git clone https://github.com/openfaas/faasd --depth=1
cd faasd
./hack/install.sh
```

最后的输出将显示一个登录命令，你可以用它来确定OpenFaaS网关的密码。你的GitHub Action将需要这个密码。

运行一些测试命令以检查工作是否正常进行。

```
faas-cli version
faas-cli store deploy nodeinfo

curl http://127.0.0.1:8080/function/nodeinfo
```

你现在有了一个带有OpenFaaS的功能齐全的边缘设备，并部署了你的第一个函数。你可以通过运行`faas-cli store list`找到其他的示例函数。

#### 与GITHub连接

为了连接到GitHub，或任何其他现有系统的内部抑或SaaS系统，你需要一个公共端点。我们将使用inlets来做这件事，它可以按月订阅，供个人或企业使用。

inlets是一个专门为云原生系统和容器设计的隧道。它是安全的，与VPN不同的是，它将单个端点如你的OpenFaaS网关连接到其他网络或互联网。它完全在用户空间运行，就流量传入速率来看，这可能是最快的选择。

现在你可以在你喜欢的云上创建一个服务器，并安装inlets服务器进程，或者使用inlets CLI来自动化进程。

- [手动创建一个隧道服务器](https://docs.inlets.dev/)
- [自动创建一个隧道服务器](https://docs.inlets.dev/)

在上面的步骤中，你已经创建了一个sub-domain和一个DNS记录，如`edge.example.com`。你还会得到一个用于inlet服务器控制平面的URL和一个用于inlet客户端的token。

我们将在边缘设备上运行inlets客户端，并通过systemd单元文件进行安装，因此它总是为我们运行，并且会在在因任何原因断开连接时重新启动。

```
inlets http client \
  --token $TOKEN \
  --url $URL \
  --upstream http://127.0.0.1:8080 \
  --generate=systemd > ./inlets.service
```

接下来检查inlets.service文件，并安装它:

```
sudo cp ./inlets.service /etc/systemd/system/inlets.service
sudo systemctl enable inlets.service
sudo systemctl start inlets.service
```

检查它的启动是否正常: `sudo systemctl status inlets.service`

现在你可以在任何地方使用其认证的公共HTTPS URL访问你的OpenFaaS用户界面。

![portal-ui](https://github.com/wenchajun/docs/blob/master/docs/images/portal-ui.png)

创建一个函数来处理仓库中的webhooks

我们将创建一个新的函数来处理仓库的webhooks，主要处理发生在GitHub仓库上的事件，如推送、PR和问题事件。

创建一个新的GitHub仓库并将其克隆到你的系统中。

现在由你来决定你想用什么语言来写函数。Go和Python往往使用最少的资源，所以如果你想装入大量的函数，它们可能是你最好的选择。Node.js也很流行，但对内存的需求可能会更大一些。

探索这些模板，并注意不是每个模板都支持你的Raspberry Pi这样的arm设备。如果你是OpenFaaS的新手，最好坚持使用官方选项。

我将使用`golang-http`模板创建一个名为`repo-events`的函数。

```
faas-cli template store list
faas-cli template store pull golang-http

# Scaffold a function
faas-cli new \
  --lang golang-http \
  --prefix ghcr.io/alexellis \
  repo-events
```

这将创建一个名为` repo-events.yml` 的文件和一个 `repo-events/handler.go` 文件，所以你可以用 Go编写代码。

```yaml
version: 1.0
provider:
  name: openfaas
  gateway: http://127.0.0.1:8080
functions:
  repo-events:
    lang: golang-http
    handler: ./repo-events
    image: ghcr.io/alexellis/repo-events:latest
```

```go
package function

import (
        "fmt"
        "net/http"

        handler "github.com/openfaas/templates-sdk/go-http"
)

// Handle a function invocation
func Handle(req handler.Request) (handler.Response, error) {
        var err error

        message := fmt.Sprintf("Body: %s", string(req.Body))

        return handler.Response{
                Body:       []byte(message),
                StatusCode: http.StatusOK,
        }, err
}
```

为方便起见，将 `repo-events.yml `重命名为 `stack.yml`:

```
mv repo-events.yml stack.yml
```

为你的函数初始化一个Go模块:

```
cd repo-events/

go mod init
go mod tidy
cd .
```

在你的笔记本或客户机上（不是在faasd主机上），运行以下内容:

```
DOCKER_BUILDKIT=1 \
  faas-cli build \
  --build-arg GO111MODULE=on

#28 writing image sha256:e5ff71c8fa666f6cffc866ee8339b01fcb7074c23deaddbd00e1056519e784d4 done
#28 naming to ghcr.io/alexellis/repo-events:latest done
#28 DONE 0.1s
Image: ghcr.io/alexellis/repo-events:latest built.
[0] < Building repo-events done in 10.62s.
[0] Worker done.

Total build time: 10.62s
```

你需要在本地安装Docker，这样才能发挥作用。

后续的构建会更快，因为构建的各个部分都会被缓存。

更新stack.yml文件并添加以下内容，这样CI系统就不需要为Go HTTP模板设置`faas-cli template store pull`命令。

```
configuration:
  templates:
    - name: golang-http
```

#### 用GitHub Action构建和部署该函数

- 为你的账户或仓库启用GitHub Action
- 在设置页面创建以下secrets
- Name: `OPENFAAS_URL`, value: the gateway’s public HTTPS URL or inlets PRO tunnel URL i.e. `https://edge.example.com`
- Name: `OPENFAAS_PASSWORD`, value: faasd’s basic auth password
- 创建如下文件`.github/workflows/build.yml`

这是文件的顶部，它在每次提交PR时触发一个名为build的构建，并推送到远程分支。

```
name: build

on:
  push:
    branches:
      - '*'
  pull_request:
    branches:
      - '*'
```

然后添加GitHub令牌的权限，这样该动作就可以使用自己的临时令牌推送到GHCR。这比生成你自己的个人访问令牌更安全，后者可以访问你的整个账户，而且可能不会过期。

```
permissions:
  actions: read
  checks: write
  contents: read
  packages: write
```

接下来是job的步骤：

```
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@master
        with:
          fetch-depth: 1
      - name: Get faas-cli
        run: curl -sLSf https://cli.openfaas.com | sudo sh
      - name: Pull custom templates from stack.yml
        run: faas-cli template pull stack
      - name: Set up QEMU
        uses: docker/setup-qemu-action@v1
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v1
      - name: Get TAG
        id: get_tag
        run: echo ::set-output name=TAG::latest-dev
      - name: Get Repo Owner
        id: get_repo_owner
        run: >
          echo ::set-output name=repo_owner::$(echo ${{ github.repository_owner }} |
          tr '[:upper:]' '[:lower:]')
      - name: Docker Login
        run: >
          echo ${{secrets.GITHUB_TOKEN}} | 
          docker login ghcr.io --username 
          ${{ steps.get_repo_owner.outputs.repo_owner }} 
          --password-stdin
      - name: Publish functions
        run: >
          OWNER="${{ steps.get_repo_owner.outputs.repo_owner }}" 
          TAG="latest"
          faas-cli publish
          --extra-tag ${{ github.sha }}
          --build-arg GO111MODULE=on
          --platforms linux/amd64,linux/arm/v7,linux/arm64
      - name: Login
        run: >
          echo ${{secrets.OPENFAAS_PASSWORD}} | 
          faas-cli login --gateway ${{secrets.OPENFAAS_URL}} --password-stdin
      - name: Deploy
        run: >
          OWNER="${{ steps.get_repo_owner.outputs.repo_owner }}"
          TAG="${{ github.sha }}"
          faas-cli deploy --gateway ${{secrets.OPENFAAS_URL}}
```

前几步是用buildx设置Docker，这样它就可以为不同的系统交叉编译容器，如果你只想部署到云或英特尔兼容的faasd实例，这一步可以跳过。

接下来的步骤是使用GitHub Action附带的GitHub令牌登录ghcr.io注册表，然后运行`faas-cli publish`，构建并推送一个多架构镜像，然后登录你的远程网关，使用`faas-cli login`和`faas-cli deploy`做部署。

建议使用--platforms标志，以使你的构建更有效率。目前，它正在为下列平台进行构建：

- `linux/amd64` - 普通PC和云计算机
- `linux/arm/v7` -  32-bit arm Raspberry Pi OS
- `linux/arm64` - 64-bit arm 服务器 或者在 Raspberry Pi上运行的 Ubuntu 系统

因此，如果你只是将你的函数部署到带有32位操作系统的Raspberry Pi上，只需将这一行改为`--platforms linux/arm/v7`。

你会注意到文件中的最后一步是进行部署。如果你不希望自动部署构建，想把CI和CD步骤分开,那么你可以把它放到一个单独的文件中，只有在GitHub仓库中进行发布时才会运行。

```
- name: Login
        run: >
          echo ${{secrets.OPENFAAS_PASSWORD}} | 
          faas-cli login --gateway ${{secrets.OPENFAAS_URL}} --password-stdin
      - name: Deploy
        run: >
          OWNER="${{ steps.get_repo_owner.outputs.repo_owner }}"
          TAG="${{ github.sha }}"
          faas-cli deploy --gateway ${{secrets.OPENFAAS_URL}}
```

现在运行以下命令：

```
git add .
git commit
git push origin master
```

几分钟后， GitHub Action将启动。如果你已经正确地复制了所有内容，那么它应该向GHCR发布一个镜像。

如果镜像显示为私有，那么你需要将该镜像公开，以便faasd能够访问它。进入你的用户帐户的软件包标签，并使其公开。

然后重新启动作业，或将另一个更改推送到仓库。

![fixed](https://github.com/wenchajun/docs/blob/master/docs/images/fixed.png)

> 构建失败几次并不丢人。一旦它能正常运行，它通常能够在一分钟内构建、发布和部署你的函数。

几分钟后，你会看到函数出现在你的本地faasd实例上，运行`faas-cli list`来验证函数的部署。

我现在能够调用我的URL了。

``` 
curl -sL https://edge.example.com/function/repo-events
```

由你来扩展这个函数：

谷歌提供了一个优秀的库，叫做github-go，可以用来解析和响应从GitHub发送的webhook事件。

作为一个练习，为什么不在你的GitHub仓库的webhooks页面上输入你的函数的URL（https://edge.example.com/function/repo-events）？让它在你觉得有趣的事件中启动，比如一个新的PR，或者一个新的issue。

使用github-go库，使用HMAC验证事件，然后解析事件。你接下来要做什么，取决于你自己。你想向Slack发送一条消息吗？发出电子邮件？用一个链接到你的discord社区来回应一个评论？你想把一个issue标记为一个问题吗？

关于一个用Go编写的GitHub机器人的例子，请查看[Derek](https://github.com/alexellis/derek)。

![docker commenting](https://github.com/wenchajun/docs/blob/master/docs/images/fixed.png)

> Derek commenting on PRs

让我们从这个翻译成OpenFaaS模板的例子开始。它验证传入的消息已经用你在 repo 的 webhooks 页面上输入的 HMAC 秘密签名，然后解析事件，并将在函数的日志中打印一条消息，或者对于其他事件类型打印一个错误。

```go
package function

import (
	"bytes"
	"fmt"
	"net/http"
	"os"

	"github.com/google/go-github/v40/github" // with go modules enabled (GO111MODULE=on or outside GOPATH)
	handler "github.com/openfaas/templates-sdk/go-http"
)

// Handle a function invocation
func Handle(req handler.Request) (handler.Response, error) {

	webhookSecretKey, err := os.ReadFile("/var/openfaas/secrets/webhook-secret")
	if err != nil {
		return handler.Response{
			StatusCode: http.StatusInternalServerError,
			Body:       []byte(fmt.Sprintf("Error reading webhook secret: %s", err)),
		}, fmt.Errorf("error reading webhook secret: %w", err)
	}

	payload, err := github.ValidatePayloadFromBody(req.Header.Get("Content-Type"),
		bytes.NewBuffer(req.Body),
		req.Header.Get(github.SHA256SignatureHeader),
		webhookSecretKey)
	if err != nil {
		return handler.Response{
			StatusCode: http.StatusBadRequest,
			Body:       []byte(fmt.Sprintf("Error validating payload: %s", err.Error())),
		}, fmt.Errorf("error validating payload: %w", err)
	}

	eventType := req.Header.Get(github.EventTypeHeader)
	event, err := github.ParseWebHook(eventType, payload)
	if err != nil {
		return handler.Response{
			StatusCode: http.StatusBadRequest,
			Body:       []byte(fmt.Sprintf("Error parsing webhook: %s", err.Error())),
		}, fmt.Errorf("error parsing webhook: %w", err)
	}

	switch event := event.(type) {
	case *github.IssueCommentEvent:
		fmt.Printf("Issue comment body: %s\n", event.GetComment().GetBody())
	default:
		return handler.Response{
			StatusCode: http.StatusBadRequest,
			Body:       []byte(fmt.Sprintf("Event type not supported: %s", eventType)),
		}, fmt.Errorf("event type not supported: %s", eventType)
	}

	return handler.Response{
		Body:       []byte("Accepted webhook"),
		StatusCode: http.StatusAccepted,
	}, nil
}
```

然后为你的webhook创建一个secret:

```
faas-cli secret create webhook-secret \
 --from-literal "S_3PPzytjNcgVfW"

Creating secret: webhook-secret
Created: 200 OK
```

编辑你的stack.yml文件并添加一个 "secret":

```
  image: ghcr.io/alexellis/repo-events:latest
    secrets:
    - webhook-secret
```

然后向版本库推送一个提交，触发一个新的构建。

在你的版本库上创建一个issue，并留下评论来触发这个函数。

你会在函数的日志中看到结果。

````
pi@faasd-pi:~ $ faas-cli logs repo-events
2021-11-29T12:03:43Z Issue comment body: faasd and Go at the edge
2021-11-29T12:03:43Z 2021/11/29 12:03:43 POST / - 202 Accepted - ContentLength: 16
````

#### 更进一步

我们现在已经建立了一个完整的管道，每次你使用Actions和GHCR提交到GitHub仓库时都会部署新版本的代码。Inlets为我们的部署和调用提供了一个安全的上行链路。我向你展示了如何将GitHub仓库的事件连接到你的函数中，但许多平台都提供webhook，如Gumroad和Stripe。

- 向客户发送电子邮件，并为Gumroad制作自定义折扣
- 用Slack和faasd跟踪Stripe支付的情况

你也可以决定从一些内部系统（如Jenkins或企业内部的BitBucket）调用你的函数，而不必让这些事件通过互联网。如果你的平台不支持webhooks，那么我的eBook将告诉你如何使用定时调度规则来触发函数，这样它们就可以从你心中的任何来源收集数据。

当一个事件源没有API或SDK可用时，那么faasd也可以用来搜刮网站。网络搜刮，正好可以用Puppeteer的OpenFaaS。只需注意，如果不做一些额外的努力，这篇博文在你的Raspberry Pi上是无法工作的。

在我的电子书《Serverless For Everyone Else》中，我展示了如何使用OpenFaaS REST API、连接数据库、异步功能、私有镜像和仓库、自定义域和监控的实际例子。你还可以得到我走过所有这些步骤的视频。

如果你真的想使用K3s或Kubernetes呢？这也没问题。我向你展示的一切，包括GitHub行动和示例代码，都可以在你的树莓派或边缘设备上运行的K3s或Kubernetes中使用。你将在CPU、内存和SD卡的磨损方面付出代价，但如果你已经对Kubernetes有了很大的依赖，那么这些折衷可能是值得的。我在KubeCon 2020的演讲中谈到了这些，并展示了真实世界的使用案例。

