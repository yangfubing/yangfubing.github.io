
## Flannel 

> 我们知道 Docker 的网络模式可以解决一个节点上的容器之间的网络通信问题，但是对于跨主机的容器之间的通信就无能为力了，就需要借助第三方的工具来实现容器的跨主机通信。为解决容器跨主机通信问题，社区出现了很多种网络解决方案，不同的方案工作原理各有不同，对于网络环境的要求也各有不同，我们这里以使用最多的 `Flannel` 这个网络插件为例，来给大家讲解该经典网络插件的工作原理。

> `Flannel` 是 CoreOS（Etcd 的公司）推出的一个 Overlay 类型的容器网络插件，目前支持三种后端实现：`UDP`、`VXLAN`、`host-gw` 三种方式。UDP 是最开始支持的最简单的但是却是性能最差的一种方式，因此基本上在正式使用的时候不会使用这种方式，不过该方式由于非常简单所有可以有助于我们来理解容器跨主机网络通信的实现原理，所以我们先来和大家了解下 UDP 方式的实现方式。

### UDP 方式 

> 还是要 `UDP` 模式我们需要在 Flannel 的配置文件中指定 `Backend type` 为 `UDP`，可以直接修改 Flannel 的方式实现：

```
$ kubectl edit cm kube-flannel-cfg -n kube-system
apiVersion: v1
data:
  cni-conf.json: |
    {
      "cniVersion": "0",
      "name": "cbr0",
      "plugins": [
        {
          "type": "flannel",
          "delegate": {
            "hairpinMode": true,
            "isDefaultGateway": true
          }
        },
        {
          "type": "portmap",
          "capabilities": {
            "portMappings": true
          }
        }
      ]
    }
  net-conf.json: |
    {
        {
      "Network": "20/16",
      "Backend": {
        "Type": "udp"  # 修改后端类型为 UDP
      }
    }
kind: ConfigMap
......
```

> 需要将 `Backend` 的类型更改为 `udp`，采用 UDP 模式时后端默认为端口为 8285，即 Flanneld 的监听端口。当采用 UDP 模式时，Flanneld 进程在启动时会通过打开 `/dev/net/tun` 的方式生成一个 `TUN` 设备，`TUN` 设备可以简单理解为 Linux 当中提供的一种内核网络与用户空间通信的一种机制，即应用可以通过直接读写 TUN 设备的方式收发 RAW IP 包。所以我们还需要将宿主机的 `/dev/net/tun` 文件挂载到容器中去：

```
$ kubectl edit ds kube-flannel-ds-amd64 -n kube-system
......
  volumeMounts:
    - mountPath: /run/flannel
    name: run
    - mountPath: /etc/kube-flannel/
    name: flannel-cfg
    - mountPath: /dev/net
    name: tun
......
volumes:
- hostPath:
    path: /run/flannel
    type: ""
  name: run
- hostPath:
    path: /etc/cni/net.d
    type: ""
  name: cni
- hostPath:
    path: /dev/net  # 挂载宿主机的 /dev/net/tun 文件
    type: ""
  name: tun
......
```

> 然后 Flanneld 的 Pod 会自动重建，重建完成后，可以随便查看一个 Pod 的日志：

```
$ kubectl logs -f kube-flannel-ds-amd64-5bk4dmd64 -n kube-system
I1128 08:26:663566       1 main.go:527] Using interface with name eth0 and address 111
I1128 08:26:663838       1 main.go:544] Defaulting external address to interface address (111)
I1128 08:26:857634       1 kube.go:126] Waiting 10m0s for node controller to sync
I1128 08:26:857805       1 kube.go:309] Starting kube subnet manager
I1128 08:26:858137       1 kube.go:133] Node controller sync successful
I1128 08:26:858324       1 main.go:244] Created subnet manager: Kubernetes Subnet Manager - ydzs-master
I1128 08:26:858357       1 main.go:247] Installing signal handlers
I1128 08:26:858933       1 main.go:386] Found network config - Backend type: udp
I1128 08:26:089114       1 main.go:317] Wrote subnet file to /run/flannel/subnet.env
I1128 08:26:089177       1 main.go:321] Running backend.
I1128 08:26:089227       1 main.go:339] Waiting for all goroutines to exit
I1128 08:26:089280       1 udp_network_amdgo:100] Watching for new subnet leases
I1128 08:26:089350       1 udp_network_amdgo:195] Subnet added: 20/24
I1128 08:26:089443       1 udp_network_amdgo:195] Subnet added: 20/24
I1128 08:26:089487       1 udp_network_amdgo:195] Subnet added: 20/24
I1128 08:26:089518       1 udp_network_amdgo:195] Subnet added: 20/24
I1128 08:27:936553       1 udp_network_amdgo:195] Subnet added: 20/24
I1128 08:27:341176       1 udp_network_amdgo:195] Subnet added: 20/24
I1128 08:27:136280       1 udp_network_amdgo:195] Subnet added: 20/24
I1128 08:27:863379       1 udp_network_amdgo:195] Subnet added: 20/24
```

> 看到

```
Found network config - Backend type: udp
```

这个信息证明我们现在已经变成了 `UDP` 模式了。

> Flanneld 进程启动后通过 `ip a` 命令可以发现节点当中已经多了一个叫 `flannel0` 的网络设备：

```
$ ip -d link show flannel0
210: flannel0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1472 qdisc pfifo_fast state UNKNOWN mode DEFAULT group default qlen 500
    link/none  promiscuity 0
    tun addrgenmode random numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535
```

> 由于是 UDP 的服务，所以我们需要通过 `netstat -ulnp` 命令查看进程：

```
$ netstat -ulnp | grep flanneld
udp        0      0 111:8285       0:*                           24844/flanneld
```

> 现在我们有两个 Pod，分别在节点 ydzs-node1 和节点 ydzs-node2 上面，现在让 pod-a（10.244.1.236）向 pod-b（10.244.2.123）发送一个请求报文（ping），我们来分析以下报文是如何从 pod-a 到达 pod-b 的：

```
$ kubectl get pods -o wide
NAME                        READY   STATUS    RESTARTS   AGE     IP             NODE         NOMINATED NODE   READINESS GATES
pod-a                       1/1     Running   0          73s     2236   ydzs-node1   <none>           <none>
pod-b                       1/1     Running   0          38s     2123   ydzs-node2   <none>           <none>
```

> 1、在 pod-a 当中发出 ICMP 请求报文，其源地址就是 10.244.1.236，目标地址是 10.244.2.123，此时通过 pod-a 内的路由表匹配到应该将该 IP 包发送到 ydzs-node1 节点上网关 10.244.1.1（`cni0`网桥）。

```
$ kubectl exec pod-a -- route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0         21      0         UG    0      0        0 eth0
20      21      220     UG    0      0        0 eth0
20      0         2220   U     0      0        0 eth0
```

> 在 ydzs-node1 节点上可以查看到 pod-a 中的网关 10.244.1.1 就是节点上的 `cni0` 网桥：

```
[root@ydzs-node1 ~]# ifconfig -a
cni0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        inet 21  netmask 2220  broadcast 0
        inet6 fe80::64c2:65ff:fe15:3669  prefixlen 64  scopeid 0x20<link>
        ether f6:f9:99:71:81:a2  txqueuelen 1000  (Ethernet)
        RX packets 33385207  bytes 24883992070 (1 GiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 31653673  bytes 19703556786 (3 GiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
......
```

> 2、这个时候，IP 包的下一个目的地，就取决于宿主机上面的路由规则了，Flanneld 进程已经在宿主机上面创建了一系列的路由规则，比如当前的 ydzs-node1 节点：

```
[root@ydzs-node1 ~]# ip route
default via 111 dev eth0  proto static  metric 100
10/24 dev eth0  proto kernel  scope link  src 122  metric 100
20/16 dev flannel0
20/24 dev cni0  proto kernel  scope link  src 21
10/16 dev docker0  proto kernel  scope link  src 11
```

> 此时到达 `cni0` 的 IP 包目标地址 10.244.2.123 匹配不到本机的 `cni0` 网桥对应的 `10.244.1.0/24` 网段，只能匹配到第三条 `10.244.0.0/16` 对应的这条路由规则，这个时候内核将 RAW IP 包发送给 `flannel0` 设备。

> `flannel0` 设备它是一个 `TUN` 设备（Tunnel 设备）。在 Linux 中，TUN 设备是一种工作在三层（Network Layer）的虚拟网络设备，TUN 设备的功能非常简单，即：`在操作系统内核和用户应用程序之间传递 IP 包`。

> 3、由于 `flannel0` 是一个 `TUN` 设备，发送给 `flannel0` 接口的 `RAW IP` 包将被 Flanneld 进程接收到，然后在原有的基础上进行 UDP 封包，然后发送到 ydzs-node2 节点上的 `Flanneld` 进行解包。

> 封包

> UDP 封包的形式为：10.151.30.22:src port -> 10.151.30.23:8285。

> 这里最关键的就是 UDP 封包发送到我们的目标 IP 10.244.2.123 这个容器所在的节点，但是是如何知道这个节点的呢？

> 这个就需要了解一个非常重要的概念*_子网（Subnet）_，Flannel 管理的容器网络，一台宿主机上的所有容器，都属于该宿主机被分配的一个子网，比如我们这里的 ydzs-node1 节点的子网是 `10.244.1.0/24`（10.244.1.1-10.244.1.254），pod-a 的 IP 地址是 `10.244.1.236`；ydzs-node2 节点的子网是 `10.244.2.0/24`（10.244.2.1-10.244.2.254），pod-b 的 IP 地址是 `10.244.2.123`，这些子网信息是当 Flanneld 进程在启动时通过 api-server 保存到 etcd 当中，所以在发送报文时可以通过目的地址 `10.244.2.123` 匹配到对应的子网是 `10.244.2.0/24`，这个时候查询 etcd 得到这个子网对应的宿主机的 IP 地址 10.151.30.23，也就是 ydzs-node2 节点。

> 4、ydzs-node2 节点收到 UDP 报文过后经过 Linux 内核通过 UDP 端口 8285 将包交给节点上的 Flanneld 进程。

> 5、然后 ydzs-node2 节点上的 Flanneld 进程将接收到的 UDP 包解包后得到 RAW IP 包：

```
2236 -> 2123
```

。

> 6、解包后的 RAW IP 包匹配到 ydzs-node2 节点上的路由规则（10.244.2.0/24），内核将 RAW IP 包发送给 `cni0` 设备

```
[root@ydzs-node2 ~]# ip route
default via 111 dev eth0  proto static  metric 100
10/24 dev eth0  proto kernel  scope link  src 123  metric 100
20/16 dev flannel0
20/24 dev cni0  proto kernel  scope link  src 21
10/16 dev docker0  proto kernel  scope link  src 11
```

> 7、`cni0` 将 IP 包转发给连接在 `cni0` 网桥上的 pod-b，这样就完成了这个通信过程：

```
$ kubectl exec pod-a ping 2123
PING 2123 (2123): 56 data bytes
64 bytes from 2123: seq=0 ttl=62 time=452 ms
64 bytes from 2123: seq=1 ttl=62 time=160 ms
64 bytes from 2123: seq=2 ttl=62 time=853 ms
```

> 上面就是基于 Flannel UDP 模式的跨主通信的基本流程了，从上面的整个流程来看，`Flanneld` 主要有两方面的功能：

*   UDP 封包解包
*   节点上的路由表的动态更新

> 我们可以明显看出来数据包是通过 tun 设备从内核态复制到用户态的应用中的，然后再通过用户态复制到内核态，仅一次网络传输就进行了两次用户态和内核态的切换，显然这种效率是不会很高的，由于低效率所以这种方式基本上不是呀，要提高效率最简单的方式就是把封包解包这些事情都交给内核去干好了，事实上 Linux 内核本身也提供了比较成熟的网络封包解包（隧道传输）实现方案 `VXLAN`，Flanneld 也实现了基于 `VXLAN` 的方案，该方案在我们日常使用的时候也是最普遍的。

### VXLAN 方式 

> `VXLAN`，即 Virtual Extensible LAN（虚拟可扩展局域网），是 Linux 内核本身就支持的一种网络虚似化技术。所以说，VXLAN 可以完全在内核态实现上述封装和解封装的工作，从而通过与前面相似的隧道机制，构建出覆盖网络（Overlay Network）。

> 同样的当我们使用 `VXLAN` 模式的时候需要将 Flanneld 的 Backend 类型修改为 `vxlan`：

```
$ kubectl edit cm kube-flannel-cfg -n kube-system
apiVersion: v1
data:
  cni-conf.json: |
    {
      "cniVersion": "0",
      "name": "cbr0",
      "plugins": [
        {
          "type": "flannel",
          "delegate": {
            "hairpinMode": true,
            "isDefaultGateway": true
          }
        },
        {
          "type": "portmap",
          "capabilities": {
            "portMappings": true
          }
        }
      ]
    }
  net-conf.json: |
    {
        {
      "Network": "20/16",
      "Backend": {
        "Type": "vxlan"  # 修改后端类型为 vxlan
      }
    }
kind: ConfigMap
......
```

> 将类型修改为 `vxlan` 过后，需要重建下 Flanneld 的所有 Pod 才能生效：

```
$ kubectl delete pod -n kube-system -l app=flannel
```

> 重建完成后同样可以随便查看一个 Pod 的日志，出现如下

```
Found network config - Backend type: vxlan
```

的日志信息就证明已经配置成功了：

```
$ kubectl logs -f kube-flannel-ds-amd64-pfb8h -n kube-system
I1129 03:33:588549       1 main.go:527] Using interface with name eth0 and address 123
I1129 03:33:588893       1 main.go:544] Defaulting external address to interface address (123)
I1129 03:33:698562       1 kube.go:126] Waiting 10m0s for node controller to sync
I1129 03:33:698726       1 kube.go:309] Starting kube subnet manager
I1129 03:33:698910       1 kube.go:133] Node controller sync successful
I1129 03:33:698980       1 main.go:244] Created subnet manager: Kubernetes Subnet Manager - ydzs-node2
I1129 03:33:699000       1 main.go:247] Installing signal handlers
I1129 03:33:699375       1 main.go:386] Found network config - Backend type: vxlan
I1129 03:33:699553       1 vxlan.go:120] VXLAN config: VNI=1 Port=0 GBP=false DirectRouting=false
I1129 03:33:703715       1 main.go:317] Wrote subnet file to /run/flannel/subnet.env
I1129 03:33:703785       1 main.go:321] Running backend.
I1129 03:33:703825       1 main.go:339] Waiting for all goroutines to exit
I1129 03:33:703940       1 vxlan_network.go:60] watching for new subnet leases
```

> Flanneld 在启动时会通过 `Netlink` 机制与 Linux 内核通信，建立一个 

```
VTEP（Virtual Tunnel Access End Point）
```

 设备 `flannel.1`（命名规则为`flannel.[VNI]`，VNI 默认为1），类似于交换机当中的一个网口。我们可以通过 `ip -d link` 命令查看 `VTEP` 设备 `flannel.1` 的配置信息：

```
$ ip -d link show flannel.1
6: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN mode DEFAULT
    link/ether 72:f7:e9:40:97:1e brd ff:ff:ff:ff:ff:ff promiscuity 0
    vxlan id 1 local 122 dev eth0 srcport 0 0 dstport 8472 nolearning ageing 300 addrgenmode eui64
```

> 从上面输出可以看到，`VTEP` 的 local IP 为 10.151.30.22，destination port 为 `8472`。同样我们也可以在节点上查看进程监听情况：

```
$ netstat -ulnp | grep 8472
udp        0      0 0:8472            0:*                           -
```

> 我们仔细看和 UDP 模式下查看 Flanneld 监听的端口是有区别的，最后一栏显示的不是进程的 ID 和名称，而是一个破折号`-`，这说明 UDP 的8472端口不是由用户态的进程在监听的，也证实了`VXLAN`模块工作在内核态模式下。

> 在 UDP 模式下由 Flanneld 进程进行网络包的封包和解包的工作，而在 `VXLAN` 模式下解封包的事情交由内核处理，下面我们来看下 Flanneld 后端的具体工具流程。当 Flanneld 启动时将创建 `VTEP` 设备 `flannel.1`，并将 `VTEP` 设备的相关信息上报到 etcd 当中，而当在 Flannel 网络中有新的节点发现时，各个节点上的 Flanneld 进程将依次执行以下流程：

> 1、在节点当中创建一条该节点所属网段的路由表，主要是能让 Pod 当中的流量路由到 `flannel.1` 接口。通过`route -n`可以查看到节点当中已经有四条 `flannel.1` 接口的路由：

```
$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0         111    0         UG    100    0        0 eth0
10     0         2220   U     100    0        0 eth0
20      20      2220   UG    0      0        0 flannel.1
20      0         2220   U     0      0        0 cni0
20      20      2220   UG    0      0        0 flannel.1
20      20      2220   UG    0      0        0 flannel.1
20      20      2220   UG    0      0        0 flannel.1
10      0         220     U     0      0        0 docker0
```

> 比如 `10.244.2.0` 这条路由规则，他的意思就是发往 `10.244.2.0/24` 网段的 IP 包，都需要经过 `flannel.1` 设备发出，而且最后被发送到的网关地址是 `10.244.2.0`。而其实这个网关地址就是 ydzs-node2 节点上的 VTEP 设备（也就是 flannel.1）的 IP 地址：

```
[root@ydzs-node2 ~]# ifconfig
flannel.1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        inet 20  netmask 222255  broadcast 0
        inet6 fe80::2467:52ff:fe6b:1bf9  prefixlen 64  scopeid 0x20<link>
        ether 26:67:52:6b:1b:f9  txqueuelen 0  (Ethernet)
        RX packets 42050890  bytes 27691009839 (7 GiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 36287976  bytes 46894346975 (6 GiB)
        TX errors 0  dropped 163 overruns 0  carrier 0  collisions 0
......
```

> 2、上面知道了目的 VTEP 设备的 IP 地址了，这个时候就需要知道目的 MAC 地址，才能把数据包发送过去，这个时候其实 Flanneld 进程就会在节点当中维护所有节点的 IP 以及 `VTEP` 设备的静态 `ARP` 缓存。可通过 `arp -n` 命令查看到当前节点当中已经缓存了另外四个节点以及 `VTEP` 的 ARP 信息。

```
$ arp -n
Address                  HWtype  HWaddress           Flags Mask            Iface
20               ether   32:f1:e0:a9:97:ab   CM                    flannel.1
20               ether   26:67:52:6b:1b:f9   CM                    flannel.1
20               ether   0a:24:5e:40:ff:da   CM                    flannel.1
20               ether   4a:09:1f:42:ed:c1   CM                    flannel.1
......
```

> 这里我们可以看到 IP 地址 `10.244.2.0` 对应的 MAC 地址是 `26:67:52:6b:1b:f9`，这样我们就知道了目的 VTEP 设备的 MAC 地址。但是现在我们只是知道了目标设备的 MAC 地址，却不知道对应的宿主机的地址是什么？

> 3、这个时候 Flanneld 进程还会在节点当中添加一条该节点的转发表，通过 `bridge` 命令查看节点上的 VXLAN 转发表（FDB entry），MAC 为 `VTEP` 设备即 `flannel.1` 的 MAC 地址，IP 为 VTEP 对应的对外 IP（可通过 Flanneld 的启动参数 `--iface=eth0` 指定，若不指定则按默认网关查找网络接口对应的 IP），可以看到已经有四条转发表。

```
$ bridge fdb show dev flannel.1
32:f1:e0:a9:97:ab dst 157 self permanent
26:67:52:6b:1b:f9 dst 123 self permanent
4a:09:1f:42:ed:c1 dst 159 self permanent
0a:24:5e:40:ff:da dst 111 self permanent
```

> 这样我们就找到了上面目的 VTEP 设备的 MAC 地址对应的 IP 地址为 10.151.30.23 的主机，这就是我们的 ydzs-node2 节点，所以我们就找到了要发往的目的地址。

> 这个时候容器跨节点网络通信实现的完整流程为：

*   和 UDP 模式一样，pod-a（10.244.1.236）当中的 IP 包通过 pod-a 内的路由表被发送到 `cni0`
*   到达 `cni0` 当中的 IP 包通过匹配节点 ydzs-node1 当中的路由表发现通往 10.244.2.13 的 IP 包应该交给 `flannel.1` 接口
*   `flannel.1` 作为一个 VTEP 设备，收到报文后将按照 `VTEP` 的配置进行封包，通过 ydzs-node1 节点上的 arp 和转发表得知 10.244.2.123 属于节点 ydzs-node2，并且会将 ydzs-node2 节点对应的 VTEP 设备的 MAC 地址，根据 `flannel.1` 设备创建时的设置的参数（VNI、local IP、Port）进行 VXLAN 封包
*   通过节点 ydzs-node2 跟 ydzs-node1 之间的网络连接，VXLAN 包到达 ydzs-node2 的 eth0 接口
*   通过端口 8472，VXLAN 包被转发给 VTEP 设备 `flannel.1` 进行解包
*   解封装后的 IP 包匹配节点 ydzs-node2 当中的路由表（10.244.2.0），内核将 IP 包转发给`cni0`
*   `cni0`将 IP 包转发给连接在 `cni0` 上的 pod-b

### host-gw 

> `host-gw` 即 Host Gateway，从名字中就可以想到这种方式是通过把主机当作网关来实现跨节点网络通信的。那么具体如何实现跨节点通信呢？

> 同理 UDP 模式和 VXLAN 模式，首先将 Backend 中的 type 改为`host-gw`，这里就不再赘述，更新完成后，随便查看一个 flannel 的 Pod 日志，如果出现如下所示的 

```
Found network config - Backend type: host-gw
```

 日志就证明已经是 `host-gw` 模式了：

```
$ kubectl logs -f kube-flannel-ds-amd64-642l6 -n kube-system
I1129 04:53:309992       1 main.go:527] Using interface with name eth0 and address 159
I1129 04:53:310292       1 main.go:544] Defaulting external address to interface address (159)
I1129 04:53:510618       1 kube.go:126] Waiting 10m0s for node controller to sync
I1129 04:53:607860       1 kube.go:309] Starting kube subnet manager
I1129 04:53:511530       1 kube.go:133] Node controller sync successful
I1129 04:53:511624       1 main.go:244] Created subnet manager: Kubernetes Subnet Manager - ydzs-node4
I1129 04:53:511646       1 main.go:247] Installing signal handlers
I1129 04:53:511899       1 main.go:386] Found network config - Backend type: host-gw
I1129 04:53:611040       1 main.go:317] Wrote subnet file to /run/flannel/subnet.env
I1129 04:53:611120       1 main.go:321] Running backend.
I1129 04:53:611165       1 main.go:339] Waiting for all goroutines to exit
I1129 04:53:611280       1 route_network.go:53] Watching for new subnet leases
I1129 04:53:611768       1 route_network.go:85] Subnet added: 20/24 via 111
W1129 04:53:612268       1 route_network.go:102] Replacing existing route to 20/24 via 20 dev index 6 with 20/24 via 111 dev index 
I1129 04:53:612705       1 route_network.go:85] Subnet added: 20/24 via 122
W1129 04:53:612777       1 route_network.go:88] Ignoring non-host-gw subnet: type=vxlan
I1129 04:53:612829       1 route_network.go:85] Subnet added: 20/24 via 123
W1129 04:53:612868       1 route_network.go:88] Ignoring non-host-gw subnet: type=vxlan
I1129 04:53:612935       1 route_network.go:85] Subnet added: 20/24 via 157
W1129 04:53:612995       1 route_network.go:88] Ignoring non-host-gw subnet: type=vxlan
I1129 04:53:808989       1 route_network.go:85] Subnet added: 20/24 via 157
W1129 04:53:809279       1 route_network.go:102] Replacing existing route to 20/24 via 20 dev index 6 with 20/24 via 157 dev index 
I1129 04:53:012832       1 route_network.go:85] Subnet added: 20/24 via 122
W1129 04:53:013132       1 route_network.go:102] Replacing existing route to 20/24 via 20 dev index 6 with 20/24 via 122 dev index 
I1129 04:53:713684       1 route_network.go:85] Subnet added: 20/24 via 123
W1129 04:53:714022       1 route_network.go:102] Replacing existing route to 20/24 via 20 dev index 6 with 20/24 via 123 dev index
```

> 采用 `host-gw` 模式后 Flanneld 的唯一作用就是`负责主机上路由表的动态更新`，其实就是将每个 Flannel 子网（Flannel Subnet，比如：10.244.1.0/24）的`下一跳`，设置成了该子网对应的宿主机的 IP 地址，当然，Flannel 子网和主机的信息，都是保存在 etcd 当中的。Fanneld 只需要 WACTH 这些数据的变化，然后实时更新路由表即可。主要流程如下所示：

> 1、同 UDP、VXLAN 模式一致，通过容器A 的路由表 IP 包到达`cni0` 2、到达 `cni0` 的 IP 包匹配到 ydzs-node1 当中的路由规则（10.244.2.0），并且网关为 10.151.30.23，即节点 ydzs-node2，所以内核将 IP 包发送给节点 ydzs-node2（10.151.30.23）:

```
$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0         111    0         UG    100    0        0 eth0
10     0         2220   U     100    0        0 eth0
20      111    2220   UG    0      0        0 eth0
20      0         2220   U     0      0        0 cni0
20      123    2220   UG    0      0        0 eth0
20      157    2220   UG    0      0        0 eth0
20      159    2220   UG    0      0        0 eth0
10      0         220     U     0      0        0 docker0
```

> 3、IP 包通过物理网络到达节点的 ydzs-node2 的 eth0 设备 4、到达 ydzs-node2 节点 eth0 的 IP 包匹配到节点当中的路由表（10.244.2.0/24），IP 包被转发给 `cni0` 设备 5、`cni0` 将 IP 包转发给连接在 `cni0` 上的 pod-b

> 这样就完成了整个跨主机通信流程，这个流程可能是最简单最容器理解的模式了，而且容器通信的过程还免除了额外的封包和解包带来的性能损耗，所以理论上性能肯定要更好，但是该模式是通过节点上的`路由表`来实现各个节点之间的跨节点网络通信，那么就得保证两个节点是可以`直接路由`过去的。按照内核当中的路由规则，网关必须在跟主机当中至少一个 IP 处于同一网段，故造成的结果就是采用`host-gw` 这种模式的时候，集群中所有的节点必须处于同一个网络当中，这对于集群规模比较大时需要对节点进行网段划分的话会存在一定的局限性，另外一个则是随着集群当中节点规模的增大，Flanneld 需要维护主机上成千上万条路由表的动态更新也是一个不小的压力。

> 除了 Flannel 之外，`Calico` 这种网络插件和 Flannel 的 `host-gw` 模式基本上是一样的，都是在每台书主机上面添加子网和对应的书主机的 IP 地址为网关这样的路由信息，不过，不同于 Flannel 通过 Etcd 和宿主机上的 flanneld 来维护路由信息的做法，Calico 使用 `bgp` 来自动地在整个集群中分发路由信息。