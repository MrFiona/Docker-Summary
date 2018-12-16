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

这就要用到 Docker 的另一种存储机制：Data Volume，下一节我们讨论。