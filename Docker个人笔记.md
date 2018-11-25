# Docker个人笔记

本文主要是个人对Docker的一些学习总结。


## Docker简介

Docker是一个开源的引擎，可以轻松的为任何应用创建一个轻量级的、可移植的、自给自足的容器。开发者在笔记本上编译测试通过的容器可以批量地在生产环境中部署，包括VMs（虚拟机）、bare metal、OpenStack 集群和其他的基础应用平台。

### Docker 架构详解

Docker 的核心组件包括：

- Docker 客户端 - Client
- Docker 服务器 - Docker daemon
- Docker 镜像 - Image
- Registry 镜像仓库
- Docker 容器 - Container
    
Docker 架构如下图所示：

![Docker架构图](/assets/Dcoker架构图.jpg)
![Docker图解](/assets/20170214190946299.png)

Docker 采用的是 Client/Server 架构。客户端向服务器发送请求，服务器负责构建、运行和分发容器。客户端和服务器可以运行在同一个 Host 上，客户端也可以通过 socket 或 REST API 与远程的服务器通信。

#### Docker 客户端

最常用的 Docker 客户端是 docker 命令。通过 docker 我们可以方便地在 Host 上构建和运行容器。除了 docker 命令行工具，用户也可以通过 REST API 与服务器通信。

#### Docker 服务器

Docker daemon 是服务器组件，以 Linux 后台服务的方式运行。

```
systemctl status docker.service
```

Docker daemon 运行在 Docker host 上，负责创建、运行、监控容器，构建、存储镜像。


#### Docker 镜像

可将 Docker 镜像看着只读模板，通过它可以创建 Docker 容器。

例如某个镜像可能包含一个 Ubuntu 操作系统、一个 Apache HTTP Server 以及用户开发的 Web 应用。

镜像有多种生成方法：

- 可以从无到有开始创建镜像
- 也可以下载并使用别人创建好的现成的镜像
- 还可以在现有镜像上创建新的镜像

我们可以将镜像的内容和创建步骤描述在一个文本文件中，这个文件被称作 Dockerfile，通过执行 docker build <docker-file> 命令可以构建出 Docker 镜像，后面我们会讨论。

#### Docker 容器

Docker 容器就是 Docker 镜像的运行实例。

用户可以通过 CLI（docker）或是 API 启动、停止、移动或删除容器。可以这么认为，对于应用软件，镜像是软件生命周期的构建和打包阶段，而容器则是启动和运行阶段。

#### Registry 镜像仓库

Registry 是存放 Docker 镜像的仓库，Registry 分私有和公有两种。

Docker Hub（https://hub.docker.com/）是默认的 Registry，由 Docker 公司维护，上面有数以万计的镜像，用户可以自由下载和使用。

出于对速度或安全的考虑，用户也可以创建自己的私有 Registry。后面我们会学习如何搭建私有 Registry。

docker pull 命令可以从 Registry 下载镜像。
docker run 命令则是先下载镜像（如果本地没有），然后再启动容器。
