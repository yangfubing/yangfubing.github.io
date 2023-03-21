
## 示例

> 前面介绍了 Helm 的基本使用，以及 Helm Chart 包开发相关的一些知识点，下面我们用一个实例来演示下如何开发一个真正的 Helm Chart 包。

### 应用

> 我们这里为前面的完整示例 Wordpress 开发一个 Chart 包，在开发 Chart 包之前很明显我们最需要的就是要知道我们自己的应用应该如何使用，如何部署，不然是不可能编写出对应的 Chart 包的。

> 我们可以用 `helm create` 命令来创建一个 Chart 包，这里我们就完全手动来创建，首先创建一个名为 wordpress 的文件夹:

```
$ mkdir wordpress && cd wordpress
```

> 然后在目录下面创建如下所示的 `Chart.yaml` 文件：

```
apiVersion: v2
name: wordpress
description: A Helm chart for Kubernetes
home: https://wordpress.org/
type: application
# chart 版本号
version: 0
# wordpress 的应用版本
appVersion: 2
```

> 由于 Wordpress 应用本身是一来 MySQL 数据库的，所以同样需要添加一个依赖说明 `requirements.yaml`：

```
dependencies:
- name: mysql
  version: 2
  repository: http://mirror.azure.cn/kubernetes/charts/
  condition: mysql.enabled
  tags:
    - wordpress-database
```

> 这里依赖的应用我们可以通过 `helm search` 命令来获取，当然也可以随意指定一个 Chart 版本：

```
$ helm search repo stable/mysql
NAME                    CHART VERSION   APP VERSION     DESCRIPTION                                       
stable/mysql            2           28          Fast, reliable, scalable, and easy to use open-...
```

> 需要注意的是在依赖的文件中我们添加了一个 

```
condition: mysql.enabled
```

 条件，这意味着当 Values 值 `mysql.enabled` 为 true 的时候才会真正去依赖这个子 Chart，因为我们完全可以直接使用一个外部的数据库。所以首先我们要先添加一个 `mysql.enabled` 的 Values 值，添加如下所示的 `values.yaml` 文件：

```
##
## MySQL Chart 配置，参考如下 values.yaml
## https://github.com/helm/charts/blob/master/stable/mysql/values.yaml
##
mysql:
  ## 是否部署mysql服务来满足wordpress的需求。如果要使用外部数据库，将 enabled 设置为 false 并配置 externalDatabase 参数。
  enabled: true
  ## todo，其他 mysql 配置

## 当 mysql.enabled=false 的时候使用外部数据库
externalDatabase:
  ## 数据库地址
  host: localhost:3306

  ## Wordpress 数据库的非root用户
  user: wordpress

  ## 数据库密码
  password: wordpress

  ## 数据库名称
  database: wordpress
```

> 上面的 Values 配置很好理解，就是如果 `mysql.enabled=true` 则我们就用 mysql 这个子 Chart 去安装一个 MySQL 数据库，如果为 false 则使用下面 `externalDatabase` 的外部数据库信息来作为 wordpress 的数据库配置，虽然现在这些配置还完全没有任何作用，但是这些却是一开始就很容易考虑到的事情。

> 接下来创建一个 `templates` 和 `charts` 目录，并在 `charts` 目录下面获取 mysql 这个子 chart:

```
$ mkdir templates && mkdir charts
$ cd charts && helm fetch stable/mysql && cd ..
```

> 现在的整个 Chart 包目录结构如下所示：

```
$ tree .
.
├── Chart.yaml
├── charts
│   └── mysql-tgz
├── requirements.yaml
├── templates
└── values.yaml

2 directories, 4 files
```

> 接下来就是我们的重头戏模板的开发了。我们可以将前面课程示例中的 Wordpress 资源清单直接拷贝到 `templates` 目录下面，将 `Deployment` 和 `Service` 分别放在不同的 YAML 文件中，同样还有数据持久化的资源清单，但是这里我们就需要先将 MySQL 的部分移除掉，因为我们会通过外部数据库或者子 Chart 来渲染，不需要在这里显示的声明了，另外还需要将所有资源对象的命名空间去掉，因为我们可以通过 `helm install` 命令直接指定即可，现在我们的模板目录结构就如下所示了：

```
$ tree templates 
templates
├── deployment.yaml
├── pvc.yaml
└── service.yaml

0 directories, 4 files
```

### 名称

> 这个时候我们可以尝试去安装调试下这个 Chart 模板：

```
$ helm install --generate-name --dry-run --debug .
install.go:148: [debug] Original chart version: ""
install.go:165: [debug] CHART PATH: /Users/ych/devs/workspace/yidianzhishi/course/k8strain/content/helm/manifests/wordpress

load.go:112: Warning: Dependencies are handled in Chart.yaml since apiVersion "v2". We recommend migrating dependencies to Chart.yaml.
Error: rendered manifests contain a resource that already exists. Unable to continue with install: existing resource conflict: kind: Middleware, namespace: default, name: redirect-https
helm.go:76: [debug] existing resource conflict: kind: Middleware, namespace: default, name: redirect-https
rendered manifests contain a resource that already exists. Unable to continue with install
helm.sh/helm/v3/pkg/action.(*Install).Run
        /home/circleci/helm.sh/helm/pkg/action/install.go:242
main.runInstall
        /home/circleci/helm.sh/helm/cmd/helm/install.go:209
main.newInstallCmd.func1
        /home/circleci/helm.sh/helm/cmd/helm/install.go:115
github.com/spf13/cobra.(*Command).execute
        /go/pkg/mod/github.com/spf13/cobra@v5/command.go:826
github.com/spf13/cobra.(*Command).ExecuteC
        /go/pkg/mod/github.com/spf13/cobra@v5/command.go:914
github.com/spf13/cobra.(*Command).Execute
        /go/pkg/mod/github.com/spf13/cobra@v5/command.go:864
main.main
        /home/circleci/helm.sh/helm/cmd/helm/helm.go:75
runtime.main
        /usr/local/go/src/runtime/proc.go:203
runtime.goexit
        /usr/local/go/src/runtime/asm_amds:1357
```

> 我们看到出现了错误，其中还有这样的一条警告信息 

```
load.go:112: Warning: Dependencies are handled in Chart.yaml since apiVersion "v2". We recommend migrating dependencies to Chart.yaml.
```

，大概意思就是在 Helm3 中依赖项的配置被移动到了 `Chart.yaml` 文件中，也就是在 Helm2 版本的时候依赖才会在 `requirements.yaml` 中声明，这里我们可以将 `requirements.yaml` 文件中的内容全部拷贝到 `Chart.yaml` 文件中去，然后删除 `requirements.yaml` 文件，这个时候重新 DEBUG 下就没有这个提示信息了。

> 下面的错误提示基本上都是类似于 

```
rendered manifests contain a resource that already exists.
```

 这样的信息，意思就是这些资源对象在现有集群中已经包含了，这很正常，因为我们的资源对象名称都是写死的，是非常有可能和现有的集群冲突的，所以接下来我们要给这些资源清单改名，如何才能避免冲突又具有很好的扩展性呢？这里我们创建一个如下所示的命名模板：

```
{{/*
创建一个默认的应用名称，截取63个字符是因为 Kubernetes 的 name 属性的限制（DNS 命名规范）。
*/}}
{{- define "wordpress.fullname" -}}
{{- if .Values.fullnameOverride -}}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" -}}
{{- else -}}
{{- $name := default .Chart.Name .Values.nameOverride -}}
{{- if contains $name .Release.Name -}}
{{- .Release.Name | trunc 63 | trimSuffix "-" -}}
{{- else -}}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" -}}
{{- end -}}
{{- end -}}
{{- end -}}
```

> 根据规范命名模板的名称最好以 Chart 名作为前缀，这里我们命名成 `wordpress.fullname`，首先就是判断是否定义了 `fullnameOverride` 这个 Values 值，如果定义了则截取 63 个字符并移除 `-` 这样的前后缀作为名称，如果没有定义呢？就要检查 Chart 名是否包含 Release 名称，如果包含则截取 Release 名的 63 个字符并移除 `-` 这样的前后缀作为名称，如果不包含，则将 Release 名称和 Chart 名称用 `-` 连接起来作为名称，这个命名的方式也是符合大部分 Chart 模板中的命名，以后的模板开发中都可以直接使用。在 `templates` 目录下面新建一个 `_helpers.tpl` 这样的 partials 文件，然后将上面定义的命名模板放置到里面去。后面我们所有的命名模板都将在该文件中完成。

> 接下来将 `templates` 目录下面的所有资源对象 name 属性全都换成上面我们定义的命名模板，由于这里并没有什么空格控制之类的，我们直接使用 `template` 函数即可，同样作为惯例，也习惯将 Release 和 Chart 名称之类作为 Label 标签，所以最终，这些资源清单的 Meta 信息如下所示，如果有其他额外的信息当然可以随意添加：

```
metadata:
  name: {{ template "wordpress.fullname" . }}
  labels:
    app: "{{ template "wordpress.fullname" . }}"
    chart: "{{ template "wordpress.chart" . }}"
    release: {{ .Release.Name | quote }}
    heritage: {{ .Release.Service | quote }}
```

> 然后当然也需要将 Deployment 的 matchLabels 和 template 下面的 Label 标签进行修改，以及 Service 的 `selector` 匹配的标签。

> 同样还有 PVC 对象，以前我们直接使用的一个确定名称的 PVC 对象：

```
volumes:
- name: wordpress-data
  persistentVolumeClaim:
    claimName: wordpress-pvc
```

> 但是现在我们这里作为模板就要考虑到各种情况了，很有可能使用我们模板的用户根本就不需要持久化，也有可能直接传递一个存在的 PVC 对象进来使用，所以这里做了如下改变：

```
volumes:
- name: wordpress-data
{{- if .Values.persistence.enabled }}
  persistentVolumeClaim:
    claimName: {{ .Values.persistence.existingClaim | default (include "wordpress.fullname" .) }}
{{- else }}
  emptyDir: {}
{{ end }}
```

> 当我们定义了 Values 值 

```
persistence.enabled=true
```

 为 true 时候就表示要使用持久化了，所以要指定下面的 `claimName` 属性，但是还需要判断 

```
persistence.existingClaim
```

 这个 Values 值是否存在，如果存在则表示直接使用，如果不存在则使用我们模板里面渲染的 PVC 对象，如果不需要持久化，则直接使用 `emptyDir:{}` 即可。最后我们的 PVC 对象模板变成了如下所示：

```
{{- if and .Values.persistence.enabled (not .Values.persistence.existingClaim) }}
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: {{ template "wordpress.fullname" . }}
  labels:
    app: "{{ template "wordpress.fullname" . }}"
    chart: "{{ template "wordpress.chart" . }}"
    release: {{ .Release.Name | quote }}
    heritage: {{ .Release.Service | quote }}
spec:
  {{- if .Values.persistence.storageClass }}
  storageClassName: {{ .Values.persistence.storageClass | quote }}
  {{- end }}
  accessModes:
  - {{ .Values.persistence.accessMode | quote }}
  resources:
    requests:
      storage: {{ .Values.persistence.size | quote }}
{{- end -}}
```

> 其中访问模式、存储容量、StorageClass、存在的 PVC 都通过 Values 来指定，增加了灵活性。对应的 `values.yaml` 配置部分我们可以给一个默认的配置：

```
## 是否使用 PVC 开启数据持久化
persistence:
  enabled: true
  ## 是否使用 storageClass，如果不适用则补配置
  # storageClass: "xxx"
  ##
  ## 如果想使用一个存在的 PVC 对象，则直接传递给下面的 existingClaim 变量
  # existingClaim: your-claim
  accessMode: ReadWriteMany  # 访问模式
  size: 2Gi  # 存储容量
```

> 现在我们去重新做一次 DEBUG，可以看到正常了，但是还远远不够，接下来我们就来定制其他部分。

### 定制

> 比如副本数我们可以通过 Values 来指定，变成 

```
replicas: {{ .Values.replicaCount }}
```

。

> 更新策略也可以，因为更新策略并不是一层不变的，这里和之前不太一样，我们需要用到一个新的函数 `toYaml`：

```
{{- if .Values.updateStrategy }}
strategy: {{ toYaml .Values.updateStrategy | nindent 4 }}
{{- end }}
```

> 意思就是我们将 `updateStrategy` 这个 Values 值转换成 YAML 格式，并保留4个空格。然后添加其他的配置，比如是否需要添加 nodeSelector、容忍、亲和性这些，这里我们都是使用 `toYaml` 函数来控制空格，如下所示：

```
{{- if .Values.nodeSelector }}
    nodeSelector:
{{ toYaml .Values.nodeSelector | indent 8 }}
    {{- end -}}
  {{- with .Values.affinity }}
    affinity:
{{ toYaml . | indent 8 }}
  {{- end }}
  {{- with .Values.tolerations }}
    tolerations:
{{ toYaml . | indent 8 }}
  {{- end }}
```

> 接下来当然就是镜像的配置了，如果是私有仓库还需要指定 `imagePullSecrets`：

```
{{- if .Values.image.pullSecrets }}
imagePullSecrets:
{{- range .Values.image.pullSecrets }}
- name: {{ . }}
{{- end }}
{{- end }}
containers:
- name: wordpress
  image: {{ printf "%s:%s" .Values.image.name .Values.image.tag }}
  imagePullPolicy: {{ .Values.image.pullPolicy | quote }}
  ports:
  - containerPort: 80
    name: web
```

> 对应的 Values 值如下所示：

```
image:
  name: wordpress
  tag: 2-apache
  ## 指定 imagePullPolicy 默认为 Always
  pullPolicy: IfNotPresent
  ## 如果是私有仓库，需要指定 imagePullSecrets
  # pullSecrets:
  #   - myRegistryKeySecretName
```

> 然后就是最重要的环境变量配置部分了，因为涉及到数据库的配置，同样最核心的三个环境变量配置 `WORDPRESS_DB_HOST`、`WORDPRESS_DB_USER`、

```
WORDPRESS_DB_PASSWORD
```

，由于可能使用我们模板的用户可能使用外部数据库，也有可能使用我们依赖的 mysql 这个子 Chart，所以我们需要分别判断来进行渲染：

```
env:
- name: WORDPRESS_DB_HOST
{{- if .Values.mysql.enabled }}
  value: {{ printf "%s:%d" (include "mysql.fullname" .) (int64 .Values.mysql.service.port) }}
{{- else }}
  value: {{ .Values.externalDatabase.host | quote }}
{{- end }}
- name: WORDPRESS_DATABASE_NAME
{{- if .Values.mysql.enabled }}
  value: {{ .Values.mysql.mysqlDatabase | quote }}
{{- else }}
  value: {{ .Values.externalDatabase.database | quote }}
{{- end }}
- name: WORDPRESS_DB_USER
{{- if .Values.mysql.enabled }}
  value: {{ .Values.mysql.mysqlUser | quote }}
{{- else }}
  value: {{ .Values.externalDatabase.user | quote }}
{{- end }}
- name: WORDPRESS_DB_PASSWORD
  valueFrom:
    secretKeyRef:
    {{- if .Values.mysql.enabled }}
      name: {{ template "mysql.fullname" . }}
      key: mysql-password
    {{- else }}
      name: {{ printf "%s-%s" .Release.Name "externaldb" }}
      key: db-password
    {{- end }}
```

> 每一个环境变量首先都是判断 `mysql.enabled` 是否为 true，才表示使用子 Chart 来渲染，而对应的值都是子 Chart 中渲染过后的值，所以也需要我们去了解子 Chart 的渲染行为，如果使用外部的数据库就要简单很多，因为只需要读取 Values 值即可。另外一个值得注意的是如果是配置的密码，我们还需要去创建一个 Secret 资源对象来引用。所以我们在 templates 目录下面创建了一个 

```
externaldb-secrets.yaml
```

 的资源文件，里面配置的密码通过 `b64enc` 函数转换为 Base64 编码格式。

```
{{- if not .Values.mysql.enabled }}
apiVersion: v1
kind: Secret
metadata:
  name: {{ printf "%s-%s" .Release.Name "externaldb"  }}
  labels:
    app: {{ printf "%s-%s" .Release.Name "externaldb"  }}
    chart: "{{ template "wordpress.chart" . }}"
    release: {{ .Release.Name | quote }}
    heritage: {{ .Release.Service | quote }}
type: Opaque
data:
  db-password: {{ .Values.externalDatabase.password | b64enc | quote }}
{{- end }}
```

> 然后就是 resource 资源声明，这里我们定义一个默认的 resources 值，同样用 `toYaml` 函数来控制空格：

```
resources:
{{ toYaml .Values.resources | indent 10 }}
```

> 最后是健康检查部分，虽然我们之前没有做 livenessProbe，但是我们开发 Chart 模板的时候就要尽可能考虑周全一点，这里我们加上存活性和可读性两个探针，并且根据 

```
livenessProbe.enabled
```

 和 

```
readinessProbe.enabled
```

 两个 Values 值来判断是否需要添加探针，探针对应的参数也都通过 Values 值来配置：

```
{{- if .Values.livenessProbe.enabled }}
livenessProbe:
  initialDelaySeconds: {{ .Values.livenessProbe.initialDelaySeconds }}
  periodSeconds: {{ .Values.livenessProbe.periodSeconds }}
  timeoutSeconds: {{ .Values.livenessProbe.timeoutSeconds }}
  successThreshold: {{ .Values.livenessProbe.successThreshold }}
  failureThreshold: {{ .Values.livenessProbe.failureThreshold }}
  httpGet:
    path: /wp-login.php
    port: 80
{{- end }}
{{- if .Values.readinessProbe.enabled }}
readinessProbe:
  initialDelaySeconds: {{ .Values.readinessProbe.initialDelaySeconds }}
  periodSeconds: {{ .Values.readinessProbe.periodSeconds }}
  timeoutSeconds: {{ .Values.readinessProbe.timeoutSeconds }}
  successThreshold: {{ .Values.readinessProbe.successThreshold }}
  failureThreshold: {{ .Values.readinessProbe.failureThreshold }}
  httpGet:
    path: /wp-login.php
    port: 80
{{- end }}
```

> 这样我们的 Deployment 的模板就基本上完成了，我们可以通过 DEBUG 模式来模拟渲染：

```
$ helm install --generate-name --dry-run --debug .
```

> 可以根据结果来判断是否符合我们的需求。

> 最后我们还需要来对 Service 对象做模板化，因为前面我们是默认的 `NodePort` 类型，我们需要通过 Values 来定制：

```
apiVersion: v1
kind: Service
metadata:
  name: {{ template "wordpress.fullname" . }}
  labels:
    app: "{{ template "wordpress.fullname" . }}"
    chart: "{{ template "wordpress.chart" . }}"
    release: {{ .Release.Name | quote }}
    heritage: {{ .Release.Service | quote }}
spec:
  selector:
    app: "{{ template "wordpress.fullname" . }}"
  type: {{ .Values.service.type }}
  {{- if (or (eq .Values.service.type "LoadBalancer") (eq .Values.service.type "NodePort")) }}
  externalTrafficPolicy: {{ .Values.service.externalTrafficPolicy | quote }}
  {{- end }}
  ports:
  - name: web
    port: {{ .Values.service.port }}
    targetPort: web
    {{- if (and (eq .Values.service.type "NodePort") (not (empty .Values.service.nodePort)))}}
    nodePort: {{ .Values.service.nodePort }}
    {{- end }}
```

> 这样我们就可以通过配置 Values 值来配置 Service 对象了。最后就是 `Ingress/IngressRoute` 对象了，大家可以自己尝试讲这部分补齐。

> 最后在 `templates` 目录下面加上 `NOTES.txt` 文件来说明如何使用我们的 Chart 包就可以了：

```
Get the WordPress Manifests Objects:

$ kubectl get all -l app={{ .Release.Name }}
```

> 最后我们来真正的使用我们的 Chart 包安装一次来测试下：

```
$ helm install mychart . --set service.type=NodePort
NAME: mychart
LAST DEPLOYED: Sun Mar  8 17:52:04 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
Get the WordPress Manifests Objects:
  $ kubectl get all -l app=mychart
$ helm ls  
NAME                    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART            APP VERSION
mychart                 default         1               2020-03-08 17:52:826185 +0800 CST    deployed        wordpress-0  2
```

> 安装完成后可以查看我们的资源对象：

```
$ kubectl get all -l app=mychart-wordpress
NAME                                     READY   STATUS    RESTARTS   AGE
pod/mychart-wordpress-5f65786d89-2m45s   1/1     Running   0          70s

NAME                        TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
service/mychart-wordpress   NodePort   229   <none>        80:30427/TCP   70s

NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/mychart-wordpress   1/1     1            1           70s

NAME                                           DESIRED   CURRENT   READY   AGE
replicaset.apps/mychart-wordpress-5f65786d89   1         1         1       70s
```

> 这个时候我们通过上面的 NodePort 就可以去访问到我们的应用了，当然还有很多配置我们都是可以直接通过 Values 去进行定制的。