# Docker镜像

镜像是 Docker 容器的基石，容器是镜像的运行实例，有了镜像才能启动容器。

## base镜像

base 镜像有两层含义：

- 不依赖其他镜像，从 scratch 构建。
- 其他镜像可以之为基础进行扩展。

所以，能称作 base 镜像的通常都是各种 Linux 发行版的 Docker 镜像，比如 Ubuntu, Debian, CentOS 等。

![base镜像](/assets/base镜像.png)

从官网上拉下来的centos镜像仅仅200MB，平时我们安装一个 CentOS 至少都有几个 GB，相信这是几乎所有 Docker 初学者都会有的疑问，包括我自己。下面我们来解释这个问题。

Linux 操作系统由内核空间和用户空间组成。如下图所示：