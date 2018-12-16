# Docker存储

Docker 为容器提供了两种存放数据的资源：

- 由 storage driver 管理的镜像层和容器层。
- Data Volume。

我们会详细讨论它们的原理和特性。

## storage driver

在前面镜像章节我们学习到 Docker 镜像的分层结构，简单回顾一下

![Docker分层结构](/assets/Docker分层结构.PNG)