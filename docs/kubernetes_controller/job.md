## Job 与 CronJob¶

> 今天我们来给大家介绍另外一类资源对象：`Job`，我们在日常的工作中经常都会遇到一些需要进行批量数据处理和分析的需求，当然也会有按时间来进行调度的工作，在我们的 Kubernetes 集群中为我们提供了 `Job` 和 `CronJob` 两种资源对象来应对我们的这种需求。

> `Job` 负责处理任务，即仅执行一次的任务，它保证批处理任务的一个或多个 Pod 成功结束。而`CronJob` 则就是在 `Job` 上加上了时间调度。

### Job¶

> 我们用 `Job` 这个资源对象来创建一个任务，我们定义一个 `Job` 来执行一个倒计时的任务，对应的资源清单如下所示：(job-demo.yaml)

```
apiVersion: batch/v1
kind: Job
metadata:
  name: job-demo
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: counter
        image: busybox
        command:
        - "bin/sh"
        - "-c"
        - "for i in 9 8 7 6 5 4 3 2 1; do echo $i; done"
```

> 我们可以看到 `Job` 中也是一个 Pod 模板，和之前的 Deployment、StatefulSet 之类的是一致的，只是 Pod 中的容器要求是一个任务，而不是一个常驻前台的进程了，因为需要退出，另外值得注意的是 `Job` 的 `RestartPolicy` 仅支持 `Never` 和 `OnFailure` 两种，不支持 `Always`，我们知道 `Job` 就相当于来执行一个批处理任务，执行完就结束了，如果支持 `Always` 的话是不是就陷入了死循环了？

> 直接创建这个 Job 对象：

```
$ kubectl apply -f job-demo.yaml
job.batch/job-demo created
$ kubectl get job 
NAME       COMPLETIONS   DURATION   AGE
job-demo   1/1           26s        3m4s
$ kubectl get pods         
NAME                      READY   STATUS      RESTARTS   AGE
job-demo-frs2r            0/1     Completed   0          2m41s
```

> `Job` 对象创建成功后，我们可以查看下对象的详细描述信息：

```
$ kubectl describe job job-demo
Name:           job-demo
Namespace:      default
Selector:       controller-uid=b1191631-c71c-45e7-86c9-ebca69cb6125
Labels:         controller-uid=b1191631-c71c-45e7-86c9-ebca69cb6125
                job-name=job-demo
.......
Pod Template:
  Labels:  controller-uid=b1191631-c71c-45e7-86c9-ebca69cb6125
           job-name=job-demo
  Containers:
   ......
Events:
  Type    Reason            Age   From            Message
  ----    ------            ----  ----            -------
  Normal  SuccessfulCreate  49s   job-controller  Created pod: job-demo-z48h2
```

> 可以看到，Job 对象在创建后，它的 Pod 模板，被自动加上了一个 

```
controller-uid=< 一个随机字符串 >
```

 这样的 Label 标签，而这个 Job 对象本身，则被自动加上了这个 Label 对应的 Selector，从而 保证了 Job 与它所管理的 Pod 之间的匹配关系。而 Job 控制器之所以要使用这种携带了 UID 的 Label，就是为了避免不同 Job 对象所管理的 Pod 发生重合。

> 我们可以看到很快 Pod 变成了 `Completed` 状态，这是因为容器的任务执行完成正常退出了，我们可以查看对应的日志：

```
$ kubectl logs job-demo-frs2r
9
8
7
6
5
4
3
2
1
```

> 如果的任务执行失败了，我们这里定义了 `restartPolicy=Never`，那么任务在执行失败后 Job 控制器就会不断地尝试创建一个新 Pod，当然，这个尝试肯定不能无限进行下去。我们可以通过 Job 对象的 `spec.backoffLimit` 字段来定义重试次数，另外需要注意的是 Job 控制器重新创建 Pod 的间隔是呈指数增加的，即下一次重新创建 Pod 的动作会分别发生在 10s、20s、40s… 后。

> 如果我们定义的 

```
restartPolicy=OnFailure
```

，那么任务执行失败后，Job 控制器就不会去尝试创建新的 Pod了，它会不断地尝试重启 Pod 里的容器。

> 上面我们这里的 Job 任务对应的 Pod 在运行结束后，会变成 `Completed` 状态，但是如果执行任务的 Pod 因为某种原因一直没有结束怎么办呢？同样我们可以在 Job 对象中通过设置字段 

```
spec.activeDeadlineSeconds
```

 来限制任务运行的最长时间，比如：

```
spec:
 activeDeadlineSeconds: 100
```

> 那么当我们的任务 Pod 运行超过了 100s 后，这个 Job 的所有 Pod 都会被终止，并且， Pod 的终止原因会变成 `DeadlineExceeded`。

> 除此之外，我们还可以通过设置 `spec.parallelism` 参数来进行并行控制，该参数定义了一个 Job 在任意时间最多可以有多少个 Pod 同时运行。`spec.completions` 参数可以定义 Job 至少要完成的 Pod 数目。

### CronJob¶

> `CronJob` 其实就是在 `Job` 的基础上加上了时间调度，我们可以在给定的时间点运行一个任务，也可以周期性地在给定时间点运行。这个实际上和我们 Linux 中的 `crontab` 就非常类似了。

> 一个 `CronJob` 对象其实就对应中 `crontab` 文件中的一行，它根据配置的时间格式周期性地运行一个`Job`，格式和 `crontab` 也是一样的。

> crontab 的格式为：`分 时 日 月 星期 要运行的命令` 。

*   第1列分钟0～59
*   第2列小时0～23）
*   第3列日1～31
*   第4列月1～12
*   第5列星期0～7（0和7表示星期天）
*   第6列要运行的命令

> 现在，我们用 `CronJob` 来管理我们上面的 `Job` 任务，定义如下所示的资源清单：(cronjob-demo.yaml)

```
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: cronjob-demo
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: hello
            image: busybox
            args:
            - "bin/sh"
            - "-c"
            - "for i in 9 8 7 6 5 4 3 2 1; do echo $i; done"
```

> 这里的 Kind 变成了 `CronJob` 了，要注意的是`.spec.schedule`字段是必须填写的，用来指定任务运行的周期，格式就和 `crontab` 一样，另外一个字段是`.spec.jobTemplate`, 用来指定需要运行的任务，格式当然和 `Job` 是一致的。还有一些值得我们关注的字段 

```
.spec.successfulJobsHistoryLimit
```

 和 

```
.spec.failedJobsHistoryLimit
```

，表示历史限制，是可选的字段，指定可以保留多少完成和失败的 `Job`，默认没有限制，所有成功和失败的 `Job` 都会被保留。然而，当运行一个 `CronJob` 时，`Job` 可以很快就堆积很多，所以一般推荐设置这两个字段的值。如果设置限制的值为 0，那么相关类型的 `Job` 完成后将不会被保留。

> 我们直接新建上面的资源对象：

```
$ kubectl create -f cronjob-demo.yaml
cronjob "cronjob-demo" created
```

> 然后可以查看对应的 Cronjob 资源对象：

```
$ kubectl get cronjob
NAME           SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
cronjob-demo   */1 * * * *   False     0        <none>          7s
```

> 稍微等一会儿查看可以发现多了几个 Job 资源对象，这个就是因为上面我们设置的 CronJob 资源对象，每1分钟执行一个新的 Job：

```
$ kubectl get job 
NAME                      COMPLETIONS   DURATION   AGE
cronjob-demo-1574147280   1/1           6s         2m45s
cronjob-demo-1574147340   1/1           11s        105s
cronjob-demo-1574147400   1/1           5s         45s
$ kubectl get pods
NAME                      READY   STATUS      RESTARTS   AGE
cronjob-demo-1574147340-ksd5x   0/1     Completed           0          3m7s
cronjob-demo-1574147400-pts94   0/1     Completed           0          2m7s
cronjob-demo-1574147460-t5hcd   0/1     Completed           0          67s
cronjob-demo-1574147520-vmjfr   0/1     ContainerCreating   0          7s
```

> 这个就是 CronJob 的基本用法，一旦不再需要 CronJob，我们可以使用 kubectl 命令删除它：

```
$ kubectl delete cronjob cronjob-demo
cronjob "cronjob-demo" deleted
```

> 不过需要注意的是这将会终止正在创建的 Job，但是运行中的 Job 将不会被终止，不会删除 Job 或 它们的 Pod。