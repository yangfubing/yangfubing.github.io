
## NetworkPolicy 

> 在 Kubernetes 中要实现容器之间网络的隔离，是通过一个专门的 API 对象 `NetworkPolicy`（网络策略）来实现的，要让网络策略生效，就需要特定的网络插件支持，目前已经实现了 `NetworkPolicy` 的网络插件包括 Calico、Weave 和 kube-router 等项目，但是并不包括 Flannel 项目。所以说，如果想要在使用 Flannel 的同时还使用 NetworkPolicy 的话，你就需要再额外安装一个网络插件，比如 Calico 项目，来负责执行 NetworkPolicy。由于我们这里使用的是 Flannel 网络插件，所以首先需要安装 Calico 来负责网络策略。

### 安装 Calico 

> 首先确定 kube-controller-manager 配置了如下的两个参数：

```
......
spec:
  containers:
  - command:
    - kube-controller-manager
    - --allocate-node-cidrs=true
    - --cluster-cidr=20/16
......
```

> 然后下载需要使用的资源清单文件：

```
$ curl https://docs.projectcalico.org/v10/manifests/canal.yaml -O
```

> 如果之前配置的 pod CIDR 就是 `10.244.0.0/16` 网段，则可以跳过下面的配置，如果不同则可以使用如下方式进行替换：

```
$ POD_CIDR="<your-pod-cidr>" \\
$ sed -i -e "s?20/16?$POD_CIDR?g" canal.yaml
```

> 然后直接安装即可：

```
$ kubectl apply -f canal.yaml
```

### 网络策略 

> 默认情况下 Pod 是可以接收来自任何发送方的请求，也可以向任何接收方发送请求。而如果我们要对这个情况作出限制，就必须通过 `NetworkPolicy` 对象来指定。

> 如下资源清单文件所示，我们这里定义了一个网络策略资源清单文件：(test-networkpolicy.yaml)

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 10/16
        except:
        - 10/24
    - namespaceSelector:
        matchLabels:
          project: myproject
    - podSelector:
        matchLabels:
          role: frontend
  - ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - ipBlock:
        cidr: 0/24
    ports:
    - protocol: TCP
      port: 5978
```

> 与所有其他的 Kubernetes 资源对象一样，NetworkPolicy 需要 `apiVersion`、`kind` 和 `metadata` 字段，我们通过 `spec.podSelector` 字段定义这个 NetworkPolicy 的限制范围，因为 NetworkPolicy 目前只支持定义 `ingress` 规则，所以这里的 `podSelector` 本质上是为该策略定义 目标pod, 比如我们这里 `matchLabels:role=db` 表示的就是当前 Namespace 里携带了 `role=db` 标签的 Pod。而如果你把 `podSelector` 字段留空：

```
spec: 
  podSelector: {}
```

> 那么这个 NetworkPolicy 就会作用于当前 Namespace 下的所有 Pod。

> 然后每个 NetworkPolicy 包含一个 `policyTypes` 列表，可以是一个 `Ingress`、`Egress` 或者都包含，该字段表示给当前策略是否应用于所匹配的 Pod 的入口流量、出口流量或者二者都包含，如果没有指定 `policyTypes`，则默认情况下表示 `Ingress` 入口流量，如果配置了任何出口流量规则，则将指定为 `Egress`。

> 规则

> 一旦 Pod 被 NetworkPolicy 选中，那么这个 Pod 就会进入`拒绝所有（Deny All）`的状态，即这个 Pod 既不允许被外界访问，也不允许对外界发起访问，所以 NetworkPolicy 定义的规则，其实就是白名单了。

> 比如上面示例表示的是该隔离规则只对 default 命名空间下的，携带了 `role=db` 标签的 Pod 有效。限制的请求类型包括 ingress（流入）和 egress（流出）。

> `ingress`: 每个 NetworkPolicy 包含一个 ingress 规则的白名单列表。其中的规则允许同时匹配 `from` 和 `ports` 部分的流量。比如上面示例中我们配置的入口流量规则如下所示：

```
ingress:
  - from:
    - ipBlock:
        cidr: 10/16
        except:
        - 10/24
    - namespaceSelector:
        matchLabels:
          project: myproject
    - podSelector:
        matchLabels:
          role: frontend
  - ports:
    - protocol: TCP
      port: 80
```

> 这里的 ingress 规则中我们定义了 `from` 和 `ports`，表示允许流入的`白名单`和端口，上面我们也说了 Kubernetes 会拒绝任何访问被隔离 Pod 的请求，除非这个请求来自于以下白名单里的对象，并且访问的是被隔离 Pod 的 8- 端口。而这个允许流入的`白名单`中指定了三种并列的情况，分别是：`ipBlock`、`namespaceSelector` 和 `podSelector`：

*   default 命名空间下面带有 `role=frontend` 标签的 Pod
*   带有 `project=myproject` 标签的 Namespace 下的任何 Pod
*   任何源地址属于 `172.17.0.0/16` 网段，且不属于 `172.17.1.0/24` 网段的请求。

> `egress`: 每个 NetworkPolicy 包含一个 egress 规则的白名单列表。每个规则都允许匹配 `to` 和 `port` 部分的流量。比如我们这里示例规则的配置：

```
egress:
  - to:
    - ipBlock:
        cidr: 0/24
    ports:
    - protocol: TCP
      port: 5978
```

> 表示 Kubernetes 会拒绝被隔离 Pod 对外发起任何请求，除非请求的目的地址属于 `10.0.0.0/24` 网段，并且访问的是该网段地址的 `5978` 端口。

### 测试 

> 比如现在我们创建一个 Pod，带有 `role=db` 的 Label 标签：(test-networkpolicy-pod.yaml)

```
apiVersion: v1
kind: Pod
metadata:
  name: test-networkpolicy
  labels:
    role: db
spec:
  containers:
  - name: testnp
    image: nginx
```

> 直接创建这个 Pod：

```
$ kubectl apply -f test-networkpolicy-pod.yaml
$ kubectl get pods -o wide
NAME                        READY   STATUS    RESTARTS   AGE     IP             NODE         NOMINATED NODE   READINESS GATES
pod-a                       1/1     Running   4          4h3m    2236   ydzs-node1   <none>           <none>
pod-b                       1/1     Running   4          4h3m    2123   ydzs-node2   <none>           <none>
test-networkpolicy          1/1     Running   0          4m49s   22     ydzs-node3   <none>           <none>
```

> 这个时候我们用一个已经存在的 Pod 来对这个 Pod 发起一个网络请求：

```
$ kubectl exec pod-a wget 22
Connecting to 22 (22:80)
saving to 'index.html'
index.html           100% |********************************|   612  0:00:00 ETA
'index.html' saved
```

> 我们可以看到可以成功请求，这个时候我们来创建上面的 NetworkPolicy 对象：

```
$ kubectl apply -f test-networkpolicy.yaml
networkpolicy.networking.k8s.io/test-network-policy created
```

> 这个时候我们创建了一个网络策略，由于匹配了网络策略的就会拒绝所有的网络请求，需要通过白名单来进行开启请求，由于我们这里的测试 Pod 明显没有在白名单之中，所以就会拒绝网络请求了：

```
$ kubectl exec pod-a wget 22
Connecting to 22 (22:80)
# 请求会一直 hang 住
```

> 这个时候我们可以用 NetworkPolicy 白名单里面匹配的 Pod 来对上面的 Pod 发起网络请求，比如在带有标签 `project=myproject` 的 Namespace 下面的 Pod 来发起网络请求，或者给 pod-a 加上一个 `role=frontend` 的标签：

```
$ kubectl label pod pod-a role=frontend
pod/pod-a labeled
```

> 这个时候重新发起一个网络请求可以看到已经可以成功了，因为我们访问的 Pod 进程默认的端口就是 80 端口，是匹配白名单的：

```
$ kubectl exec pod-a -- wget 22 -O-
Connecting to 22 (22:80)
writing to stdout
-                    100% |********************************|   612  0:00:00 ETA
written to stdout
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

> 我们可以看到已经成功了，Kubernetes 的 NetworkPolicy 实现了访问控制，依赖的底层是依靠网络插件添加 iptables 规则来进行控制的，但是在每个节点上都需要配置大量 iptables 规则，加上不同维度控制的增加，导致运维、排障难度较大，所以如果不是特别需要的场景，最好不要使用了。