# Docker 网络

本文主要介绍Docker 提供的几种原生网络，以及如何创建自定义网络, 容器之间如何通信，以及容器与外界如何交互。

## none 网络

故名思议，none 网络就是什么都没有的网络。挂在这个网络下的容器除了 lo，没有其他任何网卡。容器创建时，可以通过 --network=none 指定使用 none 网络。

![none网络](/assets/none网络.PNG)

> **应用场景**：一些对安全性要求高并且不需要联网的应用可以使用 none 这样一个封闭的网络。比如某个容器的唯一用途是生成随机密码，就可以放到 none 网络中避免密码被窃取。当然大部分容器是需要网络的，我们接着看 host 网络。

## host 网络

连接到 host 网络的容器共享 Docker host 的网络栈，容器的网络配置与 host 完全一样。可以通过 --network=host 指定使用 host 网络。

![host网络](/assets/host网络-1.PNG)

在容器中可以看到 host 的所有网卡，并且连 hostname 也是 host 的。

> **应用场景**：直接使用 Docker host 的网络最大的好处就是性能，如果容器对网络传输效率有较高要求，则可以选择 host 网络。当然不便之处就是牺牲一些灵活性，比如要考虑端口冲突问题，Docker host 上已经使用的端口就不能再用了。Docker host 的另一个用途是让容器可以直接配置 host 网路。比如某些跨 host 的网络解决方案，其本身也是以容器方式运行的，这些方案需要对网络进行配置，比如管理 iptables，大家将会在后面进阶技术章节看到。

## bridge 网络

Docker 安装时会创建一个命名为 docker0 的 linux bridge。如果不指定--network，创建的容器默认都会挂到 docker0 上。

下面以当创建一个容器为例讲解过程:

![bridge网络1](/assets/bridge网络1.PNG)

- 一个新的网络接口 veth37e6429 被挂到了 docker0 上，veth37e6429 就是新创建容器的虚拟网卡。
- 新创建的容器内有一个虚拟网卡 eth0@if2744

实际上 eth0@if2744 和 veth37e6429 是一对 veth pair。veth pair 是一种成对出现的特殊网络设备，可以把它们想象成由一根虚拟网线连接起来的一对网卡，网卡的一头（eth0@if2744）在容器中，另一头（veth37e6429）挂在网桥 docker0 上，其效果就是将 eth0@if2744 也挂在了 docker0 上。

![bridge网络2](/assets/bridge网络2.PNG)

我们还看到 eth0@if2744 已经配置了 IP 172.17.0.4，为什么是这个网段呢？让我们通过 docker network inspect bridge 看一下 bridge 网络的配置信息:

![bridge网络3](/assets/bridge网络3.PNG)

原来 bridge 网络配置的 subnet 就是 172.17.0.0/16，并且网关是 172.17.0.1。这个网关在哪儿呢？大概你已经猜出来了，就是 docker0。

![bridge网络4](/assets/bridge网络4.PNG)

当前容器网络拓扑结构如图所示：

![bridge网络5](/assets/bridge网络5.png)

容器创建时，docker 会自动从 172.17.0.0/16 中分配一个 IP，这里 16 位的掩码保证有足够多的 IP 可以供容器使用。

## user-defined 网络

除了 none, host, bridge 这三个自动创建的网络，用户也可以根据业务需要创建 user-defined 网络。

Docker 提供三种 user-defined 网络驱动：bridge, overlay 和 macvlan。overlay 和 macvlan 用于创建跨主机的网络，我们后面有章节单独讨论。

我们可通过 bridge 驱动创建类似前面默认的 bridge 网络，例如：

![自定义网络1](/assets/自定义网络1.PNG)

查看一下当前 host 的网络结构变化：

![自定义网络2](/assets/自定义网络2.PNG)

新增了一个网桥 br-f74dcd35e69b，这里 f74dcd35e69b 正好新建 bridge 网络 zp_net 的短 id。执行 docker network inspect 查看一下 zp_net 的配置信息：

![自定义网络3](/assets/自定义网络3.PNG)

这里 172.18.0.0/16 是 Docker 自动分配的 IP 网段。当然也可以指定IP网段，只需在创建网段时指定 --subnet 和 --gateway 参数：

![自定义网络4](/assets/自定义网络4.PNG)

这里我们创建了新的 bridge 网络 zp_net2，网段为 172.22.16.0/24，网关为 172.22.16.1。与前面一样，网关在 zp_net2 对应的网桥 br-9244654a9f6f 上：

![自定义网络5](/assets/自定义网络5.PNG)

容器要使用新的网络，需要在启动时通过 --network 指定：

![自定义网络6](/assets/自定义网络6.PNG)

容器分配到的 IP 为 172.22.16.2。

到目前为止，容器的 IP 都是 docker 自动从 subnet 中分配，我们也可以通过--ip指定一个静态 IP。

![自定义网络7](/assets/自定义网络7.PNG)

> 注：只有使用 --subnet 创建的网络才能指定静态 IP。

zp_net 创建时没有指定 --subnet，如果指定静态 IP 报错如下：

![自定义网络8](/assets/自定义网络8.PNG)

![自定义网络10](/assets/自定义网络10.PNG)

![自定义网络9](/assets/自定义网络-9.PNG)

当前 docker host 的网络拓扑结构如下图:

![自定义网络11](/assets/自定义网络11.PNG)

两个tomcat-1和tomcat-2同期都是挂在zp_net2上，应该能够互通，验证如下如所示：

![自定义网络12](/assets/自定义网络12.PNG)

可见同一网络中的容器，网管之间都是可以通信的。

从拓扑图可以看出zp_net2和默认的bridge网络属于不同的网桥，应该不能通信，验证如下图所示：

![自定义网络13](/assets/自定义网络13.PNG)

可见不同网络中的容器之间是不能够通信的，如果要让它们之间可以通信则可以通过增加路由的方式实现，请大家思考下怎么实现？？


## 容器间通信

容器之间可通过 IP，Docker DNS Server 或 joined 容器三种方式通信。

### IP 通信

两个容器要能通信，必须要有属于同一个网络的网卡。

满足这个条件后，容器就可以通过 IP 交互了。具体做法是在容器创建时通过 --network 指定相应的网络，或者通过 docker network connect 将现有容器加入到指定网络。可参考上一节 httpd 和 busybox 的例子，这里不再赘述。

### 容器与外部通信

容器与外部通信主要有两个方向：
- 容器访问外部世界
- 外部世界访问容器

#### 容器访问外部世界

这里的外部世界指的是容器网络以外的网络环境，并非特指Internet。

下面模拟容器访问外部网络的过程

查看docker host上的iptables规则，如下图所示：

![NAT](/assets/NAT.png)

在NAT表中，又这么一条规则：

其含义是：如果网桥docker0收到来自172.17.0.0/16网段的外出包，把它交给MASQUERADE处理。而MASQUERADE的处理方式则是将包的源地址替换成host的地址发送出去，即做一次网络地址转换(NAT)。

下面我们通过tcpdump查看地址是如何转换的，先查看docker host的路由表，如下图所示：

默认路由通过eth0发出去，所以我们要同时监控eth0和docker0上的icmp(ping)数据包。

当centos ping 10.25.80.224时，tcpdump输出如下：

docker0收到centos的ping包，源地址为容器的IP 172.17.0.2，交给MASQUERADE处理，这时，在eth0上我们看到了变化，如下图所示：

ping包的源地址变成了eth0的IP 10.25.73.60，这就是iptable NAT规则处理的结果，从而保证数据包能够到达外网。

1. centos发送ping包：