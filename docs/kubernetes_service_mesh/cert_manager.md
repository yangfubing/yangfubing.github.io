
## CertManager 自动 HTTPS

> cert-manager 是一种自动执行证书管理的工具，它可以与 Istio Gateway 集成以管理 TLS 证书，当然也可以很方便地和前面我们配置的 ingress-nginx 或者 traefik 配合使用。对于和 Istio 的集成使用，无需特殊配置即可。

> 注意

> 在进行本节实验之前记得将前面章节的实验内容进行清空。

### 安装

> 要在 Kubernetes 集群上安装 cert-manager 也非常简单，官方提供了一个单一的资源清单文件，包含了所有的资源对象，所以直接安装即可：

```
# Kubernetes 15+
$ kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1/cert-manager.yaml
# Kubernetes <15
$ kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1/cert-manager-legacy.yaml
```

> 上面的命令会创建一个名为 cert-manager 的命名空间，安装大量的 CRD 以及 AdmissionWebhook 对象，可以通过如下命令来查看是否安装成功：

```
$ kubectl get pods --namespace cert-manager
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-7ddc5b4db-56nln               1/1     Running   0          4m46s
cert-manager-cainjector-6644dc4975-zthcj   1/1     Running   0          4m47s
cert-manager-webhook-7b887475fb-9fbp4      1/1     Running   0          4m45s
```

> 正常情况下可以看到 cert-manager、cert-manager-cainjector 以及 cert-manager-webhook 这几个 Pod 处于 Running 状态。我们可以通过下面的测试来验证下是否可以签发基本的证书类型，创建一个 Issuer 资源对象来测试 webhook 工作是否正常(在开始签发证书之前，必须在群集中至少配置一个 Issuer 或 ClusterIssuer 资源)：

```
$ cat <<EOF > test-resources.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: cert-manager-test
---
apiVersion: cert-manager.io/v1alpha2
kind: Issuer
metadata:
  name: test-selfsigned
  namespace: cert-manager-test
spec:
  selfSigned: {}  # 配置自签名的证书机构类型
---
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  name: selfsigned-cert
  namespace: cert-manager-test
spec:
  dnsNames:
  - example.com
  secretName: selfsigned-cert-tls
  issuerRef:
    name: test-selfsigned
EOF
```

> 这里我们创建了一个名为 `cert-manager-test` 的命名空间，创建了一个 Issuer 的证书颁发机构，然后使用这个 Issuer 来创建一个证书 Certificate 对象，直接创建上面的资源清单即可：

```
$ kubectl apply -f test-resources.yaml 
namespace/cert-manager-test created
issuer.cert-manager.io/test-selfsigned created
certificate.cert-manager.io/selfsigned-cert created
```

> 创建完成后可以检查新创建的证书状态，在 cert-manager 处理证书请求之前，可能需要稍微等几秒：

```
$ kubectl describe certificate -n cert-manager-test
Name:         selfsigned-cert
Namespace:    cert-manager-test
......
Spec:
  Dns Names:
    example.com
  Issuer Ref:
    Name:       test-selfsigned
  Secret Name:  selfsigned-cert-tls
Status:
  Conditions:
    Last Transition Time:  2020-08-16T01:50:40Z
    Message:               Certificate is up to date and has not expired
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2020-11-14T01:50:39Z
  Not Before:              2020-08-16T01:50:39Z
  Renewal Time:            2020-10-15T01:50:39Z
  Revision:                1
Events:
  Type    Reason     Age   From          Message
  ----    ------     ----  ----          -------
  Normal  Issuing    64s   cert-manager  Issuing certificate as Secret does not exist
  Normal  Generated  64s   cert-manager  Stored new private key in temporary Secret resource "selfsigned-cert-25ftw"
  Normal  Requested  64s   cert-manager  Created new CertificateRequest resource "selfsigned-cert-p2sbx"
  Normal  Issuing    63s   cert-manager  The certificate has been successfully issued
```

> 从上面的 Events 事件中我们可以证书已经成功签发了，到这里证明我们的 cert-manager 已经安装成功了。而且我们需要注意的是 cert-manager 的功能非常强大，不只是可以支持 ACME 类型的证书签发，还支持其他众多的类型，比如 SelfSigned(自签名)、CA、Vault、Venafi、External、ACME，只是我们一般主要是使用 ACME 来帮我们生成自动化的证书。

### 环境配置

> 由于通过 ACME 做自动化证书的时候，需要暴露 80 和 443 端口，当然如果我们使用 DNS 校验方式也可以，但是有时候我们根本就没有域名的情况下想要实现自动化证书，我们可以使用 `xip.io` 这类的服务来实现。前面我们部署 istio-ingressgateway 的时候是通过 NodePort 类暴露的服务，所以我们需要在前面加一个 LB 来转发下请求。这里为了简单，我直接使用 haproxy 来监听节点的 80 和 443 端口，将请求转发到后端的 NodePort 端口。

> 首先安装 haproxy:

```
$ yum install -y haproxy
```

> 然后配置 haproxy，配置文件 

```
/etc/haproxy/haproxy.cfg
```

，内容如下所示：

```
listen stats
  bind    *:9000
  mode    http
  stats   enable
  stats   hide-version
  stats   uri       /stats
  stats   refresh   30s
  stats   realm     Haproxy\\ Statistics
  stats   auth      Admin:Password

frontend istio-https
    bind *:443
    mode tcp
    option tcplog
    tcp-request inspect-delay 5s
    tcp-request content accept if { req.ssl_hello_type 1 }
    default_backend istio-https-svc

backend istio-https-svc
    mode tcp
    option tcplog
    option tcp-check
    balance roundrobin
    default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
    server istio-http-svc-1 11:30951 check

frontend istio-http
    bind *:80
    mode tcp
    option tcplog
    default_backend istio-http-svc

backend istio-http-svc
    mode tcp
    option tcplog
    option tcp-check
    balance roundrobin
    default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
    server istio-http-svc-1 11:32193 check
```

> 其中的 32193 与 30951 端口是 istio-ingressgateway 的 NodePort 端口：

```
$ kubectl get svc istio-ingressgateway -n istio-system
NAME                   TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                                      AGE
istio-ingressgateway   NodePort   11128   <none>        15020:31093/TCP,80:32193/TCP,443:30951/TCP,31400:31871/TCP,15443:31367/TCP   68d
```

> 配置完成后直接启动 haproxy 即可：

```
$ sudo systemctl start haproxy
$ sudo systemctl enable haproxy
$ sudo systemctl status haproxy
```

> 然后我们可以通过上面 9000 端口监控我们的 haproxy 的运行状态:

> ![](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200816120647.png)

### 与 Istio 集成

> 这里我们来通过 cert-manager 为前面的 httpbin 应用配置自动的 HTTPS，首先单独创建一个用户 cert-manager 的自定义的 Gateay，如下所示：

```
$ cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: acme-gateway
  labels:
    app: ingressgateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - hosts:
    - "*.xip.io"
    port:
      number: 80
      name: http
      protocol: HTTP
    # tls:
    #   httpsRedirect: true
  - hosts:
    - "*.xip.io"
    port:
      name: https
      number: 443
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: "xip-cert"
---
EOF
gateway.networking.istio.io/acme-gateway created
```

> 需要注意的是在配置 Gateway 的时候 tls 的 credentialName 代表的是 cert-manager 自动生成的证书名称。

> 接下来创建 VirtulService 对象，这里我们使用 https 方式的 ACME 证书校验方式，除了 http 方式之外还有 tls 与 dns 方式的校验，dns 方式的证书校验支持通配符的域名。所以我们需要为 `/.well-known/` 这个 PATH 路径做一个正确的配置，方便进行 http 校验：

```
$ cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  gateways:
  - acme-gateway
  hosts:
  - httpbin.11xip.io
  http:
  - route:
    - destination:
        host: httpbin
        port:
          number: 8000
EOF
virtualservice.networking.istio.io/httpbin created
```

> 部署完成后我们可以通过访问 

```
http://httpbin.11xip.io
```

 来验证是否已经部署成功：

> ![](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200816121734.png)

> 接下来创两个 ClusterIssuer，一个用于测试，一个用于正式使用：

```
$ cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-vapi.letsencrypt.org/directory
    email: icnych@xxx.com
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - http01:
        ingress:
          class: istio  
EOF
clusterissuer.cert-manager.io/letsencrypt-staging created
$ cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1beta1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-vapi.letsencrypt.org/directory
    email: icnych@xxx.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: istio  
EOF
clusterissuer.cert-manager.io/letsencrypt-prod created
$ kubectl describe clusterissuer                
......
Status:
  Acme:
    Last Registered Email:  icnych@xxx.com
    Uri:                    https://acme-vapi.letsencrypt.org/acme/acct/94057742
  Conditions:
    Last Transition Time:  2020-08-16T07:37:40Z
    Message:               The ACME account was registered with the ACME server
    Reason:                ACMEAccountRegistered
    Status:                True
    Type:                  Ready
Events:                    <none>
```

> 然后创建一个 Certificate 对象来获取证书，由于 Istio 需要在它的命名空间下面有证书，所以我们需要在 istio-system 这个命名空间下面创建:

```
$ cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1beta1
kind: Certificate
metadata:
  name: httpbin-cert
  namespace: istio-system  # 这里必须是 istio-system 空间
spec:
  secretName: xip-cert  # 这个就是上面 gateway 所配置的证书名称
  issuerRef:
    kind: ClusterIssuer
    name: letsencrypt-staging
  commonName: httpbin.11xip.io
  dnsNames:
  - httpbin.11xip.io
EOF
certificate.cert-manager.io/httpbin-cert created
```

> 创建完成后这时候查看 istio-system 空间应该会有一个 `cm-acme-http-solver-` 这个开头的 Pod、Servie、Ingress 资源对象：

```
$ kubectl get pods -n istio-system
NAME                                    READY   STATUS    RESTARTS   AGE
cm-acme-http-solver-8tpbn               1/1     Running   0          16s
$ kubectl get svc -n istio-system
NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                                      AGE
cm-acme-http-solver-w6sf7   NodePort    1120    <none>        8089:32135/TCP
$ kubectl get ingress -n istio-system
NAME                        HOSTS                          ADDRESS   PORTS   AGE
cm-acme-http-solver-jkrs2   httpbin.11xip.io             80      45s
```

> 隔一小会儿去查看上面部署的 Certificate 对象的状态：

```
$ kubectl describe certificate httpbin-cert -n istio-system
......
Events:
  Type    Reason     Age    From          Message
  ----    ------     ----   ----          -------
  Normal  Issuing    13m    cert-manager  Issuing certificate as Secret does not exist
  Normal  Generated  13m    cert-manager  Stored new private key in temporary Secret resource "http-cert-6cml8"
  Normal  Requested  13m    cert-manager  Created new CertificateRequest resource "http-cert-9wcg2"
  Normal  Issuing    6m17s  cert-manager  The certificate has been successfully issued
```

> 看到最后的信息 

```
The certificate has been successfully issued
```

 证明我们的证书获取成功了，但是这个时候如果我们通过 https 去访问的话还是会提示证书错误的，因为我们获取的是 staging 环境的证书：

> ![](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200816122317.png)

> 这个时候我们重新更新 httpbin-cert 这个 Certificate 对象中引用的 issuer，更改为正式环境的 issuer，或者使用正式的签名机构新建一个证书对象，正常就可以获得线上环境的证书了。