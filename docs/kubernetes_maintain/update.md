 K8S训练营

GitHub

*   1000 Stars
*   303 Forks

*   介绍
*   [ ]  Docker 基础

    Docker 基础

    *   简介
    *   安装
    *   基本操作
    *   Dockerfile
    *   Dockerfile最佳实践

*   [ ]  Kubernetes 基础

    Kubernetes 基础

    *   简介
    *   安装
    *   资源清单
    *   Pod 原理
    *   Pod 生命周期
    *   Pod 使用进阶

*   [ ]  Kubernetes 控制器

    Kubernetes 控制器

    *   ReplicaSet
    *   Deployment
    *   StatefulSet
    *   DaemonSet
    *   Job
    *   HPA

*   [ ]  Kubernetes 配置管理

    Kubernetes 配置管理

    *   ConfigMap
    *   Secret
    *   ServiceAccount

*   [ ]  Kubernetes 安全

    Kubernetes 安全

    *   RBAC
    *   Security Context
    *   准入控制器

*   [ ]  Kubernetes 网络

    Kubernetes 网络

    *   网络插件
    *   网络策略
    *   Service 服务
    *   [ ]  Ingress

        Ingress

        *   NGINX Controller
        *   Traefik

*   [ ]  Kubernetes 调度器

    Kubernetes 调度器

    *   调度器介绍
    *   Pod 调度

*   [ ]  Kubernetes 存储

    Kubernetes 存储

    *   Local 本地存储
    *   Ceph 存储
    *   存储原理

*   [ ]  应用示例

    应用示例

    *   应用部署

*   [ ]  Helm 包管理

    Helm 包管理

    *   Helm
    *   Charts
    *   [ ]  模板开发

        模板开发

        *   内置对象
        *   Values
        *   函数与管道
        *   流程控制
        *   变量
        *   命名模板
        *   访问文件
        *   NOTES.txt
        *   子chart与全局值
        *   模板调试

    *   Chart Hooks
    *   示例

*   [ ]  Kubernetes 监控

    Kubernetes 监控

    *   Prometheus
    *   Grafana
    *   PromQL
    *   AlertManager
    *   Thanos
    *   Prometheus Adpater
    *   [ ]  Prometheus Operator

        Prometheus Operator

        *   安装
        *   自定义监控报警
        *   高级配置

*   [ ]  Kubernetes 日志

    Kubernetes 日志

    *   日志收集架构
    *   EFK

*   [ ]  Kubernetes DevOps

    Kubernetes DevOps

    *   Jenkins
    *   GitLab
    *   Harbor
    *   Jenkins 流水线
    *   Tekton

*   [ ]  Service Mesh 实践

    Service Mesh 实践

    *   Istio 简介
    *   流量管理
    *   可观测性
    *   安全管控
    *   Certmanager

*   [ ]  Kubernetes 多租户

    Kubernetes 多租户

    *   多租户介绍
    *   资源配额
    *   Pod 安全策略

*   [ ]  Operator

    Operator

    *   CRD
    *   Operator

*   [x]  实践技巧

    实践技巧

    *   技巧
    *   [ ]  集群升级 集群升级

        目录

        *   更新证书

            *   手动更新证书
            *   用 Kubernetes 证书 API 更新证书(不推荐)

        *   集群升级

    *   升级为集群
    *   资源预留

*   [ ]  源码

    源码

    *   kubelet 源码
    *   [ ]  client-go

        client-go

        *   workqueue
        *   Indexer
        *   DeltaFIFO

目录

*   更新证书

    *   手动更新证书
    *   用 Kubernetes 证书 API 更新证书(不推荐)

*   集群升级


## 集群升级

### 更新证书

> 使用 kubeadm 安装 kubernetes 集群非常方便，但是也有一个比较烦人的问题就是默认的证书有效期只有一年时间，所以需要考虑证书升级的问题，本文的演示集群版本为 v1.26.2 版本，不保证下面的操作对其他版本也适用，`在操作之前一定要先对证书目录进行备份，防止操作错误进行回滚`。本文主要介绍两种方式来更新集群证书。

### 手动更新证书

> 由 kubeadm 生成的客户端证书默认只有一年有效期，我们可以通过 `check-expiration` 命令来检查证书是否过期：

```
$ kubeadm alpha certs check-expiration
CERTIFICATE                EXPIRES                  RESIDUAL TIME   EXTERNALLY MANAGED
admin.conf                 Nov 07, 2020 11:59 UTC   73d             no
apiserver                  Nov 07, 2020 11:59 UTC   73d             no
apiserver-etcd-client      Nov 07, 2020 11:59 UTC   73d             no
apiserver-kubelet-client   Nov 07, 2020 11:59 UTC   73d             no
controller-manager.conf    Nov 07, 2020 11:59 UTC   73d             no
etcd-healthcheck-client    Nov 07, 2020 11:59 UTC   73d             no
etcd-peer                  Nov 07, 2020 11:59 UTC   73d             no
etcd-server                Nov 07, 2020 11:59 UTC   73d             no
front-proxy-client         Nov 07, 2020 11:59 UTC   73d             no
scheduler.conf             Nov 07, 2020 11:59 UTC   73d             no
```

> 该命令显示 `/etc/kubernetes/pki` 文件夹中的客户端证书以及 kubeadm 使用的 `KUBECONFIG` 文件中嵌入的客户端证书的到期时间/剩余时间。

> 注意

> `kubeadm` 不能管理由外部 CA 签名的证书，如果是外部得证书，需要自己手动去管理证书的更新。

> 另外需要说明的是上面的列表中没有包含 `kubelet.conf`，因为 kubeadm 将 kubelet 配置为自动更新证书。

> 另外 kubeadm 会在控制面板升级的时候自动更新所有证书，所以使用 kubeadm 搭建得集群最佳的做法是经常升级集群，这样可以确保你的集群保持最新状态并保持合理的安全性。但是对于实际的生产环境我们可能并不会去频繁得升级集群，所以这个时候我们就需要去手动更新证书。

> 要手动更新证书也非常方便，我们只需要通过 

```
kubeadm alpha certs renew
```

 命令即可更新你的证书，这个命令用 CA（或者 front-proxy-CA ）证书和存储在 `/etc/kubernetes/pki` 中的密钥执行更新。

> 注意

> 如果你运行了一个高可用的集群，这个命令需要在所有控制面板节点上执行。

> 接下来我们来更新我们的集群证书，下面的操作都是在 master 节点上进行，首先备份原有证书：

```
$ mkdir /etc/kubernetes.bak
$ cp -r /etc/kubernetes/pki/ /etc/kubernetes.bak
$ cp /etc/kubernetes/*.conf /etc/kubernetes.bak
```

> 然后备份 etcd 数据目录：

```
$ cp -r /var/lib/etcd /var/lib/etcd.bak
```

> 接下来执行更新证书的命令：

```
$ kubeadm alpha certs renew all --config=kubeadm.yaml
kubeadm alpha certs renew all --config=kubeadm.yaml
certificate embedded in the kubeconfig file for the admin to use and for kubeadm itself renewed
certificate for serving the Kubernetes API renewed
certificate the apiserver uses to access etcd renewed
certificate for the API server to connect to kubelet renewed
certificate embedded in the kubeconfig file for the controller manager to use renewed
certificate for liveness probes to healthcheck etcd renewed
certificate for etcd nodes to communicate with each other renewed
certificate for serving etcd renewed
certificate for the front proxy client renewed
certificate embedded in the kubeconfig file for the scheduler manager to use renewed
```

> 通过上面的命令证书就一键更新完成了，这个时候查看上面的证书可以看到过期时间已经是一年后的时间了：

```
$ kubeadm alpha certs check-expiration
CERTIFICATE                EXPIRES                  RESIDUAL TIME   EXTERNALLY MANAGED
admin.conf                 Aug 26, 2021 03:47 UTC   364d            no
apiserver                  Aug 26, 2021 03:47 UTC   364d            no
apiserver-etcd-client      Aug 26, 2021 03:47 UTC   364d            no
apiserver-kubelet-client   Aug 26, 2021 03:47 UTC   364d            no
controller-manager.conf    Aug 26, 2021 03:47 UTC   364d            no
etcd-healthcheck-client    Aug 26, 2021 03:47 UTC   364d            no
etcd-peer                  Aug 26, 2021 03:47 UTC   364d            no
etcd-server                Aug 26, 2021 03:47 UTC   364d            no
front-proxy-client         Aug 26, 2021 03:47 UTC   364d            no
scheduler.conf             Aug 26, 2021 03:47 UTC   364d            no
```

> 然后记得更新下 kubeconfig 文件：

```
$ kubeadm init phase kubeconfig all --config kubeadm.yaml
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Using existing kubeconfig file: "/etc/kubernetes/admin.conf"
[kubeconfig] Using existing kubeconfig file: "/etc/kubernetes/kubelet.conf"
[kubeconfig] Using existing kubeconfig file: "/etc/kubernetes/controller-manager.conf"
[kubeconfig] Using existing kubeconfig file: "/etc/kubernetes/scheduler.conf"
```

> 将新生成的 admin 配置文件覆盖掉原本的 admin 文件:

```
$ mv $HOME/.kube/config $HOME/.kube/config.old
$ cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ chown $(id -u):$(id -g) $HOME/.kube/config
```

> 完成后重启 kube-apiserver、kube-controller、kube-scheduler、etcd 这4个容器即可，我们可以查看 apiserver 的证书的有效期来验证是否更新成功：

```
$ docker restart `docker ps | grep etcd  | awk '{ print $1 }'`
$ docker restart `docker ps | grep kube-apiserver  | awk '{ print $1 }'`
$ docker restart `docker ps | grep kube-scheduler  | awk '{ print $1 }'`
$ docker restart `docker ps | grep kube-controller  | awk '{ print $1 }'`
systemctl restart kubelet
$ echo | openssl s_client -showcerts -connect 11:6443 -servername api 2>/dev/null | openssl x509 -noout -enddate
notAfter=Aug 26 03:47:23 2021 GMT
```

> 可以看到现在的有效期是一年过后的，证明已经更新成功了。

### 用 Kubernetes 证书 API 更新证书(不推荐)

> 除了上述的一键手动更新证书之外，还可以使用 Kubernetes 证书 API 执行手动证书更新。对于线上环境我们可能并不会去冒险经常更新集群或者去更新证书，这些毕竟是有风险的，所以我们希望生成的证书有效期足够长，虽然从安全性角度来说不推荐这样做，但是对于某些场景下一个足够长的证书有效期也是非常有必要的。有很多管理员就是去手动更改 kubeadm 的源码为10年，然后重新编译来创建集群，这种方式虽然可以达到目的，但是不推荐使用这种方式，特别是当你想要更新集群的时候，还得用新版本进行更新。其实 Kubernetes 提供了一种 API 的方式可以来帮助我们生成一个足够长证书有效期。

> 要使用内置的 API 方式来签名，首先我们需要配置 kube-controller-manager 组件的 

```
--experimental-cluster-signing-duration
```

 参数，将其调整为10年，我们这里是 kubeadm 安装的集群，所以直接修改静态 Pod 的 yaml 文件即可:

```
$ vi /etc/kubernetes/manifests/kube-controller-manager.yaml
......
spec:
  containers:
  - command:
    - kube-controller-manager
    # 设置证书有效期为 10 年
    - --experimental-cluster-signing-duration=87600h 
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
......
```

> 修改完成后 kube-controller-manager 会自动重启生效。然后我们需要使用下面的命令为 Kubernetes 证书 API 创建一个证书签名请求。如果您设置例如 `cert-manager` 等外部签名者，则会自动批准证书签名请求（CSRs）。否者，您必须使用 `kubectl certificate` 命令手动批准证书。以下 kubeadm 命令输出要批准的证书名称，然后等待批准发生：

```
$ kubeadm alpha certs renew all --use-api --config kubeadm.yaml &
```

> 输出类似于以下内容：

```
[1] 2890
[certs] Certificate request "kubeadm-cert-kubernetes-admin-pn99f" created
```

> 然后接下来我们需要去手动批准证书：

```
$ kubectl get csr
NAME                                  AGE   REQUESTOR          CONDITION
kubeadm-cert-kubernetes-admin-pn99f   64s   kubernetes-admin   Pending
# 手动批准证书
$ kubectl certificate approve kubeadm-cert-kubernetes-admin-pn99f
certificatesigningrequest.certificates.k8s.io/kubeadm-cert-kubernetes-admin-pn99f approved
```

> 用同样的方式为处于 Pending 状态的 csr 执行批准操作，直到所有的 csr 都批准完成为止。最后所有的 csr 列表状态如下所示：

```
$ kubectl get csr
NAME                                                AGE     REQUESTOR          CONDITION
kubeadm-cert-front-proxy-client-llhrj               30s     kubernetes-admin   Approved,Issued
kubeadm-cert-kube-apiserver-2s6kf                   2m43s   kubernetes-admin   Approved,Issued
kubeadm-cert-kube-apiserver-etcd-client-t9pkx       2m7s    kubernetes-admin   Approved,Issued
kubeadm-cert-kube-apiserver-kubelet-client-pjbjm    108s    kubernetes-admin   Approved,Issued
kubeadm-cert-kube-etcd-healthcheck-client-8dcn8     64s     kubernetes-admin   Approved,Issued
kubeadm-cert-kubernetes-admin-pn99f                 4m29s   kubernetes-admin   Approved,Issued
kubeadm-cert-system:kube-controller-manager-mr86h   79s     kubernetes-admin   Approved,Issued
kubeadm-cert-system:kube-scheduler-t8lnw            17s     kubernetes-admin   Approved,Issued
kubeadm-cert-ydzs-master-cqh4s                      52s     kubernetes-admin   Approved,Issued
kubeadm-cert-ydzs-master-lvbr5                      41s     kubernetes-admin   Approved,Issued
```

> 批准完成后检查证书的有效期：

```
$ kubeadm alpha certs check-expiration
CERTIFICATE                EXPIRES                  RESIDUAL TIME   EXTERNALLY MANAGED
admin.conf                 Nov 05, 2029 11:53 UTC   9y              no
apiserver                  Nov 05, 2029 11:54 UTC   9y              no
apiserver-etcd-client      Nov 05, 2029 11:53 UTC   9y              no
apiserver-kubelet-client   Nov 05, 2029 11:54 UTC   9y              no
controller-manager.conf    Nov 05, 2029 11:54 UTC   9y              no
etcd-healthcheck-client    Nov 05, 2029 11:53 UTC   9y              no
etcd-peer                  Nov 05, 2029 11:53 UTC   9y              no
etcd-server                Nov 05, 2029 11:54 UTC   9y              no
front-proxy-client         Nov 05, 2029 11:54 UTC   9y              no
scheduler.conf             Nov 05, 2029 11:53 UTC   9y              no
```

> 我们可以看到已经延长小10年了，这是因为 ca 证书的有效期只有10年。

> 但是现在我们还不能直接重启控制面板的几个组件，这是因为使用 kubeadm 安装的集群对应的 etcd 默认是使用的 

```
/etc/kubernetes/pki/etcd/ca.crt
```

 这个证书进行前面的，而上面我们用命令 

```
kubectl certificate approve
```

 批准过后的证书是使用的默认的 

```
/etc/kubernetes/pki/ca.crt
```

 证书进行签发的，所以我们需要替换 etcd 中的 ca 机构证书:

```
# 先拷贝静态 Pod 资源清单
$ cp -r /etc/kubernetes/manifests/ /etc/kubernetes/manifests.bak
$ cp /etc/kubernetes/pki/ca.crt /etc/kubernetes/pki/etcd/ca.crt
$ cp /etc/kubernetes/pki/ca.key /etc/kubernetes/pki/etcd/ca.key
```

> 除此之外还需要替换 

```
requestheader-client-ca-file
```

 文件，默认是 

```
/etc/kubernetes/pki/front-proxy-ca.crt
```

 文件，现在也需要替换成默认的 CA 文件，否则使用聚合 API，比如安装了 metrics-server 后执行 `kubectl top` 命令就会报错：

```
$ cp /etc/kubernetes/pki/ca.crt /etc/kubernetes/pki/front-proxy-ca.crt
$ cp /etc/kubernetes/pki/ca.key /etc/kubernetes/pki/front-proxy-ca.key
```

> 由于是静态 Pod，修改完成后上面的组件都会自动重启生效。由于我们当前版本的 kubelet 默认开启了证书自动轮转，所以 kubelet 的证书也不用再去管理了，这样我就将证书更新成10有效期了。`在操作之前一定要先对证书目录进行备份，防止操作错误进行回滚`。

### 集群升级

> 最新的 Kubernetes 版本是 v1.19.0，我们这里的环境是 v1.26.2，由于我们这里版本跨度太大，不能直接从 1.16.x 更新到 1.19.x，kubeadm 的更新是不支持跨多个主版本的，所以我们可以一个版本一个版本的升级，不过版本更新的方式方法基本上都是一样的，所以后面要更新的话也挺简单了，下面我们就先将集群更新到 v1.26.14 版本。

> 首先查看当前集群版本：

```
$ kubectl version
Client Version: version.Info{Major:"1", Minor:"16", GitVersion:"v2", GitCommit:"c97fe5036ef3df2967d086711e6c0c405941e14b", GitTreeState:"clean", BuildDate:"2019-10-15T19:18:23Z", GoVersion:"go10", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"16", GitVersion:"v2", GitCommit:"c97fe5036ef3df2967d086711e6c0c405941e14b", GitTreeState:"clean", BuildDate:"2019-10-15T19:09:08Z", GoVersion:"go10", Compiler:"gc", Platform:"linux/amd64"}
```

> 首先我们保留 kubeadm config 文件：

```
$ kubeadm config view > kubeadm-config.yaml
apiServer:
  extraArgs:
    authorization-mode: Node,RBAC
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.aliyuncs.com/k8sxio  # 修改成阿里云镜像源
kind: ClusterConfiguration
kubernetesVersion: v14
networking:
  dnsDomain: cluster.local
  podSubnet: 20/16
  serviceSubnet: 0/12
scheduler: {}
```

> 将上面的 `imageRepository` 值更改为：

```
registry.aliyuncs.com/k8sxio
```

，然后保存内容到文件 kubeadm-config.yaml 中（当然如果你的集群可以获取到 grc.io 的镜像可以不用更改）。

> 然后更新 kubeadm:

```
$ yum makecache fast && yum install -y kubeadm-14-0 kubectl-14-0
$ kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"16", GitVersion:"v14", GitCommit:"d2a081c8e14e21e28fe5bdfa38a817ef9c0bb8e3", GitTreeState:"clean", BuildDate:"2020-08-13T12:31:14Z", GoVersion:"go15", Compiler:"gc", Platform:"linux/amd64"}
```

> 因为 `kubeadm upgrade plan` 命令执行过程中会去 `dl.k8s.io` 获取版本信息，这个地址是需要科学方法才能访问的，所以我们可以先将 kubeadm 更新到目标版本，然后就可以查看到目标版本升级的一些信息了。

> 执行 `upgrade plan` 命令查看是否可以升级：

```
$ kubeadm upgrade plan
[upgrade/config] Making sure the configuration is correct:
[upgrade/config] Reading configuration from the cluster...
[upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[preflight] Running pre-flight checks.
[upgrade] Making sure the cluster is healthy:
[upgrade] Fetching available versions to upgrade to
[upgrade/versions] Cluster version: v2
[upgrade/versions] kubeadm version: v14
I0827 15:46:805052   11355 version.go:251] remote version is much newer: v0; falling back to: stable-16
[upgrade/versions] Latest stable version: v14
[upgrade/versions] Latest version in the v16 series: v14

Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   CURRENT       AVAILABLE
Kubelet     7 x v2   v14

Upgrade to the latest version in the v16 series:

COMPONENT            CURRENT   AVAILABLE
API Server           v2   v14
Controller Manager   v2   v14
Scheduler            v2   v14
Kube Proxy           v2   v14
CoreDNS              2     2
Etcd                 15    15-0

You can now apply the upgrade by executing the following command:

    kubeadm upgrade apply v14

_____________________________________________________________________
```

> 我们可以先使用 `dry-run` 命令查看升级信息：

```
$ kubeadm upgrade apply v14 --config kubeadm-config.yaml --dry-run
```

> 注意要通过 `--config` 指定上面保存的配置文件，该配置文件信息包含了上一个版本的集群信息以及修改过后的镜像地址。

> 查看了上面的升级信息确认无误后就可以执行升级操作了，我们可以先提前下载所需镜像：

```
$ kubeadm config images pull --config kubeadm-config.yaml
[config/images] Pulled registry.aliyuncs.com/k8sxio/kube-apiserver:v14
[config/images] Pulled registry.aliyuncs.com/k8sxio/kube-controller-manager:v14
[config/images] Pulled registry.aliyuncs.com/k8sxio/kube-scheduler:v14
[config/images] Pulled registry.aliyuncs.com/k8sxio/kube-proxy:v14
[config/images] Pulled registry.aliyuncs.com/k8sxio/pause:1
[config/images] Pulled registry.aliyuncs.com/k8sxio/etcd:15-0
[config/images] Pulled registry.aliyuncs.com/k8sxio/coredns:2
```

> 然后就可以执行真正的升级命令：

```
$ kubeadm upgrade apply v14 --config kubeadm-config.yaml
kubeadm upgrade apply v14 --config kubeadm-config.yaml
[upgrade/config] Making sure the configuration is correct:
[preflight] Running pre-flight checks.
[upgrade] Making sure the cluster is healthy:
[upgrade/version] You have chosen to change the cluster version to "v14"
[upgrade/versions] Cluster version: v2
[upgrade/versions] kubeadm version: v14
[upgrade/confirm] Are you sure you want to proceed with the upgrade? [y/N]:y
......
```

> 隔一段时间看到如下信息就证明集群升级成功了：

```
......
[addons]: Migrating CoreDNS Corefile
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

[upgrade/successful] SUCCESS! Your cluster was upgraded to "v14". Enjoy!

[upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
```

> 由于上面我们已经更新过 kubectl 了，现在我们用 kubectl 来查看下版本信息：

```
$ kubectl version
Client Version: version.Info{Major:"1", Minor:"16", GitVersion:"v14", GitCommit:"d2a081c8e14e21e28fe5bdfa38a817ef9c0bb8e3", GitTreeState:"clean", BuildDate:"2020-08-13T12:33:34Z", GoVersion:"go15", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"16", GitVersion:"v14", GitCommit:"d2a081c8e14e21e28fe5bdfa38a817ef9c0bb8e3", GitTreeState:"clean", BuildDate:"2020-08-13T12:24:51Z", GoVersion:"go15", Compiler:"gc", Platform:"linux/amd64"}
$ kubectl get nodes
NAME          STATUS   ROLES    AGE    VERSION
ydzs-master   Ready    master   292d   v2
ydzs-node1    Ready    <none>   292d   v2
ydzs-node2    Ready    <none>   292d   v2
ydzs-node3    Ready    <none>   290d   v2
ydzs-node4    Ready    <none>   290d   v2
ydzs-node5    Ready    <none>   218d   v2
ydzs-node6    Ready    <none>   218d   v2
```

> 可以看到版本并没有更新，这是因为节点上的 kubelet 还没有更新的，我们可以通过 kubelet 查看下版本：

```
$ kubelet --version
Kubernetes v2
```

> 这个时候我们去手动更新下 kubelet：

```
$ yum install -y kubelet-14-0
# 安装完成后查看下版本
$ kubelet --version
Kubernetes v14
# 然后重启 kubelet 服务
$ systemctl daemon-reload
$ systemctl restart kubelet
$ kubectl get nodes
NAME          STATUS   ROLES    AGE    VERSION
ydzs-master   Ready    master   292d   v14
ydzs-node1    Ready    <none>   292d   v2
ydzs-node2    Ready    <none>   292d   v2
ydzs-node3    Ready    <none>   290d   v2
ydzs-node4    Ready    <none>   290d   v2
ydzs-node5    Ready    <none>   218d   v2
ydzs-node6    Ready    <none>   218d   v2
```

> 可以看到 master 节点已经更新到 v1.26.14 版本了，然后就可以去升级节点了，升级节点的时候最好先驱逐节点，然后逐个升级：

```
$ kubectl drain ydzs-node1 --ignore-daemonsets
node/ydzs-node1 cordoned
error: unable to drain node "ydzs-node1", aborting command...

There are pending nodes to be drained:
 ydzs-node1
error: cannot delete Pods with local storage (use --delete-local-data to override): rook-ceph/csi-cephfsplugin-provisioner-56c8b7ddf4-n96kk, rook-ceph/csi-rbdplugin-provisioner-6ff4dd4b94-2bl82
$ kubectl get nodes
NAME          STATUS                     ROLES    AGE    VERSION
ydzs-master   Ready                      master   292d   v14
ydzs-node1    Ready,SchedulingDisabled   <none>   292d   v2
ydzs-node2    Ready                      <none>   292d   v2
ydzs-node3    Ready                      <none>   290d   v2
ydzs-node4    Ready                      <none>   290d   v2
ydzs-node5    Ready                      <none>   218d   v2
ydzs-node6    Ready                      <none>   218d   v2
```

> 然后在 ydzs-node1 节点上执行更新命令：

```
$ kubeadm upgrade node
[upgrade] Reading configuration from the cluster...
[upgrade] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[upgrade] Skipping phase. Not a control plane node[kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-16" ConfigMap in the kube-system namespace
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[upgrade] The configuration for this node was successfully updated!
[upgrade] Now you should go ahead and upgrade the kubelet package using your package manager.
# 更新软件包
$ yum install -y kubeadm-14-0 kubectl-14-0 kubelet-14-0
# 安装完成后重启 kubelet
$ systemctl daemon-reload
$ systemctl restart kubelet
```

> 更新完成后，确认节点升级成功：

```
$ kubectl get nodes
NAME          STATUS                     ROLES    AGE    VERSION
ydzs-master   Ready                      master   292d   v14
ydzs-node1    Ready,SchedulingDisabled   <none>   292d   v14
ydzs-node2    Ready                      <none>   292d   v2
ydzs-node3    Ready                      <none>   290d   v2
ydzs-node4    Ready                      <none>   290d   v2
ydzs-node5    Ready                      <none>   218d   v2
ydzs-node6    Ready                      <none>   218d   v2
```

> 然后解除禁止调度：

```
$ kubectl uncordon ydzs-node1
node/ydzs-node1 uncordoned
```

> 用同样的方式升级其他节点即可升级成功：

```
$ kubectl get nodes
NAME          STATUS   ROLES    AGE    VERSION
ydzs-master   Ready    master   292d   v14
ydzs-node1    Ready    <none>   292d   v14
ydzs-node2    Ready    <none>   292d   v14
ydzs-node3    Ready    <none>   290d   v14
ydzs-node4    Ready    <none>   290d   v14
ydzs-node5    Ready    <none>   218d   v14
ydzs-node6    Ready    <none>   218d   v14
```

Copyright © 2019-2020 优点知识

powered by MkDocs and Material for MkDocs