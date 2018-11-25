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

构建过程如下图所示：

![镜像构建过程](/assets/镜像构建过程.png)

可以看到，新镜像是从 base 镜像一层一层叠加生成的。每安装一个软件，就在现有镜像的基础上增加一层。

这样的分层结构最大的好处就是共享资源。

比如：有多个镜像都从相同的 base 镜像构建而来，那么 Docker Host 只需在磁盘上保存一份 base 镜像；同时内存中也只需加载一份 base 镜像，就可以为所有容器服务了，而且镜像的每一层都可以被共享。

如果多个容器共享一份基础镜像，当某个容器修改了基础镜像的内容，比如 /etc 下的文件，这时其他容器的 /etc 是不会被修改的，修改会被限制在单个容器内。
这就是容器 Copy-on-Write 特性。

## 可写的容器层

当容器启动时，一个新的可写层被加载到镜像的顶部。
这一层通常被称作“容器层”，“容器层”之下的都叫“镜像层”。

![可写的容器层](/assets/可写的容器层.png)

所有对容器的改动 - 无论添加、删除、还是修改文件都只会发生在容器层中。

**只有容器层是可写的，容器层下面的所有镜像层都是只读的。**

镜像层数量可能会很多，所有镜像层会联合在一起组成一个统一的文件系统。如果不同层中有一个相同路径的文件，比如 /a，上层的 /a 会覆盖下层的 /a，也就是说用户只能访问到上层中的文件 /a。在容器层中，用户看到的是一个叠加之后的文件系统。

- 添加文件
    在容器中创建文件时，新文件被添加到容器层中。

- 读取文件 
    在容器中读取某个文件时，Docker 会从上往下依次在各镜像层中查找此文件。一旦找到，打开并读入内存。

- 修改文件 
    在容器中修改已存在的文件时，Docker 会从上往下依次在各镜像层中查找此文件。一旦找到，立即将其复制到容器层，然后修改之。

- 删除文件 
    在容器中删除文件时，Docker 也是从上往下依次在镜像层中查找此文件。找到后，会在容器层中记录下此删除操作。

只有当需要修改时才复制一份数据，这种特性被称作 Copy-on-Write。可见，容器层保存的是镜像变化的部分，不会对镜像本身进行任何修改。

这样就解释了我们前面提出的问题：容器层记录对镜像的修改，所有镜像层都是只读的，不会被容器修改，所以镜像可以被多个容器共享。

## 构建镜像

Docker 提供了两种构建镜像的方法：

- docker commit 命令
- Dockerfile 构建文件

### docker commit 构建

docker commit 命令是创建新镜像最直观的方法，其过程包含三个步骤：

1. 运行容器
2. 修改容器
3. 将容器保存为新的镜像

Docker 并不建议用户通过这种方式构建镜像。原因如下：

1. 这是一种手工创建镜像的方式，容易出错，效率低且可重复性弱。比如要在 debian base 镜像中也加入 vi，还得重复前面的所有步骤。
2. 更重要的：使用者并不知道镜像是如何创建出来的，里面是否有恶意程序。也就是说无法对镜像进行审计，存在安全隐患。

既然 docker commit 不是推荐的方法，我们干嘛还要花时间学习呢？

原因是：即便是用 Dockerfile（推荐方法）构建镜像，底层也 docker commit 一层一层构建新镜像的。学习 docker commit 能够帮助我们更加深入地理解构建过程和镜像的分层结构。

### Dockerfile 构建

Dockerfile 是一个文本文件，记录了镜像构建的所有步骤。

![Dockerfile构建镜像1](/assets/Dockerfile构建镜像1.PNG)

![Dockerfile构建镜像2](/assets/Dockerfile构建镜像2.PNG)

构建过程：

    1. 运行 docker build 命令，-t 将新镜像命名为 goujian:1.0，命令末尾的 . 指明 build context 为当前目录。Docker 默认会从 build context 中查找 Dockerfile 文件，我们也可以通过 -f 参数指定 Dockerfile 的位置。
    2. 从这步开始就是镜像真正的构建过程。首先 Docker 将 build context 中的所有文件发送给 Docker daemon。build context 为镜像构建提供所需要的文件或目录。Dockerfile 中的 ADD、COPY 等命令可以将 build context 中的文件添加到镜像。此例中，build context 为当前目录 /root/zhangpeng/test，该目录下的所有文件和子目录都会被发送给 Docker daemon。所以，使用 build context 就得小心了，不要将多余文件放到 build context，特别不要把 /、/usr 作为 build context，否则构建过程会相当缓慢甚至失败。
    3. 执行 FROM，将 hub.yun.paic.com.cn/anbot-ci/tomcat:3.1 作为 base 镜像。ubuntu 镜像 ID 为 4a521e439418。
    4. 执行 RUN，打印hello world!!!，启动 ID 为 8a2d49a04152 的临时容器，在容器中通过 echo "hello world!!!"，执行成功后，将容器保存为镜像，其 ID 为 c344d9f990ab。**备注：这一步底层使用的是类似 docker commit 的命令。**
    
    
docker history 会显示镜像的构建历史，也就是 Dockerfile 的执行过程。

![构建历史](/assets/构建历史.PNG)

goujian:1.0 与 hub.yun.paic.com.cn/anbot-ci/tomcat:3.1 镜像相比，确实只是多了顶部的一层 c344d9f990ab，由 echo 命令创建，大小为 0B。docker history 也向我们展示了镜像的分层结构，每一层由上至下排列。
<missing>标记的历史表示无法获取 IMAGE ID，通常从 Docker Hub 下载的镜像会有这个问题。

### 镜像的缓存特性

Docker 会缓存已有镜像的镜像层，构建新镜像时，如果某镜像层已经存在，就直接使用，无需重新创建。

![构建历史2](/assets/构建历史2.PNG)

1. 之前已经运行过相同的 RUN 指令，这次直接使用缓存中的镜像层 c344d9f990ab。
2. 执行 新的 RUN 指令。其过程是启动临时容器，创建history.txt并将process写入，提交新的镜像层 316c44fb45bda，删除临时容器 0d858a4b49f7。

在 goujian:1.0 镜像上直接添加一层就得到了新的镜像 goujian:2.0。

如果我们希望在构建镜像时不使用缓存，可以在 docker build 命令中加上 --no-cache 参数。

Dockerfile 中每一个指令都会创建一个镜像层，上层是依赖于下层的。无论什么时候，只要某一层发生变化，其上面所有层的缓存都会失效。

也就是说，如果我们改变 Dockerfile 指令的执行顺序，或者修改或添加指令，都会使缓存失效。

除了构建时使用缓存，Docker 在下载镜像时也会使用。例如我们下载 httpd 镜像。

![Docker缓存](/assets/Docker缓存.PNG)