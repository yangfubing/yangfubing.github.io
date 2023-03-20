
## Local 存储 

> 前面我们有通过 `hostPath` 或者 `emptyDir` 的方式来持久化我们的数据，但是显然我们还需要更加可靠的存储来保存应用的持久化数据，这样容器在重建后，依然可以使用之前的数据。但是存储资源和 CPU 资源以及内存资源有很大不同，为了屏蔽底层的技术实现细节，让用户更加方便的使用，Kubernetes 便引入了 `PV` 和 `PVC` 两个重要的资源对象来实现对存储的管理。

### 概念 

> `PV` 的全称是：`PersistentVolume`（持久化卷），是对底层共享存储的一种抽象，PV 由管理员进行创建和配置，它和具体的底层的共享存储技术的实现方式有关，比如 `Ceph`、`GlusterFS`、`NFS`、`hostPath` 等，都是通过插件机制完成与共享存储的对接。

> `PVC` 的全称是：

```
PersistentVolumeClaim
```

（持久化卷声明），PVC 是用户存储的一种声明，PVC 和 Pod 比较类似，Pod 消耗的是节点，PVC 消耗的是 PV 资源，Pod 可以请求 CPU 和内存，而 PVC 可以请求特定的存储空间和访问模式。对于真正使用存储的用户不需要关心底层的存储实现细节，只需要直接使用 PVC 即可。

> 但是通过 PVC 请求到一定的存储空间也很有可能不足以满足应用对于存储设备的各种需求，而且不同的应用程序对于存储性能的要求可能也不尽相同，比如读写速度、并发性能等，为了解决这一问题，Kubernetes 又为我们引入了一个新的资源对象：`StorageClass`，通过 `StorageClass` 的定义，管理员可以将存储资源定义为某种类型的资源，比如快速存储、慢速存储等，用户根据 StorageClass 的描述就可以非常直观的知道各种存储资源的具体特性了，这样就可以根据应用的特性去申请合适的存储资源了，此外 `StorageClass` 还可以为我们自动生成 PV，免去了每次手动创建的麻烦。

### hostPath 

> 我们上面提到了 PV 是对底层存储技术的一种抽象，PV 一般都是由管理员来创建和配置的，我们首先来创建一个 `hostPath` 类型的 `PersistentVolume`。Kubernetes 支持 hostPath 类型的 PersistentVolume 使用节点上的文件或目录来模拟附带网络的存储，但是需要注意的是在生产集群中，我们不会使用 hostPath，集群管理员会提供网络存储资源，比如 NFS 共享卷或 Ceph 存储卷，集群管理员还可以使用 `StorageClasses` 来设置动态提供存储。因为 Pod 并不是始终固定在某个节点上面的，所以要使用 hostPath 的话我们就需要将 Pod 固定在某个节点上，这样显然就大大降低了应用的容错性。

> 比如我们这里将测试的应用固定在节点 ydzs-node1 上面，首先在该节点上面创建一个 

```
/data/k8s/test/hostpath
```

 的目录，然后在该目录中创建一个 `index.html` 的文件：

```
$ echo 'Hello from Kubernetes hostpath storage' > /data/k8s/test/hostpath/index.html
```

> 然后接下来创建一个 hostPath 类型的 PV 资源对象：（pv-hostpath.yaml）

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-hostpath
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/data/k8s/test/hostpath"
```

> 配置文件中指定了该卷位于集群节点上的 

```
/data/k8s/test/hostpath
```

 目录，还指定了 10G 大小的空间和 `ReadWriteOnce` 的访问模式，这意味着该卷可以在单个节点上以读写方式挂载，另外还定义了名称为 `manual` 的 `StorageClass`，该名称用来将 

```
PersistentVolumeClaim
```

 请求绑定到该 `PersistentVolum`。下面是关于 PV 的这些配置属性的一些说明：

*   > Capacity（存储能力）：一般来说，一个 PV 对象都要指定一个存储能力，通过 PV 的 `capacity` 属性来设置的，目前只支持存储空间的设置，就是我们这里的 `storage=10Gi`，不过未来可能会加入 `IOPS`、吞吐量等指标的配置。

*   > AccessModes（访问模式）：用来对 PV 进行访问模式的设置，用于描述用户应用对存储资源的访问权限，访问权限包括下面几种方式：

    *   ReadWriteOnce（RWO）：读写权限，但是只能被单个节点挂载
    *   ReadOnlyMany（ROX）：只读权限，可以被多个节点挂载
    *   ReadWriteMany（RWX）：读写权限，可以被多个节点挂载

    > 注意

    > 一些 PV 可能支持多种访问模式，但是在挂载的时候只能使用一种访问模式，多种访问模式是不会生效的。

    > 下图是一些常用的 Volume 插件支持的访问模式： ![pv access modes](../assets/img/kubernetes_stroage/pv-access-modes.png)

> 直接创建上面的资源对象：

```
$ kubectl apply -f pv-hostpath.yaml
persistentvolume/pv-hostpath created
```

> 创建完成后查看 PersistentVolume 的信息，输出结果显示该 `PersistentVolume` 的状态（STATUS） 为 `Available`。 这意味着它还没有被绑定给 

```
PersistentVolumeClaim
```

：

```
$ kubectl get pv pv-hostpath
NAME          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
pv-hostpath   10Gi       RWO            Retain           Available           manual                  58s
```

> 其中有一项 `RECLAIM POLICY` 的配置，同样我们可以通过 PV 的 

```
persistentVolumeReclaimPolicy
```

（回收策略）属性来进行配置，目前 PV 支持的策略有三种：

*   Retain（保留）：保留数据，需要管理员手工清理数据
*   Recycle（回收）：清除 PV 中的数据，效果相当于执行 `rm -rf /thevoluem/*`
*   Delete（删除）：与 PV 相连的后端存储完成 volume 的删除操作，当然这常见于云服务商的存储服务，比如 ASW EBS。

> 不过需要注意的是，目前只有 `NFS` 和 `HostPath` 两种类型支持回收策略，当然一般来说还是设置为 `Retain` 这种策略保险一点。

> 注意

> `Recycle` 策略会通过运行一个 busybox 容器来执行数据删除命令，默认定义的 busybox 镜像是：

```
gcr.io/google_containers/busybox:latest
```

，并且 

```
imagePullPolicy: Always
```

，如果需要调整配置，需要增加

```
kube-controller-manager
```

 启动参数：

```
--pv-recycler-pod-template-filepath-hostpath
```

 来进行配置。

> 关于 PV 的状态，实际上描述的是 PV 的生命周期的某个阶段，一个 PV 的生命周期中，可能会处于4种不同的阶段：

*   Available（可用）：表示可用状态，还未被任何 PVC 绑定
*   Bound（已绑定）：表示 PVC 已经被 PVC 绑定
*   Released（已释放）：PVC 被删除，但是资源还未被集群重新声明
*   Failed（失败）： 表示该 PV 的自动回收失败

> 现在我们创建完成了 PV，如果我们需要使用这个 PV 的话，就需要创建一个对应的 PVC 来和他进行绑定了，就类似于我们的服务是通过 Pod 来运行的，而不是 Node，只是 Pod 跑在 Node 上而已。

> 现在我们来创建一个 

```
PersistentVolumeClaim
```

，Pod 使用 PVC 来请求物理存储，我们这里创建的 PVC 请求至少 3G 容量的卷，该卷至少可以为一个节点提供读写访问，下面是 PVC 的配置文件：(pvc-hostpath.yaml)

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-hostpath
spec:
  storageClassName: manual
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
```

> 同样我们可以直接创建这个 PVC 对象：

```
$ kubectl create -f pvc-hostpath.yaml
persistentvolumeclaim/pvc-hostpath created
```

> 创建 PVC 之后，Kubernetes 就会去查找满足我们声明要求的 PV，比如 `storageClassName`、`accessModes` 以及容量这些是否满足要求，如果满足要求就会将 PV 和 PVC 绑定在一起。

> 注意

> 需要注意的是目前 PV 和 PVC 之间是一对一绑定的关系，也就是说一个 PV 只能被一个 PVC 绑定。

> 我们现在再次查看 PV 的信息：

```
$ kubectl get pv -l type=local
NAME          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                  STORAGECLASS   REASON   AGE
pv-hostpath   10Gi       RWO            Retain           Bound    default/pvc-hostpath   manual                  81m
```

> 现在输出的 STATUS 为 `Bound`，查看 PVC 的信息：

```
$ kubectl get pvc pvc-hostpath
NAME           STATUS   VOLUME        CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-hostpath   Bound    pv-hostpath   10Gi       RWO            manual         6m47s
```

> 输出结果表明该 PVC 绑定了到了上面我们创建的 `pv-hostpath` 这个 PV 上面了，我们这里虽然声明的3G的容量，但是由于 PV 里面是 10G，所以显然也是满足要求的。

> PVC 准备好过后，接下来我们就可以来创建 Pod 了，该 Pod 使用上面我们声明的 PVC 作为存储卷：(pv-hostpath-pod.yaml)

```
apiVersion: v1
kind: Pod
metadata:
  name: pv-hostpath-pod
spec:
  volumes:
  - name: pv-hostpath
    persistentVolumeClaim:
      claimName: pvc-hostpath
  nodeSelector:
    kubernetes.io/hostname: ydzs-node1
  containers:
  - name: task-pv-container
    image: nginx
    ports:
    - containerPort: 80
    volumeMounts:
    - mountPath: "/usr/share/nginx/html"
      name: pv-hostpath
```

> 这里需要注意的是，由于我们创建的 PV 真正的存储在节点 ydzs-node1 上面，所以我们这里必须把 Pod 固定在这个节点下面，另外可以注意到 Pod 的配置文件指定了 

```
PersistentVolumeClaim
```

，但没有指定 `PersistentVolume`，对 Pod 而言，`PVC` 就是一个存储卷。直接创建这个 Pod 对象即可：

```
$ kubectl create -f pv-hostpath-pod.yaml
pod/pv-hostpath-pod created
$ kubectl get pod pv-hostpath-pod
NAME              READY   STATUS    RESTARTS   AGE
pv-hostpath-pod   1/1     Running   0          105s
```

> 运行成功后，我们可以打开一个 shell 访问 Pod 中的容器：

```
$ kubectl exec -it pv-hostpath-pod -- /bin/bash
```

> 在 shell 中，我们可以验证 nginx 的数据 是否正在从 hostPath 卷提供 index.html 文件：

```
root@pv-hostpath-pod:/# apt-get update
root@pv-hostpath-pod:/# apt-get install curl -y
root@pv-hostpath-pod:/# curl localhost
Hello from Kubernetes hostpath storage
```

> 我们可以看到输出结果是我们前面写到 hostPath 卷种的 index.html 文件中的内容，同样我们可以把 Pod 删除，然后再次重建再测试一次，可以发现内容还是我们在 hostPath 种设置的内容。

> 我们在持久化容器数据的时候使用 PV/PVC 有什么好处呢？比如我们这里之前直接在 Pod 下面也可以使用 hostPath 来持久化数据，为什么还要费劲去创建 PV、PVC 对象来引用呢？PVC 和 PV 的设计，其实跟`面向对象`的思想完全一致，PVC 可以理解为持久化存储的接口，它提供了对某种持久化存储的描述，但不提供具体的实现；而这个持久化存储的实现部分则由 PV 负责完成。这样做的好处是，作为应用开发者，我们只需要跟 PVC 这个接口打交道，而不必关心具体的实现是 hostPath、NFS 还是 Ceph。毕竟这些存储相关的知识太专业了，应该交给专业的人去做，这样对于我们的 Pod 来说就不用管具体的细节了，你只需要给我一个可用的 PVC 即可了，这样是不是就完全屏蔽了细节和解耦了啊，所以我们更应该使用 PV、PVC 这种方式。

### Local PV 

> 上面我们创建了后端是 hostPath 类型的 PV 资源对象，我们也提到了，使用 hostPath 有一个局限性就是，我们的 Pod 不能随便漂移，需要固定到一个节点上，因为一旦漂移到其他节点上去了宿主机上面就没有对应的数据了，所以我们在使用 hostPath 的时候都会搭配 nodeSelector 来进行使用。但是使用 hostPath 明显也有一些好处的，因为 PV 直接使用的是本地磁盘，尤其是 SSD 盘，它的读写性能相比于大多数远程存储来说，要好得多，所以对于一些对磁盘 IO 要求比较高的应用比如 etcd 就非常实用了。不过呢，相比于正常的 PV 来说，使用了 hostPath 的这些节点一旦宕机数据就可能丢失，所以这就要求使用 hostPath 的应用必须具备数据备份和恢复的能力，允许你把这些数据定时备份在其他位置。

> 所以在 hostPath 的基础上，Kubernetes 依靠 PV、PVC 实现了一个新的特性，这个特性的名字叫作：

```
Local Persistent Volume
```

，也就是我们说的 `Local PV`。

> 其实 `Local PV` 实现的功能就非常类似于 `hostPath` 加上 `nodeAffinity`，比如，一个 Pod 可以声明使用类型为 Local 的 PV，而这个 PV 其实就是一个 hostPath 类型的 Volume。如果这个 hostPath 对应的目录，已经在节点 A 上被事先创建好了，那么，我只需要再给这个 Pod 加上一个 `nodeAffinity=nodeA`，不就可以使用这个 Volume 了吗？理论上确实是可行的，但是事实上，我们绝不应该把一个宿主机上的目录当作 PV 来使用，因为本地目录的存储行为是完全不可控，它所在的磁盘随时都可能被应用写满，甚至造成整个宿主机宕机。所以，一般来说 `Local PV` 对应的存储介质是一块额外挂载在宿主机的磁盘或者块设备，我们可以认为就是`一个 PV 一块盘`。

> 另外一个 `Local PV` 和普通的 PV 有一个很大的不同在于 `Local PV` 可以保证 Pod 始终能够被正确地调度到它所请求的 `Local PV` 所在的节点上面，对于普通的 PV 来说，Kubernetes 都是先调度 Pod 到某个节点上，然后再持久化节点上的 Volume 目录，进而完成 Volume 目录与容器的绑定挂载，但是对于 `Local PV` 来说，节点上可供使用的磁盘必须是提前准备好的，因为它们在不同节点上的挂载情况可能完全不同，甚至有的节点可以没这种磁盘，所以，这时候，调度器就必须能够知道所有节点与 `Local PV` 对应的磁盘的关联关系，然后根据这个信息来调度 Pod，实际上就是在调度的时候考虑 Volume 的分布。

> 接下来我们来测试下 `Local PV` 的使用，当然按照上面我们的分析我们应该给宿主机挂载并格式化一个可用的磁盘，我们这里就暂时将 ydzs-node1 节点上的 `/data/k8s/localpv` 这个目录看成是挂载的一个独立的磁盘。现在我们来声明一个 `Local PV` 类型的 PV，如下所示：(pv-local.yaml)

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-local
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /data/k8s/localpv  # ydzs-node1节点上的目录
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - ydzs-node1
```

> 和前面我们定义的 PV 不同，我们这里定义了一个 `local` 字段，表明它是一个 `Local PV`，而 path 字段，指定的正是这个 PV 对应的本地磁盘的路径，即：`/data/k8s/localpv`，这也就意味着如果 Pod 要想使用这个 PV，那它就必须运行在 ydzs-node1 节点上。所以，在这个 PV 的定义里，添加了一个节点亲和性 `nodeAffinity` 字段指定 ydzs-node1 这个节点。这样，调度器在调度 Pod 的时候，就能够知道一个 PV 与节点的对应关系，从而做出正确的选择。

> 直接创建上面的资源对象：

```
$ kubectl apply -f pv-local.yaml 
persistentvolume/pv-local created
$ kubectl get pv
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS  CLAIM      STORAGECLASS      REASON   AGE
pv-local  5Gi        RWO            Delete           Available          local-storage              24s
```

> 可以看到，这个 PV 创建后，进入了 `Available`（可用）状态。这个时候如果按照前面提到的，我们要使用这个 `Local PV` 的话就需要去创建一个 PVC 和他进行绑定：(pvc-local.yaml)

```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc-local
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: local-storage
```

> 同样要注意声明的这些属性需要和上面的 PV 对应，直接创建这个资源对象：

```
$ kubectl apply -f pvc-local.yaml 
persistentvolumeclaim/pvc-local created
$ kubectl get pvc
NAME           STATUS   VOLUME        CAPACITY   ACCESS MODES   STORAGECLASS    AGE
pvc-local      Bound    pv-local      5Gi        RWO            local-storage   38s
```

> 可以看到现在 PVC 和 PV 已经处于 `Bound` 绑定状态了。但实际上这是不符合我们的需求的，比如现在我们的 Pod 声明使用这个 pvc-local，并且我们也明确规定，这个 Pod 只能运行在 ydzs-node2 这个节点上，如果按照上面我们这里的操作，这个 pvc-local 是不是就和我们这里的 pv-local 这个 `Local PV` 绑定在一起了，但是这个 PV 的存储券又在 ydzs-node1 这个节点上，显然就会出现冲突了，那么这个 Pod 的调度肯定就会失败了，所以我们在使用 `Local PV` 的时候，必须想办法延迟这个`绑定`操作。

> 要怎么来实现这个延迟绑定呢？我们可以通过创建 `StorageClass` 来指定这个动作，在 StorageClass 种有一个 

```
volumeBindingMode=WaitForFirstConsumer
```

 的属性，就是告诉 Kubernetes 在发现这个 StorageClass 关联的 PVC 与 PV 可以绑定在一起，但不要现在就立刻执行绑定操作（即：设置 PVC 的 VolumeName 字段），而是要等到第一个声明使用该 PVC 的 Pod 出现在调度器之后，调度器再综合考虑所有的调度规则，当然也包括每个 PV 所在的节点位置，来统一决定，这个 Pod 声明的 PVC，到底应该跟哪个 PV 进行绑定。通过这个延迟绑定机制，原本实时发生的 PVC 和 PV 的绑定过程，就被延迟到了 Pod 第一次调度的时候在调度器中进行，从而保证了这个绑定结果不会影响 Pod 的正常调度。

> 所以我们需要创建对应的 `StorageClass` 对象：（local-storageclass.yaml）

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

> 这个 `StorageClass` 的名字，叫作 local-storage，也就是我们在 PV 中声明的，需要注意的是，在它的 `provisioner` 字段，我们指定的是 `no-provisioner`。这是因为我们这里是手动创建的 PV，所以不需要动态来生成 PV，另外这个 StorageClass 还定义了一个 

```
volumeBindingMode=WaitForFirstConsumer
```

 的属性，它是 `Local PV` 里一个非常重要的特性，即：`延迟绑定`。通过这个延迟绑定机制，原本实时发生的 PVC 和 PV 的绑定过程，就被延迟到了 Pod 第一次调度的时候在调度器中进行，从而保证了这个绑定结果不会影响 Pod 的正常调度。

> 现在我们来创建这个 StorageClass 资源对象：

```
$ kubectl apply -f local-storageclass.yaml 
storageclass.storage.k8s.io/local-storage created
```

> 现在我们重新删除上面声明的 PVC 对象，重新创建：

```
$ kubectl delete -f pvc-local.yaml 
persistentvolumeclaim "pvc-local" deleted
$ kubectl create -f pvc-local.yaml
persistentvolumeclaim/pvc-local created
$ kubectl get pvc
NAME           STATUS    VOLUME        CAPACITY   ACCESS MODES   STORAGECLASS    AGE
pvc-local      Pending                                           local-storage   3s
```

> 我们可以发现这个时候，集群中即使已经存在了一个可以与 PVC 匹配的 PV 了，但这个 PVC 依然处于 `Pending` 状态，也就是等待绑定的状态，这就是因为上面我们配置的是延迟绑定，需要在真正的 Pod 使用的时候才会来做绑定。

> 同样我们声明一个 Pod 来使用这里的 pvc-local 这个 PVC，资源对象如下所示：(pv-local-pod.yaml)

```
apiVersion: v1
kind: Pod
metadata:
  name: pv-local-pod
spec:
  volumes:
  - name: example-pv-local
    persistentVolumeClaim:
      claimName: pvc-local
  containers:
  - name: example-pv-local
    image: nginx
    ports:
    - containerPort: 80
    volumeMounts:
    - mountPath: /usr/share/nginx/html
      name: example-pv-local
```

> 直接创建这个 Pod：

```
$ kubectl apply -f pv-local-pod.yaml 
pod/pv-local-pod created
```

> 创建完成后我们这个时候去查看前面我们声明的 PVC，会立刻变成 `Bound` 状态，与前面定义的 PV 绑定在了一起：

```
$ kubectl get pvc
NAME           STATUS   VOLUME        CAPACITY   ACCESS MODES   STORAGECLASS    AGE
pvc-local      Bound    pv-local      5Gi        RWO            local-storage   4m59s
```

> 这时候，我们可以尝试在这个 Pod 的 Volume 目录里，创建一个测试文件，比如：

```
$ kubectl exec -it pv-local-pod /bin/sh
# cd /usr/share/nginx/html
# echo "Hello from Kubernetes local pv storage" > test.txt  
#
```

> 然后，登录到 ydzs-node1 这台机器上，查看一下它的 `/data/k8s/localpv` 目录下的内容，你就可以看到刚刚创建的这个文件：

```
# 在ydzs-node1节点上
$ ls /data/k8s/localpv
test.txt
$ cat /data/k8s/localpv/test.txt 
Hello from Kubernetes local pv storage
```

> 如果重新创建这个 Pod 的话，就会发现，我们之前创建的测试文件，依然被保存在这个持久化 Volume 当中：

```
$ kubectl delete -f pv-local-pod.yaml  
$ kubectl apply -f pv-local-pod.yaml 
$ kubectl exec -it pv-local-pod /bin/sh
# ls /usr/share/nginx/html
test.txt
# cat /usr/share/nginx/html/test.txt
Hello from Kubernetes local pv storage
#
```

> 到这里就说明基于本地存储的 Volume 是完全可以提供容器持久化存储功能的，对于 StatefulSet 这样的有状态的资源对象，也完全可以通过声明 Local 类型的 PV 和 PVC，来管理应用的存储状态。

> 需要注意的是，我们上面手动创建 PV 的方式，即静态的 PV 管理方式，在删除 PV 时需要按如下流程执行操作：

*   删除使用这个 PV 的 Pod
*   从宿主机移除本地磁盘
*   删除 PVC
*   删除 PV

> 如果不按照这个流程的话，这个 PV 的删除就会失败。