
## kubelet 源码

> 进入 cmd/kubelet/app/server.go 调用 NewKubeletCommand 函数，使用默认的参数创建一个 cobra.Command 指针：

```
func NewKubeletCommand() *cobra.Command {
    ...
    cmd := &cobra.Command{
        Use: componentKubelet,
        ...
        // Kubelet具有特殊的flag解析要求来强制执行flag优先级规则，因此我们在下面的Run中手动完成所有解析。DisableFlagParsing=true提供传递给`args` arg中的kubelet的完整flagset来运行，没有Cobra的干扰。
        DisableFlagParsing: true,
        Run: func(cmd *cobra.Command, args []string) {
            // 初始化flag解析，因为我们禁用了cobra的flag解析
            if err := cleanFlagSet.Parse(args); err != nil {
                cmd.Usage()
                klog.Fatal(err)
            }
            ......
            // 校验初始的 KubeletFlags
            // 加载 kubelet config 文件(如果开启)
            // 使用动态的 kubelet config(如果开启)
            // 通过kubeletFlags和kubeletConfig封装KubeletServer结构
            // 使用 kubeletServer 来构造默认的 KubeletDeps
            // 添加 kubelet 配置控制器到 kubeletDeps
            // 添加 stopCh Channel 供 kubelet 和 docker shim 复用
            // 开启docker shim特性(如果开启)
            if kubeletServer.KubeletFlags.ExperimentalDockershim {
                if err := RunDockershim(&kubeletServer.KubeletFlags, kubeletConfig, stopCh); err != nil {
                    klog.Fatal(err)
                }
                // 注意这里的return，意味着启用 dockershim only 模式，kubelet只会启动 dockershim 进程。此标志仅用于测试目的，除非您意识到自己在做什么，否则请不要使用它。 [默认= FALSE]
                return
            }
            // 运行 kubelet
            if err := Run(kubeletServer, kubeletDeps, stopCh); err != nil {
                klog.Fatal(err)
            }
        }
    }
    ...
    return cmd
}
```

> 其中 RunDockershim 函数仅在当前进程中启动 dockershim。这仅用于验证测试目的。

```
func RunDockershim(f *options.KubeletFlags, c *kubeletconfiginternal.KubeletConfiguration, stopCh <-chan struct{}) error {
    // 初始化 docker 客户端配置
    // 初始化网络插件配置
    // 初始化 streaming 配置（现在不使用TLS）
    // 独立的 dockershim 将始终会启动本地的streaming server。
    ds, err := dockershim.NewDockerService(dockerClientConfig, r.PodSandboxImage, streamingConfig, &pluginSettings,
        f.RuntimeCgroups, c.CgroupDriver, r.DockershimRootDirectory, true /*startLocalStreamingServer*/)
    if err != nil {
        return err
    }
    // 为 docker CRI shim 启动 GRPC 服务器
    server := dockerremote.NewDockerServer(f.RemoteRuntimeEndpoint, ds)
    if err := server.Start(); err != nil {
        return err
    }
    <-stopCh
    return nil
}
```

> 然后是最核心的 kubelet 启动函数 Run：

```
// Run 函数运行一个具有给定依赖项的 KubeletServer，永远不会退出。
// kubeDeps 参数可能为 nil - 如果为空，则需要从 KubeletServer 上的设置进行初始化。
// 如果调用者已经设置了依赖对象了则不会生成默认的对象了。
func Run(s *options.KubeletServer, kubeDeps *kubelet.Dependencies, stopCh <-chan struct{}) error {
    if err := initForOS(s.KubeletFlags.WindowsService); err != nil {
        return fmt.Errorf("failed OS init: %v", err)
    }
    if err := run(s, kubeDeps, stopCh); err != nil {
        return fmt.Errorf("failed to run Kubelet: %v", err)
    }
    return nil
}
```

> 调用下面的核心的 run 函数：

```
func run(s *options.KubeletServer, kubeDeps *kubelet.Dependencies, stopCh <-chan struct{}) (err error) {
    // 基于初始的KubeletServer的值设置全局的 feature gates 特性
    // 校验初始KubeletServer（我们首先设置feature gates，因为该验证依赖feature gates）
    if err := options.ValidateKubeletServer(s); err != nil {
        return err
    }
    // 获取 Kubelet 锁文件
    // 使用 /configz 端口注册当前配置
    // 获取客户端等信息，检测独立(standaloneMode)模式
    // 获取主机名、获取节点名
    // 如果是独立模式，尽可能将所有客户端设置成nil
    // 否则
    //      创建 kubelet 客户端
    //      为事件创建一个独立的客户端
    //      为禁用限流和附加超时的心跳创建一个单独的客户端
    //          如果启用了NodeLease功能，则超时是租约持续时间和状态更新频率的最小值
    //          NodeLease: Kubelet 使用新的 Lease API 报告节点心跳，节点生命周期控制器使用这些心跳作为节点健康信号。
    // 如果 kubelet 配置控制器可用，并且启用了动态配置，则启动配置和状态同步循环
    // DynamicConfigDir:
    //      Kubelet 将使用此目录检查下载的配置并跟踪配置运行状况。
    //      Kubelet 将创建此目录（如果该目录尚不存在）, 路径可以是绝对的或相对的; 相对路径位于 Kubelet 的当前工作目录下。
    //      提供此标志可启用动态 kubelet 配置，要使用此标志，必须启用DynamicKubeletConfig feature gates。
    if utilfeature.DefaultFeatureGate.Enabled(features.DynamicKubeletConfig) && len(s.DynamicConfigDir.Value()) > 0 &&
        kubeDeps.KubeletConfigController != nil && !standaloneMode && !s.RunOnce {
        // StartSync 函数告诉控制器开一个goroutine从apiserver同步status/config 到 /（位于pkg/kubelt/kubeletconfig/controller.go）
        if err := kubeDeps.KubeletConfigController.StartSync(kubeDeps.KubeClient, kubeDeps.EventClient, string(nodeName)); err != nil {
            return err
        }
    }

    // 然后调用 BuildAuth 函数创建一个验证器，一个授权器，以及一个 kubelet的authorizer.RequestAttributesGetter
    // 安装事件记录器
    // 解析资源预留配置
    // 构造 ContainerManager

    // 然后调用 RunKubelet 函数
    if err := RunKubelet(s, kubeDeps, s.RunOnce); err != nil {
        return err
    }

    // 如果开启了 healthzPort 端口（本地 healthz endpint 端口，0表示禁用）,则开启 healthz server

    // 如果使用了 systemd , 通知它我们已经启动了
    go daemon.SdNotify(false, "READY=1")

    select {
    case <-done:
        break
    case <-stopCh:
        break
    }

    return nil
}
```

> 其中校验 KubeletServer 的函数 ValidateKubeletServer 位于

```
cmd/kubelet/app/options/options.go
```

：

```
// ValidateKubeletServer 函数校验 KubeletServer 的配置，如果配置无效则返回error。
func ValidateKubeletServer(s *KubeletServer) error {
    // 任何的 KubeletConfiguration 校验都使用kubeletconfigvalidation.ValidateKubeletConfiguration函数
    // ValidateKubeletConfiguration 函数位于 pkg/kubelet/apis/config/validation/validation.go
    if err := kubeletconfigvalidation.ValidateKubeletConfiguration(&s.KubeletConfiguration); err != nil {
        return err
    }
    // ValidateKubeletFlags 函数校验 Kubelet 的配置 flag，如果无效则返回 error
    if err := ValidateKubeletFlags(&s.KubeletFlags); err != nil {
        return err
    }
    return nil
}
```

> 其中启动 Kubelet 核心的 RunKubelet 函数如下：

```
// RunKubelet 函数负责配置和运行 kubelet。它用于三种不同的应用：
//   集成测试
//   Kubelet 二进制
//   独立的 kubernetes 二进制
// 最终，#2将会被#3的实例所替换
func RunKubelet(kubeServer *options.KubeletServer, kubeDeps *kubelet.Dependencies, runOnce bool) error {
    ......
    // 核心仍然是调用 createAndInitKubelet 函数将 Kubelet 配置、依赖等构造成 kubelet 的启动接口

    // 然后调用 startKubelet 函数启动 Kubelet
    startKubelet(k, podCfg, &kubeServer.KubeletConfiguration, kubeDeps, kubeServer.EnableCAdvisorJSONEndpoints, kubeServer.EnableServer)

    return nil
}
```

> 其中 createAndInitKubelet 函数调用了 kubelet.NewMainKubelet 函数来构造 kubelet 的启动接口，NewMainKubelet 函数位于

```
pkg/kubelet/kubelet.go
```

：

```
// NewMainKubelet 函数实例化一个新的 Kubelet 对象以及所有必需的内部模块。此处不会初始化 Kubelet 及其模块。
func NewMainKubelet(...) {
    // 一些常规校验
    // 指定容器垃圾回收策略
    // 指定镜像垃圾回收策略
    // 配置驱逐相关策略
    // 配置kubelet客户端
    // 配置集群DNS
    // 组装Kubelet结构体
    // 设置 secretManager
    // 设置 configMapManager
    // 设置 livenessManager
    // 设置 podManager，podManager 还负责保持 secretManager 和 configMapManager 内容为最新。
    // 设置 statusManager
    // 启动 docker CRI shim GRPC 服务(包括RuntimeService和ImageService两个客户端):
    //      默认 remoteRuntimeEndpoint=unix:///var/run/dockershim.sock，即默认使用本地的docker作为容器运行时；如果remoteImageEndpoint没有指定，则和remoteRuntimeEndpoint保持一致
    // 然后创建一个新的 kubeGenericRuntimeManager 
    // 初始化一个新的 GenericPLEG 对象，用于配置pod生命周期事件生成器
    // 然后配置容器回收策略
    // 配置镜像回收策略
    // 如果配置了 Kubelet 证书自动轮转(开启RotateKubeletServerCertificate特性)，则初始化证书管理器
    // 然后配置probe管理器(探针管理)
    // 生成一个对pod的service account token的管理器
    // 调用如下方法配置volume插件管理器：
    //      NewInitializedVolumePluginMgr函数初始化Kubelet runtimeState上的一些storageErrors（在csi_plugin.go init中），这会影响节点d的就绪状态。必须在初始化Kubelet之前调用此函数，以便节点ReadyState在存储状态下准确无误。
    klet.volumePluginMgr, err =
        NewInitializedVolumePluginMgr(klet, secretManager, configMapManager, tokenManager, kubeDeps.VolumePlugins, kubeDeps.DynamicPluginProber)
    // 然后配置volume管理器
    // 配置eviction(驱逐)管理器
    // 如果开启了NodeLease特性(节点生命周期控制器使用lease api报告的心跳来作为节点健康的信号))，则配置nodeLease控制器
    // 最后，将最新版本的配置放在 Kubelet 上，以便可以看到它的配置方式:
    klet.kubeletConfiguration = *kubeCfg
    // 最后的最后，生成状态函数应该是我们做的最后一件事，因为这依赖于已经构造的Kubelet的其他部分。
    // defaultNodeStatusFuncs 是一个工厂函数(位于pkg/kubelet/kubelet_node_status.go)，它生成一组默认的 setNodeStatus 函数
    klet.setNodeStatusFuncs = klet.defaultNodeStatusFuncs()

    return klet, nil
}
```

> 其中 startKubelet 函数在调用上面的 createAndInitKubelet 函数获取到最新的 kubelet 启动对象以后，直接启动 kubelet 的一个server：

```
// startKubelet 函数很简单就是直接启动一个 kubelet 的server，
func startKubelet(k kubelet.Bootstrap, podCfg *config.PodConfig, kubeCfg *kubeletconfiginternal.KubeletConfiguration, kubeDeps *kubelet.Dependencies, enableCAdvisorJSONEndpoints, enableServer bool) {
    // 开启一个goroutine，调用Run函数，不退出。
    go wait.Until(func() {
        k.Run(podCfg.Updates())
    }, 0, wait.NeverStop)

    // 开启 kubelet server
    if enableServer {
        go k.ListenAndServe(net.ParseIP(kubeCfg.Address), uint(kubeCfg.Port), kubeDeps.TLSOptions, kubeDeps.Auth, enableCAdvisorJSONEndpoints, kubeCfg.EnableDebuggingHandlers, kubeCfg.EnableContentionProfiling)

    }
    // 如果开启了只读端口，然后也会在一个gorouting 里面开启一个只读的 server
    if kubeCfg.ReadOnlyPort > 0 {
        go k.ListenAndServeReadOnly(net.ParseIP(kubeCfg.Address), uint(kubeCfg.ReadOnlyPort), enableCAdvisorJSONEndpoints)
    }
    // 如果开启了 kubelet 的 pod 资源 grpc 服务，则同样运行 一个kubelet 的 podresource grpc 服务
    if utilfeature.DefaultFeatureGate.Enabled(features.KubeletPodResources) {
        go k.ListenAndServePodResources()
    }
}
```

> 其中 Run 函数，位于

```
pkg/kubelet/kubelet.go
```

，运行启动 kubelet 响应配置更新：

```
func (kl *Kubelet) Run(updates <-chan kubetypes.PodUpdate) {
    // 配置 logServer
    // 检查 kubeClient
    // 开启 cloud provider 同步管理器
    // 调用 initializeModules 函数初始化内部模块（不需要启动容器运行时）
    if err := kl.initializeModules(); err != nil {
        kl.recorder.Eventf(kl.nodeRef, vEventTypeWarning, events.KubeletSetupFailed, err.Error())
        klog.Fatal(err)
    }
    // 启动 volume 管理器
    if kl.kubeClient != nil {
        // 立即开始同步节点状态，这可能会设置运行时需要运行的内容
        go wait.Until(kl.syncNodeStatus, kl.nodeStatusUpdateFrequency, wait.NeverStop)

        // FastStatusUpdateOnce 函数启动一个循环，在应用CIDR时检查内部节点索引器缓存，并尝试立即更新POD CIDR。更新pod CIDR后，它会触发运行时更新和节点状态更新。函数在一次成功的节点状态更新后直接返回。该功能仅在 kubelet 启动期间执行，通过尽快更新 pod cidr、运行时状态和节点状态来提高准备就绪节点的延迟。
        go kl.fastStatusUpdateOnce()

        // 开始同步 lease
        if utilfeature.DefaultFeatureGate.Enabled(features.NodeLease) {
            go kl.nodeLeaseController.Run(wait.NeverStop)
        }
    }
    // 每5s调用一次updateRuntimeUp函数检查容器运行时状态
    go wait.Until(kl.updateRuntimeUp, 5*time.Second, wait.NeverStop)

    // 启动循环同步 iptables 规则
    if kl.makeIPTablesUtilChains {
        // 不过当前syncNetworkUtil函数是一个空实现
        go wait.Until(kl.syncNetworkUtil, 1*time.Minute, wait.NeverStop)
    }

    // 当 pod 没有被 podworker 正确处理的时候，启动一个 goroutine 杀死pod。
    go wait.Until(kl.podKiller, 1*time.Second, wait.NeverStop)

    // 启动组件同步循环
    //      使用apiserver同步pods状态; 也用作状态缓存。
    kl.statusManager.Start()
    //      处理容器探针
    kl.probeManager.Start()

    // 启动 RuntimeClass 控制器
    if kl.runtimeClassManager != nil {
        kl.runtimeClassManager.Start(wait.NeverStop)
    }

    // 启动 Pod 生命周期事件生成器。
    // PodLifecycleEventGenerator 是一个包含下面函数的pod生命周期事件生成器接口：
    // type PodLifecycleEventGenerator interface {
    //     Start()
    //     Watch() chan *PodLifecycleEvent
    //     Healthy() (bool, error)
    // }
    kl.pleg.Start()
    // 调用updates的同步循环
    kl.syncLoop(updates, kl)
}
```

> 其中 initializeModules 函数用来初始化内部得一些模块（不需要启动容器运行时）

```
func (kl *Kubelet) initializeModules() error {
    // 注册 prometheus metrics 接口
    // 创建文件系统目录:
    //      根目录 pods目录 插件目录 pod-resources 目录
    if err := kl.setupDataDirs(); err != nil {
        return err
    }
    // 如果容器日志目录不存在，则创建该目录(/var/log/containers)

    // 然后启动镜像管理器
    kl.imageManager.Start()

    // 开启 certificate 管理器
    if kl.serverCertificateManager != nil {
        kl.serverCertificateManager.Start()
    }

    // 开启 out of memory 监听器
    if err := kl.oomWatcher.Start(kl.nodeRef); err != nil {
        return fmt.Errorf("Failed to start OOM watcher %v", err)
    }

    // 开启资源分析器
    kl.resourceAnalyzer.Start()

    return nil
}
```

> 然后就是调用 syncNodeStatus 函数失效节点状态同步，位于

```
pkg/kubelet/kubelet_node_status.go
```

:

```
// syncNodeStatus 函数定期被一个goroutine所调用。
// 如果和上次同步对比有任何变化或者上次同步经过的时间足够长，它会将节点状态同步到 master，如果需要请先注册 kubelet 。
func (kl *Kubelet) syncNodeStatus() {
    kl.syncNodeStatusMux.Lock()
    defer kl.syncNodeStatusMux.Unlock()

    if kl.kubeClient == nil || kl.heartbeatClient == nil {
        return
    }
    // 如果开启了registerNode
    if kl.registerNode {
        // 如果不需要执行任何操作，它将立即退出。registerWithApiServer 函数向群集 APIServer 注册节点。可以安全地多次调用，但不能同时调用（kl.registrationcompleted 未加锁）。
        kl.registerWithAPIServer()
    }
    if err := kl.updateNodeStatus(); err != nil {
        klog.Errorf("Unable to update node status: %v", err)
    }
}
```

> 调用 updateNodeStatus 函数更新节点状态:

```
// updateNodeStatus 函数更新节点状态到 master，如果上次同步有任何更改或经过足够的时间，则重试一次，最多重试5次
func (kl *Kubelet) updateNodeStatus() error {
    klog.V(5).Infof("Updating node status")
    for i := 0; i < nodeStatusUpdateRetry; i++ {
        // 调用 tryUpdateNodeStatus 函数尝试进行状态更新
        if err := kl.tryUpdateNodeStatus(i); err != nil {
            // 当心跳操作失败多次时，调用 onRepeatedHeartBeatFailure 函数
            if i > 0 && kl.onRepeatedHeartbeatFailure != nil {
                kl.onRepeatedHeartbeatFailure()
            }
            klog.Errorf("Error updating node status, will retry: %v", err)
        } else {
            return nil
        }
    }
    return fmt.Errorf("update node status exceeds retry count")
}
```

> 调用 tryUpdateNodeStatus 函数尝试去进行真正的节点状态更新：

```
func (kl *Kubelet) tryUpdateNodeStatus(tryNumber int) error {
    // 在大型集群中，这里的节点对象的 GET 和 PUT 操作是 apiserver 和 etcd 上的主要负载。
    // 为了减少 etcd 上的负载，我们提供了从 apiserver 缓存获取操作（数据可能稍有延迟，但似乎不会造成更多冲突-延迟非常小）。
    // 如果导致冲突，则所有重试都直接从 etcd 提供。
    opts := metavGetOptions{}
    if tryNumber == 0 {
        util.FromApiserverCache(&opts)
    }
    node, err := kl.heartbeatClient.CoreV1().Nodes().Get(string(kl.nodeName), opts)
    if err != nil {
        return fmt.Errorf("error getting node %q: %v", kl.nodeName, err)
    }

    originalNode := node.DeepCopy()
    if originalNode == nil {
        return fmt.Errorf("nil %q node object", kl.nodeName)
    }

    podCIDRChanged := false
    if node.Spec.PodCIDR != "" {
        // Pod CIDR 之前就已经更新过了，因此我们需要判断node.Spec.PodCIDR是否为空，还需要知道 pod CIDR 是否真的已经改变了。
        if podCIDRChanged, err = kl.updatePodCIDR(node.Spec.PodCIDR); err != nil {
            klog.Errorf(err.Error())
        }
    }

    // setNodeStatus 函数就是调用上面我们设置的 setNodeStatusFuncs 函数列表来进行状态更新
    kl.setNodeStatus(node)

    // 如果开启了 NodeLease 特性
    now := kl.clock.Now()
    if utilfeature.DefaultFeatureGate.Enabled(features.NodeLease) && now.Before(kl.lastStatusReportTime.Add(kl.nodeStatusReportFrequency)) {
        if !podCIDRChanged && !nodeStatusHasChanged(&originalNode.Status, &node.Status) {
            // 将指定的卷标记为已在节点的卷状态中成功报告为“正在使用”。
            kl.volumeManager.MarkVolumesAsReportedInUse(node.Status.VolumesInUse)
            return nil
        }
    }

    // Patch 现在节点的状态到 APIServer
    updatedNode, _, err := nodeutil.PatchNodeStatus(kl.heartbeatClient.CoreV1(), types.NodeName(kl.nodeName), originalNode, node)
    if err != nil {
        return err
    }
    // 设置当前时间为最近一次状态上报时间
    kl.lastStatusReportTime = now
    kl.setLastObservedNodeAddresses(updatedNode.Status.Addresses)
    // 如果更新成功完成，请将卷使用标记为ReportedInUse以表明这些卷已在节点状态下更新
    kl.volumeManager.MarkVolumesAsReportedInUse(updatedNode.Status.VolumesInUse)
    return nil
}
```