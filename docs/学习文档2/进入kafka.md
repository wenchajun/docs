# 进入kafka查看消息

安装kafka集群：

使用该脚本安装kafka

https://github.com/kubesphere-sigs/fluent-operator-walkthrough/blob/master/deploy-kafka.sh

```shell
#!/bin/bash
set -eu
# Simple script to deploy Kafka to a Kubernetes cluster with context already set
KAFKA_NAMESPACE=${KAFKA_NAMESPACE:-kafka}
CLUSTER_NAME=${CLUSTER_NAME:-fluent}

if [[ "${INSTALL_HELM:-no}" == "yes" ]]; then
    # See https://helm.sh/docs/intro/install/
    curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
fi

# docker pull quay.io/strimzi/kafka:0.28.0-kafka-3.1.0
# docker pull quay.io/strimzi/operator:0.28.0
# kind load docker-image quay.io/strimzi/kafka:0.28.0-kafka-3.1.0 --name fluent
# kind load docker-image quay.io/strimzi/operator:0.28.0 --name fluent

kubectl create ns $KAFKA_NAMESPACE
kubectl create -f 'https://strimzi.io/install/latest?namespace=kafka' -n $KAFKA_NAMESPACE
kubectl apply -f https://strimzi.io/examples/latest/kafka/kafka-persistent-single.yaml -n $KAFKA_NAMESPACE
kubectl wait kafka/my-cluster --for=condition=Ready --timeout=300s -n $KAFKA_NAMESPACE 
```

部署后查看pod和svc

```
root@i-f1or71kf:~# kubectl get pod -n kafka
NAME                                         READY   STATUS    RESTARTS   AGE
my-cluster-entity-operator-6d5ff97f6-tc4c4   3/3     Running   0          23h
my-cluster-kafka-0                           1/1     Running   0          23h
my-cluster-zookeeper-0                       1/1     Running   0          23h
strimzi-cluster-operator-587cb79468-66t6b    1/1     Running   0          23h
root@i-f1or71kf:~# kubectl get svc -n kafka
NAME                          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                               AGE
my-cluster-kafka-bootstrap    ClusterIP   10.96.76.79     <none>        9091/TCP,9092/TCP,9093/TCP            23h
my-cluster-kafka-brokers      ClusterIP   None            <none>        9090/TCP,9091/TCP,9092/TCP,9093/TCP   23h
my-cluster-zookeeper-client   ClusterIP   10.96.190.170   <none>        2181/TCP                              23h
my-cluster-zookeeper-nodes    ClusterIP   None            <none>        2181/TCP,2888/TCP,3888/TCP            23h


```

进入kafka容器

```
kubectl exec -it my-cluster-kafka-0 -n kafka -- sh
```

运行脚本

```
sh-4.4$ ./bin/kafka-topics.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --list            
__consumer_offsets
__strimzi-topic-operator-kstreams-topic-store-changelog
__strimzi_store_topic
fluent-log

```

或者

```
kubectl -n kafka exec -it my-cluster-kafka-0 -- sh bin/kafka-topics.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --list
```

