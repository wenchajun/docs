## 给节点添加标签

1. 列出集群中的节点及其标签：

   ```shell
   kubectl get nodes --show-labels
   ```

   输出类似于此：

   ```
   NAME      STATUS    ROLES    AGE     VERSION        LABELS
   worker0   Ready     <none>   1d      v1.13.0        ...,kubernetes.io/hostname=worker0
   worker1   Ready     <none>   1d      v1.13.0        ...,kubernetes.io/hostname=worker1
   worker2   Ready     <none>   1d      v1.13.0        ...,kubernetes.io/hostname=worker2
   ```

2. 选择一个节点，给它添加一个标签：

   ```shell
   kubectl label nodes <your-node-name> disktype=ssd
   ```

```
kubectl label nodes node4 KubeSphere=logging
```

3.验证你所选节点具有 `disktype=ssd` 标签：

```shell
kubectl get nodes --show-labels
```

4. 输出类似于此：

```
NAME      STATUS    ROLES    AGE     VERSION        LABELS
worker0   Ready     <none>   1d      v1.13.0        ...,disktype=ssd,kubernetes.io/hostname=worker0
worker1   Ready     <none>   1d      v1.13.0        ...,kubernetes.io/hostname=worker1
worker2   Ready     <none>   1d      v1.13.0        ...,kubernetes.io/hostname=worker2
```

在前面的输出中，可以看到 `worker0` 节点有一个 `disktype=ssd` 标签。

```
AME    STATUS   ROLES                  AGE   VERSION   LABELS
node2   Ready    control-plane,master   8d    v1.21.5   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=node2,kubernetes.io/os=linux,logging=true,node-role.kubernetes.io/control-plane=,node-role.kubernetes.io/master=,node.kubernetes.io/exclude-from-external-load-balancers=,topology.disk.csi.qingcloud.com/instance-type=Standard,topology.disk.csi.qingcloud.com/zone=ap2a
node3   Ready    worker                 8d    v1.21.5   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,hydro-node=true,kubernetes.io/arch=amd64,kubernetes.io/hostname=node3,kubernetes.io/os=linux,node-role.kubernetes.io/worker=,topology.disk.csi.qingcloud.com/instance-type=Standard,topology.disk.csi.qingcloud.com/zone=ap2a
node4   Ready    worker                 8d    v1.21.5   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,hydro-node=true,hydro-role=master,kubernetes.io/arch=amd64,kubernetes.io/hostname=node4,kubernetes.io/os=linux,node-role.kubernetes.io/worker=,topology.disk.csi.qingcloud.com/instance-type=Standard,topology.disk.csi.qingcloud.com/zone=ap2a

```

将这个

```yaml
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app.kubernetes.io/component: operator
      app.kubernetes.io/name: fluentbit-operator
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app.kubernetes.io/component: operator
        app.kubernetes.io/name: fluentbit-operator
    spec:
      affinity: {}
      containers:
      - image: kubesphere/fluentbit-operator:v0.11.0
        imagePullPolicy: IfNotPresent
        name: fluentbit-operator
        resources:
          limits:
            cpu: 100m
            memory: 50Mi
          requests:
            cpu: 10m
            memory: 20Mi
        terminationMessagePath: /dev/termination-log

```

改为

```yaml
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app.kubernetes.io/component: operator
      app.kubernetes.io/name: fluentbit-operator
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app.kubernetes.io/component: operator
        app.kubernetes.io/name: fluentbit-operator
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: logging
                operator: In
                values:
                - true
      containers:
      - image: kubesphere/fluentbit-operator:v0.11.0
        imagePullPolicy: IfNotPresent
        name: fluentbit-operator
        resources:


```

