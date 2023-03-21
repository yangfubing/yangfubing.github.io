 
## 可观测性

> 前面我们学习了 Istio 中的流量管理功能，本节我们来学习如何配置 Istio 来自动收集网格中的服务遥测，通过监控指标、日志、分布式追踪等方式来实现应用的可观测性。

> ![可观测性](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200626091556.png)

### 使用 Kiali 观测微服务

> 由于微服务之间的调用关系错综复杂，排查问题就更加困难了，为了使服务之间的关系更加清晰明了，了解应用的行为和状态，我们有必要使用一些可视化的方案来观测我们的微服务应用，其中 Kiali 就是这样的一个工具。

> Kiali 是 Istio 的一个可观测工具，提供服务拓扑展示服务网格的结构，提供网格的健康状态视图，配置信息验证功能，此外还具有服务网格配置的功能。

> 默认情况下，Kiali 服务已经安装了，可以通过 istioctl 命令来访问，比如我们可以使用如下命令来打开 kiali 的 web UI：

```
$ istioctl dashboard kiali --address=0
http://localhost:44759/kiali
```

> 然后就可以通过 `44759` 端口访问到 kiali 服务了。此外我们也可以通过修改 Kiali 的服务为 NodePort 类型来访问：

```
$ kubectl get pods -n istio-system -l app=kiali
NAME                     READY   STATUS    RESTARTS   AGE
kiali-6f457f5964-8p8gr   1/1     Running   0          32d
$ kubectl get svc -n istio-system -l app=kiali
NAME    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)     AGE
kiali   ClusterIP   178   <none>        20001/TCP   32d
$ kubectl edit svc kiali -n istio-system
......
spec:
  clusterIP: 178
  ports:
  - name: http-kiali
    port: 20001
    protocol: TCP
    targetPort: 20001
  selector:
    app: kiali
  sessionAffinity: None
  type: NodePort
......
$ kubectl get svc -n istio-system -l app=kiali
NAME    TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)           AGE
kiali   NodePort   178   <none>        20001:31098/TCP   32d
```

> ![](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200711121832.png)

> 默认的用户名和密码存储在名为 kiali 的 Secret 对象之中，需要将其中的 username 与 passphrase 做 base64 解码获得用户名和密码：

```
$ kubectl get secret -n istio-system kiali -o yaml
apiVersion: v1
data:
  passphrase: YWRtaW4=
  username: YWRtaW4=
kind: Secret
......
```

> ![](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200711122104.png)

> 默认会安装命名空间进行分类，我们的 BookInfo 应用部署在 default 命名空间之下的，我们可以点击 default 命名空间之下的应用：

> ![](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200711122733.png)

> 还切换到 Graph 页面下面查看微服务应用的整个调用链：

> ![](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200711122921.png)

> 我们可以看到上图非常清晰的把 BookInfo 应用的调用链给展示出来了，这对于我们了解我们的应用是非常有帮助的。还可以查看服务的详细信息：

> ![](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200711145119.png)

> 在该页面之中还可以查看应用的指标、链路追踪等信息。此外在 `Istio Config` 路径下面还可以查看整个网格中的配置校验情况：

> ![](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200711145702.png)

> 红色标记的表示验证未通过的配置，可以点击进去编辑错误的配置，此外还可以在页面上新建 Istio 的配置：

> ![](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200711145955.png)

### 监控指标

> 默认情况下 Istio 就部署了用于收集监控指标的 Prometheus 应用:

```
$ kubectl get pods -n istio-system -l app=prometheus
NAME                          READY   STATUS    RESTARTS   AGE
prometheus-6dd77d88cf-nmsl7   2/2     Running   0          32d
$ kubectl get svc -n istio-system -l app=prometheus
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
prometheus   ClusterIP   1202   <none>        9090/TCP   32d
```

> 为了测试方便我们可以将 Prometheus 的 Service 更改为 NodePort 类型：

```
$ kubectl edit svc prometheus -n istio-system
......
spec:
  type: NodePort
......
$ kubectl get svc -n istio-system -l app=prometheus
NAME         TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
prometheus   NodePort   1202   <none>        9090:31128/TCP   32d
```

> 然后我们就可以通过 

```
http://<NodeIP>:31128
```

 访问 Prometheus 页面，默认情况下就已经有了很多抓取任务了：

> ![](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200711102255.png)

> 默认就已经收集了很多网格的指标，包括服务级别和代理级别（sidecar）的数据，自 Istio 1.5 起，Istio 标准指标由 Envoy 代理直接获取，之前是通过 Mixer 组件生成的。

> ![](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200711152018.png)

> 从上图可以看出现在的架构不需要 Mixer 这样的组件来完成了，在性能上得到了大幅度的提升，但是数据都需要通过 Envoy 代理来自己上传，扩展起来不是很方便，现在是通过 `STATS` 和 `METADATA EXCHANGE` 这两个插件来完成的，但是扩展性没有以前的 Mixer 组件强大了，后续或许会使用 WebAssembly 这样的技术来解决这个问题。

> 对于 HTTP，HTTP/2 和 GRPC 流量，Istio 生成以下指标：

*   请求数（istio_requests_total）：这是一个用于累加每个由 Istio 代理所处理请求的 COUNTER 指标。
*   请求时长（istio_request_duration_seconds）：这是一个用于测量请求的持续时间的指标。
*   请求大小（istio_request_bytes）：这是一个用于测量 HTTP 请求 body 大小的指标。
*   响应大小（istio_response_bytes）：这是一个用于测量 HTTP 响应 body 大小的指标。

> 对于 TCP 流量，Istio 生成以下指标： * TCP 发送字节数（istio_tcp_sent_bytes_total）：这是一个用于测量在 TCP 连接下响应期间发送的总字节数的 COUNTER 指标。 * TCP 接收字节数（istio_tcp_received_bytes_total）：这是一个用于测量在 TCP 连接下请求期间接收的总字节数的 COUNTER 指标。 * TCP 打开连接数（istio_tcp_connections_opened_total）：这是一个用于累加每个打开连接的 COUNTER 指标。 * TCP 关闭连接数 (istio_tcp_connections_closed_total) : 这是一个用于累加每个关闭连接的 COUNTER 指标。

### 使用 Grafana 可视化系统监控

> 上面我们了解到了 Istio 网格中默认通过 Prometheus 收集了很多服务和代理相关的指标数据，此外 Istio 还默认开启了 Grafana 工具，我们可以通过 Grafana 来可视化查看网格的监控状态。

```
$ kubectl get pods -n istio-system -l app=grafana
NAME                       READY   STATUS    RESTARTS   AGE
grafana-74dc798895-mns2v   1/1     Running   0          32d
$ kubectl get svc -n istio-system -l app=grafana
NAME      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
grafana   ClusterIP   254   <none>        3000/TCP   32d
```

> 同样我们这里将 grafana 的 Service 对象修改为 NodePort 类型的服务：

```
$ kubectl edit svc grafana -n istio-system
......
spec:
  type: NodePort
......
$ kubectl get svc -n istio-system -l app=grafana
NAME      TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
grafana   NodePort   254   <none>        3000:30692/TCP   32d
```

> 然后就可以通过 

```
http://<NodeIP>:30692
```

 访问 Grafana 应用了：

> ![](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200711153433.png)

> 默认情况下 Grafana 中就已经导入了 Istio 的几个 Dashboard： * Mesh Dashboard：主要是查看应用的数据，包括网格数据总览、服务视图、工作负载视图等 * Performance Dashboard：主要是用于查看 Istio 本身的监控数据。

> ![](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200711153944.png)

### 访问日志

> 日志也是应用可观测性中一个非常重要的方式，也是我们传统调试应用非常重要的手段，在 Istio 中使用 Sidecar 容器对请求进行了拦截，无疑也增大了调试难度，但是同时 Istio 也可以监测到网格内的服务通信的流转情况，并生成详细的遥测日志数据，任何请求与事件的元信息都可以获取到，所以我们也非常有必要来查看下 Istio 中的代理日志数据。Istio 最简单的日志类型是 `Envoy 的访问日志`，Envoy 代理打印访问日志信息到标准输出，然后我们就可以通过 `kubectl logs` 命令打印出来查看了。

> 默认情况下 Istio 已经开启了 Envoy 访问日志，我们也可以通过 Istio 的 ConfigMap 配置来修改日志的格式：

```
$ kubectl edit cm istio -n istio-system 
apiVersion: v1
data:
  mesh: |-
    accessLogEncoding: JSON
    accessLogFile: /dev/stdout
    accessLogFormat: ""
    outboundTrafficPolicy:  
      mode: REGISTRY_ONLY
    defaultConfig:
......
```

> 默认情况下日志就是输出到 stdout 上的 TEXT 文本格式，为了方便显示，这里我们将其设置为 JSON 格式，如果要想修改访问日志的格式可以设置 `accessLogFormat` 属性，具体的访问日志格式可以查看 Envoy 官方文档了解配置规则。

> ![istio log config](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200719152344.png)

> 现在我们去访问 BookInfo 服务，同时来查看 productpage 的 sidecar 日志：

```
$ kubectl get pods -l app=productpage
NAME                              READY   STATUS    RESTARTS   AGE
productpage-v1-5f96867467-nsclh   2/2     Running   0          8d
$ kubectl logs -f productpage-v1-5f96867467-nsclh -c istio-proxy
{"user_agent":"Mozilla/0 (Macintosh; Intel Mac OS X 10_15_6) AppleWebKit/536 (KHTML, like Gecko) Chrome/41116 Safari/536","response_code":"200","response_flags":"-","start_time":"2020-07-19T06:48:074Z","method":"GET","request_id":"6df02877-0e8d-99f4-a9ee-0f9479d648dd","upstream_host":"2180:9080","x_forwarded_for":"-","requested_server_name":"-","bytes_received":"0","istio_policy_status":"-","bytes_sent":"178","upstream_cluster":"outbound|9080||details.default.svc.cluster.local","downstream_remote_address":"2227:47244","authority":"details:9080","path":"/details/0","protocol":"HTTP/1","upstream_service_time":"17","upstream_local_address":"2227:42248","duration":"18","upstream_transport_failure_reason":"-","route_name":"default","downstream_local_address":"11166:9080"}
{"user_agent":"Mozilla/0 (Macintosh; Intel Mac OS X 10_15_6) AppleWebKit/536 (KHTML, like Gecko) Chrome/41116 Safari/536","response_code":"200","response_flags":"-","start_time":"2020-07-19T06:48:104Z","method":"GET","request_id":"6df02877-0e8d-99f4-a9ee-0f9479d648dd","upstream_host":"2182:9080","x_forwarded_for":"-","requested_server_name":"-","bytes_received":"0","istio_policy_status":"-","bytes_sent":"295","upstream_cluster":"outbound|9080||reviews.default.svc.cluster.local","downstream_remote_address":"2227:36718","authority":"reviews:9080","path":"/reviews/0","protocol":"HTTP/1","upstream_service_time":"20","upstream_local_address":"2227:42884","duration":"21","upstream_transport_failure_reason":"-","route_name":"default","downstream_local_address":"144:9080"}
{"bytes_sent":"4183","upstream_cluster":"inbound|9080|http|productpage.default.svc.cluster.local","downstream_remote_address":"20:0","authority":"k8s.qikqiak.com:32193","path":"/productpage","protocol":"HTTP/1","upstream_service_time":"70","upstream_local_address":"11:36436","duration":"71","upstream_transport_failure_reason":"-","route_name":"default","downstream_local_address":"2227:9080","user_agent":"Mozilla/0 (Macintosh; Intel Mac OS X 10_15_6) AppleWebKit/536 (KHTML, like Gecko) Chrome/41116 Safari/536","response_code":"200","response_flags":"-","start_time":"2020-07-19T06:48:061Z","method":"GET","request_id":"6df02877-0e8d-99f4-a9ee-0f9479d648dd","upstream_host":"11:9080","x_forwarded_for":"20","requested_server_name":"outbound_.9080_._.productpage.default.svc.cluster.local","bytes_received":"0","istio_policy_status":"-"}
```

> > 这里我们查询的容器名 `istio-proxy` 其实就是 Envoy sidecar 代理，Envoy 将请求和响应日志都进行了打印并输出至 stdout ，所以可以通过 `kubectl logs` 查询。

> 正常就会收到3条 JSON 格式的日志，productpage 服务会调用 details 与 reviews 服务，所以我们可以看到一条访问 `/reviews/0` 与 `/details/0` 的 `outbound` 请求，和最终的 `/productpage` 的 `inbound` 请求记录。

> 为了方便查看，这里我们将第一条日志进行格式化，如下所示： ![json format](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200719150158.png)

> 这里面的日志信息其实是一个标准的 Envoy 流量模型，所以如果我们要深入研究 Istio，对应 Envoy 的了解是必不可少的，其中最重要的几个字段就是 Envoy 流量五元组中的几个信息。

> 在 Envoy 中接受请求流量叫做 `Downstream（下游）`，Envoy 发出请求流量叫做 `Upstream（上游）`，在处理 `Downstream` 和 `Upstream` 过程中，分别会涉及2个流量端点，即请求的发起端和接收端：

> ![envoy request model](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200719150602.png)

> 在这个过程中，Envoy 会根据用户规则，计算出符合条件的转发目的主机集合，这个集合叫做 `UPSTREAM_CLUSTER`, 并根据负载均衡规则，从这个集合中选择一个 host 作为流量转发的接收端点，这个 host 就是 `UPSTREAM_HOST`。这就是 Envoy 请求处理的`流量五元组`信息，这是 Envoy 日志里最重要的部分，通过这个五元组我们可以准确的观测流量`「从哪里来」和「到哪里去」`：

*   downstream_remote_address
*   downstream_local_address
*   upstream_local_address
*   upstream_host
*   upstream_cluster

> 除了五元组信息之外，还有 `request_id` 属性，该属性主要用于链路追踪，可以将一条请求在不同的服务中的调用串联起来。

> 还有一个非常重要的属性 `response_flags`，可以通过该属性来查看当前请求的状态，用于调试请求的时候非常有用： ![response flag](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200719151232.png)

> 不过需要注意的是，当 Pod 被销毁后，旧的日志将不复存在，也无法通过 `kubectl logs` 就行查看了。所以如果要查看历史的的日志数据，我们可以使用 `EFK` 方案将日志进行收集，前面我们已经仔细讲解过了，这里就不再赘述了。

### 分布式追踪

> 相比传统的单体应用，微服务的一个主要变化是将应用中的不同模块拆分为了独立的服务，在微服务架构下，原来进程内的方法调用成为了跨进程的远程方法调用。相对于单一进程内的方法调用而言，跨进程调用的调试和故障分析是非常困难的，难以使用传统的代码调试程序或者日志打印来对分布式的调用过程进行查看和分析。一个来自客户端的请求在其业务处理过程中很有可能需要经过多个微服务，我们如果想要对该请求的端到端调用过程进行完整的分析，则必须将该请求经过的所有进程的相关信息都收集起来并关联在一起，这就是`分布式追踪`，也是应用可观测性中非常重要的手段。

> 在 Istio 中通过分布式追踪我们可以让用户对跨多个分布式服务网格的一个请求进行追踪分析，进而可以通过可视化的方式更加深入地了解请求的`延迟、序列化和并行度`等。

> Istio 利用 Envoy 的分布式追踪功能提供了开箱即用的追踪集成，默认 Istio 提供了集成各种追踪后端服务的选项，并且可以通过配置代理来自动发送追踪 `span` 到追踪后端服务。

> 在使用 Istio 的分布式追踪之前，我们需要先了解两个重要的概念：`Span` 与 `Trace`。

> ![tracing](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200719154443.png)

*   Span：分布式追踪的基本组成单元，表示一个分布式系统中的单独的工作单元，每一个 Span 可以包含其它 Span 的引用。多个 Span 在一起构成了 Trace。
*   Trace：微服务中记录的完整的请求执行过程，一个完整的 Trace 由一个或多个 Span 组成。

> 随着分布式追踪技术的发展，社区推出了 OpenTracing 这个规范，提供了标准的 API 规范、框架以及一些公共的库。目前比较知名的追踪工具基本上都通过 OpenTracing 进行实现，比如 Jaeger、Zipkin 等。

> 我们这里主要来给大家介绍比较流行的 Jaeger。 `Jaeger` 是由 Uber 开源的分布式追踪系统，采用 Go 语言编写，主要借鉴了 `Google Dapper` 论文和 `Zipkin` 的设计，兼容 OpenTracing 以及 Zipkin 追踪格式，目前已成为 CNCF 基金会的开源项目。

> ![jaeger logo](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200719155511.png)

> 我们这里安装 Istio 的时候使用的是 demo 这个 profile，默认情况下已经安装了 Jaeger，如果没有安装可以在使用 istioctl 命令初始化的时候添加 

```
--set values.tracing.enabled=true
```

 参数进行开启：

```
$ istioctl manifest apply --set values.tracing.enabled=true
```

> 如果集群中已经有一个可以使用的 Jaeger 服务的话，这可以使用下面的命令进行关联：

```
$ istioctl manifest apply --set values.global.tracer.zipkin.address=<jaeger-collector-service>.<jaeger-collector-namespace>:9411
```

> 我们可以查看默认已经安装的 Jaeger 服务：

```
kubectl get pods -l app=jaeger -n istio-system
NAME                             READY   STATUS    RESTARTS   AGE
istio-tracing-8584b4d7f9-vmcdx   1/1     Running   0          40d
$ kubectl get svc -l app=jaeger -n istio-system
NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                         AGE
jaeger-agent                ClusterIP   None             <none>        5775/UDP,6831/UDP,6832/UDP      40d
jaeger-collector            ClusterIP   74       <none>        14267/TCP,14268/TCP,14250/TCP   40d
jaeger-collector-headless   ClusterIP   None             <none>        14250/TCP                       40d
jaeger-query                ClusterIP   1127    <none>        16686/TCP                       40d
tracing                     ClusterIP   11125   <none>        80/TCP                          40d
zipkin                      ClusterIP   1119    <none>        9411/TCP                        40d
```

> 我们可以通过 `tracing` 这个 Service 来访问 Jaeger 的 Dashboard 页面，同样为了方便测试将其修改为 NodePort 类型：

```
$ kubectl edit svc tracing -n istio-system
......
spec:
  type: NodePort
......
service/tracing edited
$ kubectl get svc tracing -n istio-system
NAME      TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
tracing   NodePort   11125   <none>        80:31755/TCP   40d
```

> 修改完成后就可以在浏览器中通过 

```
http://<NodeIP>:31755
```

 访问 Jaeger 的 Dashboard 页面了：

> ![Jaeger Dashboard](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200719161244.png)

> 接下来我们重新访问 BookInfo 应用产生流量请求生成追踪数据。请求完成后回到 Jaeger 页面，在左侧 Servcie 区域选择一个我们要追踪的服务，比如 `details.default`，点击 `"Find Traces"` 按钮查看追踪结果。

> ![Find Traces](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200719161645.png)

> 点击列表项目还可以查看追踪详细信息，记录了一次请求涉及到的 Services、深度、Span 总数、请求总时长等信息，也可以对下方的单项服务展开，观察每一个服务的请求耗时和详情。

> ![Trace Detail](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200719161848.png)

> 此外我们还可以切换 Trace 的显示方式为 Graph，在右上角点击 `Trace Timeline` 切换为 `Trace Graph` 模式，该模式下可以更加清晰查看到每一个调用详细信息：

> ![Trace Graph](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200719163141.png)

> 此外，Jaeger 还可以展示服务依赖，点击顶部的 `Dependencies` 菜单查看，该页面可以查看服务的完整依赖调用关系：

> ![Jeager 依赖](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200719162121.png)

> 此外还可以对比两个 Trace，在顶部 `Compare` 菜单中，输入两个不同的 `Trace ID` 即可进行对比，这可以帮助我们发现不同 Trace 之间的差异：

> ![Trace Compare](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200719162545.png)

> 这就是 Jaeger 的基本使用，不过需要注意的是 Istio 默认提供的 Jaeger 采用内存的存储方式，Pod 被销毁后数据也就丢失了。在生产环境中需要单独配置持久化存储数据库，具体可查看 Jaeger 官方文档。

> 到这里我们就完成了 Istio 中可观测性的实验：指标、日志、追踪，对于微服务应用可观测性对于我们监控和调试应用是非常重要的手段，我们非常有必要掌握这些方式。