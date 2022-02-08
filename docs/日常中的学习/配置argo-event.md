本例中 k3s 的下载地址为 https://github.com/k3s-io/k3s/releases/download/v1.19.15%2Bk3s2/k3s

```shell
wget https://github.com/k3s-io/k3s/releases/download/v1.19.15%2Bk3s2/k3s
chmod +x k3s
mv k3s /usr/local/bin/
```

or

```shell
curl -sfL https://get.k3s.io | INSTALL_K3S_CHANNEL=latest sh -
```

在这里引入了环境变量,该配置会将config放到k8s中config的默认配置

```
export INSTALL_K3S_EXEC="--write-kubeconfig ~/.kube/config"
export INSTALL_K3S_VERSION=v1.20.11+k3s2
export K3S_NODE_NAME=Openfunction
```

```
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable=traefik --write-kubeconfig ~/.kube/config" INSTALL_K3S_VERSION=v1.20.11+k3s2 K3S_NODE_NAME=Openfunction  sh -s -
```



```
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable=traefik" sh -s -
```



### Server

```shell
curl -sfL https://get.k3s.io | INSTALL_K3S_CHANNEL=latest K3S_NODE_NAME=salmon sh -
```

启动后，获取 server 的地址和 token

token 位于 **/var/lib/rancher/k3s/server/node-token**

### Agent

```shell
cat /var/lib/rancher/k3s/server/node-token
curl -sfL https://get.k3s.io | INSTALL_K3S_CHANNEL=latest K3S_NODE_NAME=snail K3S_URL=https://192.168.0.3:6443 K3S_TOKEN="K102af05b2674156ae810fb474d7dca3bbd5fd7c4605ed5e817c2c618af1980e352::server:395b928545d56bc869db82b4d06bce1b" sh -
```

```
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.20.11+k3s2 K3S_NODE_NAME=raspberrypi \
K3S_URL=https://192.168.1.5:6443 \
K3S_TOKEN="K1007e23eafba4674f7ee5384b74188a6cba343ed1da11c23981b096819b64c018e::server:29b2f2f5dc00499e62edc132784fcf43"  sh -
```



现在我们有两个节点：

Server 节点 salmon

Agent 节点 snail 具备 edge=true 标签

```
kubectl label node raspberrypi edge=true
```

```
kubectl label node raspberrypi node-role.kubernetes.io/edge=true
```



```shell
~# kubectl get nodes  --show-labels
NAME     STATUS   ROLES                  AGE   VERSION        LABELS
snail    Ready    edge                   22h   v1.22.2+k3s2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=k3s,beta.kubernetes.io/os=linux,edge=true,kubernetes.io/arch=amd64,kubernetes.io/hostname=snail,kubernetes.io/os=linux,node-role.kubernetes.io/edge=true,node.kubernetes.io/instance-type=k3s
salmon   Ready    control-plane,master   22h   v1.22.2+k3s2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=k3s,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=salmon,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=true,node-role.kubernetes.io/master=true,node.kubernetes.io/instance-type=k3s
```

安装openfunction

```
wget -c  https://github.com/OpenFunction/cli/releases/download/v0.5.1/ofn_linux_amd64.tar.gz -O - | tar -xz

```



安装openfunction









可以参考https://argoproj.github.io/argo-events/installation/

第一步，创建namesapce

```
kubectl create namespace argo-events
```

第二步, 部署Argo Events, SA, ClusterRoles, Sensor Controller, EventBus Controller和EventSource Controller。

```
kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-events/stable/manifests/install.yaml
# Install with a validating admission controller
kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-events/stable/manifests/install-validating-webhook.yaml
```

本次实验中安装了

```yaml

kubectl create namespace argo-events
 kubectl apply -n argo-events -f https://raw.githubusercontent.com/argoproj/argo-events/v1.5.4/manifests/install.yaml
 kubectl apply -n argo-events -f https://raw.githubusercontent.com/argoproj/argo-events/v1.5.4/manifests/install-validating-webhook.yaml
 kubectl get pods -n  argo-events
  kubectl apply -n argo-events -f https://raw.githubusercontent.com/argoproj/argo-events/v1.5.4/examples/eventbus/native.yaml
 kubectl get pods -n  argo-events


```

https://argoproj.github.io/argo-events/eventsources/setup/github/

```
apiVersion: v1
kind: Secret
metadata:
  name: github-access
type: Opaque
data:
  token: <base64-encoded-api-token-from-previous-step>
  secret: <base64-encoded-webhook-secret-key>
```



```
kubectl -n argo-events apply -f github-access.yaml
```



