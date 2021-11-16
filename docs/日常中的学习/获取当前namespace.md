本来搜索到的，在stack上看到

https://stackoverflow.com/questions/55314152/how-to-get-namespace-from-current-context-set-in-kube-config/55315895#55315895

然后又进入

[How to get namespace from current-context set in .kube/config](https://stackoverflow.com/questions/55314152/how-to-get-namespace-from-current-context-set-in-kube-config)

感觉这个回答比较好

This code, based on answer https://stackoverflow.com/a/55315895/2784039, returns the namespace set in the current context of the `kubeconfig` file. In addition, it sets `namespace` to `default` if `namespace` is not defined in the `kubeconfig` current context:

```go
package main

import (
    "fmt"

    "k8s.io/client-go/tools/clientcmd"
)

func main() {

    clientCfg, _ := clientcmd.NewDefaultClientConfigLoadingRules().Load()
    namespace := clientCfg.Contexts[clientCfg.CurrentContext].Namespace

    if namespace == "" {
        namespace = "default"
    }
    fmt.Printf(namespace)
}
```

This code works fine on user side, it it is run outside the cluster. Check comments below to retrieve `namespace` for a pod inside the cluster.