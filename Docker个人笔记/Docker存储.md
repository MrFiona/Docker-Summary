# Docker存储

Docker 为容器提供了两种存放数据的资源：

- 由 storage driver 管理的镜像层和容器层。
- Data Volume。

我们会详细讨论它们的原理和特性。

## storage driver

在前面镜像章节我们学习到 Docker 镜像的分层结构，简单回顾一下

![host网络](/assets/Docker分层结构.webp)

容器由最上面一个可写的容器层，以及若干只读的镜像层组成，容器的数据就存放在这些层中。这样的分层结构最大的特性是 Copy-on-Write：

- 新数据会直接存放在最上面的容器层。
- 修改现有数据会先从镜像层将数据复制到容器层，修改后的数据直接保存在容器层中，镜像层保持不变。
- 如果多个层中有命名相同的文件，用户只能看到最上面那层中的文件。

分层结构使镜像和容器的创建、共享以及分发变得非常高效，而这些都要归功于 Docker storage driver。正是 storage driver 实现了多层数据的堆叠并为用户提供一个单一的合并之后的统一视图。

Docker 支持多种 storage driver，有 AUFS、Device Mapper、Btrfs、OverlayFS、VFS 和 ZFS。它们都能实现分层的架构，同时又有各自的特性。对于 Docker 用户来说，具体选择使用哪个 storage driver 是一个难题，因为：

- 没有哪个 driver 能够适应所有的场景。
- driver 本身在快速发展和迭代。

不过 Docker 官方给出了一个简单的答案：
优先使用 Linux 发行版默认的 storage driver。

Docker 安装时会根据当前系统的配置选择默认的 driver。默认 driver 具有最好的稳定性，因为默认 driver 在发行版上经过了严格的测试，运行docker info查看默认driver。


对于某些容器，直接将数据放在由 storage driver 维护的层中是很好的选择，比如那些无状态的应用。无状态意味着容器没有需要持久化的数据，随时可以从镜像直接创建。

比如 busybox，它是一个工具箱，我们启动 busybox 是为了执行诸如 wget，ping 之类的命令，不需要保存数据供以后使用，使用完直接退出，容器删除时存放在容器层中的工作数据也一起被删除，这没问题，下次再启动新容器即可。

但对于另一类应用这种方式就不合适了，它们有持久化数据的需求，容器启动时需要加载已有的数据，容器销毁时希望保留产生的新数据，也就是说，这类容器是有状态的。

这就要用到 Docker 的另一种存储机制：Data Volume。

## Data Volume

Data Volume 本质上是 Docker Host 文件系统中的目录或文件，能够直接被 mount 到容器的文件系统中。Data Volume 有以下特点：

- Data Volume 是目录或文件，而非没有格式化的磁盘（块设备）。
- 容器可以读写 volume 中的数据。
- volume 数据可以被永久的保存，即使使用它的容器已经销毁。

`现在我们有数据层（镜像层和容器层）和 volume 都可以用来存放数据，前者放在数据层中。因为这部分内容是无状态的，应该作为镜像的一部分；后者放在 Data Volume 中。这是需要持久化的数据，并且应该与镜像分开存放。`

因为 volume 实际上是 docker host 文件系统的一部分，所以 volume 的容量取决于文件系统当前未使用的空间，目前还没有方法设置 volume 的容量。

在具体的使用上，docker 提供了两种类型的 volume：bind mount 和 docker managed volume。

### bind mount

bind mount 是将 host 上已存在的目录或文件 mount 到容器。

-v 的格式为 `<host path>:<container path>`, /usr/local/tomcat/logs。由于 /usr/local/tomcat/logs 已经存在，原有数据会被隐藏起来，取而代之的是 host /docker-test 中的数据，这与 linux mount 命令的行为是一致的。

![datavolume-1](/assets/datavolume-1.PNG)

host 中的修改确实生效了，bind mount 可以让 host 与容器共享数据。这在管理上是非常方便的。

![datavolume-1](/assets/datavolume-2.PNG)

即使容器没有了，bind mount 也还在，bind mount 是 host 文件系统中的数据，不会随着容器的消失而不存在。

`使用单一文件有一点要注意：host 中的源文件必须要存在，不然会当作一个新目录 bind mount 给容器。`

mount point 有很多应用场景，比如我们可以将源代码目录 mount 到容器中，在 host 中修改代码就能看到应用的实时效果。再比如将 mysql 容器的数据放在 bind mount 里，这样 host 可以方便地备份和迁移数据。

bind mount 的使用直观高效，易于理解，但它也有不足的地方：bind mount 需要指定 host 文件系统的特定路径，这就限制了容器的可移植性，当需要将容器迁移到其他 host，而该 host 没有要 mount 的数据或者数据不在相同的路径时，操作会失败。

移植性更好的方式是 docker managed volume。

### docker managed volume

docker managed volume 与 bind mount 在使用上的最大区别是不需要指定 mount 源，指明 mount point 就行了。以 tomcat 容器为例：

我们通过 -v 告诉 docker 需要一个 data volume，并将其 mount 到 /usr/local/tomcat/logs。data volume 可以在容器的配置信息中找到，执行 docker inspect 命令：

![docker-managed-volume-1](/assets/docker-managed-volume-1.PNG)

![docker-managed-volume-23](/assets/docker-managed-volume-2.PNG)

这里会显示容器当前使用的所有 data volume，包括 bind mount 和 docker managed volume。type为volume则为docker managed volume类型，type为bind则为bind mount类型。

Source 就是该 volume 在 host 上的目录。

每当容器申请 mount docker manged volume 时，docker 都会在/var/lib/docker/volumes 下生成一个目录，这个目录就是 mount 源。

![docker-managed-volume-3](/assets/docker-managed-volume-3.PNG)

docker managed volume 的创建过程：

1. 容器启动时，简单的告诉 docker "我需要一个 volume 存放数据，帮我 mount 到目录 /abc"。
2. docker 在 /var/lib/docker/volumes 中生成一个随机目录作为 mount 源。
3. 如果 /abc 已经存在，则将数据复制到 mount 源，
4. 将 volume mount 到 /abc

除了通过 docker inspect 查看 volume，我们也可以用 docker volume 命令：

![docker-managed-volume-3](/assets/docker-managed-volume-4.PNG)

目前，docker volume 只能查看 docker managed volume，还看不到 bind mount；同时也无法知道 volume 对应的容器，这些信息还得靠docker inspect。

我们已经学习了两种 data volume 的原理和基本使用方法，下面做个对比：

- 相同点：两者都是 host 文件系统中的某个路径。

- 不同点：

    |  | bind mount | docker managed volume |
    | :------| ------: | :------: |
    | volume 位置 | 可任意指定 | /var/lib/docker/volumes/... |
    | 对已有mount point 影响	 | 隐藏并替换为 volume | 原有数据复制到 volume |
    | 是否支持单个文件 | 支持 | 不支持，只能是目录 |
    | 权限控制 | 可设置为只读，默认为读写权限 | 无控制，均为读写权限 |
    | 移植性 | 移植性弱，与 host path 绑定 | 移植性强，无需指定 host 目录 |
    
### 数据共享

数据共享是 volume 的关键特性，本节我们详细讨论通过 volume 如何在容器与 host 之间，容器与容器之间共享数据。

#### 容器与 host 共享数据

我们有两种类型的 data volume，它们均可实现在容器与 host 之间共享数据，但方式有所区别。

对于 bind mount 是非常明确的：直接将要共享的目录 mount 到容器。具体请参考前面 tomcat的例子，不再赘述。

docker managed volume 就要麻烦点。由于 volume 位于 host 中的目录，是在容器启动时才生成，所以需要将共享数据拷贝到 volume 中。

![数据共享-1](/assets/数据共享-1.PNG)

docker cp 可以在容器和 host 之间拷贝数据，当然我们也可以直接通过 Linux 的 cp 命令复制到 /var/lib/docker/volumes/xxx。

#### 容器之间共享数据

第一种方法是将共享数据放在 bind mount 中，然后将其 mount 到多个容器。还是以 tomcat 为例，不过这次的场景复杂些，我们要创建三个 tomcat 容器组，它们挂在同一个主机目录，如下图所示：

![数据共享-2](/assets/数据共享-2.PNG)

`**结论：当主机的mount目录发生变化时，三个容器的目录发生了变化；当其中一个容器的目录发生了变化时，另外两个容器以及主机mount目录都发生了变化。**`

### volume container

volume container 是专门为其他容器提供 volume 的容器。它提供的卷可以是 bind mount，也可以是 docker managed volume。下面我们创建一个 volume container：

![数据共享-3](/assets/数据共享-3.PNG)

我们将容器命名为 vc_data（vc 是 volume container 的缩写）。注意这里执行的是 docker create 命令，这是因为 volume container 的作用只是提供数据，它本身不需要处于运行状态。容器 mount 了两个 volume, 通过 docker inspect 可以查看到这两个 volume。

![数据共享-4](/assets/数据共享-4.PNG)

其他容器可以通过 --volumes-from 使用 vc_data 这个 volume container：

![数据共享-5](/assets/数据共享-5.PNG)

三个 tomcat 容器都使用了 vc_data，看看它们现在都有哪些 volume，以 vc_data_1 为例：

![数据共享-6](/assets/数据共享-6.PNG)

可见，三个容器已经成功共享了 volume container 中的 volume。

下面我们讨论一下 volume container 的特点：

- 与 bind mount 相比，不必为每一个容器指定 host path，所有 path 都在 volume container 中定义好了，容器只需与 volume container 关联，实现了容器与 host 的解耦。
- 使用 volume container 的容器其 mount point 是一致的，有利于配置的规范和标准化，但也带来一定的局限，使用时需要综合考虑。

### data-packed volume container

在上一节的例子中 volume container 的数据归根到底还是在 host 里，有没有办法将数据完全放到 volume container 中，同时又能与其他容器共享呢？

当然可以，通常我们称这种容器为 data-packed volume container。其原理是将数据打包到镜像中，然后通过 docker managed volume 共享。

我们用下面的 Dockfile 构建镜像：

![数据共享-8](/assets/数据共享-8.PNG)

ADD 将docker-test目录文件添加到容器目录 /usr/local/apache2/htdocs。
VOLUME 的作用与 -v 等效，用来创建 docker managed volume，mount point 为 /usr/local/apache2/htdocs，因为这个目录就是 ADD 添加的目录，所以会将已有数据拷贝到 volume 中。build 新镜像 test-dockerfile:1.0，用新镜像创建 data-packed volume container，因为在 Dockerfile 中已经使用了 VOLUME 指令，这里就不需要指定 volume 的 mount point 了。启动 httpd 容器并使用 data-packed volume container：

![数据共享-7](/assets/数据共享-7.PNG)

容器能够正确读取 volume 中的数据。data-packed volume container 是自包含的，不依赖 host 提供数据，具有很强的移植性，非常适合 只适用静态文件的场景，比如应用的配置信息、web server 的静态文件等。

