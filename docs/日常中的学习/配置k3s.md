安装docker

## 卸载旧版本

旧版本的 Docker 称为 `docker` 或者 `docker-engine`，使用以下命令卸载旧版本：

```
 sudo apt-get remove docker \
               docker-engine \
               docker.io
```

# 使用 APT 安装

由于 `apt` 源使用 HTTPS 以确保软件下载过程中不被篡改。因此，我们首先需要添加使用 HTTPS 传输的软件包以及 CA 证书。

```
 sudo apt-get update

 sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```



鉴于国内网络问题，强烈建议使用国内源，官方源请在注释中查看。

为了确认所下载软件包的合法性，需要添加软件源的 `GPG` 密钥。

```
$ curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg


# 官方源
# $ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```



然后，我们需要向 `sources.list` 中添加 Docker 软件源

```
$ echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://mirrors.aliyun.com/docker-ce/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null


# 官方源
# $ echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```



> 以上命令会添加稳定版本的 Docker APT 镜像源，如果需要测试版本的 Docker 请将 stable 改为 test。

## 安装 Docker

更新 apt 软件包缓存，并安装 `docker-ce`：

```
$ sudo apt-get update

$ sudo apt-get install docker-ce docker-ce-cli containerd.io
```



# 使用脚本自动安装

在测试或开发环境中 Docker 官方为了简化安装流程，提供了一套便捷的安装脚本，Ubuntu 系统上可以使用这套脚本安装，另外可以通过 `--mirror` 选项使用国内源进行安装：

> 若你想安装测试版的 Docker, 请从 test.docker.com 获取脚本

```
 $ curl -fsSL test.docker.com -o get-docker.sh//测试版
$ curl -fsSL get.docker.com -o get-docker.sh
$ sudo sh get-docker.sh --mirror Aliyun
# $ sudo sh get-docker.sh --mirror AzureChinaCloud
```



执行这个命令后，脚本就会自动的将一切准备工作做好，并且把 Docker 的稳定(stable)版本安装在系统中。

# 启动 Docker

```
$ sudo systemctl enable docker
$ sudo systemctl start docker
```



```shell
root@i-k5d6x5d6:~# sudo sh get-docker.sh 
# Executing docker install script, commit: 93d2499759296ac1f9c510605fef85052a2c32be
Warning: the "docker" command appears to already exist on this system.

If you already have Docker installed, this script can cause trouble, which is
why we're displaying this warning and provide the opportunity to cancel the
installation.

If you installed the current Docker package using this script and are using it
again to update Docker, you can safely ignore this message.

You may press Ctrl+C now to abort this script.
+ sleep 20
+ sh -c apt-get update -qq >/dev/null
+ sh -c DEBIAN_FRONTEND=noninteractive apt-get install -y -qq apt-transport-https ca-certificates curl >/dev/null
+ sh -c curl -fsSL "https://download.docker.com/linux/ubuntu/gpg" | gpg --dearmor --yes -o /usr/share/keyrings/docker-archive-keyring.gpg
+ sh -c echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu focal stable" > /etc/apt/sources.list.d/docker.list
+ sh -c apt-get update -qq >/dev/null
+ sh -c DEBIAN_FRONTEND=noninteractive apt-get install -y -qq --no-install-recommends  docker-ce-cli docker-scan-plugin docker-ce >/dev/null
+ version_gte 20.10
+ [ -z  ]
+ return 0
+ sh -c DEBIAN_FRONTEND=noninteractive apt-get install -y -qq docker-ce-rootless-extras >/dev/null
+ sh -c docker version
Client: Docker Engine - Community
 Version:           20.10.12
 API version:       1.41
 Go version:        go1.16.12
 Git commit:        e91ed57
 Built:             Mon Dec 13 11:45:33 2021
 OS/Arch:           linux/amd64
 Context:           default
 Experimental:      true

Server: Docker Engine - Community
 Engine:
  Version:          20.10.12
  API version:      1.41 (minimum version 1.12)
  Go version:       go1.16.12
  Git commit:       459d0df
  Built:            Mon Dec 13 11:43:42 2021
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.4.12
  GitCommit:        7b11cfaabd73bb80907dd23182b9347b4245eb5d
 runc:
  Version:          1.0.2
  GitCommit:        v1.0.2-0-g52b36a2
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0

================================================================================

To run Docker as a non-privileged user, consider setting up the
Docker daemon in rootless mode for your user:

    dockerd-rootless-setuptool.sh install

Visit https://docs.docker.com/go/rootless/ to learn about rootless mode.


To run the Docker daemon as a fully privileged service, but granting non-root
users access, refer to https://docs.docker.com/go/daemon-access/

WARNING: Access to the remote API on a privileged Docker daemon is equivalent
         to root access on the host. Refer to the 'Docker daemon attack surface'
         documentation for details: https://docs.docker.com/go/attack-surface/

================================================================================

root@i-k5d6x5d6:~# sudo systemctl enable docker
Synchronizing state of docker.service with SysV service script with /lib/systemd/systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install enable docker

```

访问下方链接即可了解如何安装k3d：

https://k3d.io/#installation

在本文中，我们将以curl的方式安装。

> 请注意：直接从URL运行脚本存在很严重的安全问题。所以在运行任意脚本之前，确保源是项目的网站或git在线repository。

以下是安装步骤：

访问：https://k3d.io/#installation

复制“curl“安装命令并且在你的terminal内运行它：

**curl -s https://raw.githubusercontent.com/rancher/k3d/main/install.sh | bash**

- **k3d version**：提供已安装的k3d版本
- **k3d –help**：列出可用于k3d的命令

现在k3d已经安装完成并且准备开始使用。

### Step2：从单节点集群开始

在我们创建一个HA集群之前，让我们从单节点集群开始以理解命令（“grammar”）并且查看默认情况下k3d部署了什么。

首先，语法。在V3，k3d在命令的使用方式上做了很大的改变。我们不深入研究以前的命令是怎么做的，我们将使用V3的语法。

K3s遵循“名词+动词”的句法。首先指定我们要使用的东西（集群或节点），然后指定我们要应用的操作（create、delete、start、stop）。

##### 创建一个单节点集群

我们将借助k3d使用默认值创建一个单节点集群：

安装一个集群，该集群有一个server、一个agent，名为openfunction

```
root@i-k5d6x5d6:~# k3d cluster create --agents 1 openfunction
INFO[0000] Prep: Network                                
INFO[0000] Created network 'k3d-openfunction'           
INFO[0000] Created volume 'k3d-openfunction-images'     
INFO[0000] Starting new tools node...                   
INFO[0001] Starting Node 'k3d-openfunction-tools'       
INFO[0001] Creating node 'k3d-openfunction-server-0'    
INFO[0002] Creating node 'k3d-openfunction-agent-0'     
INFO[0002] Creating LoadBalancer 'k3d-openfunction-serverlb' 
INFO[0002] Using the k3d-tools node to gather environment information 
INFO[0002] HostIP: using network gateway 172.20.0.1 address 
INFO[0002] Starting cluster 'openfunction'              
INFO[0002] Starting servers...                          
INFO[0002] Starting Node 'k3d-openfunction-server-0'    
INFO[0008] Starting agents...                           
INFO[0008] Starting Node 'k3d-openfunction-agent-0'     
INFO[0020] Starting helpers...                          
INFO[0020] Starting Node 'k3d-openfunction-serverlb'    
INFO[0027] Injecting '172.20.0.1 host.k3d.internal' into /etc/hosts of all nodes... 
INFO[0027] Injecting records for host.k3d.internal and for 3 network members into CoreDNS configmap... 
INFO[0028] Cluster 'openfunction' created successfully! 
INFO[0028] You can now use it like this:                
kubectl cluster-info
```

此时使用`kubectl cluster-info`并不能获取信息，那么我们需要安装kubectl

```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

接着安装

```
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

或者将其放于` ~/.local/bin`目录下，一样的效果。

我们还可以从不同的角度看看到底部署了什么。

让我们从头开始，看看K3s集群里面有什么（pods、服务、部署等）：

**kubectl get all --all-namespaces**

```
root@i-k5d6x5d6:~# kubectl get all --all-namespaces
NAMESPACE     NAME                                          READY   STATUS      RESTARTS   AGE
kube-system   pod/coredns-7448499f4d-7gnv2                  1/1     Running     0          21m
kube-system   pod/local-path-provisioner-5ff76fc89d-sqhnq   1/1     Running     0          21m
kube-system   pod/helm-install-traefik-crd-jwfgp            0/1     Completed   0          21m
kube-system   pod/metrics-server-86cbb8457f-9n7q8           1/1     Running     0          21m
kube-system   pod/helm-install-traefik-fg7xq                0/1     Completed   6          21m
kube-system   pod/svclb-traefik-jmq74                       2/2     Running     0          12m
kube-system   pod/svclb-traefik-9pffl                       2/2     Running     0          12m
kube-system   pod/traefik-6b84f7cbc-x78th                   1/1     Running     0          12m

NAMESPACE     NAME                     TYPE           CLUSTER-IP      EXTERNAL-IP             PORT(S)                      AGE
default       service/kubernetes       ClusterIP      10.43.0.1       <none>                  443/TCP                      21m
kube-system   service/kube-dns         ClusterIP      10.43.0.10      <none>                  53/UDP,53/TCP,9153/TCP       21m
kube-system   service/metrics-server   ClusterIP      10.43.228.242   <none>                  443/TCP                      21m
kube-system   service/traefik          LoadBalancer   10.43.36.4      172.20.0.2,172.20.0.3   80:32224/TCP,443:30510/TCP   12m

NAMESPACE     NAME                           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
kube-system   daemonset.apps/svclb-traefik   2         2         2       2            2           <none>          12m

NAMESPACE     NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
kube-system   deployment.apps/coredns                  1/1     1            1           21m
kube-system   deployment.apps/local-path-provisioner   1/1     1            1           21m
kube-system   deployment.apps/metrics-server           1/1     1            1           21m
kube-system   deployment.apps/traefik                  1/1     1            1           12m

NAMESPACE     NAME                                                DESIRED   CURRENT   READY   AGE
kube-system   replicaset.apps/coredns-7448499f4d                  1         1         1       21m
kube-system   replicaset.apps/local-path-provisioner-5ff76fc89d   1         1         1       21m
kube-system   replicaset.apps/metrics-server-86cbb8457f           1         1         1       21m
kube-system   replicaset.apps/traefik-6b84f7cbc                   1         1         1       12m

NAMESPACE     NAME                                 COMPLETIONS   DURATION   AGE
kube-system   job.batch/helm-install-traefik-crd   1/1           5m52s      21m
kube-system   job.batch/helm-install-traefik       1/1           8m28s      21m

```

k3d  cluster delete xxx

删除xxx集群
