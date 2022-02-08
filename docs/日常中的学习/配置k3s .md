参考[K3s: Lightweight Kubernetes](https://k3s.io/)

在[Releases · k3s-io/k3s (github.com)](https://github.com/k3s-io/k3s/releases)下载 latest 版本的代码



## 环境1

x86 * 2

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
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable=traefik --write-kubeconfig ~/.kube/config" INSTALL_K3S_VERSION=v1.20.11+k3s2 K3S_NODE_NAME=Openfunction  sh -s -
```



```
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.20.11+k3s2 K3S_NODE_NAME=Openfunction sh -
```



### Server

```shell
curl -sfL https://get.k3s.io | INSTALL_K3S_CHANNEL=latest K3S_NODE_NAME=salmon sh -
```

启动后，获取 server 的地址和 token

token 位于 **/var/lib/rancher/k3s/server/node-token**

### Agent

```shell
curl -sfL https://get.k3s.io | INSTALL_K3S_CHANNEL=latest K3S_NODE_NAME=snail K3S_URL=https://192.168.0.3:6443 K3S_TOKEN="K102af05b2674156ae810fb474d7dca3bbd5fd7c4605ed5e817c2c618af1980e352::server:395b928545d56bc869db82b4d06bce1b" sh -
```

```
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.20.11+k3s2 K3S_NODE_NAME=raspberrypi \
K3S_URL=https://192.168.1.5:6443 \
K3S_TOKEN="K1013a3603ddd28ad7a52e665000fdd46850e21abbe7eb61143e424cc18f12a8a44::server:baa6ce02043596b79f24a165dbacb436"  sh -
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
     wget https://raw.githubusercontent.com/OpenFunction/OpenFunction/main/hack/deploy.sh
     ls
     chmod +x deploy.sh 
     ./deploy.sh --all
     kubectl get pods -A
     kubectl create -f https://github.com/OpenFunction/OpenFunction/releases/download/v0.4.0/bundle.yaml
     kubectl create -f https://raw.githubusercontent.com/OpenFunction/OpenFunction/main/config/bundle.yaml

```





### 安装 Mosquitto

在 snail（边缘节点）上创建两个目录：

```shell
mkdir -p /data/mosquitto/data /data/mosquitto/log
```

创建 Mosquitto 服务，文件名 `mosquitto.yaml`，内容如下：

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: mosquitto
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      name: mosquitto
  template:
    metadata:
      labels:
        name: mosquitto
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/hostname
                operator: In
                values:
                - snail
      containers:
        - name: mosquitto
          image: eclipse-mosquitto:2.0
          ports:
          - containerPort: 1883
          volumeMounts:
          - name: mosquitto-config-file
            mountPath: /mosquitto/config
          - name: mosquitto-pwfile
            mountPath: /mosquitto/config/pwfile
          - name: mosquitto-log
            mountPath: /mosquitto/log
          - name: mosquitto-data
            mountPath: /mosquitto/data
      volumes:
      - name: mosquitto-config-file
        configMap:
          name: mosquitto-config
      - name: mosquitto-pwfile
        secret:
          secretName: mosquitto-pwfile
      - name: mosquitto-data
        persistentVolumeClaim:
          claimName: mosquitto-data-pvc
      - name: mosquitto-log
        persistentVolumeClaim:
          claimName: mosquitto-log-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: mosquitto
  namespace: default
spec:
  ports:
    - protocol: TCP
      name: web
      port: 1883
      targetPort: 1883
  selector:
    name: mosquitto
---
apiVersion: v1
kind: Secret
metadata:
  name: mosquitto-pwfile
  namespace: default
type: Opaque
data:
  # "admin:openfunction"
  pwfile: |
    YWRtaW46JDckMTAxJDcrdDcyUzU0ZmlaU3RiSTEkQVg5NTZZVkg5V3pnSGZQbGVoaDYyd3ZrNkNUTzE0aEREcUpIS2tSMGp2Y2lqTlc1M0Uvd3ZJeVFlQWRzYk0yWmlUYW1OQ0tzMG1ZcTAwNDNaaUFySFE9PQo=
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: mosquitto-config
  namespace: default
data:
  mosquitto.conf: |
    password_file /mosquitto/config/pwfile/pwfile
    listener 1883
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: mosquitto-data-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mosquitto-log-pv
spec:
  capacity:
    storage: 100Mi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-path
  local:
    path: /data/mosquitto/log
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - snail
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: mosquitto-log-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mosquitto-data-pv
spec:
  capacity:
    storage: 100Mi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-path
  local:
    path: /data/mosquitto/data
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - snail
```

执行：

```shell
kubectl apply -f mosquitto.yaml
```

### 创建函数

`function.yaml`:

```yaml
apiVersion: core.openfunction.io/v1alpha2
kind: Function
metadata:
  name: mqtt-handler
spec:
  version: "v1.0.0"
  image: zephyrfish/mqtt-handler:v0.4.0
  imageCredentials:
    name: push-secret
  build:
    builder: openfunctiondev/go116-builder:v0.3.0
    env:
      FUNC_NAME: "MQTTHandler"
    srcRepo:
      url: "https://github.com/tpiperatgod/mqtt-events-handler.git"
      sourceSubPath: "/"
  serving:
    runtime: "OpenFuncAsync"
    template:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/hostname
                operator: In
                values:
                - snail
      containers:
       - name: function
         imagePullPolicy: IfNotPresent
    openFuncAsync:
      dapr:
        inputs:
          - name: edge-events
            component: mqtt
            type: bindings
        annotations:
          dapr.io/log-level: "debug"
        components:
          mqtt:
            type: bindings.mqtt
            version: v1
            metadata:
              - name: consumerID
                value: "consumerA"
              - name: url
                value: "tcp://admin:public@mosquitto.default.svc.cluster.local:1883"
              - name: topic
                value: "topic1"
              - name: qos
                value: 1
              - name: retain
                value: "false"
              - name: cleanSession
                value: "false"
```

### 创建事件生产者

`events-producer.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: events-producer
  labels:
    app: eventsproducer
spec:
  replicas: 1
  selector:
    matchLabels:
      app: eventsproducer
  template:
    metadata:
      labels:
        app: eventsproducer
      annotations:
        dapr.io/enabled: "true"
        
        dapr.io/app-id: "events-producer"
        dapr.io/log-as-json: "true"
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/hostname
                operator: In
                values:
                - snail
      containers:
        - name: producer
          image: openfunctiondev/events-producer:latest
          imagePullPolicy: Always
          env:
            - name: TARGET_NAME
              value: "eventsource-my-eventsource-kafka-sample-two"
```

