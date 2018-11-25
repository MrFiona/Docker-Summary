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

![linux操作系统](/assets/linux操作系统.png)

内核空间是 kernel，Linux 刚启动时会加载 bootfs 文件系统，之后 bootfs 会被卸载掉。

用户空间的文件系统是 rootfs，包含我们熟悉的 /dev, /proc, /bin 等目录。

对于 base 镜像来说，底层直接用 Host 的 kernel，自己只需要提供 rootfs 就行了。

而对于一个精简的 OS，rootfs 可以很小，只需要包括最基本的命令、工具和程序库就可以了。相比其他 Linux 发行版，CentOS 的 rootfs 已经算臃肿的了，alpine 还不到 10MB。

我们平时安装的 CentOS 除了 rootfs 还会选装很多软件、服务、图形桌面等，需要好几个 GB 就不足为奇了。

![容器与主机共享内核](/assets/容器与主机共享内核.png)

- ① Host kernel 为 4.4.0-31
- ② 启动并进入 CentOS 容器
- ③ 验证容器是 CentOS 7
- ④ 容器的 kernel 版本与 Host 一致

容器只能使用 Host 的 kernel，并且不能修改。
所有容器都共用 host 的 kernel，在容器中没办法对 kernel 升级。如果容器对 kernel 版本有要求（比如应用只能在某个 kernel 版本下运行），则不建议用容器，这种场景虚拟机可能更合适。

## 镜像的分层结构

Docker 支持通过扩展现有镜像，创建新的镜像。

实际上，Docker Hub 中 99% 的镜像都是通过在 base 镜像中安装和配置需要的软件构建出来的。比如我们现在构建一个新的镜像，Dockerfile 如下：

![镜像的分层结构](/assets/镜像的分层结构.png)

- ① 新镜像不再是从 scratch 开始，而是直接在 Debian base 镜像上构建。
- ② 安装 emacs 编辑器。
- ③ 安装 apache2。
- ④ 容器启动时运行 bash。