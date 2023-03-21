
## Jenkins Pipeline

> 要实现在 Jenkins 中的构建工作，可以有多种方式，我们这里采用比较常用的 Pipeline 这种方式。Pipeline，简单来说，就是一套运行在 Jenkins 上的工作流框架，将原来独立运行于单个或者多个节点的任务连接起来，实现单个任务难以完成的复杂流程编排和可视化的工作。

> Jenkins Pipeline 有几个核心概念：

*   Node：节点，一个 Node 就是一个 Jenkins 节点，Master 或者 Agent，是执行 Step 的具体运行环境，比如我们之前动态运行的 Jenkins Slave 就是一个 Node 节点
*   Stage：阶段，一个 Pipeline 可以划分为若干个 Stage，每个 Stage 代表一组操作，比如：Build、Test、Deploy，Stage 是一个逻辑分组的概念，可以跨多个 Node
*   Step：步骤，Step 是最基本的操作单元，可以是打印一句话，也可以是构建一个 Docker 镜像，由各类 Jenkins 插件提供，比如命令：sh 'make'，就相当于我们平时 shell 终端中执行 make 命令一样。

> 那么我们如何创建 Jenkins Pipline 呢？

*   Pipeline 脚本是由 Groovy 语言实现的，但是我们没必要单独去学习 Groovy，当然你会的话最好
*   Pipeline 支持两种语法：Declarative(声明式)和 Scripted Pipeline(脚本式)语法
*   Pipeline 也有两种创建方法：可以直接在 Jenkins 的 Web UI 界面中输入脚本；也可以通过创建一个 Jenkinsfile 脚本文件放入项目源码库中
*   一般我们都推荐在 Jenkins 中直接从源代码控制(SCMD)中直接载入 Jenkinsfile Pipeline 这种方法

> 我们这里来给大家快速创建一个简单的 Pipeline，直接在 Jenkins 的 Web UI 界面中输入脚本运行。

*   新建 Job：在 Web UI 中点击 New Item -> 输入名称：pipeline-demo -> 选择下面的 Pipeline -> 点击 OK
*   > 配置：在最下方的 Pipeline 区域输入如下 Script 脚本，然后点击保存。

    ```
    node {
    stage('Clone') {
        echo "Clone Stage"
    }
    stage('Test') {
        echo "Test Stage"
    }
    stage('Build') {
        echo "Build Stage"
    }
    stage('Deploy') {
        echo " Deploy Stage"
    }
    }
    ```

*   > 构建：点击左侧区域的 `Build Now`，可以看到 Job 开始构建了

> 隔一会儿，构建完成，可以点击左侧区域的 ·Console Output·，我们就可以看到如下输出信息：

> console output 我们可以看到上面我们 Pipeline 脚本中的4条输出语句都打印出来了，证明是符合我们的预期的。

> 如果大家对 Pipeline 语法不是特别熟悉的，可以前往输入脚本的下面的链接 `Pipeline Syntax` 中进行查看，这里有很多关于 Pipeline 语法的介绍，也可以自动帮我们生成一些脚本。

### 在 Slave 中构建任务

> 上面我们创建了一个简单的 Pipeline 任务，但是我们可以看到这个任务并没有在 Jenkins 的 Slave 中运行，那么如何让我们的任务跑在 Slave 中呢？还记得上节课我们在添加 Slave Pod 的时候，一定要记住添加的 label 吗？没错，我们就需要用到这个 label，我们重新编辑上面创建的 Pipeline 脚本，给 node 添加一个 label 属性，如下：

```
node('ydzs-jnlp') {
  stage('Clone') {
    echo "Clone Stage"
  }
  stage('Test') {
    echo "Test Stage"
  }
  stage('Build') {
    echo "Build Stage"
  }
  stage('Deploy') {
    echo " Deploy Stage"
  }
}
```

> 我们这里只是给 node 添加了一个 `ydzs-jnlp` 这样的一个label，然后我们保存，构建之前查看下 kubernetes 集群中的 Pod：

```
$ kubectl get pods -n kube-ops
NAME                       READY     STATUS              RESTARTS   AGE
jenkins-7c85b6f4bd-rfqgv   1/1       Running             4          6d
```

> 然后重新触发立刻构建：

```
$ kubectl get pods -n kube-ops
NAME                       READY     STATUS    RESTARTS   AGE
jenkins-7c85b6f4bd-rfqgv   1/1       Running   4          6d
jnlp-0hrrz                 1/1       Running   0          23s
```

> 我们发现多了一个名叫jnlp-0hrrz的 Pod 正在运行，隔一会儿这个 Pod 就不再了：

```
$ kubectl get pods -n kube-ops
NAME                       READY     STATUS    RESTARTS   AGE
jenkins-7c85b6f4bd-rfqgv   1/1       Running   4          6d
```

> 这也证明我们的 Job 构建完成了，同样回到 Jenkins 的 Web UI 界面中查看 Console Output，可以看到如下的信息：

> pipeline demo2 是不是也证明我们当前的任务在跑在上面动态生成的这个 Pod 中，也符合我们的预期。我们回到 Job 的主界面，也可以看到大家可能比较熟悉的 Stage View 界面：

### 部署 Kubernetes 应用

> 上面我们已经知道了如何在 Jenkins Slave 中构建任务了，那么如何来部署一个原生的 Kubernetes 应用呢？ 要部署 Kubernetes 应用，我们就得对我们之前部署应用的流程要非常熟悉才行，我们之前的流程是怎样的：

*   编写代码
*   测试
*   编写 Dockerfile
*   构建打包 Docker 镜像
*   推送 Docker 镜像到仓库
*   编写 Kubernetes YAML 文件
*   更改 YAML 文件中 Docker 镜像 TAG
*   利用 kubectl 工具部署应用

> 我们之前在 Kubernetes 环境中部署一个原生应用的流程应该基本上是上面这些流程吧？现在我们就需要把上面这些流程放入 Jenkins 中来自动帮我们完成(当然编码除外)，从测试到更新 YAML 文件属于 CI 流程，后面部署属于 CD 的流程。如果按照我们上面的示例，我们现在要来编写一个 Pipeline 的脚本，应该怎么编写呢？

```
node('ydzs-jnlp') {
    stage('Clone') {
      echo "Clone Stage"
    }
    stage('Test') {
      echo "Test Stage"
    }
    stage('Build') {
      echo "Build Docker Image Stage"
    }
    stage('Push') {
      echo "Push Docker Image Stage"
    }
    stage('YAML') {
      echo "Change YAML File Stage"
    }
    stage('Deploy') {
      echo "Deploy Stage"
    }
}
```

> 现在我们创建一个流水线的作业，直接使用上面的脚本来构建，同样可以得到正确的结果：

> ![jenkins pipeline demo](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200514102548.png)

> 这里我们来将一个简单 golang 程序，部署到 kubernetes 环境中，代码链接：https://github.com/cnych/drone-k8s-demo。我们将代码推送到我们自己的 GitLab 仓库上去，地址：http://git.k8s.local/course/devops-demo，这样让 Jenkins 和 Gitlab 去进行连接进行 CI/CD。

> 如果按照之前的示例，我们是不是应该像这样来编写 Pipeline 脚本：

> 第一步，clone 代码 第二步，进行测试，如果测试通过了才继续下面的任务 第三步，由于 Dockerfile 基本上都是放入源码中进行管理的，所以我们这里就是直接构建 Docker 镜像了 第四步，镜像打包完成，就应该推送到镜像仓库中吧 第五步，镜像推送完成，是不是需要更改 YAML 文件中的镜像 TAG 为这次镜像的 TAG 第六步，万事俱备，只差最后一步，使用 kubectl 命令行工具进行部署了

> 到这里我们的整个 CI/CD 的流程是不是就都完成了。我们同样可以用上面的我们自定义的一个 jnlp 的镜像来完成我们的整个构建工作，但是我们这里的项目是 golang 代码的，构建需要相应的环境，如果每次需要特定的环境都需要重新去定制下镜像这未免太麻烦了，我们这里来采用一种更加灵活的方式，自定义 `podTemplate`。我们可以直接在 Pipeline 中去自定义 Slave Pod 中所需要用到的容器模板，这样我们需要什么镜像只需要在 `Slave Pod Template` 中声明即可，完全不需要去定义一个庞大的 Slave 镜像了。

> 首先去掉 Jenkins 中 kubernetes 插件中的 Pod Template 的定义，进入页面 http://jenkins.k8s.local/configureClouds/，删除下方的 `Pod Template` -> 保存。

> ![jenkins pod template](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200514104142.png)

> 然后新建一个名为 `devops-demo` 类型为流水线(Pipeline)的任务：

> ![devops demo pipeline](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200514104255.png)

> 然后在这里需要勾选触发远程构建的触发器，其中令牌我们可以随便写一个字符串，然后记住下面的 URL，将 JENKINS_URL 替换成 Jenkins 的地址,我们这里的地址就是：

```
http://jenkins.k8s.local/job/devops-demo/build?token=server321
```

> ![remote trigger](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200514104742.png)

> 然后在下面的流水线区域我们可以选择 `Pipeline script` 然后在下面测试流水线脚本，我们这里选择 

```
Pipeline script from SCM
```

，意思就是从代码仓库中通过 `Jenkinsfile` 文件获取 `Pipeline script` 脚本定义，然后选择 SCM 来源为 Git，在出现的列表中配置上仓库地址 

```
http://git.k8s.local/course/devops-demo.git
```

，由于我们是在一个 Slave Pod 中去进行构建，所以如果使用 SSH 的方式去访问 Gitlab 代码仓库的话就需要频繁的去更新 SSH-KEY，所以我们这里采用直接使用用户名和密码的形式来方式：

> ![devops scm](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200514105725.png)

> 我们可以看到有一个明显的错误 

```
Could not resolve host: git.k8s.local
```

 提示不能解析我们的 GitLab 域名，这是因为我们的域名都是自定义的，我们可以通过在 CoreDNS 中添加自定义域名解析来解决这个问题：

```
$ kubectl edit cm coredns -n kube-system
apiVersion: v1
data:
  Corefile: |
    .:53 {
        log
        errors
        health {
          lameduck 5s
        }
        ready
        hosts {  # 添加自定义域名解析
          111 git.k8s.local
          111 jenkins.k8s.local
          111 harbor.k8s.local
          fallthrough
        }
        kubernetes cluster.local in-addr.arpa iparpa {
           pods insecure
           upstream
           fallthrough in-addr.arpa iparpa
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
kind: ConfigMap
metadata:
  creationTimestamp: "2019-11-08T11:59:49Z"
  name: coredns
  namespace: kube-system
  resourceVersion: "67954425"
  selfLink: /api/v1/namespaces/kube-system/configmaps/coredns
  uid: 21966186-c2d9-467a-b87f-d061c5c9e4d7
```

> 修改完成后，隔一小会儿，CoreDNS 就是自动热加载，我们就可以在集群内访问我们自定义的域名了。然后肯定没有权限，所以需要配置帐号认证信息。在 `Credentials` 区域点击添加按钮添加我们访问 Gitlab 的用户名和密码：

> ![add gitlab auth](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200514105943.png)

> 然后需要我们配置用于构建的分支，如果所有的分支我们都想要进行构建的话，只需要将 `Branch Specifier` 区域留空即可，一般情况下不同的环境对应的分支才需要构建，比如 master、develop、test 等，平时开发的 feature 或者 bugfix 的分支没必要频繁构建，我们这里就只配置 master 和 develop 两个分支用于构建。

> ![add branch filter](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200514175511.png)

> 编辑完成后，这个时候 Jenkins 就可以正常访问到 GitLab 了。

> 然后前往 Gitlab 中配置项目 devops-demo 的 Webhook，settings -> Webhooks，填写上面得到的 trigger 地址：

> ![gitlab webhook settings](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200515105726.png)

> 保存后，如果出现 

```
Url is blocked: Requests to the local network are not allowed
```

 这样的报警信息，则需要进入 GitLab Admin -> Settings -> NetWork -> 勾选 `Outbound requests`，然后重新配置即可。

> ![gitlab outbount settings](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200515105932.png)

> 保存后，可以直接点击 Test -> Push Event 测试是否可以正常访问 Webhook 地址，这里需要注意的是我们需要配置下 Jenkins 的安全配置，否则这里的触发器没权限访问 Jenkins，系统管理 -> 全局安全配置：取消防止跨站点请求伪造，勾选上匿名用户具有可读权限：

> 保存后，可以直接点击 `Test -> Push Event` 测试访问，正常如果没有配置过 Jenkins 的安全选项，会出现403错误，系统管理 -> 全局安全配置：`勾选匿名用户具有读写权限`：

> ![jenkins sec settings](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200515120341.png)

> 由于我们这里的 Jenkins 是比较新的 `Jenkins ver. 2.222.3` 版本，该版本已经默认取消了 CSRF 的安全配置入口，所以我们需要手动执行一段脚本来禁用 CSRF 的跨站请求，系统管理 -> 脚本命令行：执行下图所示的命令即可。

```
import jenkins.model.Jenkins

def jenkins = Jenkins.instance

jenkins.setCrumbIssuer(null)
```

> ![disable csrf](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200515120615.png)

> 配置完成后，这个时候我们重新去 GitLab 进行测试，出现了 

```
Hook executed successfully: HTTP 201
```

 则证明 Webhook 配置成功了，否则就需要检查下 Jenkins 的安全配置是否正确了。

> 由于当前项目中还没有 `Jenkinsfile` 文件，所以触发过后会构建失败，接下来我们直接在代码仓库根目录下面添加 `Jenkinsfile` 文件，用于描述流水线构建流程。

> 首先定义最简单的流程，要注意这里和前面的不同之处，这里我们使用 `podTemplate` 来定义不同阶段使用的的容器，有哪些阶段呢？

```
Clone 代码 -> 单元测试 -> Golang 编译打包 -> Docker 镜像构建/推送 -> Kubectl 部署服务。
```

> Clone 代码在默认的 Slave 容器中即可；单元测试我们这里直接忽略，有需要这个阶段的同学自己添加上即可；Golang 编译打包肯定就需要 Golang 的容器了；Docker 镜像构建/推送是不是就需要 Docker 环境了；最后的 Kubectl 更新服务是不是就需要一个有 Kubectl 的容器环境了，所以我们这里就可以很简单的定义 `podTemplate` 了，如下定义：

```
def label = "slave-${UUID.randomUUID().toString()}"

podTemplate(label: label, containers: [
  containerTemplate(name: 'golang', image: 'golang:2-alpine11', command: 'cat', ttyEnabled: true),
  containerTemplate(name: 'docker', image: 'docker:latest', command: 'cat', ttyEnabled: true),
  containerTemplate(name: 'kubectl', image: 'cnych/kubectl', command: 'cat', ttyEnabled: true)
], serviceAccount: 'jenkins', volumes: [
  hostPathVolume(mountPath: '/home/jenkins/.kube', hostPath: '/root/.kube'),
  hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock')
]) {
  node(label) {
    def myRepo = checkout scm
    def gitCommit = myRepo.GIT_COMMIT
    def gitBranch = myRepo.GIT_BRANCH

    stage('单元测试') {
      echo "测试阶段"
    }
    stage('代码编译打包') {
      container('golang') {
        echo "代码编译打包阶段"
      }
    }
    stage('构建 Docker 镜像') {
      container('docker') {
        echo "构建 Docker 镜像阶段"
      }
    }
    stage('运行 Kubectl') {
      container('kubectl') {
        echo "查看 K8S 集群 Pod 列表"
        sh "kubectl get pods"
      }
    }
  }
}
```

> 直接在 `podTemplate` 里面定义每个阶段需要用到的容器，volumes 里面将我们需要用到的 `docker.sock` 文件，需要注意的我们使用的 label 标签是是一个随机生成的，这样有一个好处就是有多个任务来的时候就可以同时构建了。正常来说我们还需要将访问集群的 `kubeconfig` 文件拷贝到 kubectl 容器的 `~/.kube/config` 文件下面去，这样我们就可以在容器中访问 Kubernetes 集群了，但是由于我们构建是在 Slave Pod 中去构建的，Pod 就很有可能每次调度到不同的节点去，这就需要保证每个节点上有 `kubeconfig` 文件才能挂载成功，所以这里我们使用另外一种方式。

> 通过将 `kubeconfig` 文件通过凭证上传到 Jenkins 中，然后在 Jenkinsfile 中读取到这个文件后，拷贝到 kubectl 容器中的 `~/.kube/config` 文件中，这样同样就可以正常使用 kubectl 访问集群了。在 Jenkins 页面中添加凭据，选择 `Secret file` 类型，然后上传 `kubeconfig` 文件，指定 ID 即可：

> ![add kubeconfig file](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200515153957.png)

> 然后在 `Jenkinsfile` 的 kubectl 容器中读取上面添加的 `Secret file` 文件，拷贝到 `~/.kube/config` 即可：

```
stage('运行 Kubectl') {
  container('kubectl') {
    withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
      echo "查看 K8S 集群 Pod 列表"
      sh "mkdir -p ~/.kube && cp ${KUBECONFIG} ~/.kube/config"
      sh "kubectl get pods"
    }
  }
}
```

> 现在我们直接将 `Jenkinsfile` 文件提交到 GitLab 代码仓库中，正常来说就可以触发 Jenkins 的构建了：

```
$ kubectl get pods -n kube-ops
NAME                                                     READY   STATUS              RESTARTS   AGE
jenkins-68ccff445c-dk24f                                 1/1     Running             0          14h
slave-5dc89808-76f8-418f-ad31-ec34175aa192-5tmwz-2sdm8   0/4     ContainerCreating   0          3s
```

> 我们可以看到生成的 slave Pod 包含了4个容器，就是我们在 podTemplate 指定的加上 slave 的镜像，运行完成后该 Pod 也会自动销毁。

> ![](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200515154445.png)

### Pipeline

> 接下来我们就来实现具体的流水线。

> 第一个阶段：单元测试，我们可以在这个阶段是运行一些单元测试或者静态代码分析的脚本，我们这里直接忽略。

> 第二个阶段：代码编译打包，我们可以看到我们是在一个 golang 的容器中来执行的，我们只需要在该容器中获取到代码，然后在代码目录下面执行打包命令即可，如下所示：

```
stage('代码编译打包') {
  try {
    container('golang') {
      echo "代码编译打包阶段"
      sh """
        export GOPROXY=https://goproxy.cn
        GOOS=linux GOARCH=amd64 go build -v -o demo-app
        """
    }
  } catch (exc) {
    println "构建失败 - ${currentBuild.fullDisplayName}"
    throw(exc)
  }
}
```

> 第三个阶段：构建 Docker 镜像，要构建 Docker 镜像，就需要提供镜像的名称和 tag，要推送到 Harbor 仓库，就需要提供登录的用户名和密码，所以我们这里使用到了 `withCredentials` 方法，在里面可以提供一个`credentialsId` 为 dockerhub 的认证信息，如下：

```
stage('构建 Docker 镜像') {
  withCredentials([[$class: 'UsernamePasswordMultiBinding',
    credentialsId: 'docker-auth',
    usernameVariable: 'DOCKER_USER',
    passwordVariable: 'DOCKER_PASSWORD']]) {
      container('docker') {
        echo " 构建 Docker 镜像阶段"
        sh """
          docker login ${registryUrl} -u ${DOCKER_USER} -p ${DOCKER_PASSWORD}
          docker build -t ${image} .
          docker push ${image}
          """
      }
  }
}
```

> 其中 `${image}` 和 `${imageTag}` 我们可以在上面定义成全局变量：

```
// 获取 git commit id 作为镜像标签
def imageTag = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
// 仓库地址
def registryUrl = "harbor.k8s.local"
def imageEndpoint = "course/devops-demo"
// 镜像
def image = "${registryUrl}/${imageEndpoint}:${imageTag}"
```

> 这里定义的镜像名称为 `course/devops-demo`，所以需要提前在 Harbor 中新建一个名为 course 的私有项目： ![harbor project](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200520142740.png)

> Docker 的用户名和密码信息则需要通过凭据来进行添加，进入 jenkins 首页 -> 左侧菜单凭据 -> 添加凭据，选择用户名和密码类型的，其中 ID 一定要和上面的 `credentialsId` 的值保持一致：

> ![harbor auth](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200520140737.png)

> 不过需要注意的是我们这里使用的是 `Docker IN Docker` 模式来构建 Docker 镜像，通过将宿主机的 `docker.sock` 文件挂载到容器中来共享 Docker Daemon，所以我们也需要提前在节点上配置对 Harbor 镜像仓库的信任:

```
$ vi /etc/docker/daemon.json
{
  "insecure-registries" : [  # 配置忽略 Harobr 镜像仓库的证书校验
    "harbor.k8s.local"
  ],
  "storage-driver": "overlay2",
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors" : [
    "https://ot2k4dmirror.aliyuncs.com/"
  ],
}
$ systemctl daemon-reload
$ systemctl restart docker
```

> 配置生效过后我们就可以正常在流水线中去操作 Docker 命令，否则会出现如下所示的错误： ![](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200520141843.png)

> 现在镜像我们都已经推送到了 Harbor 仓库中去了，接下来就可以部署应用到 Kubernetes 集群中了，当然可以直接通过 kubectl 工具去操作 YAML 文件来部署，我们这里的示例，编写了一个 Helm Chart 模板，所以我们也可以直接通过 Helm 来进行部署，所以当然就需要一个具有 helm 命令的容器，这里我们使用 `cnych/helm` 这个镜像，这个镜像也非常简单，就是简单的将 helm 二进制文件下载下来放到 PATH 路径下面去即可，对应的 Dockerfile 文件如下所示，大家也可以根据自己的需要来进行定制：

```
FROM alpine
MAINTAINER cnych <icnych@gmail.com>
ARG HELM_VERSION="v1"
RUN apk add --update ca-certificates \\
 && apk add --update -t deps wget git openssl bash \\
 && wget https://get.helm.sh/helm-${HELM_VERSION}-linux-amdtar.gz \\
 && tar -xvf helm-${HELM_VERSION}-linux-amdtar.gz \\
 && mv linux-amd64/helm /usr/local/bin \\
 && apk del --purge deps \\
 && rm /var/cache/apk/* \\
 && rm -f /helm-${HELM_VERSION}-linux-amdtar.gz
ENTRYPOINT ["helm"]
CMD ["help"]
```

> 我们这里使用的是 Helm3 版本，所以要想用 Helm 来部署应用，同样的需要配置一个 kubeconfig 文件在容器中，这样才能访问到 Kubernetes 集群。所以我们可以将 `运行 Kubectl` 的阶段做如下更改：

```
stage('运行 Helm') {
  withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
    container('helm') {
      sh "mkdir -p ~/.kube && cp ${KUBECONFIG} ~/.kube/config"
      echo "开始 Helm 部署"
      helmDeploy(
          debug       : false,
          name        : "devops-demo",
          chartDir    : "./helm",
          namespace   : "kube-ops",
          valuePath   : "./helm/my-value.yaml",
          imageTag    : "${imageTag}"
      )
      echo "[INFO] Helm 部署应用成功..."
    }
  }
}
```

> 其中 `helmDeploy` 方法可以在全局中进行定义封装：

```
def helmLint(String chartDir) {
    println "校验 chart 模板"
    sh "helm lint ${chartDir}"
}

def helmDeploy(Map args) {
    if (args.debug) {
        println "Debug 应用"
        sh "helm upgrade --dry-run --debug --install ${args.name} ${args.chartDir} -f ${args.valuePath} --set image.tag=${args.imageTag} --namespace ${args.namespace}"
    } else {
        println "部署应用"
        sh "helm upgrade --install ${args.name} ${args.chartDir} -f ${args.valuePath} --set image.tag=${args.imageTag} --namespace ${args.namespace}"
        echo "应用 ${args.name} 部署成功. 可以使用 helm status ${args.name} 查看应用状态"
    }
}
```

> 我们在 Chart 模板中定义了一个名为 `my-values.yaml` 的 Values 文件，用来覆盖默认的值，比如这里我们需要使用 Harbor 私有仓库的镜像，则必然需要定义 `imagePullSecrets`，所以需要在目标 namespace 下面创建一个 Harbor 登录认证的 Secret 对象：

```
$ kubectl create secret docker-registry harbor-auth --docker-server=harbor.k8s.local --docker-username=admin --docker-password=Harbor12345 --docker-email=admin@admin.com --namespace kube-ops
secret/harbor-auth created
```

> 然后由于每次我们构建的镜像 tag 都会变化，所以我们可以通过 `--set` 来动态设置。

> 不过需要记得在上面容器模板中添加 helm 容器：

```
containerTemplate(name: 'helm', image: 'cnych/helm', command: 'cat', ttyEnabled: true)
```

> 对于不同的环境我们可以使用不同的 values 文件来进行区分，这样当我们部署的时候可以手动选择部署到某个环境下面去。

```
def userInput = input(
  id: 'userInput',
  message: '选择一个部署环境',
  parameters: [
      [
          $class: 'ChoiceParameterDefinition',
          choices: "Dev\\nQA\\nProd",
          name: 'Env'
      ]
  ]
)
echo "部署应用到 ${userInput} 环境"
// 选择不同环境下面的 values 文件
if (userInput == "Dev") {
    // deploy dev stuff
} else if (userInput == "QA"){
    // deploy qa stuff
} else {
    // deploy prod stuff
}
// 根据 values 文件再去使用 Helm 进行部署
```

> 然后去构建应用的时候，在 Helm 部署阶段就会看到 Stage View 界面出现了暂停的情况，需要我们选择一个环境来进行部署：

> ![](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200520161259.png)

> 选择完成后再去部署应用。最后我们还可以添加一个 kubectl 容器来查看应用的相关资源对象：

```
stage('运行 Kubectl') {
  withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
    container('kubectl') {
      sh "mkdir -p ~/.kube && cp ${KUBECONFIG} ~/.kube/config"
      echo "查看应用"
      sh "kubectl get all -n kube-ops -l app=devops-demo"
    }
  }
}
```

> 有时候我们部署的应用即使有很多测试，但是也难免会出现一些错误，这个时候如果我们是部署到线上的话，就需要要求能够立即进行回滚，这里我们同样可以使用 Helm 来非常方便的操作，添加如下一个回滚的阶段：

```
stage('快速回滚?') {
  withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
    container('helm') {
      sh "mkdir -p ~/.kube && cp ${KUBECONFIG} ~/.kube/config"
      def userInput = input(
        id: 'userInput',
        message: '是否需要快速回滚？',
        parameters: [
            [
                $class: 'ChoiceParameterDefinition',
                choices: "Y\\nN",
                name: '回滚?'
            ]
        ]
      )
      if (userInput == "Y") {
        sh "helm rollback devops-demo --namespace kube-ops"
      }
    }
  }
}
```

> 最后一条完整的流水线就完成了。

> ![](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200520163020.png)

> 我们可以在本地加上应用域名 

```
devops-demo.k8s.local
```

 的映射就可以访问应用了：

```
$ curl http://devops-demo.k8s.local
{"msg":"Hello DevOps On Kubernetes"}
```

> 完整的 Jenkinsfile 文件如下所示：

```
def label = "slave-${UUID.randomUUID().toString()}"

def helmLint(String chartDir) {
    println "校验 chart 模板"
    sh "helm lint ${chartDir}"
}

def helmDeploy(Map args) {
    if (args.debug) {
        println "Debug 应用"
        sh "helm upgrade --dry-run --debug --install ${args.name} ${args.chartDir} -f ${args.valuePath} --set image.tag=${args.imageTag} --namespace ${args.namespace}"
    } else {
        println "部署应用"
        sh "helm upgrade --install ${args.name} ${args.chartDir} -f ${args.valuePath} --set image.tag=${args.imageTag} --namespace ${args.namespace}"
        echo "应用 ${args.name} 部署成功. 可以使用 helm status ${args.name} 查看应用状态"
    }
}

podTemplate(label: label, containers: [
  containerTemplate(name: 'golang', image: 'golang:2-alpine11', command: 'cat', ttyEnabled: true),
  containerTemplate(name: 'docker', image: 'docker:latest', command: 'cat', ttyEnabled: true),
  containerTemplate(name: 'helm', image: 'cnych/helm', command: 'cat', ttyEnabled: true),
  containerTemplate(name: 'kubectl', image: 'cnych/kubectl', command: 'cat', ttyEnabled: true)
], serviceAccount: 'jenkins', volumes: [
  hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock')
]) {
  node(label) {
    def myRepo = checkout scm
    // 获取 git commit id 作为镜像标签
    def imageTag = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
    // 仓库地址
    def registryUrl = "harbor.k8s.local"
    def imageEndpoint = "course/devops-demo"
    // 镜像
    def image = "${registryUrl}/${imageEndpoint}:${imageTag}"

    stage('单元测试') {
      echo "测试阶段"
    }
    stage('代码编译打包') {
      try {
        container('golang') {
          echo "代码编译打包阶段"
          sh """
            export GOPROXY=https://goproxy.cn
            GOOS=linux GOARCH=amd64 go build -v -o demo-app
            """
        }
      } catch (exc) {
        println "构建失败 - ${currentBuild.fullDisplayName}"
        throw(exc)
      }
    }
    stage('构建 Docker 镜像') {
      withCredentials([[$class: 'UsernamePasswordMultiBinding',
        credentialsId: 'docker-auth',
        usernameVariable: 'DOCKER_USER',
        passwordVariable: 'DOCKER_PASSWORD']]) {
          container('docker') {
            echo " 构建 Docker 镜像阶段"
            sh """
              cat /etc/resolv.conf
              docker login ${registryUrl} -u ${DOCKER_USER} -p ${DOCKER_PASSWORD}
              docker build -t ${image} .
              docker push ${image}
              """
          }
      }
    }
    stage('运行 Helm') {
      withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
        container('helm') {
          sh "mkdir -p ~/.kube && cp ${KUBECONFIG} ~/.kube/config"
          echo "开始 Helm 部署"
          def userInput = input(
            id: 'userInput',
            message: '选择一个部署环境',
            parameters: [
                [
                    $class: 'ChoiceParameterDefinition',
                    choices: "Dev\\nQA\\nProd",
                    name: 'Env'
                ]
            ]
          )
          echo "部署应用到 ${userInput} 环境"
          // 选择不同环境下面的 values 文件
          if (userInput == "Dev") {
              // deploy dev stuff
          } else if (userInput == "QA"){
              // deploy qa stuff
          } else {
              // deploy prod stuff
          }
          helmDeploy(
              debug       : false,
              name        : "devops-demo",
              chartDir    : "./helm",
              namespace   : "kube-ops",
              valuePath   : "./helm/my-values.yaml",
              imageTag    : "${imageTag}"
          )
        }
      }
    }
    stage('运行 Kubectl') {
      withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
        container('kubectl') {
          sh "mkdir -p ~/.kube && cp ${KUBECONFIG} ~/.kube/config"
          echo "查看应用"
          sh "kubectl get all -n kube-ops -l app=devops-demo"
        }
      }
    }
    stage('快速回滚?') {
      withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
        container('helm') {
          sh "mkdir -p ~/.kube && cp ${KUBECONFIG} ~/.kube/config"
          def userInput = input(
            id: 'userInput',
            message: '是否需要快速回滚？',
            parameters: [
                [
                    $class: 'ChoiceParameterDefinition',
                    choices: "Y\\nN",
                    name: '回滚?'
                ]
            ]
          )
          if (userInput == "Y") {
            sh "helm rollback devops-demo --namespace kube-ops"
          }
        }
      }
    }
  }
}
```