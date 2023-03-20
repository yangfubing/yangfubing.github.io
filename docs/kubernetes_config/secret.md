
## Secret 

> 敏感信息配置管理

> 前文我们学习 `ConfigMap` 的时候，我们说 `ConfigMap` 这个资源对象是 Kubernetes 当中非常重要的一个资源对象，一般情况下 ConfigMap 是用来存储一些非安全的配置信息，如果涉及到一些安全相关的数据的话用 ConfigMap 就非常不妥了，因为ConfigMa 是明文存储的，这个时候我们就需要用到另外一个资源对象了：`Secret`，`Secret`用来保存敏感信息，例如密码、OAuth 令牌和 ssh key 等等，将这些信息放在 `Secret` 中比放在 Pod 的定义中或者 Docker 镜像中要更加安全和灵活。

> `Secret` 主要使用的有以下三种类型：

*   `Opaque`：base64 编码格式的 Secret，用来存储密码、密钥等；但数据也可以通过`base64 –decode`解码得到原始数据，所有加密性很弱。
*   ```
    kubernetes.io/dockerconfigjson
    ```

    ：用来存储私有`docker registry`的认证信息。
*   ```
    kubernetes.io/service-account-token
    ```

    ：用于被 `ServiceAccount` ServiceAccount 创建时 Kubernetes 会默认创建一个对应的 Secret 对象。Pod 如果使用了 ServiceAccount，对应的 Secret 会自动挂载到 Pod 目录 

    ```
    /run/secrets/kubernetes.io/serviceaccount
    ```

     中。
*   ```
    bootstrap.kubernetes.io/token
    ```

    ：用于节点接入集群的校验的 Secret

### Opaque Secret 

> `Opaque` 类型的数据是一个 map 类型，要求 value 必须是 `base64` 编码格式，比如我们来创建一个用户名为 admin，密码为 admin321 的 `Secret` 对象，首先我们需要先把用户名和密码做 `base64` 编码：

```
$ echo -n "admin" | base64
YWRtaW4=
$ echo -n "admin321" | base64
YWRtaW4zMjE=
```

> 然后我们就可以利用上面编码过后的数据来编写一个 YAML 文件：(secret-demo.yaml)

```
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: YWRtaW4=
  password: YWRtaW4zMjE=
```

> 然后我们就可以使用 kubectl 命令来创建了：

```
$ kubectl create -f secret-demo.yaml
secret "mysecret" created
```

> 利用`get secret`命令查看：

```
$ kubectl get secret
NAME                  TYPE                                  DATA      AGE
default-token-n9w2d   kubernetes.io/service-account-token   3         33d
mysecret              Opaque                                2         40s
```

> 其中 default-token-n9w2d 为创建集群时默认创建的 Secret，被 

```
serviceacount/default
```

 引用。我们可以使用 `describe` 命令查看详情：

```
$ kubectl describe secret mysecret
Name:         mysecret
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
password:  8 bytes
username:  5 bytes
```

> 我们可以看到利用 describe 命令查看到的 Data 没有直接显示出来，如果想看到 Data 里面的详细信息，同样我们可以输出成YAML 文件进行查看：

```
$ kubectl get secret mysecret -o yaml
apiVersion: v1
data:
  password: YWRtaW4zMjE=
  username: YWRtaW4=
kind: Secret
metadata:
  creationTimestamp: 2018-06-19T15:27:06Z
  name: mysecret
  namespace: default
  resourceVersion: "3694084"
  selfLink: /api/v1/namespaces/default/secrets/mysecret
  uid: 39c139f5-73d5-11e8-a101-525400db4df7
type: Opaque
```

> 创建好 `Secret`对象后，有两种方式来使用它：

*   以环境变量的形式
*   以Volume的形式挂载

### 环境变量 

> 首先我们来测试下环境变量的方式，同样的，我们来使用一个简单的 busybox 镜像来测试下:(secret1-pod.yaml)

```
apiVersion: v1
kind: Pod
metadata:
  name: secret1-pod
spec:
  containers:
  - name: secret1
    image: busybox
    command: [ "/bin/sh", "-c", "env" ]
    env:
    - name: USERNAME
      valueFrom:
        secretKeyRef:
          name: mysecret
          key: username
    - name: PASSWORD
      valueFrom:
        secretKeyRef:
          name: mysecret
          key: password
```

> 主要需要注意的是上面环境变量中定义的 `secretKeyRef` 字段，和我们前文的 `configMapKeyRef` 类似，一个是从 `Secret` 对象中获取，一个是从 `ConfigMap` 对象中获取，创建上面的 Pod：

```
$ kubectl create -f secret1-pod.yaml
pod "secret1-pod" created
```

> 然后我们查看Pod的日志输出：

```
$ kubectl logs secret1-pod
...
USERNAME=admin
PASSWORD=admin321
...
```

> 可以看到有 USERNAME 和 PASSWORD 两个环境变量输出出来。

### Volume 挂载 

> 同样的我们用一个 Pod 来验证下 `Volume` 挂载，创建一个 Pod 文件：(secret2-pod.yaml)

```
apiVersion: v1
kind: Pod
metadata:
  name: secret2-pod
spec:
  containers:
  - name: secret2
    image: busybox
    command: ["/bin/sh", "-c", "ls /etc/secrets"]
    volumeMounts:
    - name: secrets
      mountPath: /etc/secrets
  volumes:
  - name: secrets
    secret:
     secretName: mysecret
```

> 创建 Pod，然后查看输出日志：

```
$ kubectl create -f secret-podyaml
pod "secret2-pod" created
$ kubectl logs secret2-pod
password
username
```

> 可以看到 Secret 把两个 key 挂载成了两个对应的文件。当然如果想要挂载到指定的文件上面，是不是也可以使用上一节课的方法：在 `secretName` 下面添加 `items` 指定 `key` 和 `path`，这个大家可以参考上节课 `ConfigMap` 中的方法去测试下。

### kubernetes.io/dockerconfigjson 

> 除了上面的 `Opaque` 这种类型外，我们还可以来创建用户 `docker registry` 认证的 `Secret`，直接使用``kubectl create` 命令创建即可，如下：

```
$ kubectl create secret docker-registry myregistry --docker-server=DOCKER_SERVER --docker-username=DOCKER_USER --docker-password=DOCKER_PASSWORD --docker-email=DOCKER_EMAIL
secret "myregistry" created
```

> 除了上面这种方法之外，我们也可以通过指定文件的方式来创建镜像仓库认证信息，需要注意对应的 `KEY` 和 `TYPE`：

```
$ kubectl create secret generic myregistry --from-file=.dockerconfigjson=/root/.docker/config.json --type=kubernetes.io/dockerconfigjson
```

> 然后查看 Secret 列表：

```
$ kubectl get secret
NAME                  TYPE                                  DATA      AGE
default-token-n9w2d   kubernetes.io/service-account-token   3         33d
myregistry            kubernetes.io/dockerconfigjson        1         15s
mysecret              Opaque                                2         34m
```

> 注意看上面的 TYPE 类型，myregistry 对应的是 

```
kubernetes.io/dockerconfigjson
```

，同样的可以使用 describe 命令来查看详细信息：

```
$ kubectl describe secret myregistry
Name:         myregistry
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  kubernetes.io/dockerconfigjson

Data
====
.dockerconfigjson:  152 bytes
```

> 同样的可以看到 Data 区域没有直接展示出来，如果想查看的话可以使用 `-o yaml` 来输出展示出来：

```
$ kubectl get secret myregistry -o yaml
apiVersion: v1
data:
  .dockerconfigjson: eyJhdXRocyI6eyJET0NLRVJfU0VSVkVSIjp7InVzZXJuYW1lIjoiRE9DS0VSX1VTRVIiLCJwYXNzd29yZCI6IkRPQ0tFUl9QQVNTV09SRCIsImVtYWlsIjoiRE9DS0VSX0VNQUlMIiwiYXV0aCI6IlJFOURTMFZTWDFWVFJWSTZSRTlEUzBWU1gxQkJVMU5YVDFKRSJ9fX0=
kind: Secret
metadata:
  creationTimestamp: 2018-06-19T16:01:05Z
  name: myregistry
  namespace: default
  resourceVersion: "3696966"
  selfLink: /api/v1/namespaces/default/secrets/myregistry
  uid: f91db707-73d9-11e8-a101-525400db4df7
type: kubernetes.io/dockerconfigjson
```

> 可以把上面的 

```
data.dockerconfigjson
```

 下面的数据做一个 `base64` 解码，看看里面的数据是怎样的呢？

```
$ echo eyJhdXRocyI6eyJET0NLRVJfU0VSVkVSIjp7InVzZXJuYW1lIjoiRE9DS0VSX1VTRVIiLCJwYXNzd29yZCI6IkRPQ0tFUl9QQVNTV09SRCIsImVtYWlsIjoiRE9DS0VSX0VNQUlMIiwiYXV0aCI6IlJFOURTMFZTWDFWVFJWSTZSRTlEUzBWU1gxQkJVMU5YVDFKRSJ9fX0= | base64 -d
{"auths":{"DOCKER_SERVER":{"username":"DOCKER_USER","password":"DOCKER_PASSWORD","email":"DOCKER_EMAIL","auth":"RE9DS0VSX1VTRVI6RE9DS0VSX1BBU1NXT1JE"}}}
```

> 如果我们需要拉取私有仓库中的 Docker 镜像的话就需要使用到上面的 myregistry 这个 `Secret`：

```
apiVersion: v1
kind: Pod
metadata:
  name: foo
spec:
  containers:
  - name: foo
    image: 11100:5000/test:v1
  imagePullSecrets:
  - name: myregistry
```

> imagePullSecrets

> `ImagePullSecrets` 与 `Secrets` 不同，因为 `Secrets` 可以挂载到 Pod 中，但是 `ImagePullSecrets` 只能由 Kubelet 访问。

> 我们需要拉取私有仓库镜像 

```
11100:5000/test:v1
```

，我们就需要针对该私有仓库来创建一个如上的 `Secret`，然后在 Pod 中指定 `imagePullSecrets`。

> 除了设置 

```
Pod.spec.imagePullSecrets
```

 这种方式来获取私有镜像之外，我们还可以通过在 `ServiceAccount` 中设置 `imagePullSecrets`，然后就会自动为使用该 SA 的 Pod 注入 `imagePullSecrets` 信息：

```
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: "2019-11-08T12:00:04Z"
  name: default
  namespace: default
  resourceVersion: "332"
  selfLink: /api/v1/namespaces/default/serviceaccounts/default
  uid: cc37a719-c4fe-4ebf-92da-e92c3e24d5d0
secrets:
- name: default-token-5tsh4
imagePullSecrets:
- name: myregistry
```

### kubernetes.io/service-account-token 

> 另外一种 `Secret` 类型就是 

```
kubernetes.io/service-account-token
```

，用于被 `ServiceAccount` 引用。`ServiceAccout` 创建时 Kubernetes 会默认创建对应的 `Secret`。Pod 如果使用了 `ServiceAccount`，对应的 `Secret` 会自动挂载到 Pod 的 

```
/var/run/secrets/kubernetes.io/serviceaccount/
```

 目录中。如下所示我们随意创建一个 Pod：

```
$ kubectl run secret-pod3 --image nginx:9
deployment.apps "secret-pod3" created
$ kubectl get pods
NAME                           READY     STATUS    RESTARTS   AGE
...
secret-pod3-78c8c76db8-7zmqm   1/1       Running   0          13s
...
```

> 我们可以直接查看这个 Pod 的详细信息：

```
volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: default-token-5tsh4
      readOnly: true
  ......
  serviceAccount: default
  serviceAccountName: default
  volumes:
  - name: default-token-5tsh4
    secret:
      defaultMode: 420
      secretName: default-token-5tsh4
```

> 可以看到默认把名为 `default`（自动创建的）的 `ServiceAccount` 对应的 Secret 对象通过 Volume 挂载到了容器的 

```
/var/run/secrets/kubernetes.io/serviceaccount
```

 的目录中，对于该目录的作用我们会在后面的安全章节中继续和大家学习。

### Secret vs ConfigMap 

> 最后我们来对比下 `Secret` 和 `ConfigMap`这两种资源对象的异同点：

### 相同点 

*   key/value的形式
*   属于某个特定的命名空间
*   可以导出到环境变量
*   可以通过目录/文件形式挂载
*   通过 volume 挂载的配置信息均可热更新

### 不同点 

*   Secret 可以被 ServerAccount 关联
*   Secret 可以存储 `docker register` 的鉴权信息，用在 `ImagePullSecret` 参数中，用于拉取私有仓库的镜像
*   Secret 支持 `Base64` 加密
*   Secret 分为 

    ```
    kubernetes.io/service-account-token
    ```

    、

    ```
    kubernetes.io/dockerconfigjson
    ```

    、`Opaque` 三种类型，而 `Configmap` 不区分类型

> 使用注意

> 同样 Secret 文件大小限制为 `1MB`（ETCD 的要求）；Secret 虽然采用 `Base64` 编码，但是我们还是可以很方便解码获取到原始信息，所以对于非常重要的数据还是需要慎重考虑，可以考虑使用 Vault 来进行加密管理。