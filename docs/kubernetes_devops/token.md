
## Tekton

> Tekton 是一款功能非常强大而灵活的 CI/CD 开源的云原生框架。Tekton 的前身是 Knative 项目的 build-pipeline 项目，这个项目是为了给 build 模块增加 pipeline 的功能，但是随着不同的功能加入到 Knative build 模块中，build 模块越来越变得像一个通用的 CI/CD 系统，于是，索性将 build-pipeline 剥离出 Knative，就变成了现在的 Tekton，而 Tekton 也从此致力于提供全功能、标准化的云原生 CI/CD 解决方案。

### 安装

> 安装 Tekton 非常简单，可以直接通过 tektoncd/pipeline 的 GitHub 仓库中的 `release.yaml` 文件进行安装，如下所示的命令：

```
$ kubectl apply -f https://github.com/tektoncd/pipeline/releases/download/v0/release.yaml
```

> 由于官方使用的镜像是 `gcr` 的镜像，所以正常情况下我们是获取不到的，如果你的集群由于某些原因获取不到镜像，可以使用下面的资源清单文件，我已经将镜像替换成了 Docker Hub 上面的镜像：

```
$ kubectl apply -f https://www.qikqiak.com/k8strain/devops/manifests/tekton/release.yaml
```

> 上面的资源清单文件安装后，会创建一个名为 `tekton-pipelines` 的命名空间，在该命名空间下面会有大量和 tekton 相关的资源对象，我们可以通过在该命名空间中查看 Pod 并确保它们处于 Running 状态来检查安装是否成功：

```
$ kubectl get pods -n tekton-pipelines
NAME                                           READY   STATUS    RESTARTS   AGE
tekton-pipelines-controller-67f4dc98d8-pgxrq   1/1     Running   0          9m46s
tekton-pipelines-webhook-59df55445c-jw76v      1/1     Running   0          9m45s
```

> Tekton 安装完成后，我们还可以选择是否安装 CLI 工具，有时候可能 Tekton 提供的命令行工具比 kubectl 管理这些资源更加方便，当然这并不是强制的，我这里是 Mac 系统，所以可以使用常用的 Homebrew 工具来安装：

```
$ brew tap tektoncd/tools
$ brew install tektoncd/tools/tektoncd-cli
```

> 安装完成后可以通过如下命令验证 CLI 是否安装成功：

```
$ tkn version
Client version: 0
Pipeline version: v0
```

> 此外，还可以安装一个 Tekton 提供的一个 Dashboard，我们可以通过 Dashboard 查看 Tekton 整个任务的构建过程，直接执行下面的命令直接安装即可：

```
$ kubectl apply -f kubectl apply -f https://www.qikqiak.com/k8strain/devops/manifests/tekton/dashboard.yaml
```

> 安装完成后我们可以通过 Dashboard 的 Service 的 NodePort 来访问应用。

> ![](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200526103227.png)

### 概念

> Tekton 为 Kubernetes 提供了多种 CRD 资源对象，可用于定义我们的流水线。

> ![](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200526100411.png)

> 主要有以下几个资源对象：

*   Task：表示执行命令的一系列步骤，task 里可以定义一系列的 steps，例如编译代码、构建镜像、推送镜像等，每个 step 实际由一个 Pod 执行。
*   ClusterTask：覆盖整个集群的任务，而不是单一的某一个命名空间，这是和 Task 最大的区别，其他基本上一致的。
*   TaskRun：task 只是定义了一个模版，taskRun 才真正代表了一次实际的运行，当然你也可以自己手动创建一个 taskRun，taskRun 创建出来之后，就会自动触发 task 描述的构建任务。
*   Pipeline：一组任务，表示一个或多个 Task、PipelineResource 以及各种定义参数的集合。
*   PipelineRun：类似 task 和 taskRun 的关系，pipelineRun 也表示某一次实际运行的 pipeline，下发一个 pipelineRun CRD 实例到 Kubernetes 后，同样也会触发一次 pipeline 的构建。
*   PipelineResource：表示 pipeline 输入资源，比如 github 上的源码，或者 pipeline 输出资源，例如一个容器镜像或者构建生成的 jar 包等。

### 示例

> 在这里我们使用一个简单的 Golang 应用，可以在仓库 https://github.com/cnych/tekton-demo 下面获取应用程序代码，测试以及 Dockerfile 文件。

> 首先第一个任务就是 Clone 应用程序代码进行测试，创建一个 `task-test.yaml` 的资源文件，内容如下所示：

```
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: test
spec:
  resources:
    inputs:
    - name: repo
      type: git
  steps:
  - name: run-test
    image: golang:14-alpine
    workingDir: /workspace/repo
    command: ["go"]
    args: ["test"]
```

> 其中 `resources` 定义了我们的任务中定义的步骤所需的输入内容，这里我们的步骤需要 Clone 一个 Git 仓库作为 `go test` 命令的输入，目前支持 git、pullRequest、image、cluster、storage、cloudevent 等资源。

> Tekton 内置的 git 资源类型，它会自动将代码仓库 Clone 到 

```
/workspace/$input_name
```

 目录中，由于我们这里输入被命名成 repo，所以代码会被 Clone 到 `/workspace/repo` 目录下面。

> 然后下面的 `steps` 就是来定义执行运行测试命令的步骤，这里我们直接在代码的根目录中运行 `go test` 命令即可，需要注意的是命令和参数需要分别定义。

> 定义完成后直接使用 kubectl 创建该任务：

```
$ kubectl apply -f task-test.yaml
task.tekton.dev/test created
```

> 现在我们定义完成了一个建的 Task 任务，但是该任务并不会立即执行，我们必须创建一个 `TaskRun` 引用它并提供所有必需输入的数据才行。这里我们就需要将 git 代码库作为输入，我们必须先创建一个 `PipelineResource` 对象来定义输入信息，创建一个名为 

```
pipelineresource.yaml
```

 的资源清单文件，内容如下所示：

```
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: cnych-tekton-example
spec:
  type: git
  params:
  - name: url
    value: https://github.com/cnych/tekton-demo
  - name: revision
    value: master
```

> 直接创建上面的资源对象即可：

```
$ kubectl apply -f pipelineresource.yaml
pipelineresource.tekton.dev/cnych-tekton-example created
```

> 接下来我们就创建 `TaskRun` 对象了，创建一个名为 `taskrun.yaml` 的文件，内容如下所示：

```
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: testrun
spec:
  taskRef:
    name: test
  resources:
    inputs:
    - name: repo
      resourceRef:
        name: cnych-tekton-example
```

> 这里通过 `taskRef` 引用上面定义的 Task 和 git 仓库作为输入，`resourceRef` 也是引用上面定义的 `PipelineResource` 资源对象。现在我们创建这个资源对象过后，就会开始运行了：

```
$ kubectl apply -f taskrun.yaml
taskrun.tekton.dev/testrun created
```

> 创建后，我们可以通过查看 TaskRun 资源对象的状态来查看构建状态：

```
$ kubectl get taskrun
NAME      SUCCEEDED   REASON    STARTTIME   COMPLETIONTIME
testrun   Unknown     Pending   21s    
$ kubectl get pods   
NAME                             READY   STATUS     RESTARTS   AGE
testrun-pod-mw9bt                0/2     Init:1/2   0          59s
$ kubectl describe pod testrun-pod-mw9bt
......
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age        From                 Message
  ----    ------     ----       ----                 -------
  Normal  Scheduled  <unknown>  default-scheduler    Successfully assigned default/testrun-pod-mw9bt to ydzs-node5
  Normal  Pulling    4m20s      kubelet, ydzs-node5  Pulling image "busybox@sha256:a2490cec4484ee6c1068ba3a05f89934010c85242f736280b35343483b2264b6"
  Normal  Pulled     4m13s      kubelet, ydzs-node5  Successfully pulled image "busybox@sha256:a2490cec4484ee6c1068ba3a05f89934010c85242f736280b35343483b2264b6"
  Normal  Created    4m13s      kubelet, ydzs-node5  Created container working-dir-initializer
  Normal  Started    4m13s      kubelet, ydzs-node5  Started container working-dir-initializer
  Normal  Pulling    4m12s      kubelet, ydzs-node5  Pulling image "cnych/tekton-entrypoint:v0"
  Normal  Pulled     2m13s      kubelet, ydzs-node5  Successfully pulled image "cnych/tekton-entrypoint:v0"
  Normal  Created    2m13s      kubelet, ydzs-node5  Created container place-tools
  Normal  Started    2m12s      kubelet, ydzs-node5  Started container place-tools
  Normal  Pulling    2m12s      kubelet, ydzs-node5  Pulling image "cnych/tekton-git-init:v0"
  Normal  Pulled     33s        kubelet, ydzs-node5  Successfully pulled image "cnych/tekton-git-init:v0"
  Normal  Created    32s        kubelet, ydzs-node5  Created container step-git-source-cnych-tekton-example-d6mcz
  Normal  Started    32s        kubelet, ydzs-node5  Started container step-git-source-cnych-tekton-example-d6mcz
  Normal  Pulling    32s        kubelet, ydzs-node5  Pulling image "golang:14-alpine"
```

> 我们可以通过 `kubectl describe` 命令来查看任务运行的过程，首先就是通过 `initContainer` 中的一个 busybox 镜像将代码 Clone 下来，然后使用任务中定义的镜像来执行命令。

> 当任务执行完成后， Pod 就会变成 Completed 状态了：

```
$ kubectl get pods
NAME                READY   STATUS      RESTARTS   AGE
testrun-pod-mw9bt   0/2     Completed   0          4m27s
$ kubectl get taskrun
NAME      SUCCEEDED   REASON      STARTTIME   COMPLETIONTIME
testrun   True        Succeeded   70s         57s
```

> 我们可以查看容器的日志信息来了解任务的执行结果信息：

```
$ kubectl logs testrun-pod-mw9bt --all-containers
{"level":"info","ts":15887518779908,"caller":"git/git.go:136","msg":"Successfully cloned https://github.com/cnych/tekton-demo @ f840e0c390be9a1a6edad76abbde64e882047f05 (grafted, HEAD, origin/master) in path /workspace/repo"}
{"level":"info","ts":158875188844209,"caller":"git/git.go:177","msg":"Successfully initialized and updated submodules in path /workspace/repo"}
PASS
ok      _/workspace/repo        006s
```

> 我们可以看到我们的测试已经通过了，当然我们也可以使用 Tekton CLI 工具来运行这个任务。Tekton CLI 提供了一种更快、更方便的方式来运行任务。

> 我们不用手动编写 TaskRun 资源清单，可以运行以下命令来执行，该命令引用我们的任务（名为 test），生成一个 TaskRun 资源对象并显示其日志：

```
$ tkn task start test --inputresource repo=cnych-tekton-example --showlog
Taskrun started: test-run-z56lj
Waiting for logs to be available...
[git-source-cnych-tekton-example-8q268] {"level":"info","ts":15887526350016,"caller":"git/git.go:136","msg":"Successfully cloned https://github.com/cnych/tekton-demo @ f840e0c390be9a1a6edad76abbde64e882047f05 (grafted, HEAD, origin/master) in path /workspace/repo"}
[git-source-cnych-tekton-example-8q268] {"level":"info","ts":15887526453426,"caller":"git/git.go:177","msg":"Successfully initialized and updated submodules in path /workspace/repo"}

[run-test] PASS
[run-test] ok   _/workspace/repo    007s
```

### Docker Hub 配置

> 为了能够构建 Docker 镜像，一般来说我们需要使用 Docker 来进行，我们这里是容器，所以可以使用 `Docker In Docker 模式`，但是这种模式安全性不高，除了这种方式之外，我们还可以使用 Google 推出的 Kaniko 工具来进行构建，该工具可以在 Kubernetes 集群中构建 Docker 镜像而无需依赖 Docker 守护进程。

> 使用 Kaniko 构建镜像和 Docker 命令基本上一致，所以我们可以提前设置下 Docker Hub 的登录凭证，方便后续将镜像推送到镜像仓库。登录凭证可以保存到 Kubernetes 的 Secret 资源对象中，创建一个名为 `secret.yaml` 的文件，内容如下所示:

```
apiVersion: v1
kind: Secret
metadata:
  name: docker-auth
  annotations:
    tekton.dev/docker-0: https://index.docker.io/v1/
type: kubernetes.io/basic-auth
stringData:
    username: myusername
    password: mypassword
```

> 记得将 myusername 和 mypassword 替换成你的 Docker Hub 登录凭证。

> 我们这里在 Secret 对象中添加了一个 `tekton.dev/docker-0` 的 annotation，该注解信息是用来告诉 Tekton 这些认证信息所属的 Docker 镜像仓库。

> 然后创建一个 ServiceAccount 对象来使用上面的 `docker-auth` 这个 Secret 对象，创建一个名为 `serviceaccount.yaml` 的文件，内容如下所示：

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-sa
secrets:
- name: docker-auth
```

> 然后直接创建上面两个资源对象即可：

```
$ kubectl apply -f secret.yaml 
secret/docker-auth created
$ kubectl apply -f serviceaccount.yaml 
serviceaccount/build-sa created
```

> 创建完成后，我们就可以在运行 Tekton 的任务或者流水线的时候使用上面的 build-sa 这个 `ServiceAccount` 对象来进行 Docker Hub 的登录认证了。

### 创建镜像任务

> 现在我们创建一个 Task 任务来构建并推送 Docker 镜像，我们这里使用的示例应用根目录下面已经包含了一个 `Dockerfile` 文件了，所以我们直接 Clone 代码就可以获得：

```
FROM golang:14-alpine

WORKDIR /go/src/app
COPY . .

RUN go get -d -v ./...
RUN go install -v ./...

CMD ["app"]
```

> 创建一个名为 `task-build-push.yaml` 的文件，文件内容如下所示：

```
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: build-and-push
spec:
  resources:
    inputs:
    - name: repo
      type: git
  steps:
  - name: build-and-push
    image: cnych/kaniko-executor:v0
    env:
    - name: DOCKER_CONFIG
      value: /tekton/home/.docker
    command:
    - /kaniko/executor
    - --dockerfile=Dockerfile
    - --context=/workspace/repo
    - --destination=cnych/tekton-test:latest
```

> 和前面的测试任务类似，这里我们同样将 git 作为输入，也只定义了一个名为 build-and-push 的步骤，Kaniko 会在同一个命令中`构建并推送`，所以不需要多个步骤了，执行的命令就是 `/kaniko/executor`，通过 `--dockerfile` 指定 Dockerfile 路径，`--context` 指定构建上下文，我们这里当然就是项目的根目录了，然后 `--destination` 参数指定最终我们的镜像名称。其实 Tekton 也支持参数化的形式，这样就可以避免我们这里直接写死了，不过由于我们都还是初学者，为了避免复杂，我们直接就直接硬编码了。

> 此外我们定义了一个名为 `DOCKER_CONFIG` 的环境变量，这个变量是用于 Kaniko 去查找 Docker 认证信息的。

> 同样直接创建上面的资源对象即可：

```
$ kubectl apply -f task-build-push.yaml 
task.tekton.dev/build-and-push created
```

> 创建了 Task 任务过后，要想真正去执行这个任务，我们就需要创建一个对应的 TaskRun 资源对象。

### 执行任务

> 和前面一样，现在我们来创建一个 TaskRun 对象来触发任务，不同之处在于我们需要指定 Task 时需要的 ServiceAccount 对象。创建一个名为 

```
taskrun-build-push.yaml
```

 的文件，内容如下所示：

```
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: build-and-push
spec:
  serviceAccountName: build-sa
  taskRef:
    name: build-and-push
  resources:
    inputs:
    - name: repo
      resourceRef:
        name: cnych-tekton-example
```

> 注意这里我们通过 `serviceAccountName` 属性指定了 Docker 认证信息的 `ServiceAccount` 对象，然后通过 `taskRef` 引用我们的任务，以及下面的 `resourceRef` 关联第一部分我们声明的输入资源。

> 然后直接创建这个资源对象即可：

```
$ kubectl apply -f taskrun-build-push.yaml 
taskrun.tekton.dev/build-and-push created
```

> 创建完成后就会触发任务执行了，我们可以通过查看 Pod 对象状态来了解进度：

```
$ kubectl get pods 
NAME                             READY   STATUS      RESTARTS   AGE
build-and-push-pod-pv6mv         0/2     Init:0/2    0          32s
$ kubectl get taskrun
NAME             SUCCEEDED   REASON      STARTTIME   COMPLETIONTIME
build-and-push   Unknown     Pending     108s
```

> 现在任务执行的 Pod 还在初始化容器阶段，我们可以看到 TaskRun 的状态处于 Pending，隔一会儿正常构建就会成功了，我们可以查看构建任务的 Pod 日志信息：

```
$ kubectl get pods
NAME                             READY   STATUS      RESTARTS   AGE
build-and-push-pod-pv6mv         0/2     Completed   0          16m
$  kubectl logs -f build-and-push-pod-pv6mv --all-containers
{"level":"info","ts":158890721187525,"caller":"creds-init/main.go:44","msg":"Credentials initialized."}
{"level":"info","ts":15889074917656,"caller":"git/git.go:136","msg":"Successfully cloned https://github.com/cnych/tekton-demo @ f840e0c390be9a1a6edad76abbde64e882047f05 (grafted, HEAD, origin/master) in path /workspace/repo"}
{"level":"info","ts":158890740128217,"caller":"git/git.go:177","msg":"Successfully initialized and updated submodules in path /workspace/repo"}
INFO[0010] Retrieving image manifest golang:14-alpine
INFO[0016] Retrieving image manifest golang:14-alpine
INFO[0020] Built cross stage deps: map[]
INFO[0020] Retrieving image manifest golang:14-alpine
INFO[0023] Retrieving image manifest golang:14-alpine
INFO[0029] Executing 0 build triggers
INFO[0029] Unpacking rootfs as cmd COPY . . requires it.
INFO[0259] WORKDIR /go/src/app
INFO[0259] cmd: workdir
INFO[0259] Changed working directory to /go/src/app
INFO[0259] Creating directory /go/src/app
INFO[0259] Resolving 1 paths
INFO[0259] Taking snapshot of files...
INFO[0259] COPY . .
INFO[0259] Resolving 56 paths
INFO[0259] Taking snapshot of files...
INFO[0259] RUN go get -d -v ./...
INFO[0259] Taking snapshot of full filesystem...
INFO[0265] Resolving 11667 paths
INFO[0272] cmd: /bin/sh
INFO[0272] args: [-c go get -d -v ./...]
INFO[0272] Running: [/bin/sh -c go get -d -v ./...]
INFO[0272] Taking snapshot of full filesystem...
INFO[0275] Resolving 11667 paths
INFO[0280] RUN go install -v ./...
INFO[0280] cmd: /bin/sh
INFO[0280] args: [-c go install -v ./...]
INFO[0280] Running: [/bin/sh -c go install -v ./...]
app
INFO[0281] Taking snapshot of full filesystem...
INFO[0284] Resolving 11666 paths
INFO[0288] CMD ["app"]
$ kubectl get taskrun
NAME             SUCCEEDED   REASON      STARTTIME   COMPLETIONTIME
build-and-push   True        Succeeded   15m         2m24s
```

> 我们可以看到 TaskRun 任务已经执行成功了。 这个时候其实我们可以在 Docker Hub 上找到我们的镜像了，当然也可以直接使用这个镜像进行测试： ![](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200526110119.png)

### 创建流水线

> 到这里前面我们的两个任务 test 和 build-and-push 都已经完成了，我们还可以创建一个流水线来将这两个任务组织起来，首先运行 test 任务，如果通过了再执行后面的 build-and-push 这个任务。

> 创建一个名为 `pipeline.yaml` 的文件，内容如下所示：

```
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: test-build-push
spec:
  resources:
  - name: repo
    type: git
  tasks:
  # 运行应用测试
  - name: test
    taskRef:
      name: test
    resources:
      inputs:
      - name: repo      # Task 输入名称 
        resource: repo  # Pipeline 资源名称
  # 构建并推送 Docker 镜像
  - name: build-and-push
    taskRef:
      name: build-and-push
    runAfter:
    - test              # 测试任务执行之后
    resources:
      inputs:
      - name: repo      # Task 输入名称 
        resource: repo  # Pipeline 资源名称
```

> 首先我们需要定义流水线需要哪些资源，可以是输入或者输出的资源，在这里我们只有一个输入，那就是命名为 repo 的应用程序源码的 GitHub 仓库。接下来定义任务，每个任务都通过 taskRef 进行引用，并传递任务需要的输入参数。

> 同样直接创建这个资源对象即可：

```
$ kubectl apply -f pipeline.yaml 
pipeline.tekton.dev/test-build-push created
```

> 前面我们提到过和通过创建 TaskRun 去触发 Task 任务类似，我们可以通过创建一个 `PipelineRun` 对象来运行流水线。这里我们创建一个名为 `pipelinerun.yaml` 的 PipelineRun 对象来运行流水线，文件内容如下所示：

```
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: test-build-push-run
spec:
  serviceAccountName: build-sa
  pipelineRef:
    name: test-build-push
  resources:
  - name: repo
    resourceRef:
      name: cnych-tekton-example
```

> 定义方式和 TaskRun 几乎一样，通过 serviceAccountName 属性指定 ServiceAccount 对象，`pipelineRef` 关联流水线对象。同样直接创建这个资源，创建后就会触发我们的流水线任务了：

```
$ kubectl apply -f pipelinerun.yaml 
pipelinerun.tekton.dev/test-build-push-run created
$ kubectl get pods | grep test-build-push-run
test-build-push-run-build-and-push-xl7wp-pod-hdnbl   0/2     Completed   0          5m27s
test-build-push-run-test-4s6qh-pod-tkwzk             0/2     Completed   0          6m5s
$ kubectl logs -f test-build-push-run-build-and-push-xl7wp-pod-hdnbl --all-containers
{"level":"info","ts":15889089442572,"caller":"git/git.go:136","msg":"Successfully cloned https://github.com/cnych/tekton-demo @ f840e0c390be9a1a6edad76abbde64e882047f05 (grafted, HEAD, origin/master) in path /workspace/repo"}
{"level":"info","ts":15889089577377,"caller":"git/git.go:177","msg":"Successfully initialized and updated submodules in path /workspace/repo"}
{"level":"info","ts":15889089469531,"caller":"creds-init/main.go:44","msg":"Credentials initialized."}
INFO[0004] Retrieving image manifest golang:14-alpine
......
app
INFO[0281] Taking snapshot of full filesystem...
INFO[0287] Resolving 11666 paths
INFO[0291] CMD ["app"]
$ kubectl get taskrun |grep test-build-push-run
test-build-push-run-build-and-push-xl7wp   True        Succeeded   6m21s       65s
test-build-push-run-test-4s6qh             True        Succeeded   6m58s       6m21s
```

> 到这里证明我们的流水线执行成功了。我们将 Tekton 安装在 Kubernetes 集群上，定义了一个 Task，并通过 YAML 清单和 Tekton CLI 创建 TaskRun 对其进行了测试。我们创建了由两个任务组成的 Tektok 流水线，第一个任务是从 GitHub 克隆代码并运行应用程序测试，第二个任务是构建一个 Docker 镜像并将其推送到 Docker Hub 上。到这里我们就完成了使用 Tekton 创建 CI/CD 流水线的一个简单示例，不过这个示例还比较简单，接下来我们再通过一个稍微复杂点的应用来完成我们前面的 Jenkins 流水线。

### 触发器和事件监听器

> 上面我们都是通过创建一个 TaskRun 或者一个 PipelineRun 对象来触发任务。但是在实际的工作中更多的是开发人员提交代码过后来触发任务，这个时候就需要用到 Tekton 里面的 Triggers 了。

> Triggers 同样通过下面的几个 CRD 对象对 Tekton 进行了一些扩展：

*   TriggerTemplate: 创建资源的模板，比如用来创建 PipelineResource 和 PipelineRun
*   TriggerBinding: 校验事件并提取相关字段属性
*   ClusterTriggerBinding: 和 TriggerBinding 类似，只是是全局的
*   EventListener: 连接 TriggerBinding 和 TriggerTemplate 到事件接收器，使用从各个 TriggerBinding 中提取的参数来创建 TriggerTemplate 中指定的 resources，同样通过 `interceptor` 字段来指定外部服务对事件属性进行预处理

> ![Tekton Triggers](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200527093100.png)

> 同样要使用 Tekton Triggers 就需要安装对应的控制器，可以直接通过 tektoncd/triggers 的 GitHub 仓库说明进行安装，如下所示的命令：

```
$ kubectl apply --filename https://storage.googleapis.com/tekton-releases/triggers/latest/release.yaml
```

> 同样由于官方使用的镜像是 `gcr` 的镜像，所以正常情况下我们是获取不到的，如果你的集群由于某些原因获取不到镜像，可以使用下面的资源清单文件，我已经将镜像替换成了 Docker Hub 上面的镜像：

```
$ kubectl apply -f https://www.qikqiak.com/k8strain/devops/manifests/tekton/trigger.yaml
```

> 可以使用如下命令查看 Triggers 的相关组件安装状态，直到都为 `Running` 状态：

```
$ kubectl get pods --namespace tekton-pipelines                                       
NAME                                           READY   STATUS    RESTARTS   AGE
tekton-dashboard-69656879d9-7bbkl              1/1     Running   0          2d6h
tekton-pipelines-controller-67f4dc98d8-pgxrq   1/1     Running   0          22d
tekton-pipelines-webhook-59df55445c-jw76v      1/1     Running   0          22d
tekton-triggers-controller-779fc9f557-vj6xs    1/1     Running   0          17m
tekton-triggers-webhook-c77f8dbd6-ctmlm        1/1     Running   0          17m
```

> 现在我们来将前面的 Jenkins Pipeline 流水线转换成使用 Tekton 来构建，代码我们已经推送到了私有仓库 GitLab，地址为：

```
http://git.k8s.local/course/devops-demo.git
```

。

> 首先我们需要完成触发器的配置，当我们提交源代码到 GitLab 的时候，需要触发 Tekton 的任务运行。所以首先需要完成这个触发器。这里就可以通过 `EventListener` 这个资源对象来完成，创建一个名为 `gitlab-listener` 的 EventListener 资源对象，文件内容如下所示：(gitlab-push-listener.yaml)

```
apiVersion: triggers.tekton.dev/v1alpha1
kind: EventListener
metadata:
  name: gitlab-listener
spec:
  serviceAccountName: tekton-triggers-gitlab-sa
  triggers:
  - name: gitlab-push-events-trigger
    interceptors:
    - gitlab:
        secretRef:  # 引用 gitlab-secret 的 Secret 对象中的 secretToken 的值
          secretName: gitlab-secret 
          secretKey: secretToken
        eventTypes:
        - Push Hook  # 只接收 GitLab Push 事件
    bindings:
    - name: gitlab-push-binding  # TriggerBinding 对象
    template:
      name: gitlab-echo-template  # TriggerTemplate 对象
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: el-gitlab-listener
spec:
  routes:
  - match: Host(`tekton-trigger.k8s.local`)
    kind: Rule
    services:
    - name: el-gitlab-listener  # 关联 EventListener 服务
      port: 8080
```

> 由于 `EventListener` 创建完成后会生成一个 Listener 的服务，用来对外暴露用于接收事件响应，比如上面我们创建的对象名为 `gitlab-listener`，创建完成后会生成一个名为 `el-gitlab-listener` 的 Service 对象，这里我们通过 IngressRoute 对象来暴露该服务，当然直接修改 Service 为 NodePort 类型也可以，如果 GitLab 也在集群内部，当然我们用 Service 的 DNS 形式来访问 `EventListener` 当然也可以。

> 另外需要注意的是在上面的 `EventListener` 对象中我们添加了 `interceptors` 属性，其中有一个内置的 `gitlab` 属性可以用来配置 GitLab 相关的信息，比如配置 WebHook 的 `Secret Token`，可以通过 Secret 对象引入进来：

```
interceptors:
- gitlab:
    secretRef:  # 引用 gitlab-secret 的 Secret 对象中的 secretToken 的值
      secretName: gitlab-secret 
      secretKey: secretToken
    eventTypes:
    - Push Hook  # 只接收 GitLab Push 事件
```

> 对应的 Secret 资源对象如下所示，一个用于 WebHook 的 `Secret Token`，另外一个是用于 GitLab 登录认证使用的：(secret.yaml)

```
apiVersion: v1
kind: Secret
metadata:
  name: gitlab-secret
type: Opaque
stringData:
  secretToken: "1234567"
---
apiVersion: v1
kind: Secret
metadata:
  name: gitlab-auth
  annotations:
    tekton.dev/git-0: http://git.k8s.local
type: kubernetes.io/basic-auth
stringData:
  username: myusername
  password: mypassword
```

> 由于 `EventListener` 对象需要访问其他资源对象，所以需要声明 RBAC，如下所示：（rbac.yaml）

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tekton-triggers-gitlab-sa
secrets:
- name: gitlab-secret
- name: gitlab-auth
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: tekton-triggers-gitlab-minimal
rules:
  # Permissions for every EventListener deployment to function
  - apiGroups: ["triggers.tekton.dev"]
    resources: ["eventlisteners", "triggerbindings", "triggertemplates"]
    verbs: ["get"]
  - apiGroups: [""]
    # secrets are only needed for Github/Gitlab interceptors, serviceaccounts only for per trigger authorization
    resources: ["configmaps", "secrets", "serviceaccounts"]
    verbs: ["get", "list", "watch"]
  # Permissions to create resources in associated TriggerTemplates
  - apiGroups: ["tekton.dev"]
    resources: ["pipelineruns", "pipelineresources", "taskruns"]
    verbs: ["create"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: tekton-triggers-gitlab-binding
subjects:
  - kind: ServiceAccount
    name: tekton-triggers-gitlab-sa
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: tekton-triggers-gitlab-minimal
```

> 然后接下来就是最重要的 `TriggerBinding` 和 `TriggerTemplate` 对象了，我们在上面的 `EventListener` 对象中将两个对象组合在一起：

```
bindings:
- name: gitlab-push-binding  # TriggerBinding 对象
template:
  name: gitlab-echo-template  # TriggerTemplate 对象
```

> 这样就可以将 `TriggerBinding` 中的参数传递到 `TriggerTemplate` 对象中进行模板化。比如这里我们定义一个如下所示的 `TriggerBinding` 对象：

```
apiVersion: triggers.tekton.dev/v1alpha1
kind: TriggerBinding
metadata:
  name: gitlab-push-binding
spec:
  params:
  - name: gitrevision
    value: $(body.checkout_sha)
  - name: gitrepositoryurl
    value: $(body.repository.git_http_url)
```

> 这里需要注意的是参数的值我们是通过读取 GitLab WebHook 发送过来的数据值，通过 `$()` 包裹的 JSONPath 表达式来提取的，关于表达式的更多用法可以查看官方文档说明，至于能够提取哪些参数值，则可以查看 WebHook 的说明，比如这里我们是 GitLab Webhook 的 Push Hook，对应的请求体数据如下所示：

```
{
  "object_kind": "push",
  "before": "95790bf891e76fee5e1747ab589903a6a1f80f22",
  "after": "da1560886d4f094c3e6c9ef40349f7d38b5d27d7",
  "ref": "refs/heads/master",
  "checkout_sha": "da1560886d4f094c3e6c9ef40349f7d38b5d27d7",
  "user_id": 4,
  "user_name": "John Smith",
  "user_username": "jsmith",
  "user_email": "john@example.com",
  "user_avatar": "https://s.gravatar.com/avatar/d4c74594d841139328695756648b6bd6?s=8://s.gravatar.com/avatar/d4c74594d841139328695756648b6bd6?s=80",
  "project_id": 15,
  "project":{
    "id": 15,
    "name":"Diaspora",
    "description":"",
    "web_url":"http://example.com/mike/diaspora",
    "avatar_url":null,
    "git_ssh_url":"git@example.com:mike/diaspora.git",
    "git_http_url":"http://example.com/mike/diaspora.git",
    "namespace":"Mike",
    "visibility_level":0,
    "path_with_namespace":"mike/diaspora",
    "default_branch":"master",
    "homepage":"http://example.com/mike/diaspora",
    "url":"git@example.com:mike/diaspora.git",
    "ssh_url":"git@example.com:mike/diaspora.git",
    "http_url":"http://example.com/mike/diaspora.git"
  },
  "repository":{
    "name": "Diaspora",
    "url": "git@example.com:mike/diaspora.git",
    "description": "",
    "homepage": "http://example.com/mike/diaspora",
    "git_http_url":"http://example.com/mike/diaspora.git",
    "git_ssh_url":"git@example.com:mike/diaspora.git",
    "visibility_level":0
  },
  "commits": [
    {
      "id": "b6568db1bc1dcd7f8b4d5a946b0b91f9dacd7327",
      "message": "Update Catalan translation to e38cb\\n\\nSee https://gitlab.com/gitlab-org/gitlab for more information",
      "title": "Update Catalan translation to e38cb",
      "timestamp": "2011-12-12T14:27:31+02:00",
      "url": "http://example.com/mike/diaspora/commit/b6568db1bc1dcd7f8b4d5a946b0b91f9dacd7327",
      "author": {
        "name": "Jordi Mallach",
        "email": "jordi@softcatala.org"
      },
      "added": ["CHANGELOG"],
      "modified": ["app/controller/application.rb"],
      "removed": []
    }
  ],
  "total_commits_count": 4
}
```

> 请求体中的任何属性都可以提取出来，作为 `TriggerBinding` 的参数，如果是其他的 Hook 事件，对应的请求体结构可以查看 GitLab 文档说明。

> 这样我们就可以在 `TriggerTemplate` 对象中通过参数来读取上面 `TriggerBinding` 中定义的参数值了，定义一个如下所示的 `TriggerTemplate` 对象，声明一个 `TaskRun` 的模板，定义的 Task 任务也非常简单，只需要在容器中打印出代码的目录结构即可：

```
apiVersion: triggers.tekton.dev/v1alpha1
kind: TriggerTemplate
metadata:
  name: gitlab-echo-template
spec:
  params:  # 定义参数，和 TriggerBinding 中的保持一致
  - name: gitrevision
  - name: gitrepositoryurl
  resourcetemplates:
  - apiVersion: tekton.dev/v1alpha1
    kind: TaskRun  # 定义 TaskRun 模板
    metadata:
      generateName: gitlab-run-  # TaskRun 名称前缀
    spec:
      serviceAccountName: tekton-triggers-gitlab-sa
      taskSpec:  # Task 任务声明
        inputs:
          resources:  # 定义一个名为 source 的 git 输入资源
          - name: source
            type: git
        steps:
        - name: show-path
          image: ubuntu  # 定义一个执行步骤，列出代码目录结构
          script: |
            #! /bin/bash
            ls -la $(inputs.resources.source.path)
      inputs:  # 声明具体的输入资源参数
        resources:
        - name: source  # 和 Task 中的资源名保持一直
          resourceSpec:  # 资源声明
            type: git
            params:
            - name: revision
              value: $(params.gitrevision)  # 读取参数值
            - name: url
              value: $(params.gitrepositoryurl)
```

> 定义完过后，直接创建上面的资源对象，创建完成后会自动生成 `EventListener` 的 Pod 和 Service 对象：

```
$ kubectl get svc -l eventlistener=gitlab-listener
NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
el-gitlab-listener   ClusterIP   1130   <none>        8080/TCP   91m
$ kubectl get pod -l eventlistener=gitlab-listener
NAME                                  READY   STATUS    RESTARTS   AGE
el-gitlab-listener-6bff9b55bc-qdpjh   1/1     Running   0          91m
$ kubectl get ingressroute
NAME                 AGE
el-gitlab-listener   74m
```

> 接下来我们就可以到 GitLab 的项目中配置 WebHook，注意需要配置 `Secret Token`，我们在上面的 Secret 对象中声明过：

> ![Secret Token](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200528205104.png)

> 创建完成后，我们可以测试下该 WebHook 的 `Push events` 事件，直接点击测试即可，正常会返回 

```
Hook executed successfully: HTTP 201
```

 的提示信息，这个时候在 Kubernetes 集群中就会出现如下所示的任务 Pod：

```
$ kubectl get pod -l triggers.tekton.dev/eventlistener=gitlab-listener
NAME                         READY   STATUS      RESTARTS   AGE
gitlab-run-2jfcf-pod-8smb7   0/2     Completed   0          5m22s
$ kubectl get taskrun -l triggers.tekton.dev/eventlistener=gitlab-listener
NAME               SUCCEEDED   REASON      STARTTIME   COMPLETIONTIME
gitlab-run-2jfcf   True        Succeeded   12h         12h
$ kubectl logs -f gitlab-run-2jfcf-pod-8smb7 --all-containers
{"level":"info","ts":159071818466494,"caller":"git/git.go:136","msg":"Successfully cloned http://git.k8s.local/course/devops-demo.git @ e8b2dd4cac9bfaa79f1998215b4c3fd0e98ea84d (grafted, HEAD) in path /workspace/source"}
{"level":"info","ts":159071819744687,"caller":"git/git.go:177","msg":"Successfully initialized and updated submodules in path /workspace/source"}
{"level":"info","ts":159071798761587,"caller":"creds-init/main.go:44","msg":"Credentials initialized."}
total 36
drwxr-xr-x 4 root root  136 May 29 10:08 .
drwxrwxrwx 3 root root   19 May 29 10:08 ..
drwxr-xr-x 8 root root 4096 May 29 10:08 .git
-rw-r--r-- 1 root root  192 May 29 10:08 .gitignore
-rw-r--r-- 1 root root  376 May 29 10:08 Dockerfile
-rw-r--r-- 1 root root 4788 May 29 10:08 Jenkinsfile
-rw-r--r-- 1 root root   42 May 29 10:08 README.md
-rw-r--r-- 1 root root   97 May 29 10:08 go.mod
-rw-r--r-- 1 root root 3370 May 29 10:08 go.sum
drwxr-xr-x 3 root root   96 May 29 10:08 helm
-rw-r--r-- 1 root root  444 May 29 10:08 main.go
```

> 到这里我们就完成了通过 GitLab 的 Push 事件来触发 Tekton 的一个任务。