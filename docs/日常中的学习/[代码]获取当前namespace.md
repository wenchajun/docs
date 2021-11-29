方法一

添加字段

```yaml
    env:
        - name: MY_POD_NAMESPACE
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
```

完整如下

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: fluentbit-operator
  namespace: kubesphere-logging-system
  labels:
    app.kubernetes.io/component: operator
    app.kubernetes.io/name: fluentbit-operator
  annotations:
    deployment.kubernetes.io/revision: '5'
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/component: operator
      app.kubernetes.io/name: fluentbit-operator
  template:
    metadata:
      creationTimestamp: null
      labels:
        app.kubernetes.io/component: operator
        app.kubernetes.io/name: fluentbit-operator
    spec:
      containers:
        - name: fluentbit-operator
          image: 'wenchajun/fluentbit-operator:v1.3'
          env:
            - name: MY_POD_NAMESPACE
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: metadata.namespace
          resources:
            limits:
              cpu: 100m
              memory: 30Mi

```

代码中直接get即可

```go
		 ns:= os.Getenv("MY_POD_NAMESPACE")
	     fmt.Println("my namespace is ",ns)

```

二.读取文件

```go
  
		filePath :="/var/run/secrets/kubernetes.io/serviceaccount/namespace"
		nsRead ,err :=ioutil.ReadFile(filePath)
		if err !=nil {
			return ctrl.Result{}, err
		}  
```

