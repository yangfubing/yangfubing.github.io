
## Pod 使用进阶 

> 深入理解 Pod 对象

### 静态 Pod 

> 在 Kubernetes 集群中除了我们经常使用到的普通的 Pod 外，还有一种特殊的 Pod，叫做Static Pod，也就是我们说的静态 Pod，静态 Pod 有什么特殊的地方呢？

> 静态 Pod 直接由节点上的 kubelet 进程来管理，不通过 master 节点上的 apiserver。无法与我们常用的控制器 Deployment 或者 DaemonSet 进行关联，它由 kubelet 进程自己来监控，当 pod 崩溃时会重启该 pod，kubelet 也无法对他们进行健康检查。静态 pod 始终绑定在某一个 kubelet 上，并且始终运行在同一个节点上。kubelet 会自动为每一个静态 pod 在 Kubernetes 的 apiserver 上创建一个镜像 Pod，因此我们可以在 apiserver 中查询到该 pod，但是不能通过 apiserver 进行控制（例如不能删除）。

> 创建静态 Pod 有两种方式：`配置文件`和 `HTTP` 两种方式

#### 配置文件 

> 配置文件就是放在特定目录下的标准的 JSON 或 YAML 格式的 pod 定义文件。用 

```
kubelet --pod-manifest-path=<the directory>
```

来启动 kubelet 进程，kubelet 定期的去扫描这个目录，根据这个目录下出现或消失的 YAML/JSON 文件来创建或删除静态 pod。

> 比如我们在 node01 这个节点上用静态 pod 的方式来启动一个 nginx 的服务。我们登录到node01节点上面，可以通过下面命令找到 kubelet 对应的启动配置文件

```
$ systemctl status kubelet
```

> 配置文件路径为：

```
$ vi /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
......
Environment="KUBELET_SYSTEM_PODS_ARGS=--pod-manifest-path=/etc/kubernetes/manifests --allow-privileged=true"
```

> 打开这个文件我们可以看到其中有一个参数`--pod-manifest-path`的配置， 所以如果我们通过kubeadm 的方式来安装的集群环境，对应的 kubelet 已经配置了我们的静态 Pod 文件的路径，默认地址为 

```
/etc/kubernetes/manifests
```

，所以我们只需要在该目录下面创建一个标准的 Pod 的 JSON 或者 YAML 文件即可，如果你的 kubelet 启动参数中没有配置上面的`--pod-manifest-path` 参数的话，那么添加上这个参数然后重启 kubelet 即可：

```
[root@ node01 ~] $ cat <<EOF >/etc/kubernetes/manifest/static-web.yaml
apiVersion: v1
kind: Pod
metadata:
  name: static-web
  labels:
    app: static
spec:
  containers:
    - name: web
      image: nginx
      ports:
        - name: web
          containerPort: 80
EOF
```

#### 通过 HTTP 创建静态 Pods 

> kubelet 周期地从–manifest-url=参数指定的地址下载文件，并且把它翻译成 JSON/YAML 格式的 pod 定义。此后的操作方式与–pod-manifest-path=相同，kubelet 会不时地重新下载该文件，当文件变化时对应地终止或启动静态 pod。

> kubelet 启动时，由`--pod-manifest-path=` 或 `--manifest-url=`参数指定的目录下定义的所有 pod 都会自动创建，例如，我们示例中的 static-web。（可能要花些时间拉取nginx 镜像，耐心等待…）

```
$ docker ps
CONTAINER ID IMAGE         COMMAND  CREATED        STATUS         PORTS     NAMES
f6d05272b57e nginx:latest  "nginx"  8 minutes ago  Up 8 minutes             k8s_web.6f802af4_static-web-fk-node1_default_67e24ed9466ba55986d120c867395f3c_378e5f3c
```

> 现在我们通过kubectl工具可以看到这里创建了一个新的镜像 Pod：

```
$ kubectl get pods
NAME                       READY     STATUS    RESTARTS   AGE
static-web-my-node01        1/1       Running   0          2m
```

> 静态 pod 的标签会传递给镜像 Pod，可以用来过滤或筛选。 需要注意的是，我们不能通过 API 服务器来删除静态 pod（例如，通过kubectl命令），kubelet 不会删除它。

```
$ kubectl delete pod static-web-my-node01
$ kubectl get pods
NAME                       READY     STATUS    RESTARTS   AGE
static-web-my-node01        1/1       Running   0          12s
```

> 我们尝试手动终止容器，可以看到kubelet很快就会自动重启容器。

```
$ docker ps
CONTAINER ID        IMAGE         COMMAND                CREATED       ...
5b920cbaf8b1        nginx:latest  "nginx -g 'daemon of   2 seconds ago ...
```

#### 静态 Pod 的动态增加和删除 

> 运行中的 kubelet 周期扫描配置的目录（我们这个例子中就是 /etc/kubernetes/manifests）下文件的变化，当这个目录中有文件出现或消失时创建或删除 pods：

```
$ mv /etc/kubernetes/manifests/static-web.yaml /tmp
$ sleep 20
$ docker ps
// no nginx container is running
$ mv /tmp/static-web.yaml  /etc/kubernetes/manifests
$ sleep 20
$ docker ps
CONTAINER ID        IMAGE         COMMAND                CREATED           ...
e7a62e3427f1        nginx:latest  "nginx -g 'daemon of   27 seconds ago
```

> 其实我们用 kubeadm 安装的集群，master 节点上面的几个重要组件都是用静态 Pod 的方式运行的，我们登录到 master 节点上查看

```
/etc/kubernetes/manifests
```

目录：

```
$ ls /etc/kubernetes/manifests/
etcd.yaml  kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml
```

> 现在明白了吧，这种方式也为我们将集群的一些组件容器化提供了可能，因为这些 Pod 都不会受到 apiserver 的控制，不然我们这里kube-apiserver怎么自己去控制自己呢？万一不小心把这个 Pod 删掉了呢？所以只能有kubelet自己来进行控制，这就是我们所说的静态 Pod。

### Downward API 

> 前面我们从 Pod 的原理到生命周期介绍了 Pod 的一些使用，作为 Kubernetes 中最核心的资源对象、最基本的调度单元，我们可以发现 Pod 中的属性还是非常繁多的，前面我们使用过一个 `volumes` 的属性，表示声明一个数据卷，我们可以通过命令

```
kubectl explain pod.spec.volumes
```

去查看该对象下面的属性非常多，前面我们只是简单使用了 `hostPath` 和 `emptyDir{}` 这两种模式，其中还有一种模式叫做`downwardAPI`，这个模式和其他模式不一样的地方在于它不是为了存放容器的数据也不是用来进行容器和宿主机的数据交换的，而是让 Pod 里的容器能够直接获取到这个 Pod 对象本身的一些信息。

> 目前 `Downward API` 提供了两种方式用于将 Pod 的信息注入到容器内部：

*   环境变量：用于单个变量，可以将 Pod 信息和容器信息直接注入容器内部
*   Volume 挂载：将 Pod 信息生成为文件，直接挂载到容器内部中去

### 环境变量 

> 我们通过 `Downward API` 来将 Pod 的 IP、名称以及所对应的 namespace 注入到容器的环境变量中去，然后在容器中打印全部的环境变量来进行验证，对应资源清单文件如下：(env-pod.yaml)

```
apiVersion: v1
kind: Pod
metadata:
  name: env-pod
  namespace: kube-system
spec:
  containers:
  - name: env-pod
    image: busybox
    command: ["/bin/sh", "-c", "env"]
    env:
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: POD_NAMESPACE
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace
    - name: POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
```

> 我们可以看到上面我们使用了一种新的方式来设置 env 的值：`valueFrom`，由于 Pod 的 name 和 namespace 属于元数据，是在 Pod 创建之前就已经定下来了的，所以我们可以使用 metata 就可以获取到了，但是对于 Pod 的 IP 则不一样，因为我们知道 Pod IP 是不固定的，Pod 重建了就变了，它属于状态数据，所以我们使用 status 这个属性去获取。另外除了使用 `fieldRef`获取 Pod 的基本信息外，还可以通过 `resourceFieldRef` 去获取容器的资源请求和资源限制信息。

> 我们直接创建上面的 Pod：

```
$ kubectl create -f env-pod.yaml
pod "env-pod" created
```

> Pod 创建成功后，我们可以查看日志：

```
$ kubectl logs env-pod -n kube-system |grep POD
kubectl logs -f env-pod -n kube-system
POD_IP=2121
KUBERNETES_SERVICE_PORT=443
KUBERNETES_PORT=tcp://1:443
KUBE_DNS_SERVICE_PORT_DNS_TCP=53
HOSTNAME=env-pod
SHLVL=1
HOME=/root
KUBE_DNS_SERVICE_HOST=10
KUBE_DNS_PORT_9153_TCP_ADDR=10
KUBE_DNS_PORT_9153_TCP_PORT=9153
KUBE_DNS_PORT_9153_TCP_PROTO=tcp
KUBE_DNS_SERVICE_PORT=53
KUBE_DNS_PORT=udp://10:53
POD_NAME=env-pod
KUBERNETES_PORT_443_TCP_ADDR=1
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
KUBE_DNS_PORT_53_TCP_ADDR=10
KUBERNETES_PORT_443_TCP_PORT=443
KUBE_DNS_SERVICE_PORT_METRICS=9153
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBE_DNS_PORT_9153_TCP=tcp://10:9153
KUBE_DNS_PORT_53_UDP_ADDR=10
KUBE_DNS_PORT_53_TCP_PORT=53
KUBE_DNS_PORT_53_TCP_PROTO=tcp
KUBE_DNS_PORT_53_UDP_PORT=53
KUBE_DNS_SERVICE_PORT_DNS=53
KUBE_DNS_PORT_53_UDP_PROTO=udp
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT_443_TCP=tcp://1:443
POD_NAMESPACE=kube-system
KUBERNETES_SERVICE_HOST=1
PWD=/
KUBE_DNS_PORT_53_TCP=tcp://10:53
KUBE_DNS_PORT_53_UDP=udp://10:53
```

> 我们可以看到 Pod 的 IP、NAME、NAMESPACE 都通过环境变量打印出来了。

> 环境变量

> 上面打印 Pod 的环境变量可以看到有很多内置的变量，其中大部分是系统自动为我们添加的，Kubernetes 会把当前命名空间下面的 Service 信息通过环境变量的形式注入到 Pod 中去：

```
$ kubectl get svc -n kube-system
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10   <none>        53/UDP,53/TCP,9153/TCP   4d21h
```

### Volume 挂载 

> `Downward API`除了提供环境变量的方式外，还提供了通过 Volume 挂载的方式去获取 Pod 的基本信息。接下来我们通过`Downward API`将 Pod 的 Label、Annotation 等信息通过 Volume 挂载到容器的某个文件中去，然后在容器中打印出该文件的值来验证，对应的资源清单文件如下所示：(volume-pod.yaml)

```
apiVersion: v1
kind: Pod
metadata:
  name: volume-pod
  namespace: kube-system
  labels:
    k8s-app: test-volume
    node-env: test
  annotations:
    own: youdianzhishi
    build: test
spec:
  volumes:
  - name: podinfo
    downwardAPI:
      items:
      - path: labels
        fieldRef:
          fieldPath: metadata.labels
      - path: annotations
        fieldRef:
          fieldPath: metadata.annotations
  containers:
  - name: volume-pod
    image: busybox
    args: 
    - sleep 
    - "3600"
    volumeMounts:
    - name: podinfo
      mountPath: /etc/podinfo
```

> 我们将元数据 labels 和 annotaions 以文件的形式挂载到了 `/etc/podinfo` 目录下，创建上面的 Pod：

```
$ kubectl create -f volume-pod.yaml
pod "volume-pod" created
```

> 创建成功后，我们可以进入到容器中查看元信息是不是已经存入到文件中了：

```
$ kubectl exec -it volume-pod /bin/sh -n kube-system
/ # ls /etc/podinfo/
..2019_11_13_09_57_990445016/  annotations
..data/                           labels
/ # cat /etc/podinfo/labels
k8s-app="test-volume"
/ # cat /etc/podinfo/annotations
build="test"
kubectl.kubernetes.io/last-applied-configuration="{\\"apiVersion\\":\\"v1\\",\\"kind\\":\\"Pod\\",\\"metadata\\":{\\"annotations\\":{\\"build\\":\\"test\\",\\"own\\":\\"youdianzhishi\\"},\\"labels\\":{\\"k8s-app\\":\\"test-volume\\",\\"node-env\\":\\"test\\"},\\"name\\":\\"volume-pod\\",\\"namespace\\":\\"kube-system\\"},\\"spec\\":{\\"containers\\":[{\\"args\\":[\\"sleep\\",\\"3600\\"],\\"image\\":\\"busybox\\",\\"name\\":\\"volume-pod\\",\\"volumeMounts\\":[{\\"mountPath\\":\\"/etc/podinfo\\",\\"name\\":\\"podinfo\\"}]}],\\"volumes\\":[{\\"downwardAPI\\":{\\"items\\":[{\\"fieldRef\\":{\\"fieldPath\\":\\"metadata.labels\\"},\\"path\\":\\"labels\\"},{\\"fieldRef\\":{\\"fieldPath\\":\\"metadata.annotations\\"},\\"path\\":\\"annotations\\"}]},\\"name\\":\\"podinfo\\"}]}}\\n"
kubernetes.io/config.seen="2019-11-13T17:57:320894744+08:00"
kubernetes.io/config.source="api"
```

> 我们可以看到 Pod 的 Labels 和 Annotations 信息都被挂载到 `/etc/podinfo` 目录下面的 lables 和 annotations 文件了。

> 目前，`Downward API` 支持的字段已经非常丰富了，比如：

```
使用 fieldRef 可以声明使用:

spec.nodeName - 宿主机名字
status.hostIP - 宿主机IP
metadata.name - Pod的名字
metadata.namespace - Pod的Namespace
status.podIP - Pod的IP
spec.serviceAccountName - Pod的Service Account的名字
metadata.uid - Pod的UID
metadata.labels['<KEY>'] - 指定<KEY>的Label值
metadata.annotations['<KEY>'] - 指定<KEY>的Annotation值
metadata.labels - Pod的所有Label
metadata.annotations - Pod的所有Annotation

 使用 resourceFieldRef 可以声明使用:

容器的 CPU limit
容器的 CPU request
容器的 memory limit
容器的 memory request
```

> 注意

> 需要注意的是，`Downward API` 能够获取到的信息，一定是 Pod 里的容器进程启动之前就能够确定下来的信息。而如果你想要获取 Pod 容器运行后才会出现的信息，比如，容器进程的 PID，那就肯定不能使用 `Downward API` 了，而应该考虑在 Pod 里定义一个 sidecar 容器来获取了。

> 在实际应用中，如果你的应用有获取 Pod 的基本信息的需求，一般我们就可以利用`Downward API`来获取基本信息，然后编写一个启动脚本或者利用`initContainer`将 Pod 的信息注入到我们容器中去，然后在我们自己的应用中就可以正常的处理相关逻辑了。

> 除了通过 Downward API 可以获取到 Pod 本身的信息之外，其实我们还可以通过映射其他资源对象来获取对应的信息，比如 Secret、ConfigMap 资源对象，同样我们可以通过环境变量和挂载 Volume 的方式来获取他们的信息，但是，通过环境变量获取这些信息的方式，不具备自动更新的能力。所以，一般情况下，都建议使用 Volume 文件的方式获取这些信息，因为通过 Volume 的方式挂载的文件在 Pod 中会进行热更新。

### PodPreset 

> 我们已经学习了很多 Pod 的知识点，但是可能有部分同学还是觉得 Pod 的字段属性太多了，很多记不住，查询文档效率不高，你是不是希望 Kubernetes 能够提供一个功能为 Pod 自动填充一些字段呢？这个需求还是很实际的，比如我们按照命名空间来划分不同的环境，然后我们在不同的环境上部署 Pod 后自动为我们加上环境相关的 Labels、Annotations 等等信息，这样就大大提高了编写 YAML 的效率，为此，在 Kubernetes v1.11 版本后就提供了一个叫做 `PodPreset（Pod 预设值）`的功能可以来解决这个问题。

> Kubernetes 提供了一个 `PodPreset` 准入控制器，当启用后，PodPreset 会将应用创建请求传入到该控制器上。当有 Pod 创建请求发生时，系统将执行以下操作：

*   检索所有可用的 PodPreset
*   检查有 PodPreset 的标签选择器上的标签与正在创建的 Pod 上的标签是否匹配
*   尝试将由 PodPreset 定义的各种资源合并到正在创建的 Pod 中
*   出现错误时，在该 Pod 上引发记录合并错误的事件，PodPreset 不会注入任何资源到创建的 Pod 中
*   注释刚生成的修改过的 Pod spec，以表明它已被 PodPreset 修改过。注释的格式为 

    ```
    podpreset.admission.kubernetes.io/podpreset-<pod-preset name>": "<resource version>"
    ```

> 每个 Pod 可以匹配零个或多个 PodPrestet；并且每个 PodPreset 可以应用于零个或多个 Pod。 PodPreset 应用于一个或多个 Pod 时，Kubernetes 会修改 Pod Spec。对于 Env、EnvFrom 和 VolumeMounts 的更改，Kubernetes 修改 Pod 中所有容器的容器 spec；对于 Volume 的更改，Kubernetes 修改 Pod Spec。

> 可能在某些情况下，您希望你的 Pod 不会被任何 Pod Preset 所改变。在这些情况下，您可以在 Pod 的 Pod Spec 中添加上这样一个 annotation：

```
podpreset.admission.kubernetes.io/exclude："true"
```

。

### 启用 PodPreset 

> 要启用`PodPreset`功能，需要确保你使用的是 kubernetes 1.8版本以上，然后需要在准入控制中加入`PodPreset`，另外为了定义 PodPreset 对象，还需要其中 PodPreset 的 API 版本，在 APIServer 启动参数中添加如下配置：

```
- --enable-admission-plugins=NodeRestriction,PodPreset
- --runtime-config=settings.k8s.io/v1alpha1=true
```

> 我们这里是使用的 kubeadm 搭建的集群，所以只需要修改静态 Pod 配置即可，路径 

```
/etc/kubernetes/manifests/kube-apiserver.yaml
```

，修改完成后，将 `kube-apiserver.yaml` 文件移除 manifests 目录，然后移动回来，相当于强制重启，执行如下命令：

```
$ kubectl api-versions |grep settings
settings.k8s.io/v1alpha1
$ kubectl get podpreset
No resources found in default namespace.
```

> 出现如上的提示证明已经开启成功。

### 示例 

> 我们经常有一个需求就是需要同步 Pod 和宿主机的时间，一般情况下，我们是通过挂载宿主机的 localtime 来完成的，如下 Pod：（time-demo.yaml）

```
apiVersion: apps/v1
kind: Pod
metadata:
  name: time-demo
  labels:
    app: time
spec:
  containers:
  - name: time-demo
    image: nginx
    ports:
    - containerPort: 80
```

> 我们直接创建该 Pod：

```
$ kubectl apply -f time-demo.yaml
$ kubectl get pods
NAME         READY   STATUS    RESTARTS   AGE
time-demo    1/1     Running   0          2m8s
```

> 创建完成后，我们可以对比下节点的时间和 Pod 的时间：

```
$ date
Thu Nov 14 12:08:27 CST 2019
$ kubectl exec time-demo date
Thu Nov 14 04:08:41 UTC 2019
```

> 我们可以看到 Pod 的时间是 UTC 时间，和节点不同步，这个时候我们可以挂载宿主机的 localtime 文件到 Pod 中去，如下所示：

```
apiVersion: apps/v1
kind: Pod
metadata:
  name: time-demo
  labels:
    app: time
spec:
  volumes:
  - name: host-time
    hostPath:
      path: /etc/localtime
  containers:
  - name: time-demo
    image: nginx
    volumeMounts:
    - name: host-time
      mountPath: /etc/localtime
    ports:
    - containerPort: 80
```

> 这个时候我们重新来测试下：

```
$ kubectl delete -f time-demo.yaml
$ kubectl apply -f time-demo.yaml
$ kubectl get pods
NAME         READY   STATUS    RESTARTS   AGE
time-demo    1/1     Running   0          2m8s
```

> 这个时候我们再来看下时间是否同步了：

```
$ date
Thu Nov 14 12:13:40 CST 2019
$ kubectl exec time-demo date
Thu Nov 14 12:13:45 CST 2019
```

> 可以看到已经生效了。但是往往我们所有的 Pod 都有时间同步的需求，如果所有的 Pod 都来手动进行挂载，是不是显得繁琐多余啊，这个时候我们就可以利用 PodPreset 来预设模板，定义一个如下所示的 PodPreset 资源对象：（time-preset.yaml）

```
apiVersion: settings.k8s.io/v1alpha1
kind: PodPreset
metadata:
  name: time-preset
  namespace: default
spec:
  selector:
    matchLabels:
  volumeMounts:
  - name: localtime
    mountPath: /etc/localtime
  volumes:
  - name: localtime
    hostPath:
      path: /etc/localtime
```

> 直接创建上面的资源对象：

```
$ kubectl apply -f time-preset.yaml
podpreset.settings.k8s.io/time-preset created
$ kubectl get podpreset
NAME          CREATED AT
time-preset   2019-11-14T06:52:11Z
```

> 这个时候我们再来直接创建上面的 Pod：(time-demo.yaml)

```
apiVersion: apps/v1
kind: Pod
metadata:
  name: time-demo
  labels:
    app: time
spec:
  containers:
  - name: time-demo
    image: nginx
    ports:
    - containerPort: 80
```

> 这个 Pod 我们没有主动挂载时间文件，直接创建：

```
$ kubectl delete -f time-demo.yaml
$ kubectl apply -f time-demo.yaml
$  kubectl get pods -l app=time
NAME        READY   STATUS    RESTARTS   AGE
time-demo   1/1     Running   0          60s
```

> 这个时候验证下 Pod 时间是否和宿主机一致：

```
$ date
Thu Nov 14 14:55:02 CST 2019
$ kubectl exec time-demo date
Thu Nov 14 14:55:21 CST 2019
```

> 可以看到已经同步了~，可以直接导出 Pod 的资源清单查看：

```
$ kubectl get pod time-demo -o yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Pod","metadata":{"annotations":{},"labels":{"app":"time"},"name":"time-demo","namespace":"default"},"spec":{"containers":[{"image":"nginx","name":"time-demo","ports":[{"containerPort":80}]}]}}
    podpreset.admission.kubernetes.io/podpreset-time-preset: "1469811"
  creationTimestamp: "2019-11-14T06:54:06Z"
  labels:
    app: time
  name: time-demo
  namespace: default
  resourceVersion: "1470398"
  selfLink: /api/v1/namespaces/default/pods/time-demo
  uid: 37a7ab94-c2c3-4b10-88df-0c486fb8d422
spec:
  containers:
  - image: nginx
    imagePullPolicy: Always
    name: time-demo
    ports:
    - containerPort: 80
      protocol: TCP
    resources: {}
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /etc/localtime
      name: localtime
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: default-token-5tsh4
      readOnly: true
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  nodeName: ydzs-node3
  priority: 0
  restartPolicy: Always
  schedulerName: default-scheduler
  securityContext: {}
  serviceAccount: default
  serviceAccountName: default
  terminationGracePeriodSeconds: 30
  tolerations:
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
    tolerationSeconds: 300
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
    tolerationSeconds: 300
  volumes:
  - hostPath:
      path: /etc/localtime
      type: ""
    name: localtime
  - name: default-token-5tsh4
    secret:
      defaultMode: 420
      secretName: default-token-5tsh4
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: "2019-11-14T06:54:06Z"
    status: "True"
    type: Initialized
  - lastProbeTime: null
    lastTransitionTime: "2019-11-14T06:54:22Z"
    status: "True"
    type: Ready
  - lastProbeTime: null
    lastTransitionTime: "2019-11-14T06:54:22Z"
    status: "True"
    type: ContainersReady
  - lastProbeTime: null
    lastTransitionTime: "2019-11-14T06:54:06Z"
    status: "True"
    type: PodScheduled
  containerStatuses:
  - containerID: docker://a774e82d4ebfde98c37b675f1a46377ccf01c0bba931507172309616dbf49165
    image: nginx:latest
    imageID: docker-pullable://nginx@sha256:922c815aa4df050d4df476e92daed4231f466acc8ee90e0e774951b0fd7195a4
    lastState: {}
    name: time-demo
    ready: true
    restartCount: 0
    started: true
    state:
      running:
        startedAt: "2019-11-14T06:54:21Z"
  hostIP: 157
  phase: Running
  podIP: 264
  podIPs:
  - ip: 264
  qosClass: BestEffort
  startTime: "2019-11-14T06:54:06Z"
```

> 从上面 YAML 中也可以看出 Pod 被自动注入了 PodPreset 声明的模板，而且该命名空间下面的所有 Pod 默认都会被注入该模板。

> 注意

> 需要注意的是 PodPreset 资源对象是 namespace scope 的，就是不同的命名空间需要创建不同的 PodPreset。