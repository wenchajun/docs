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

![devices](F:\docs\docs\images\devices.jpg)

> Pictured: Intel NUC, Bitscope Edge Rack and 6x RPi 4, Turing Pi, Compute Module Carrier with NVMe, PoE Cluster Blade.

> 边缘设备可以是在离用户较近的网络内运行的任何一种计算机。有许多工业和业余级别的选项可用于在私有云上运行工作负载。这些都可以连接到一个中央系统或真实来源（如GitHub）并由其管理。

那么，哪些合理的理由会让人部署faasd而不是Kubernetes呢？

- 如果你需要一些功能来扩展SaaS或响应webhooks
- 你想运行一些定时作业，并使用JavaScript、Go或Python等代码编写它们
- 你想把函数和一些东西打包，作为一个应用来运行它们
- 你需要在边缘或物联网设备上运行

在这篇文章中，我将向你展示使用faasd和GitHub Actions构建和运送函数到你的边缘节点是什么样子。

![faasd-conceptual](C:\Users\文茶君\Desktop\faasd-conceptual.png)

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

所有我提到的模型都支持32位和64位的操作系统。faasd支持这两种类型的arm架构，但你应该注意，你需要为你选择的一种交叉编译你的函数。我们会在GitHubActions部分找到怎么做的方法。

