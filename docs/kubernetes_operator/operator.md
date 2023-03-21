
## Operator

> `Operator` 就可以看成是 CRD 和 Controller 的一种组合特例，Operator 是一种思想，它结合了特定领域知识并通过 CRD 机制扩展了 Kubernetes API 资源，使用户管理 Kubernetes 的内置资源（Pod、Deployment等）一样创建、配置和管理应用程序，Operator 是一个特定的应用程序的控制器，通过扩展 Kubernetes API 资源以代表 Kubernetes 用户创建、配置和管理复杂应用程序的实例，通常包含资源模型定义和控制器，通过 `Operator` 通常是为了实现某种特定软件（通常是有状态服务）的自动化运维。

> 我们完全可以通过上面的方式编写一个 CRD 对象，然后去手动实现一个对应的 Controller 就可以实现一个 Operator，但是我们也发现从头开始去构建一个 CRD 控制器并不容易，需要对 Kubernetes 的 API 有深入了解，并且 RBAC 集成、镜像构建、持续集成和部署等都需要很大工作量。为了解决这个问题，社区就推出了对应的简单易用的 Operator 框架，比较主流的是 kubebuilder 和 Operator Framework，这两个框架的使用基本上差别不大，我们可以根据自己习惯选择一个即可，我们这里使用 `Operator Framework` 来给大家简要说明下 Operator 的开发。

### Operator Framework

> `Operator Framework` 是 CoreOS 开源的一个用于快速开发 Operator 的工具包，该框架包含两个主要的部分：

*   Operator SDK: 无需了解复杂的 Kubernetes API 特性，即可让你根据你自己的专业知识构建一个 Operator 应用。
*   Operator Lifecycle Manager（OLM）: 帮助你安装、更新和管理跨集群的运行中的所有 Operator（以及他们的相关服务）

> ![operator sdk](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/operator-sdk-lifecycle.png)

> Operator SDK 提供以下工作流来开发一个新的 Operator：

1.  使用 SDK 创建一个新的 Operator 项目
2.  通过添加自定义资源（CRD）定义新的资源 API
3.  指定使用 SDK API 来 watch 的资源
4.  定义 Operator 的协调（reconcile）逻辑
5.  使用 Operator SDK 构建并生成 Operator 部署清单文件

### 示例

> 我们平时在部署一个简单的 Webserver 到 Kubernetes 集群中的时候，都需要先编写一个 Deployment 的控制器，然后创建一个 Service 对象，通过 Pod 的 label 标签进行关联，最后通过 Ingress 或者 type=NodePort 类型的 Service 来暴露服务，每次都需要这样操作，是不是略显麻烦，我们就可以创建一个自定义的资源对象，通过我们的 CRD 来描述我们要部署的应用信息，比如镜像、服务端口、环境变量等等，然后创建我们的自定义类型的资源对象的时候，通过控制器去创建对应的 Deployment 和 Service，是不是就方便很多了，相当于我们用一个资源清单去描述了 Deployment 和 Service 要做的两件事情。

> 这里我们将创建一个名为 AppService 的 CRD 资源对象，然后定义如下的资源清单进行应用部署：

```
apiVersion: app.example.com/v1
kind: AppService
metadata:
  name: nginx-app
spec:
  size: 2
  image: nginx:9
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30002
```

> 通过这里的自定义的 AppService 资源对象去创建副本数为2的 Pod，然后通过 `nodePort=30002` 的端口去暴露服务，接下来我们就来一步一步的实现我们这里的这个简单的 Operator 应用。

### 开发环境

> 要开发 Operator 自然 Kubernetes 集群是少不了的，还需要 Golang 的环境，这里的安装就不多说了。然后需要安装 `operator-sdk`，operator sdk 安装方法非常多，我们可以直接在 github 上面下载需要使用的版本，然后放置到 PATH 环境下面即可，当然也可以将源码 clone 到本地手动编译安装即可，如果你是 Mac，当然还可以使用常用的 brew 工具进行安装：

```
$ brew install operator-sdk
......
$ operator-sdk version
operator-sdk version: "v0", commit: "2445fcda834ca4b7cf0d6c38fba6317fb219b469", go version: "go1
.3 darwin/amd64"
$ go version
go version go3 darwin/amd64
```

> 我们这里使用的 sdk 版本是 `v0.12.0`，其他安装方法可以参考文档：https://github.com/operator-framework/operator-sdk/blob/master/doc/user/install-operator-sdk.md

### 创建项目

> 环境准备好了，接下来就可以使用 `operator-sdk` 直接创建一个新的项目了，命令格式为：`operator-sdk new`。

> 按照上面我们预先定义的 CRD 资源清单，我们这里可以这样创建：

```
# 创建项目目录
$ mkdir -p operator-learning && cd operator-learning
$ export GO111MODULE=on  # 使用gomodules包管理工具
$ export GOPROXY="https://goproxy.io" # 使用包代理，加速
# 使用 sdk 创建一个名为 opdemo 的 operator 项目，如果在 GOPATH 之外需要指定 repo 参数
$ operator-sdk new opdemo --repo github.com/cnych/opdemo
INFO[0000] Creating new Go operator 'opdemo'.           
INFO[0000] Created go.mod                               
INFO[0000] Created tools.go                             
INFO[0000] Created cmd/manager/main.go                  
INFO[0000] Created build/Dockerfile                     
INFO[0000] Created build/bin/entrypoint                 
INFO[0000] Created build/bin/user_setup                 
INFO[0000] Created deploy/service_account.yaml          
INFO[0000] Created deploy/role.yaml                     
INFO[0000] Created deploy/role_binding.yaml             
INFO[0000] Created deploy/operator.yaml                 
INFO[0000] Created pkg/apis/apis.go                     
INFO[0000] Created pkg/controller/controller.go         
INFO[0000] Created version/version.go                   
INFO[0000] Created .gitignore                           
INFO[0000] Validating project                           
......
INFO[0063] Project validation successful.               
INFO[0063] Project creation complete.  
$ cd opdemo && tree -L 2
.
├── build
│   ├── Dockerfile
│   └── bin
├── cmd
│   └── manager
├── deploy
│   ├── operator.yaml
│   ├── role.yaml
│   ├── role_binding.yaml
│   └── service_account.yaml
├── go.mod
├── go.sum
├── pkg
│   ├── apis
│   └── controller
├── tools.go
└── version
    └── version.go

9 directories, 9 files
```

> 到这里一个全新的 Operator 项目就新建完成了。

### 项目结构

> 使用 `operator-sdk new` 命令创建新的 Operator 项目后，项目目录就包含了很多生成的文件夹和文件。

*   go.mod go.sum — Go Modules 包管理清单，用来描述当前 Operator 的依赖包。
*   cmd - 包含 main.go 文件，使用 operator-sdk API 初始化和启动当前 Operator 的入口。
*   deploy - 包含一组用于在 Kubernetes 集群上进行部署的通用的 Kubernetes 资源清单文件。
*   pkg/apis - 包含定义的 API 和自定义资源（CRD）的目录树，这些文件允许 sdk 为 CRD 生成代码并注册对应的类型，以便正确解码自定义资源对象。
*   pkg/controller - 用于编写所有的操作业务逻辑的地方
*   version - 版本定义
*   build - Dockerfile 定义目录

> 我们主要需要编写的是 `pkg` 目录下面的 api 定义以及对应的 controller 实现。

### 添加 API

> 接下来为我们的自定义资源添加一个新的 API，按照上面我们预定义的资源清单文件，在 Operator 相关根目录下面执行如下命令：

```
$ operator-sdk add api --api-version=app.example.com/v1 --kind=AppService
```

> 添加完成后，我们可以看到类似于下面的这样项目结构： ![operator demo op add api](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/operator-demo-op-add-api.png)

### 添加控制器

> 上面我们添加自定义的 API，接下来可以添加对应的自定义 API 的具体实现 Controller，同样在项目根目录下面执行如下命令：

```
$ operator-sdk add controller --api-version=app.example.com/v1 --kind=AppService
```

> 这样整个 Operator 项目的脚手架就已经搭建完成了，接下来就是具体的实现了。

### 自定义 API

> 打开源文件 

```
pkg/apis/app/v1/appservice_types.go
```

，需要我们根据我们的需求去自定义结构体 `AppServiceSpec`，我们最上面预定义的资源清单中就有 `size`、`image`、`ports` 这些属性，所有我们需要用到的属性都需要在这个结构体中进行定义：

```
type AppServiceSpec struct {
    // INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
    // Important: Run "operator-sdk generate k8s" to regenerate code after modifying this file
    // Add custom validation using kubebuilder tags: https://book.kubebuilder.io/beyond_basics/generating_crd.html
    Size      *int32                      `json:"size"`
    Image     string                      `json:"image"`
    Resources corevResourceRequirements `json:"resources,omitempty"`
    Envs      []corevEnvVar             `json:"envs,omitempty"`
    Ports     []corevServicePort        `json:"ports,omitempty"`
}
```

> 代码中会涉及到一些包名的导入，由于包名较多，所以我们会使用一些别名进行区分，主要的包含下面几个：

```
import (
    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    appv1 "github.com/cnych/opdemo/pkg/apis/app/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)
```

> 这里的 resources、envs、ports 的定义都是直接引用的 `"k8s.io/api/core/v1"` 中定义的结构体，而且需要注意的是我们这里使用的是 `ServicePort`，而不是像传统的 Pod 中定义的 ContanerPort，这是因为我们的资源清单中不仅要描述容器的 Port，还要描述 Service 的 Port。

> 然后一个比较重要的结构体 `AppServiceStatus` 用来描述资源的状态，当然我们可以根据需要去自定义状态的描述，我这里就偷懒直接使用 Deployment 的状态了：

```
type AppServiceStatus struct {
    // INSERT ADDITIONAL STATUS FIELD - define observed state of cluster
    // Important: Run "operator-sdk generate k8s" to regenerate code after modifying this file
    // Add custom validation using kubebuilder tags: https://book.kubebuilder.io/beyond_basics/generating_crd.html
    appsvDeploymentStatus `json:",inline"`
}
```

> 定义完成后，在项目根目录下面执行如下命令：

```
$ operator-sdk generate k8s
INFO[0000] Running deepcopy code-generation for Custom Resource group versions: [app:[v1], ] 
INFO[0011] Code-generation complete.
```

> 该命令是用来根据我们自定义的 API 描述来自动生成一些代码，目录 `pkg/apis/app/v1/` 下面以 `zz_generated` 开头的文件就是自动生成的代码，里面的内容并不需要我们去手动编写。

> 这样我们就算完成了对自定义资源对象的 API 的声明。

### 实现业务逻辑

> 上面 API 描述声明完成了，接下来就需要我们来进行具体的业务逻辑实现了，编写具体的 controller 实现，打开源文件

```
pkg/controller/appservice/appservice_controller.go
```

，需要我们去更改的地方也不是很多，核心的就是`Reconcile` 方法，该方法就是去不断的 watch 资源的状态，然后根据状态的不同去实现各种操作逻辑，核心代码如下：

```
func (r *ReconcileAppService) Reconcile(request reconcile.Request) (reconcile.Result, error) {
    reqLogger := log.WithValues("Request.Namespace", request.Namespace, "Request.Name", request.Name)
    reqLogger.Info("Reconciling AppService")

    // Fetch the AppService instance
    instance := &appvAppService{}
    err := r.client.Get(context.TODO(), request.NamespacedName, instance)
    if err != nil {
        if errors.IsNotFound(err) {
            // Request object not found, could have been deleted after reconcile request.
            // Owned objects are automatically garbage collected. For additional cleanup logic use finalizers.
            // Return and don't requeue
            return reconcile.Result{}, nil
        }
        // Error reading the object - requeue the request.
        return reconcile.Result{}, err
    }

    if instance.DeletionTimestamp != nil {
        return reconcile.Result{}, err
    }

    // 如果不存在，则创建关联资源
    // 如果存在，判断是否需要更新
    //   如果需要更新，则直接更新
    //   如果不需要更新，则正常返回

    deploy := &appsvDeployment{}
    if err := r.client.Get(context.TODO(), request.NamespacedName, deploy); err != nil && errors.IsNotFound(err) {
        // 创建关联资源
        //  创建 Deploy
        deploy := resources.NewDeploy(instance)
        if err := r.client.Create(context.TODO(), deploy); err != nil {
            return reconcile.Result{}, err
        }
        //  创建 Service
        service := resources.NewService(instance)
        if err := r.client.Create(context.TODO(), service); err != nil {
            return reconcile.Result{}, err
        }
        //  关联 Annotations
        data, _ := json.Marshal(instance.Spec)
        if instance.Annotations != nil {
            instance.Annotations["spec"] = string(data)
        } else {
            instance.Annotations = map[string]string{"spec": string(data)}
        }

        if err := r.client.Update(context.TODO(), instance); err != nil {
            return reconcile.Result{}, nil
        }
        return reconcile.Result{}, nil
    }

    oldspec := appvAppServiceSpec{}
    if err := json.Unmarshal([]byte(instance.Annotations["spec"]), oldspec); err != nil {
        return reconcile.Result{}, err
    }

    if !reflect.DeepEqual(instance.Spec, oldspec) {
        // 更新关联资源
        newDeploy := resources.NewDeploy(instance)
        oldDeploy := &appsvDeployment{}
        if err := r.client.Get(context.TODO(), request.NamespacedName, oldDeploy); err != nil {
            return reconcile.Result{}, err
        }
        oldDeploy.Spec = newDeploy.Spec
        if err := r.client.Update(context.TODO(), oldDeploy); err != nil {
            return reconcile.Result{}, err
        }

        newService := resources.NewService(instance)
        oldService := &corevService{}
        if err := r.client.Get(context.TODO(), request.NamespacedName, oldService); err != nil {
            return reconcile.Result{}, err
        }
        oldService.Spec = newService.Spec
        if err := r.client.Update(context.TODO(), oldService); err != nil {
            return reconcile.Result{}, err
        }

        return reconcile.Result{}, nil
    }

    return reconcile.Result{}, nil

}
```

> 上面就是业务逻辑实现的核心代码，逻辑很简单，就是去判断资源是否存在，不存在，则直接创建新的资源，创建新的资源除了需要创建 Deployment 资源外，还需要创建 Service 资源对象，因为这就是我们的需求，当然你还可以自己去扩展，比如在创建一个 Ingress 对象。更新也是一样的，去对比新旧对象的声明是否一致，不一致则需要更新，同样的，两种资源都需要更新的。

> 另外两个核心的方法就是上面的 

```
resources.NewDeploy(instance)
```

 和 

```
resources.NewService(instance)
```

 方法，这两个方法实现逻辑也很简单，就是根据 CRD 中的声明去填充 Deployment 和 Service 资源对象的 Spec 对象即可。

> NewDeploy 方法实现如下：

```
func NewDeploy(app *appvAppService) *appsvDeployment {
    labels := map[string]string{"app": app.Name}
    selector := &metavLabelSelector{MatchLabels: labels}
    return &appsvDeployment{
        TypeMeta: metavTypeMeta{
            APIVersion: "apps/v1",
            Kind:       "Deployment",
        },
        ObjectMeta: metavObjectMeta{
            Name:      app.Name,
            Namespace: app.Namespace,

            OwnerReferences: []metavOwnerReference{
                *metavNewControllerRef(app, schema.GroupVersionKind{
                    Group: vSchemeGroupVersion.Group,
                    Version: vSchemeGroupVersion.Version,
                    Kind: "AppService",
                }),
            },
        },
        Spec: appsvDeploymentSpec{
            Replicas: app.Spec.Size,
            Template: corevPodTemplateSpec{
                ObjectMeta: metavObjectMeta{
                    Labels: labels,
                },
                Spec: corevPodSpec{
                    Containers: newContainers(app),
                },
            },
            Selector: selector,
        },
    }
}

func newContainers(app *vAppService) []corevContainer {
    containerPorts := []corevContainerPort{}
    for _, svcPort := range app.Spec.Ports {
        cport := corevContainerPort{}
        cport.ContainerPort = svcPort.TargetPort.IntVal
        containerPorts = append(containerPorts, cport)
    }
    return []corevContainer{
        {
            Name: app.Name,
            Image: app.Spec.Image,
            Resources: app.Spec.Resources,
            Ports: containerPorts,
            ImagePullPolicy: corevPullIfNotPresent,
            Env: app.Spec.Envs,
        },
    }
}
```

> newService 对应的方法实现如下：

```
func NewService(app *vAppService) *corevService {
    return &corevService {
        TypeMeta: metavTypeMeta {
            Kind: "Service",
            APIVersion: "v1",
        },
        ObjectMeta: metavObjectMeta{
            Name: app.Name,
            Namespace: app.Namespace,
            OwnerReferences: []metavOwnerReference{
                *metavNewControllerRef(app, schema.GroupVersionKind{
                    Group: vSchemeGroupVersion.Group,
                    Version: vSchemeGroupVersion.Version,
                    Kind: "AppService",
                }),
            },
        },
        Spec: corevServiceSpec{
            Type: corevServiceTypeNodePort,
            Ports: app.Spec.Ports,
            Selector: map[string]string{
                "app": app.Name,
            },
        },
    }
}
```

> 这样我们就实现了 AppService 这种资源对象的业务逻辑。

### 调试

> 如果我们本地有一个可以访问的 Kubernetes 集群，我们也可以直接进行调试，在本地用户 `~/.kube/config` 文件中配置集群访问信息，下面的信息表明可以访问 Kubernetes 集群：

```
$ kubectl cluster-info
Kubernetes master is running at https://ydzs-master:6443
KubeDNS is running at https://ydzs-master:6443/api/v1/namespaces/kube-system/services/kube-dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

> 首先，在集群中安装 CRD 对象：

```
$ kubectl apply -f deploy/crds/app.example.com_appservices_crd.yaml 
customresourcedefinition.apiextensions.k8s.io/appservices.app.example.com created
$ kubectl get crd |grep appservice
appservices.app.example.com                      2019-12-19T04:31:12Z
```

> 当我们通过 `kubectl get crd` 命令获取到我们定义的 CRD 资源对象，就证明我们定义的 CRD 安装成功了。其实现在只是 CRD 的这个声明安装成功了，但是我们这个 CRD 的具体业务逻辑实现方式还在我们本地，并没有部署到集群之中，我们可以通过下面的命令来在本地项目中启动 Operator 的调试：

```
$ operator-sdk up local                                                     
operator-sdk up local 
INFO[0000] Running the operator locally.                
INFO[0000] Using namespace default.                     
{"level":"info","ts":15767300743257,"logger":"cmd","msg":"Operator Version: 1"}
{"level":"info","ts":15767300743318,"logger":"cmd","msg":"Go Version: go3"}
{"level":"info","ts":15767300743336,"logger":"cmd","msg":"Go OS/Arch: darwin/amd64"}
{"level":"info","ts":157673007433379,"logger":"cmd","msg":"Version of operator-sdk: v0"}
{"level":"info","ts":15767300744948,"logger":"leader","msg":"Trying to become the leader."}
{"level":"info","ts":15767300744969,"logger":"leader","msg":"Skipping leader election; not running in a cluster."}
......
```

> 上面的命令会在本地运行 Operator 应用，通过 `~/.kube/config` 去关联集群信息，现在我们去添加一个 AppService 类型的资源然后观察本地 Operator 的变化情况，资源清单文件就是我们上面预定义的（deploy/crds/app.example.com_v1_appservice_cr.yaml）:

```
apiVersion: app.example.com/v1
kind: AppService
metadata:
  name: nginx-app
spec:
  size: 2
  image: nginx:9
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30002
```

> 直接创建这个资源对象：

```
$ kubectl apply -f deploy/crds/app.example.com_v1_appservice_cr.yaml 
appservice.app.example.com/nginx-app created
```

> 我们可以看到我们的应用创建成功了，这个时候查看 Operator 的调试窗口会有如下的信息出现：

```
......
{"level":"info","ts":15592074670523,"logger":"controller_appservice","msg":"Reconciling AppService","Request.Namespace":"default","Request.Name":"nginx-app"}
{"level":"info","ts":15592074004226,"logger":"controller_appservice","msg":"Reconciling AppService","Request.Namespace":"default","Request.Name":"nginx-app"}
{"level":"info","ts":15592074004331,"logger":"controller_appservice","msg":"Reconciling AppService","Request.Namespace":"default","Request.Name":"nginx-app"}
{"level":"info","ts":1559207433779,"logger":"controller_appservice","msg":"Reconciling AppService","Request.Namespace":"default","Request.Name":"nginx-app"}
{"level":"info","ts":15592074951193,"logger":"controller_appservice","msg":"Reconciling AppService","Request.Namespace":"default","Request.Name":"nginx-app"}
......
```

> 然后我们可以去查看集群中是否有符合我们预期的资源出现：

```
$ kubectl get AppService
NAME        AGE
nginx-app   2m8s
$ kubectl get deploy
NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
nginx-app                2/2     2            2           2m20s
$ kubectl get svc   
NAME             TYPE           CLUSTER-IP       EXTERNAL-IP             PORT(S)          AGE
nginx-app        NodePort       110     <none>                  80:30002/TCP     2m23s
```

> 看到了吧，我们定义了两个副本（size=2），这里就出现了两个 Pod，还有一个 NodePort=30002 的 Service 对象，我们可以通过该端口去访问下应用：

> ![operator demo op demo](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/operator-demo-op-demo.png)

> 如果应用在安装过程中出现了任何问题，我们都可以通过本地的 Operator 调试窗口找到有用的信息，然后调试修改即可。

> 清理：

```
$ kubectl delete -f deploy/crds/app.example.com_v1_appservice_cr.yaml
$ kubectl delete -f deploy/crds/app.example.com_appservices_crd.yaml
```

### 部署

> 自定义的资源对象现在测试通过了，但是如果我们将本地的 

```
operator-sdk up local
```

 命令终止掉，我们可以猜想到就没办法处理 AppService 资源对象的一些操作了，所以我们需要将我们的业务逻辑实现部署到集群中去。

> 执行下面的命令构建 Operator 应用打包成 Docker 镜像：

```
$ operator-sdk build cnych/opdemo:1
......
Successfully built 29cd605c4ad2
Successfully tagged cnych/opdemo:1
INFO[0041] Operator build complete.
```

> 镜像构建成功后，推送到 docker hub：

```
$ docker push cnych/opdemo:1
```

> 镜像推送成功后，使用上面的镜像地址更新 Operator 的资源清单：

```
$ sed -i 's|REPLACE_IMAGE|cnych/opdemo:1|g' deploy/operator.yaml
# 如果你使用的是 Mac 系统，使用下面的命令
$ sed -i "" 's|REPLACE_IMAGE|cnych/opdemo:1|g' deploy/operator.yaml
```

> 现在 Operator 的资源清单文件准备好了，然后创建对应的 RBAC 的对象：

```
# Setup Service Account
$ kubectl apply -f deploy/service_account.yaml
# Setup RBAC
$ kubectl apply -f deploy/role.yaml
$ kubectl apply -f deploy/role_binding.yaml
```

> 权限相关声明已经完成，接下来安装 CRD 和 Operator：

```
$ kubectl apply -f deploy/crds/app.example.com_appservices_crd.yaml
$ kubectl get crd |grep appservices
appservices.app.example.com                      2019-12-19T05:25:55Z
......
# Deploy the Operator
$ kubectl apply -f deploy/operator.yaml
deployment.apps/opdemo created
$ kubectl get pods
NAME                                      READY   STATUS              RESTARTS   AGE
opdemo-7565595bbc-p7f5c                   1/1     Running             0          5s
```

> 到这里我们的 CRD 和 Operator 实现都已经安装成功了。

> 现在我们再来部署我们的 AppService 资源清单文件，现在的业务逻辑就会在上面的 

```
opdemo-7565595bbc-p7f5c
```

 的 Pod 中去处理了。

```
$ kubectl apply -f deploy/crds/app.example.com_v1_appservice_cr.yaml
appservice.app.example.com/nginx-app created
$ kubectl get appservice
NAME        AGE
nginx-app   18s
$  kubectl get deploy
NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
nginx-app                2/2     2            2           22s
opdemo                   1/1     1            1           5m51s
$  kubectl get svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   1       <none>        443/TCP        76d
nginx-app    NodePort    1182   <none>        80:30002/TCP   29s
opdemo       ClusterIP   1251   <none>        8383/TCP       4m25s
$  kubectl get pods
NAME                                      READY   STATUS    RESTARTS   AGE
nginx-app-76b6449498-ffhgx                1/1     Running   0          32s
nginx-app-76b6449498-wzjq2                1/1     Running   0          32s
opdemo-7565595bbc-p7f5c                   1/1     Running   0          5m59s
$ kubectl describe appservice nginx-app
Name:         nginx-app
Namespace:    default
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"app.example.com/v1","kind":"AppService","metadata":{"annotations":{},"name":"nginx-app","namespace":"default"},"spec":{"ima...
              spec: {"size":2,"image":"nginx:9","resources":{},"ports":[{"protocol":"TCP","port":80,"targetPort":80,"nodePort":30002}]}
API Version:  app.example.com/v1
Kind:         AppService
Metadata:
  Creation Timestamp:  2019-12-19T05:30:26Z
  Generation:          2
  Resource Version:    12364119
  Self Link:           /apis/app.example.com/v1/namespaces/default/appservices/nginx-app
  UID:                 1bb6d5fa-9f94-4eaf-ad55-643568690bab
Spec:
  Image:  nginx:9
  Ports:
    Node Port:    30002
    Port:         80
    Protocol:     TCP
    Target Port:  80
  Resources:
  Size:  2
Events:  <none>
```

> 然后同样的可以通过 30002 这个 NodePort 端口去访问应用，到这里应用就部署成功了。

> 如果需要清理，则直接删除即可：

```
$ kubectl delete -f deploy/crds/app.example.com_v1_appservice_cr.yaml
$ kubectl delete -f deploy/operator.yaml
$ kubectl delete -f deploy/role.yaml
$ kubectl delete -f deploy/role_binding.yaml
$ kubectl delete -f deploy/service_account.yaml
$ kubectl delete -f deploy/crds/app.example.com_appservices_crd.yaml
```