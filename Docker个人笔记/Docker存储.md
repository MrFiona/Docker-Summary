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