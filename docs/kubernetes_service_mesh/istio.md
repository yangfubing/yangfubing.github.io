
## Istio

> 现在最火的后端架构无疑是`微服务`了，微服务将之前的单体应用拆分成了许多独立的服务，好处自然很多，但是随着应用的越来越大，微服务暴露出来的问题也就随之而来了，微服务越来越多，管理越来越麻烦，特别是要你部署一套新环境的时候，你就能体会到这种痛苦了，随之而来的`服务发现、负载均衡、Trace跟踪、流量管理、安全认证等等`问题。如果从头到尾完成过一套微服务框架的话，你就会知道这里面涉及到的东西真的非常多。对于 Java 领域来说还有 Spring Cloud 这种完整的微服务框架，但是也仅仅局限于 Java 语言。当然随着微服务的不断发展，微服务的生态也不断完善，最近就发现新一代的微服务开发就悄然兴起了，那就是 Service Mesh（服务网格）。

> 而 istio 是现在最热门的 Service Mesh 工具，istio 是由 Google、IBM、Lyft 等共同开源的 Service Mesh（服务网格）框架，于2017年初开始进入大众视野。Kubernetes 解决了云原生应用的部署问题，istio 解决的是应用的服务（流量）治理问题。现在最新的版本1.6版本已经将之前的架构改成了单体应用，性能也已经大幅度提升，所以我们完全有必要去学习了解 istio，因为这就是微服务领域的下一个标准。

### 安装

```
$ export ISTIO_VERSION=1
$ curl -L https://istio.io/downloadIstio | sh -
```

> 如果安装失败，可以用手动方式进行安装，在 GitHub Release 页面获取对应系统的下载地址：

```
$ wget https://github.com/istio/istio/releases/download/1/istio-1-linux-amdtar.gz
$ tar -xzf istioctl-1-linux-amdtar.gz
$ cp istio-1/bin/istioctl /usr/local/bin/
# 进入到 istio 解压的目录
$ cd istio-1 && ls -la
total 32
drwxr-x---   6 root root   108 Jun  5 00:05 .
dr-xr-x---. 19 root root  4096 Jun  8 18:32 ..
drwxr-x---   2 root root    21 Jun  5 00:05 bin
-rw-r--r--   1 root root 11348 Jun  5 00:05 LICENSE
drwxr-xr-x   7 root root   104 Jun  5 00:05 manifests
-rw-r-----   1 root root   786 Jun  5 00:05 manifest.yaml
-rw-r--r--   1 root root  6121 Jun  5 00:05 README.md
drwxr-xr-x  20 root root  4096 Jun  5 00:05 samples
drwxr-x---   3 root root   128 Jun  5 00:05 tools
```

> 安装 istio 的工具和文件准备好过后，直接执行如下所示的安装命令即可：

```
$ istioctl manifest apply --set profile=demo
✔ Istio core installed                                            
✔ Istiod installed                                                
- Processing resources for Addons, Egress gateways, Ingress gat...
✔ Ingress gateways installed                                      
✔ Egress gateways installed                                       
- Processing resources for Addons. Waiting for Deployment/istio...
✘ Addons encountered an error: failed to wait for resource: resources not ready after 5m0s: timed out waiting for the condition
Deployment/istio-system/kiali
Deployment/istio-system/prometheus
# 如果出现了错误可以重新执行一次
$ istioctl manifest apply --set profile=demo
✔ Istio core installed                                                                               
✔ Istiod installed                                                                                   
✔ Egress gateways installed                                                                          
✔ Ingress gateways installed                                                                         
✔ Addons installed                                                                                   
✔ Installation complete
```

> 安装完成后我们可以查看 istio-system 命名空间下面的 Pod 运行状态：

```
$ kubectl get pods -n istio-system
NAME                                    READY   STATUS    RESTARTS   AGE
grafana-74dc798895-mns2v                1/1     Running   0          9m51s
istio-egressgateway-7c4f7c7b8c-wjq68    1/1     Running   0          9m54s
istio-ingressgateway-56d5c6d74c-bbtgs   1/1     Running   0          9m54s
istio-tracing-8584b4d7f9-vmcdx          1/1     Running   0          9m51s
istiod-5f47bf5895-pnrsz                 1/1     Running   0          13m
kiali-6f457f5964-8p8gr                  1/1     Running   0          9m51s
prometheus-6dd77d88cf-nmsl7             2/2     Running   0          9m50s
```

> 如果都是 Running 状态证明 istio 就已经安装成功了。

### 示例安装

> 然后我们可以来安装官方提供的一个非常经典的 Bookinfo 应用示例，这个示例部署了一个用于演示多种 Istio 特性的应用，该应用由四个单独的微服务构成。 这个应用模仿在线书店的一个分类，显示一本书的信息。页面上会显示一本书的描述，书籍的细节（ISBN、页数等），以及关于这本书的一些评论。

> Bookinfo 应用分为四个单独的微服务：

*   productpage：这个微服务会调用 details 和 reviews 两个微服务，用来生成页面。
*   details：这个微服务中包含了书籍的信息。
*   reviews：这个微服务中包含了书籍相关的评论，它还会调用 ratings 微服务。
*   ratings：这个微服务中包含了由书籍评价组成的评级信息。

> reviews 微服务有 3 个版本：

*   v1 版本不会调用 ratings 服务。
*   v2 版本会调用 ratings 服务，并使用 1 到 5 个黑色星形图标来显示评分信息。
*   v3 版本会调用 ratings 服务，并使用 1 到 5 个红色星形图标来显示评分信息。

> ![istio bookinfo](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200608185158.png)

> 上图展示了使用 istio 后，整个应用实际的结构。所有的微服务都和一个 `Envoy sidecar` 封装到一起，sidecar 拦截所有到达和离开服务的请求。

> 进入解压的 istio 目录，执行如下命令：

```
$ kubectl apply -f <(istioctl kube-inject -f samples/bookinfo/platform/kube/bookinfo.yaml)
service/details created
serviceaccount/bookinfo-details created
deployment.apps/details-v1 created
service/ratings created
serviceaccount/bookinfo-ratings created
deployment.apps/ratings-v1 created
service/reviews created
serviceaccount/bookinfo-reviews created
deployment.apps/reviews-v1 created
deployment.apps/reviews-v2 created
deployment.apps/reviews-v3 created
service/productpage created
serviceaccount/bookinfo-productpage created
deployment.apps/productpage-v1 created
```

> 这里我们部署的 `bookinfo.yaml` 资源清单文件就是普通的 Kubernetes 的 Deployment 和 Service 的 yaml 文件，而 `istioctl kube-inject` 会在这个文件的基础上向其中的 Deployment 追加一个镜像为 

```
docker.io/istio/proxyv2:1
```

 的 sidecar 容器。

> 过一会儿就可以看到如下 service 和 pod 启动:

```
$ kubectl get po
NAME                              READY     STATUS    RESTARTS   AGE
details-v1-6c77545767-dj2k5       2/2       Running   1          4h
productpage-v1-79bd99d8c5-v95wf   2/2       Running   1          4h
ratings-v1-6df6bcf5ff-8ktgb       2/2       Running   1          4h
reviews-v1-84648f754c-wwkdg       2/2       Running   1          4h
reviews-v2-878d96859-cw6p5        2/2       Running   1          4h
reviews-v3-6c4489748c-pfwxk       2/2       Running   1          4h
$ kubectl get svc
NAME          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
details       ClusterIP   1128    <none>        9080/TCP   5h
kubernetes    ClusterIP   1        <none>        443/TCP    21h
productpage   ClusterIP   11117   <none>        9080/TCP   5h
ratings       ClusterIP   1115    <none>        9080/TCP   5h
reviews       ClusterIP   1121    <none>        9080/TCP   5h
```

> 现在应用的服务都部署成功并启动了，如果我们需要在集群外部访问，就需要添加一个 istio gateway。gateway 相当于 k8s 的 ingress controller 和 ingress。它为 HTTP/TCP 流量配置负载均衡，通常在服务网格边缘作为应用的 ingress 流量管理。

> 创建一个 gateway:

```
$ kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
gateway.networking.istio.io/bookinfo-gateway created
virtualservice.networking.istio.io/bookinfo created
```

> 验证 gateway 是否启动成功:

```
$ kubectl get gateway
NAME               AGE
bookinfo-gateway   4h
```

> 要想取访问这个应用，这里我们需要更改下 istio 提供的 istio-ingressgateway 这个 Service 对象，默认是 LoadBalancer 类型的服务：

```
$ kubectl get svc -n istio-systemNAME                        TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                                      AGE
istio-ingressgateway        LoadBalancer   11128   <pending>     15020:30844/TCP,80:31585/TCP,443:32239/TCP,31400:30183/TCP,15443:31664/TCP   23m
......
```

> LoadBalancer 类型的服务，实际上是用来对接云服务厂商的，如果我们没有对接云服务厂商的话，可以将这里类型改成 `NodePort`，但是这样当访问我们的服务的时候就需要加上 nodePort 端口了：

```
$ kubectl edit svc istio-ingressgateway -n istio-system
......
type: NodePort  # 修改成 NodePort 类型
status:
  loadBalancer: {}
```

> 这样我们就可以通过 

```
http://<NodeIP>:<nodePort>/productpage
```

 访问应用了：

> ![bookinfo demo](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200608195347.png)

> 刷新页面可以看到 Book Reviews 发生了改变，因为每次请求会被路由到到了不同的 Reviews 服务版本去：

> ![bookinfo demo](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200608195420.png)

> 至此，整个 istio 就安装并验证成功了，后面将进一步分析 Bookinfo 应用来学习 istio 的基本使用方式。