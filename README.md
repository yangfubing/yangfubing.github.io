# [yangfubing.github.io](https://yangfubing.github.io)

## 云原生

---
本仓库下存放个人博客的源文件。持续更新，欢迎 star。

如果大家觉得那里写的不合适的可以给我提 Issue

[Github](https://github.com/burningmyself)

[Gitee](https://gitee.com/https://gitee.com/burningmyself)

![GitHub issues](https://img.shields.io/github/issues/burningmyself/burningmyself.github.io)
![GitHub forks](https://img.shields.io/github/forks/burningmyself/burningmyself.github.io)
![GitHub stars](https://img.shields.io/github/stars/burningmyself/burningmyself.github.io)
![GitHub license](https://img.shields.io/github/license/burningmyself/burningmyself.github.io)
![Twitter](https://img.shields.io/twitter/url?url=https%3A%2F%2Fgithub.com%2Fburningmyself%2Fburningmyself.github.io)

## 捐赠

如果你觉得这写文章能帮助到了你，你可以帮作者买一杯果汁表示鼓励
![pay](docs/assets/pay.png)

[Paypal Me](https]//paypal.me/yangfubing)

## 目录

* [介绍](docs/index.md)
* [Docker基础]
  * [简介](docs/docker/overview.md)
  * [安装](docs/docker/install.md)
  * [基本操作](docs/docker/basic.md)
  * [Dockerfile](docs/docker/dockerfile_usage.md)
  * [Dockerfile实践](docs/docker/dockerfile_practice.md)
* [Kubernetes基础]
  * [简介](docs/kubernetes_basic/overview.md)
  * [安装](docs/kubernetes_basic/install.md)  
  * [资源清单](docs/kubernetes_basic/yaml.md)  
  * [Pod原理](docs/kubernetes_basic/pod.md)
  * [Pod的生命周期](docs/kubernetes_basic/pod_life.md)
  * [Pod使用进阶](docs/kubernetes_basic/pod_deep.md)
* [Kubernetes控制器]
  * [ReplicaSet](docs/kubernetes_controller/replicaset.md)
  * [Deployment](docs/kubernetes_controller/deployment.md)
  * [StatefulSet](docs/kubernetes_controller/statefulset.md)
  * [DaemonSet](docs/kubernetes_controller/daemonset.md)
  * [Job](docs/kubernetes_controller/job.md)
  * [HPA](docs/kubernetes_controller/hpa.md)
* [Kubernetes配置管理]
  * [ConfigMap](docs/kubernetes_config/config_map.md)
  * [Secret](docs/kubernetes_config/secret.md)
  * [ServiceAccount](docs/kubernetes_config/service_account.md)
* [Kubernetes安全]  
  * [RBAC](docs/kubernetes_security/rbac.md)
  * [Security Context](docs/kubernetes_security/security_context.md)
  * [Spring常用注解](docs/kubernetes_security/admission.md)
* [Kubernetes网络]
  * [网络插件](docs/kubernetes_network/flannel.md)
  * [网络策略](docs/kubernetes_network/policy.md)
  * [Service 服务](docs/kubernetes_network/service.md)
  * [Ingress]
    * [NGINX Controller](docs/kubernetes_network/ingress/nginx.md)
    * [Traefik](docs/kubernetes_network/ingress/traefik.md)
* [Kubernetes调度器]
  * [调度器介绍](docs/kubernetes_scheduler/overview.md)
  * [Pod 调度](docs/kubernetes_scheduler/usage.md)
* [Kubernetes存储]  
  * [Local 本地存储](docs/kubernetes_stroage/local.md)
  * [Ceph 存储](docs/kubernetes_stroage/ceph.md)
  * [存储原理](docs/kubernetes_stroage/csi.md)
* [Helm 包管理]
  * [Helm](DOCS/kubernetes_package/helm.md)
  * [Charts](DOCS/kubernetes_package/charts.md)
  * [模板开发]
    * [内置对象](DOCS/kubernetes_package/templates/objects.md)
    * [Values](DOCS/kubernetes_package/templates/values.md)
    * [函数和管道](DOCS/kubernetes_package/templates/function.md)
    * [流程控制](DOCS/kubernetes_package/templates/flow.md)
    * [变量](DOCS/kubernetes_package/templates/variables.md)
    * [命名模板](DOCS/kubernetes_package/templates/named_templates.md)
    * [访问文件](DOCS/kubernetes_package/templates/access_files.md)
    * [NOTES.txt 文件](DOCS/kubernetes_package/templates/notes_files.md)
    * [子chart与全局值](DOCS/kubernetes_package/templates/subcharts_and_globals.md)
    * [模板调试](DOCS/kubernetes_package/templates/debug.md)
  * [Chart Hooks](DOCS/kubernetes_package/chart_hooks.md)
  * [示例](DOCS/kubernetes_package/example.md)
* [Kubernetes 监控]
  * [Prometheus](DOCS/kubernetes_monitor/prometheus.md)
  * [Grafana](DOCS/kubernetes_monitor/grafana.md)
  * [Promql](DOCS/kubernetes_monitor/promql.md)
  * [Alertmanager](DOCS/kubernetes_monitor/alertmanager.md)
  * [Prometheus Thanos](DOCS/kubernetes_monitor/thanos.md)
  * [Prometheus Adpater](DOCS/kubernetes_monitor/adapter.md)
* [Prometheus Operator]
  * [安装](DOCS/kubernetes_monitor/operator/install.md)
  * [自定义监控报警](DOCS/kubernetes_monitor/operator/custom.md)
  * [高级配置](DOCS/kubernetes_monitor/operator/advance.md)  
* [Kubernetes 日志]
  * [Grafana](DOCS/kubernetes_logs/architec.md)
  * [efk](DOCS/kubernetes_logs/efk.md)
* [Kubernetes DevOps]
  * [Jenins](DOCS/kubernetes_devops/jenkins.md)
  * [Gitlab](DOCS/kubernetes_devops/gitlab.md)
  * [Harbor](DOCS/kubernetes_devops/harbor.md)
  * [Jenkins Pipeline](DOCS/kubernetes_devops/jenkins_pipeline.md)
  * [Tekton](DOCS/kubernetes_devops/token.md)
* [Service Mesh 实践]
  * [Istio](DOCS/ kubernetes_service_mesh/istio.md)
  * [流量管理](DOCS/kubernetes_service_mesh/traffic.md)
  * [可观测性](DOCS/kubernetes_service_mesh/observability.md)
  * [安全管控](DOCS/kubernetes_service_mesh/security.md)
  * [CertManager](DOCS/kubernetes_service_mesh/cert_manager.md)
* [Kubernetes 多租户]
  * [多租户介绍](DOCS/kubernetes_tenant/tenant.md)
  * [多租户介绍](DOCS/kubernetes_tenant/quota.md)
  * [多租户介绍](DOCS/kubernetes_tenant/psp.md)
* [Operator]
  * [CRD](DOCS/kubernetes_operator/crd.md)
  * [Operator](DOCS/kubernetes_operator/operator.md)
* [实践技巧]
  * [技巧](DOCS/kubernetes_maintain/skill.md)  
  * [技巧](DOCS/kubernetes_maintain/update.md)
  * [技巧](DOCS/kubernetes_maintain/update_cluster.md)
  * [技巧](DOCS/kubernetes_maintain/reserved.md)
* [源码]
  * [kubelet源码](DOCS/kubernetes_code/kubelet.md)
  * [client-go]
    * [workqueue](DOCS/kubernetes_code/client_go/workqueue.md)
    * [indexer](DOCS/kubernetes_code/client_go/indexer.md)
    * [workqueue](DOCS/kubernetes_code/client_go/deltafifo.md)
* [应用示例]
  * [应用部署](DOCS/kubernetes_example/wordpress.md)
