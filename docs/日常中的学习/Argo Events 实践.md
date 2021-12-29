# Argo Events 对接 OpenFunction

## 部署 Argo Events

基于 v1.5.4 版本

### 安装

```shell
kubectl create namespace argo-events
kubectl apply -n argo-events -f https://raw.githubusercontent.com/argoproj/argo-events/v1.5.4/manifests/install.yaml
kubectl apply -n argo-events -f https://raw.githubusercontent.com/argoproj/argo-events/v1.5.4/manifests/install-validating-webhook.yaml
```

### 部署 eventBus

```shell
kubectl apply -n argo-events -f https://raw.githubusercontent.com/argoproj/argo-events/v1.5.4/examples/eventbus/native.yaml
```

## 设置 GitHub 事件源

1. 创建 GitHub API Token，参考 https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token

   Settings -> Developer settings -> Personal access tokens

   勾选 `repo_hook` 权限

   > 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

2. 对 token 进行 base64 编码

   ```shell
   echo -n <api-token-key> | base64
   ```

   > Z2hwX2tVZlc0T1hIODNNVTlwMlRrcXpuZ1Q1VFB5UlYyeTRTb2szWg==

3. 创建一个名为 `github-access` 的 secret 资源

   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: github-access
   type: Opaque
   data:
     token: <base64-encoded-api-token-from-previous-step>
     secret: <base64-encoded-webhook-secret-key>
   ```


4. 修改 events-webhook svc 为 nodeport 模式
