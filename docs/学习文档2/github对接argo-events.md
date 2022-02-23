参考[K3s: Lightweight Kubernetes](https://k3s.io/)

在[Releases · k3s-io/k3s (github.com)](https://github.com/k3s-io/k3s/releases)下载 latest 或者某一版本k3s的代码

## 环境

x86 * 2

本例中 k3s 的下载地址为 https://github.com/k3s-io/k3s/releases/download/v1.20.11%2Bk3s2/k3s

```shell
wget https://github.com/k3s-io/k3s/releases/download/v1.20.11%2Bk3s2/k3s
chmod +x k3s
mv k3s /usr/local/bin/
```

or

```shell
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.20.11+k3s2 sh -
```

### Server

在这里我们采用了第二种方法安装server端，通过提供的bash脚本进行安装，同时我们可以通过设置参数来对k3s进行自定义安装。

```
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable=traefik --write-kubeconfig ~/.kube/config" INSTALL_K3S_VERSION=v1.20.11+k3s2 K3S_NODE_NAME=Openfunction  sh -s -
```

这里对参数进行说明，`--disable=traefik`表示禁用traefik，在本文中将使用更通用的ingress-nginx来控制网络网络流量。 `--write-kubeconfig ~/.kube/config`则表示将吧kubeconfig文件写入`~/.kube/config`中，此举是为了在下面的步骤中方便安装OpenFunction，因为`ofn`将读取`~/.kube/config`以安装openfunction。`INSTALL_K3S_VERSION=v1.20.11+k3s2`则表示将安装v1.20.11版本的k3s。`K3S_NODE_NAME=Openfunction`顾名思义，表示它的名字是openfunction。

启动后，获取 server 的地址和 token

token 位于 **/var/lib/rancher/k3s/server/node-token**

### Agent

```
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.20.11+k3s2 K3S_NODE_NAME=raspberrypi \
K3S_URL=https://192.168.1.5:6443 \
K3S_TOKEN="K1013a3603ddd28ad7a52e665000fdd46850e21abbe7eb61143e424cc18f12a8a44::server:baa6ce02043596b79f24a165dbacb436"  sh -
```

这里`K3S_TOKEN`是server端的token，而`K3S_URL`是server端的ip地址。

现在我们有两个节点：

Server 节点 openfunction

Agent 节点 raspberrypi 

```
root@i-k5d6x5d6:~# kubectl get nodes
NAME           STATUS   ROLES                  AGE     VERSION
raspberrypi    Ready    <none>                 2d19h   v1.20.11+k3s2
openfunction   Ready    control-plane,master   4d14h   v1.20.11+k3s2
```

安装openfunction,下载ofn客户端进行安装

```
wget -c  https://github.com/OpenFunction/cli/releases/download/v0.5.1/ofn_linux_amd64.tar.gz -O - | tar -xz
```

此时下载ofn客户端，安装openfunction

```
./ofn install --all
```

接下来安装argo-events，具体安装参考https://argoproj.github.io/argo-events/installation/

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

第三步.如果您没有 API 令牌，请创建一个。按照[说明](https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line)创建新的 GitHub API 令牌。授予它`repo_hook`权限。

第四步.Base64 对您的 API 令牌密钥进行编码。

```
echo -n <api-token-key> | base64
```

第五步.创建一个名为的密钥`github-access`，其中包含您编码的 GitHub API 令牌。`base64`您还可以包含一个为您的 webhook编码的密钥（如果有）。

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

第六步.将密钥部署到 K8s 集群中。

```
kubectl -n argo-events apply -f github-access.yaml
```

第七步. GitHub 的事件源创建一个 pod 并通过服务公开它。服务的名称采用`<event-source-name>-eventsource-svc`格式。您需要为事件源服务创建一个 Ingress 或 Openshift Route，以便可以从 GitHub 访问它。您可以在线找到有关 Ingress 或 Route 的更多信息。

因为使用argo-events需要由github发送消息由服务器接收，所以需要安装配置相应的ingress，以便可以从外网访问。根据  https://kubernetes.github.io/ingress-nginx/deploy/#bare-metal  安装nginx 

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.1.1/deploy/static/provider/cloud/deploy.yaml
```

此时安装负载均衡插件，因为我使用的是qingcloud的服务器，所以参考：

https://github.com/yunify/qingcloud-cloud-controller-manager进行配置。

```
kubectl edit svc ingress-nginx-controller -n ingress-nginx
```

```
apiVersion: v1
kind: Service
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"app.kubernetes.io/component":"controller","app.kubernetes.io/instance":"ingress-nginx","app.kubernetes.io/
    service.beta.kubernetes.io/qingcloud-load-balancer-eip-ids: eip-aisa58w7 ##注解写绑定的青云的eip
    service.beta.kubernetes.io/qingcloud-load-balancer-type: "0"
  creationTimestamp: "2022-02-07T08:49:52Z"
  finalizers:
  - service.kubernetes.io/load-balancer-cleanup
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/version: 1.1.1
    helm.sh/chart: ingress-nginx-4.0.15
  name: ingress-nginx-controller
  namespace: ingress-nginx
  resourceVersion: "2141992"
  uid: 966f7ae3-abfb-4ffe-9e02-1e3392b91dd3
spec:
  clusterIP: 10.43.64.4
  clusterIPs:
  - 10.43.64.4
  externalTrafficPolicy: Local
  healthCheckNodePort: 31819
  ports:
  - appProtocol: http
    name: http
    nodePort: 32226
    port: 80
```

配置ingress，以便github的webhook可以访问

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  namespace: argo-events
  name: ingress-webhook
spec:
  ingressClassName: nginx
  rules:
  - host: webhook.103.61.37.229.sslip.io
    http:
      paths:
      - path: /push
        pathType: Prefix
        backend:
          service:
            name: github-eventsource-svc
            port:
              number: 12000
```

创建事件源，确保更换了url 字段

```
apiVersion: argoproj.io/v1alpha1
kind: EventSource
metadata:
  name: github
  namespace: argo-events
spec:
  service:
    ports:
      - port: 12000
        targetPort: 12000
  github:
    example:
      repositories:
        - owner: wenchajun
          names:
            - argotest
      # Github will send events to following port and endpoint
      webhook:
        # endpoint to listen to events on
        endpoint: /push
        # port to run internal HTTP server on
        port: "12000"
        # HTTP request method to allow. In this case, only POST requests are accepted
        method: POST
        # url the event-source will use to register at Github.
        # This url must be reachable from outside the cluster.
        # The name for the service is in `<event-source-name>-eventsource-svc` format.
        # You will need to create an Ingress or Openshift Route for the event-source service so that it can be reached from GitHub.
        url: http://webhook.103.61.37.229.sslip.io
      # type of events to listen to.
      # following listens to everything, hence *
      # You can find more info on https://developer.github.com/v3/activity/events/types/
      events:
        - "*"

      # apiToken refers to K8s secret that stores the github api token
      # if apiToken is provided controller will create webhook on GitHub repo
      # +optional
      apiToken:
        # Name of the K8s secret that contains the access token
        name: github-access
        # Key within the K8s secret whose corresponding value (must be base64 encoded) is access token
        key: token

#      # webhookSecret refers to K8s secret that stores the github hook secret
#      # +optional
#      webhookSecret:
#        # Name of the K8s secret that contains the hook secret
#        name: github-access
#        # Key within the K8s secret whose corresponding value (must be base64 encoded) is hook secret
#        key: secret

      # type of the connection between event-source and Github.
      # You should set it to false to avoid man-in-the-middle and other attacks.
      insecure: true
      # Determines if notifications are sent when the webhook is triggered
      active: true
      # The media type used to serialize the payloads
      contentType: json

    example-without-api-credentials:
      owner: "wenchajun"
      repository: "argotest"
      webhook:
        endpoint: "/push"
        port: "13000"
        method: "POST"
        url: "http://webhook.103.61.37.229.sslip.io"
      events:
        - "*"
      webhookSecret:
        name: github-access
        key: secret
      insecure: true
      active: true
      contentType: "json"

```

```
kubectl apply -n argo-events -f <event-source-file-updated-in-previous-step>
```

第八步，转到`Webhooks`GitHub 上的项目设置下并验证 webhook 是否已注册。您也可以通过查看事件源 pod 日志来执行相同操作。

![image-20220208155419248](docs\images\github-action.png)

至此github-Action已经对接到主机端的openfunction客户端

下一步我们需要将argo-events中收到的数据进行处理，然后发送到openfunction

Header的数据：

```

Request URL: http://webhook.103.61.39.15.sslip.io/push
Request method: POST
Accept: */*
content-type: application/json
User-Agent: GitHub-Hookshot/5fb002b
X-GitHub-Delivery: 4cf09b40-882f-11ec-9803-99d0b409fba0
X-GitHub-Event: ping
X-GitHub-Hook-ID: 342057160
X-GitHub-Hook-Installation-Target-ID: 447054217
X-GitHub-Hook-Installation-Target-Type: repository
```

Payload

```
{
  "zen": "Half measures are as bad as nothing at all.",
  "hook_id": 342057160,
  "hook": {
    "type": "Repository",
    "id": 342057160,
    "name": "web",
    "active": true,
    "events": [
      "*"
    ],
    "config": {
      "content_type": "json",
      "insecure_ssl": "1",
      "url": "http://webhook.103.61.39.15.sslip.io/push"
    },
    "updated_at": "2022-02-07T16:02:07Z",
    "created_at": "2022-02-07T16:02:07Z",
    "url": "https://api.github.com/repos/wenchajun/argotest/hooks/342057160",
    "test_url": "https://api.github.com/repos/wenchajun/argotest/hooks/342057160/test",
    "ping_url": "https://api.github.com/repos/wenchajun/argotest/hooks/342057160/pings",
    "deliveries_url": "https://api.github.com/repos/wenchajun/argotest/hooks/342057160/deliveries",
    "last_response": {
      "code": null,
      "status": "unused",
      "message": null
    }
  },
  "repository": {
    "id": 447054217,
    "node_id": "R_kgDOGqWBiQ",
    "name": "argotest",
    "full_name": "wenchajun/argotest",
    "private": false,
    "owner": {
      "login": "wenchajun",
      "id": 39892300,
      "node_id": "MDQ6VXNlcjM5ODkyMzAw",
      "avatar_url": "https://avatars.githubusercontent.com/u/39892300?v=4",
      "gravatar_id": "",
      "url": "https://api.github.com/users/wenchajun",
      "html_url": "https://github.com/wenchajun",
      "followers_url": "https://api.github.com/users/wenchajun/followers",
      "following_url": "https://api.github.com/users/wenchajun/following{/other_user}",
      "gists_url": "https://api.github.com/users/wenchajun/gists{/gist_id}",
      "starred_url": "https://api.github.com/users/wenchajun/starred{/owner}{/repo}",
      "subscriptions_url": "https://api.github.com/users/wenchajun/subscriptions",
      "organizations_url": "https://api.github.com/users/wenchajun/orgs",
      "repos_url": "https://api.github.com/users/wenchajun/repos",
      "events_url": "https://api.github.com/users/wenchajun/events{/privacy}",
      "received_events_url": "https://api.github.com/users/wenchajun/received_events",
      "type": "User",
      "site_admin": false
    },
    "html_url": "https://github.com/wenchajun/argotest",
    "description": null,
    "fork": false,
    "url": "https://api.github.com/repos/wenchajun/argotest",
    "forks_url": "https://api.github.com/repos/wenchajun/argotest/forks",
    "keys_url": "https://api.github.com/repos/wenchajun/argotest/keys{/key_id}",
    "collaborators_url": "https://api.github.com/repos/wenchajun/argotest/collaborators{/collaborator}",
    "teams_url": "https://api.github.com/repos/wenchajun/argotest/teams",
    "hooks_url": "https://api.github.com/repos/wenchajun/argotest/hooks",
    "issue_events_url": "https://api.github.com/repos/wenchajun/argotest/issues/events{/number}",
    "events_url": "https://api.github.com/repos/wenchajun/argotest/events",
    "assignees_url": "https://api.github.com/repos/wenchajun/argotest/assignees{/user}",
    "branches_url": "https://api.github.com/repos/wenchajun/argotest/branches{/branch}",
    "tags_url": "https://api.github.com/repos/wenchajun/argotest/tags",
    "blobs_url": "https://api.github.com/repos/wenchajun/argotest/git/blobs{/sha}",
    "git_tags_url": "https://api.github.com/repos/wenchajun/argotest/git/tags{/sha}",
    "git_refs_url": "https://api.github.com/repos/wenchajun/argotest/git/refs{/sha}",
    "trees_url": "https://api.github.com/repos/wenchajun/argotest/git/trees{/sha}",
    "statuses_url": "https://api.github.com/repos/wenchajun/argotest/statuses/{sha}",
    "languages_url": "https://api.github.com/repos/wenchajun/argotest/languages",
    "stargazers_url": "https://api.github.com/repos/wenchajun/argotest/stargazers",
    "contributors_url": "https://api.github.com/repos/wenchajun/argotest/contributors",
    "subscribers_url": "https://api.github.com/repos/wenchajun/argotest/subscribers",
    "subscription_url": "https://api.github.com/repos/wenchajun/argotest/subscription",
    "commits_url": "https://api.github.com/repos/wenchajun/argotest/commits{/sha}",
    "git_commits_url": "https://api.github.com/repos/wenchajun/argotest/git/commits{/sha}",
    "comments_url": "https://api.github.com/repos/wenchajun/argotest/comments{/number}",
    "issue_comment_url": "https://api.github.com/repos/wenchajun/argotest/issues/comments{/number}",
    "contents_url": "https://api.github.com/repos/wenchajun/argotest/contents/{+path}",
    "compare_url": "https://api.github.com/repos/wenchajun/argotest/compare/{base}...{head}",
    "merges_url": "https://api.github.com/repos/wenchajun/argotest/merges",
    "archive_url": "https://api.github.com/repos/wenchajun/argotest/{archive_format}{/ref}",
    "downloads_url": "https://api.github.com/repos/wenchajun/argotest/downloads",
    "issues_url": "https://api.github.com/repos/wenchajun/argotest/issues{/number}",
    "pulls_url": "https://api.github.com/repos/wenchajun/argotest/pulls{/number}",
    "milestones_url": "https://api.github.com/repos/wenchajun/argotest/milestones{/number}",
    "notifications_url": "https://api.github.com/repos/wenchajun/argotest/notifications{?since,all,participating}",
    "labels_url": "https://api.github.com/repos/wenchajun/argotest/labels{/name}",
    "releases_url": "https://api.github.com/repos/wenchajun/argotest/releases{/id}",
    "deployments_url": "https://api.github.com/repos/wenchajun/argotest/deployments",
    "created_at": "2022-01-12T02:46:42Z",
    "updated_at": "2022-01-12T02:46:42Z",
    "pushed_at": "2022-01-13T05:58:39Z",
    "git_url": "git://github.com/wenchajun/argotest.git",
    "ssh_url": "git@github.com:wenchajun/argotest.git",
    "clone_url": "https://github.com/wenchajun/argotest.git",
    "svn_url": "https://github.com/wenchajun/argotest",
    "homepage": null,
    "size": 3,
    "stargazers_count": 0,
    "watchers_count": 0,
    "language": null,
    "has_issues": true,
    "has_projects": true,
    "has_downloads": true,
    "has_wiki": true,
    "has_pages": false,
    "forks_count": 0,
    "mirror_url": null,
    "archived": false,
    "disabled": false,
    "open_issues_count": 1,
    "license": null,
    "allow_forking": true,
    "is_template": false,
    "topics": [

    ],
    "visibility": "public",
    "forks": 0,
    "open_issues": 1,
    "watchers": 0,
    "default_branch": "main"
  },
  "sender": {
    "login": "wenchajun",
    "id": 39892300,
    "node_id": "MDQ6VXNlcjM5ODkyMzAw",
    "avatar_url": "https://avatars.githubusercontent.com/u/39892300?v=4",
    "gravatar_id": "",
    "url": "https://api.github.com/users/wenchajun",
    "html_url": "https://github.com/wenchajun",
    "followers_url": "https://api.github.com/users/wenchajun/followers",
    "following_url": "https://api.github.com/users/wenchajun/following{/other_user}",
    "gists_url": "https://api.github.com/users/wenchajun/gists{/gist_id}",
    "starred_url": "https://api.github.com/users/wenchajun/starred{/owner}{/repo}",
    "subscriptions_url": "https://api.github.com/users/wenchajun/subscriptions",
    "organizations_url": "https://api.github.com/users/wenchajun/orgs",
    "repos_url": "https://api.github.com/users/wenchajun/repos",
    "events_url": "https://api.github.com/users/wenchajun/events{/privacy}",
    "received_events_url": "https://api.github.com/users/wenchajun/received_events",
    "type": "User",
    "site_admin": false
  }
}
```

