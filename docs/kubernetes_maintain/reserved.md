
## 资源预留

> Kubernetes 的节点可以按照节点的资源容量进行调度，默认情况下 Pod 能够使用节点全部可用容量。这样就会造成一个问题，因为节点自己通常运行了不少驱动 OS 和 Kubernetes 的系统守护进程。除非为这些系统守护进程留出资源，否则它们将与 Pod 争夺资源并导致节点资源短缺问题。

> 当我们在线上使用 Kubernetes 集群的时候，如果没有对节点配置正确的资源预留，我们可以考虑一个场景，由于某个应用无限制的使用节点的 CPU 资源，导致节点上 CPU 使用持续100%运行，而且压榨到了 kubelet 组件的 CPU 使用，这样就会导致 kubelet 和 apiserver 的心跳出问题，节点就会出现 Not Ready 状况了。默认情况下节点 Not Ready 过后，5分钟后会驱逐应用到其他节点，当这个应用跑到其他节点上的时候同样100%的使用 CPU，是不是也会把这个节点搞挂掉，同样的情况继续下去，也就导致了整个集群的`雪崩`，集群内的节点一个一个的 Not Ready 了，后果是非常严重的，或多或少的人遇到过 Kubernetes 集群雪崩的情况，这个问题也是面试的时候镜像询问的问题。

> 要解决这个问题就需要为 Kubernetes 集群配置资源预留，kubelet 暴露了一个名为 `Node Allocatable` 的特性，有助于为系统守护进程预留计算资源，Kubernetes 也是推荐集群管理员按照每个节点上的工作负载来配置 Node Allocatable。

> > 本文的操作环境为 Kubernetes V1.17.11 版本，Docker 和 Kubelet 采用的 cgroup 驱动为 `systemd`。

### Node Allocatable

> Kubernetes 节点上的 Allocatable 被定义为 Pod 可用计算资源量，调度器不会超额申请 Allocatable,目前支持 CPU, memory 和 ephemeral-storage 这几个参数。

> 我们可以通过 

```
kubectl describe node
```

 命令查看节点可分配资源的数据：

```
$ kubectl describe node ydzs-node4
......
Capacity:
  cpu:                4
  ephemeral-storage:  17921Mi
  hugepages-2Mi:      0
  memory:             8008820Ki
  pods:               110
Allocatable:
  cpu:                4
  ephemeral-storage:  16912377419
  hugepages-2Mi:      0
  memory:             7906420Ki
  pods:               110
......
```

> 可以看到其中有 `Capacity` 与 `Allocatable` 两项内容，其中的 `Allocatable` 就是节点可被分配的资源，我们这里没有配置资源预留，所以默认情况下 `Capacity` 与 `Allocatable` 的值基本上是一致的。下图显示了可分配资源和资源预留之间的关系：

> ![Node Allocatable](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200811141848.png)

*   Kubelet Node Allocatable 用来为 Kube 组件和 System 进程预留资源，从而保证当节点出现满负荷时也能保证 Kube 和 System 进程有足够的资源。
*   目前支持 cpu, memory, ephemeral-storage 三种资源预留。
*   Node Capacity 是节点的所有硬件资源，`kube-reserved` 是给 kube 组件预留的资源，`system-reserved` 是给系统进程预留的资源，`eviction-threshold` 是 kubelet 驱逐的阈值设定，allocatable 才是真正调度器调度 Pod 时的参考值（保证节点上所有 Pods 的 request 资源不超过Allocatable）。

> 节点可分配资源的计算方式为：

```
Node Allocatable Resource = Node Capacity - Kube-reserved - system-reserved - eviction-threshold
```

### 配置资源预留

### Kube 预留值

> 首先我们来配置 Kube 预留值，kube-reserved 是为了给诸如 kubelet、容器运行时、node problem detector 等 kubernetes 系统守护进程争取资源预留。要配置 Kube 预留，需要把 kubelet 的 

```
--kube-reserved-cgroup
```

 标志的值设置为 kube 守护进程的父控制组。

> 不过需要注意，如果 

```
--kube-reserved-cgroup
```

 不存在，Kubelet 不会创建它，启动 Kubelet 将会失败。

> 比如我们这里修改 node-ydzs4 节点的 Kube 资源预留，我们可以直接修改 

```
/var/lib/kubelet/config.yaml
```

 文件来动态配置 kubelet，添加如下所示的资源预留配置：

```
apiVersion: kubelet.config.k8s.io/v1beta1
......
enforceNodeAllocatable:
- pods
- kube-reserved  # 开启 kube 资源预留
kubeReserved:
  cpu: 500m
  memory: 1Gi
  ephemeral-storage: 1Gi
kubeReservedCgroup: /kubelet.slice  # 指定 kube 资源预留的 cgroup
```

> 修改完成后，重启 kubelet，如果没有创建上面的 kubelet 的 cgroup，启动会失败：

```
$ systemctl restart kubelet
$ journalctl -u kubelet -f
......
Aug 11 15:04:13 ydzs-node4 kubelet[28843]: F0811 15:04:653476   28843 kubelet.go:1380] Failed to start ContainerManager Failed to enforce Kube Reserved Cgroup Limits on "/kubelet.slice": ["kubelet"] cgroup does not exist
```

> 上面的提示信息很明显，我们指定的 kubelet 这个 cgroup 不存在，但是由于子系统较多，具体是哪一个子系统不存在不好定位，我们可以将 kubelet 的日志级别调整为 `v=4`，就可以看到具体丢失的 cgroup 路径：

```
$ vi /var/lib/kubelet/kubeadm-flags.env
KUBELET_KUBEADM_ARGS="--v=4 --cgroup-driver=systemd --network-plugin=cni"
```

> 然后再次重启 kubelet：

```
$ systemctl daemon-reload
$ systemctl restart kubelet
```

> 再次查看 kubelet 日志：

```
$ journalctl -u kubelet -f
......
Sep 09 17:57:36 ydzs-node4 kubelet[20427]: I0909 17:57:382811   20427 cgroup_manager_linux.go:273] The Cgroup [kubelet] has some missing paths: [/sys/fs/cgroup/cpu,cpuacct/kubelet.slice /sys/fs/cgroup/memory/kubelet.slice /sys/fs/cgroup/systemd/kubelet.slice /sys/fs/cgroup/pids/kubelet.slice /sys/fs/cgroup/cpu,cpuacct/kubelet.slice /sys/fs/cgroup/cpuset/kubelet.slice]
Sep 09 17:57:36 ydzs-node4 kubelet[20427]: I0909 17:57:383002   20427 factory.go:170] Factory "systemd" can handle container "/system.slice/run-docker-netns-db100461211c.mount", but ignoring.
Sep 09 17:57:36 ydzs-node4 kubelet[20427]: I0909 17:57:383025   20427 manager.go:908] ignoring container "/system.slice/run-docker-netns-db100461211c.mount"
Sep 09 17:57:36 ydzs-node4 kubelet[20427]: F0909 17:57:383046   20427 kubelet.go:1381] Failed to start ContainerManager Failed to enforce Kube Reserved Cgroup Limits on "/kubelet.slice": ["kubelet"] cgroup does not exist
```

> > `注意`：systemd 的 cgroup 驱动对应的 cgroup 名称是以 `.slice` 结尾的，比如如果你把 cgroup 名称`配置成`kubelet.service
> 
> ```
> ，那么对应的创建的 cgroup 名称应该为
> ```
> 
> kubelet.service.slice\`。如果你配置的是 cgroupfs 的驱动，则用配置的值即可。无论哪种方式，通过查看错误日志都是排查问题最好的方式。

> 现在可以看到具体的 cgroup 不存在的路径信息了：

```
The Cgroup [kubelet] has some missing paths: [/sys/fs/cgroup/cpu,cpuacct/kubelet.slice /sys/fs/cgroup/memory/kubelet.slice /sys/fs/cgroup/systemd/kubelet.slice /sys/fs/cgroup/pids/kubelet.slice /sys/fs/cgroup/cpu,cpuacct/kubelet.slice /sys/fs/cgroup/cpuset/kubelet.slice]
```

> 所以要解决这个问题也很简单，我们只需要创建上面的几个路径即可：

```
$ mkdir -p /sys/fs/cgroup/cpu,cpuacct/kubelet.slice
$ mkdir -p /sys/fs/cgroup/memory/kubelet.slice
$ mkdir -p /sys/fs/cgroup/systemd/kubelet.slice
$ mkdir -p /sys/fs/cgroup/pids/kubelet.slice
$ mkdir -p /sys/fs/cgroup/cpu,cpuacct/kubelet.slice
$ mkdir -p /sys/fs/cgroup/cpuset/kubelet.slice
```

> 创建完成后，再次重启：

```
$ systemctl restart kubelet
$ journalctl -u kubelet -f
......
Sep 09 17:59:41 ydzs-node4 kubelet[21462]: F0909 17:59:291957   21462 kubelet.go:1381] Failed to start ContainerManager Failed to enforce Kube Reserved Cgroup Limits on "/kubelet.slice": failed to set supported cgroup subsystems for cgroup [kubelet]: failed to set config for supported subsystems : failed to write 0 to hugetlb.2MB.limit_in_bytes: open /sys/fs/cgroup/hugetlb/kubelet.slice/hugetlb.2MB.limit_in_bytes: no such file or directory
```

> 可以看到还有一个 `hugetlb` 的 cgroup 路径不存在，所以继续创建这个路径：

```
$ mkdir -p /sys/fs/cgroup/hugetlb/kubelet.slice
$ systemctl restart kubelet
```

> 重启完成后就可以正常启动了，启动完成后我们可以通过查看 cgroup 里面的限制信息校验是否配置成功，比如我们查看内存的限制信息：

```
$ cat /sys/fs/cgroup/memory/kubelet.slice/memory.limit_in_bytes
1073741824  # 1Gi
```

> 现在再次查看节点的信息：

```
$ kubectl describe node ydzs-node4
......
Addresses:
  InternalIP:  159
  Hostname:    ydzs-node4
Capacity:
  cpu:                4
  ephemeral-storage:  17921Mi
  hugepages-2Mi:      0
  memory:             8008820Ki
  pods:               110
Allocatable:
  cpu:                3500m
  ephemeral-storage:  15838635595
  hugepages-2Mi:      0
  memory:             6857844Ki
  pods:               110
......
```

> 可以看到可以分配的 Allocatable 值就变成了 Kube 预留过后的值了，证明我们的 Kube 预留成功了。

### 系统预留值

> 我们也可以用同样的方式为系统配置预留值，`system-reserved` 用于为诸如 sshd、udev 等系统守护进程争取资源预留，system-reserved 也应该为 kernel 预留 内存，因为目前 kernel 使用的内存并不记在 Kubernetes 的 pod 上。但是在执行 `system-reserved` 预留操作时请加倍小心，因为它可能导致节点上的关键系统服务 CPU 资源短缺或因为内存不足而被终止，所以如果不是自己非常清楚如何配置，可以不用配置系统预留值。

> 同样通过 kubelet 的参数 `--system-reserved` 配置系统预留值，但是也需要配置 

```
--system-reserved-cgroup
```

 参数为系统进程设置 cgroup。

> > 请注意，如果 
> 
> ```
> --system-reserved-cgroup
> ```
> 
>  不存在，Kubelet 不会创建它，kubelet 会启动失败。

### 驱逐阈值

> 上面我们还提到可分配的资源还和 kubelet 驱逐的阈值有关。节点级别的内存压力将导致系统内存不足，这将影响到整个节点及其上运行的所有 Pod，节点可以暂时离线直到内存已经回收为止，我们可以通过配置 kubelet 驱逐阈值来防止系统内存不足。驱逐操作只支持内存和 ephemeral-storage 两种不可压缩资源。当出现内存不足时，调度器不会调度新的 Best-Effort QoS Pods 到此节点，当出现磁盘压力时，调度器不会调度任何新 Pods 到此节点。

> 我们这里为 ydzs-node4 节点配置如下所示的硬驱逐阈值：

```
# /var/lib/kubelet/config.yaml
......
evictionHard:  # 配置硬驱逐阈值
  memory.available: "300Mi"
  nodefs.available: "10%"
enforceNodeAllocatable:
- pods
- kube-reserved
kubeReserved:
  cpu: 500m
  memory: 1Gi
  ephemeral-storage: 1Gi
kubeReservedCgroup: /kubelet.slice
......
```

> 我们通过 `--eviction-hard` 预留一些内存后，当节点上的可用内存降至保留值以下时，kubelet 将尝试驱逐 Pod，

```
$ kubectl describe node ydzs-node4
......
Addresses:
  InternalIP:  159
  Hostname:    ydzs-node4
Capacity:
  cpu:                4
  ephemeral-storage:  17921Mi
  hugepages-2Mi:      0
  memory:             8008820Ki
  pods:               110
Allocatable:
  cpu:                3500m
  ephemeral-storage:  15838635595
  hugepages-2Mi:      0
  memory:             6653044Ki
  pods:               110
......
```

> 配置生效后再次查看节点可分配的资源可以看到内存减少了，临时存储没有变化是因为硬驱逐的默认值就是 10%。也是符合可分配资源的计算公式的：

```
Node Allocatable Resource = Node Capacity - Kube-reserved - system-reserved - eviction-threshold
```

> 到这里我们就完成了 Kubernetes 资源预留的配置。