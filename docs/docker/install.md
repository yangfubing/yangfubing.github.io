
## 安装

> 要安装 Docker CE（社区）版本，推荐使用 CentOS 7，使用 `overlay2` 存储驱动程序。

### 卸载旧版本 

> 旧版本的 Docker 叫 `docker` 或者 `docker-engine`，现在的版本被称为`docker-ce`，如果之前安装过，则先卸载掉之前的版本：

```
$ sudo yum remove docker \\
                  docker-client \\
                  docker-client-latest \\
                  docker-common \\
                  docker-latest \\
                  docker-latest-logrotate \\
                  docker-logrotate \\
                  docker-engine
```

### 安装 Docker-CE 

> 要安装 Docker 有多种方法，大多数用户会设置 Docker 的存储仓库来进行安装，这种方式可以简化我们的安装和升级，当然也推荐使用这种方式。

> 但是如果你的服务器无法访问互联网则可以使用手动下载 RPM 安装包进行安装，如果你要使用这种方式安装，可以在页面 https://download.docker.com/linux/centos/7/x86_64/stable/Packages/ 下载自己想要安装的 `.rpm` 文件然后直接安装即可。

> 我们这里通过在线方式进行安装，首先安装仓库，先安装一些依赖包，其中`yum-utils`提供了`yum-config-manager`的一些实用的功能：

```
$ sudo yum install -y yum-utils
```

> 然后用下面的命令添加 `stable` 版本的 docker 仓库：

```
$ sudo yum-config-manager \\
    --add-repo \\
    https://download.docker.com/linux/centos/docker-ce.repo
```

> 添加完成后就可以安装了，如果要安装最新版本，直接执行下面的命令即可：

```
$ yum install docker-ce docker-ce-cli containerd.io
```

> 如果要安装指定的版本，可以通过如下命令来获取可安装的版本：

```
$ yum list docker-ce --showduplicates | sort -r
 * updates: mirrors.tuna.tsinghua.edu.cn
Loading mirror speeds from cached hostfile
Loaded plugins: fastestmirror, langpacks
 * extras: mirrors.huaweicloud.com
docker-ce.x86_64            3:4-el7                     docker-ce-stable
......
docker-ce.x86_64            3:9-el7                     docker-ce-stable
docker-ce.x86_64            3:8-el7                     docker-ce-stable
docker-ce.x86_64            3:7-el7                     docker-ce-stable
docker-ce.x86_64            3:6-el7                     docker-ce-stable
......
docker-ce.x86_64            ce-el7                    docker-ce-stable
```

> 该软件包名称是软件包名称（docker-ce）加上版本字符串（第二列），从第一个冒号（:）开始，直至第一个连字符，并用连字符（-）分隔 ）。 比如这里我们来安装 docker-ce-18.09.9 版本：

```
$ sudo yum install docker-ce-9 docker-ce-cli-9 containerd.io -y
```

> 安装完成，但是还没有启动，由于我这里的磁盘空间不大，所以需要更改下 Docker 的根目录，将默认的 `/var/lib/docker` 更改为 `/data/docker` 目录，添加如下配置文件：

```
$ mkdir -p /etc/docker
$ vi /etc/docker/daemon.json
{
  "registry-mirrors" : [
    "https://ot2k4dmirror.aliyuncs.com/"
  ],
  "graph": "/data/docker"
}
```

> 镜像加速器

> 使用加速器可以提升获取 Docker 官方镜像的速度，建议注册阿里云帐号，使用阿里云提供的镜像加速服务，地址：https://cr.console.aliyun.com/cn-beijing/instances/mirrors。

> 然后执行下面的命令启动 docker：

```
# 设置为开机启动
$ systemctl enable docker  
$ systemctl daemon-reload
# 启动 docker
$ systemctl start docker
```

> 启动成功后，可以通过如下命令查看 Docker 相关信息：

```
$ docker info
docker info
Containers: 0
 Running: 0
 Paused: 0
 Stopped: 0
Images: 0
Server Version: 9
Storage Driver: overlay2
 Backing Filesystem: extfs
 Supports d_type: true
 Native Overlay Diff: false
Logging Driver: json-file
Cgroup Driver: cgroupfs
Plugins:
 Volume: local
 Network: bridge host macvlan null overlay
 Log: awslogs fluentd gcplogs gelf journald json-file local logentries splunk syslog
Swarm: inactive
Runtimes: runc
Default Runtime: runc
Init Binary: docker-init
containerd version: b34a5c8af56e510852c35414db4c1f4fa6172339
runc version: 3e425f80a8c931f88e6d94a8c831b9d5aa481657
init version: fec3683
Security Options:
 seccomp
  Profile: default
Kernel Version: 0-3elx86_64
Operating System: CentOS Linux 7 (Core)
OSType: linux
Architecture: x86_64
CPUs: 4
Total Memory: 64GiB
Name: pnhmltdr
ID: VIBG:JCTE:KBVO:ETXD:OZH4:SXCH:MA7Y:YFFB:KQEX:OLAM:I25P:F2TM
Docker Root Dir: /data/docker
Debug Mode (client): false
Debug Mode (server): false
Registry: https://index.docker.io/v1/
Labels:
Experimental: false
Insecure Registries:
 10/8
Registry Mirrors:
 https://ot2k4dmirror.aliyuncs.com/
Live Restore Enabled: false
Product License: Community Engine
```

> 可以看到安装的版本是 18.09.9，默认的存储驱动程序是 `overlay2`，而且 `Root Dir` 也已经是我们更改后的目录了。

### 测试 

> Docker 启动成功后，我们可以使用下面的命令来运行一个测试容器：

```
$ docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
1b930d010525: Pull complete
Digest: sha256:c3b4ada4687bbaa170745b3e4dd8ac3f194ca95b2d0518b417fb47e5879d9b5f
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
  The Docker client contacted the Docker daemon.
  The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
  The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
  The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

> 看到如上的一些信息打印出来证明我们的 Docker 就已经安装成功了。
