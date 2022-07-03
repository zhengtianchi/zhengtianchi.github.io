# kubernetes cni




# k8s cni explain in detail

## 1. Network

**在没有搭建 kubernetes 集群时，如果想让两台不同的物理节点上的容器之间进行通信，这就需要第三方工具来实现，常用的 calico 和 flannel。**

## 2. docker network mode

docker 有四种网络模式：

1. **none** :
   - 容器有独立的 Network namespace.
   - 将容器添加到一个容器专门的网络堆栈中，没有对外连接。
   - 只能使用 loopback 网络设备，容器只能使用 127.0.0.1 的本机网络。
2. **host** :
   - 此网络驱动直接使用宿主机的网络
   - 将容器添加到主机的网络堆栈中，没有隔离
   - 容器将不会虚拟出自己的网卡，配置自己的 IP 等，而是使用宿主机的 IP 和端口。
3. **bridge** :
   - 默认网络方式
   - 为每一个容器分配 IP ，并将容器连接到一个 docker0 虚拟网桥
4. **自定义网桥** :
   - 用户自定义网桥，有更多的灵活性、隔离性等。
5. **Network plugins** :
   - 可以安装和使用第三方的网络插件

## 3. CNI

**CNI(container network interface) CNCF下的一个项目，由coreOS提出**

通过插件的方式统一配置:

- **flannel** : 基于overlay 不支持网络策略
- **calico** : 基于BGP 支持网络策略
- **canal** : 支持网络策略

通过给 Kubelet 传递 CNI 选项：

- **–network-plugin=cni** 命令行选项来选择 CNI 插件。
- **–cni-conf-dir** （默认是 /etc/cni/net.d） 读取文件并使用该文件中的 CNI 配置来设置每个 pod 的网络。
- **–cni-bin-dir** （默认是 /opt/cni/bin）配置引用的任何所需的 CNI 插件都必须存在于该目录下。

**各种解决方案对比如下：**

| 特性/方案            | FLANNEL                                     | CALICO                                | MACVLAN                                          | OPENVSWITCH                        | 直接路由                                                     |
| -------------------- | ------------------------------------------- | ------------------------------------- | ------------------------------------------------ | ---------------------------------- | ------------------------------------------------------------ |
| **方案特性**         | 通过虚拟设备 flannel0 实现对 docker0 的管理 | 基于 BGP 协议的纯三层的网络方案       | 基于 Linux Kernel 的 macvlan 技术                | 基于隧道的虚拟路由技术             | 基于 Linux Kernel 的 vRouter 技术                            |
| **对底层网络的要求** | 三层互通(vxlan,udp) 二层互通(host-gw)       | 三层互通（ipip） 二层互通(bgp)        | 二层互通                                         | 三层互通                           | 二层互通                                                     |
| **配置难易度**       | 简单，基于 etcd                             | 简单，基于 etcd                       | 简单，直接使用宿主机网络，需要仔细规划IP地址范围 | 复杂，需要手工配置各个节点的bridge | 简单，使用宿主机 vRoute 功能，需要仔细规划每个node的IP地址范围 |
| **网络性能**         | host-gw > Vxlan > UDP                       | BGP 模式性能损失小，IPIP模式较小      | 性能损失可忽略                                   | 效能损失较小                       | 性能损失小                                                   |
| **网络连通性限制**   | 无                                          | 在不支持 BGP 协议的网络环境中无法使用 | 基于 macvlan 的容器无法与宿主机网络通信          | 无                                 | 在无法实现大二层互通的网络环境下无法使用                     |

## 4. Flannel 网络原理

### 4.1 Flannel 简介

Flannel 是 CoreOS 团队针对 Kubernetes 设计的一个网络规划服务，简单来说，它的功能是让集群中的不同节点主机创建的 Docker 容器都具有全集群唯一的虚拟 IP 地址。

Flannel 实质上是一种**覆盖网络(overlaynetwork)**。

> overlay 覆盖网络：
>
> - 也就是将 TCP 数据包装在另一种网络包里面进行路由转发和通信
> - 在基础网络的基础上叠加的一种虚拟网络技术模式，该网络的主机通过虚拟链路连接起来。

目前已经支持udp、vxlan、host-gw、aws-vpc、gce 和 alloc 路由等数据转发方式，默认的节点间数据通信方式是 UDP 转发。

#### 4.1.1 flannel 模式

flanneld可以在启动时通过配置文件指定不同的backend进行网络通信，目前比较成熟的backend有UDP、VXLAN和host-gateway三种。目前，**VXLAN是官方比较推崇的一种backend实现方式**。

- UDP模式和VXLAN模式基于**三层网络层**即可实现，而host-gateway模式就必须要求集群所有机器在同一个广播域，也就是需要在二层网络同一个交换机下才能实现。 

- host-gateway一般用于对网络性能要求比较高的场景，但需要基础网络架构的支持；UDP则用于测试及一般比较老的不支持VXLAN的Linux内核。

**Flannel host-gw 模式必须要求集群宿主机之间是二层连通的.就是node1和node2在一个局域网.通过arp协议可以访问到.**

总结：

- 三层通，二层不通，使用` VXLAN`或`UDP`， 推荐使用 `VXLAN`
- 二层通的情况下，考虑性能问题，推荐使用` host-gw` 模式，当然，二层都通了，vxlan 和 udp 也是可以使用的
- 其中 UDP 和 VXLAN 都是隧道模式，host-gw 则是纯三层网络方案。



##### Host-gw模式

<img src="/images/img/image-20220508120521811.png" alt="image-20220508120521811" style="zoom:200%;" />

你设置 Flannel 使用 host-gw 模式之后，flanneld 会在宿主机上创建这样一条规则，以 Node 1 为例：

当需要从容器1请求到容器2时.

```
$ ip route
...
10.244.1.0/24 via 10.168.0.3 dev eth0
```

1. 根据配置,会从eth0出去,mac地址是node2地址,目的ip肯定是10.244.1.3

2. node2收到后,会根据路由,将包转发给容器cni0,cni0转发给10.244.1.3.

这条路由规则的含义是：目的 IP 地址属于 10.244.1.0/24 网段的 IP 包，应该经过本机的 eth0 设备发出去（即：dev eth0）；并且，它下一跳地址（next-hop）是 10.168.0.3（即：via 10.168.0.3）。

所谓下一跳地址就是：如果 IP 包从主机 A 发到主机 B，需要经过路由设备 X 的中转。那么 X 的 IP 地址就应该配置为主机 A 的下一跳地址。

> 而从 host-gw 示意图中我们可以看到，这个下一跳地址对应的，正是我们的目的宿主机 Node 2。

可以看到node2充当了容器间请求的网关!这也是flannel-gw的由来

一旦配置了下一跳地址，那么接下来，当 IP 包从网络层进入链路层封装成帧的时候，eth0 设备就会使用下一跳地址对应的 MAC 地址，作为该数据帧的目的 MAC 地址。显然，这个 MAC 地址，正是 Node 2 的 MAC 地址。

这样，这个数据帧就会从 Node 1 通过宿主机的二层网络顺利到达 Node 2 上。

而 Node 2 的内核网络栈从二层数据帧里拿到 IP 包后，会“看到”这个 IP 包的目的 IP 地址是 10.244.1.3，即 Infra-container-2 的 IP 地址。这时候，根据 Node 2 上的路由表，该目的地址会匹配到第二条路由规则（也就是 10.244.1.0 对应的路由规则），从而进入 cni0 网桥，进而进入到 Infra-container-2 当中。

可以看到，**host-gw 模式的工作原理，其实就是将每个 Flannel 子网（Flannel Subnet，比如：10.244.1.0/24）的“下一跳”，设置成了该子网对应的宿主机的 IP 地址**。

也就是说，这台“主机”（Host）会充当这条容器通信路径里的“网关”（Gateway）。这也正是“host-gw”的含义。

而且 Flannel 子网和主机的信息，都是保存在 Etcd 当中的。flanneld 只需要 WACTH 这些数据的变化，然后实时更新路由表即可。

而在这种模式下，容器通信的过程就免除了额外的封包和解包带来的性能损耗。根据实际的测试，host-gw 的性能损失大约在 10% 左右，而其他所有基于 VXLAN“隧道”机制的网络方案，性能损失都在 20%~30% 左右。

**host-gw 模式能够正常工作的核心，就在于 IP 包在封装成帧发送出去的时候，会使用路由表里的“下一跳”来设置目的 MAC 地址**。这样，它就会经过二层网络到达目的宿主机。

> **所以说，Flannel host-gw 模式必须要求集群宿主机之间是二层连通的**。



这种模式下，容器通信的过程就免除了额外的封包和解包带来的性能损耗。根据实际的测试，host-gw 的性能损失大约在 10% 左右，而其他所有基于 VXLAN“隧道”机制的网络方案，性能损失都在 20%~30% 左右

Flannel host-gw 模式必须要求集群宿主机之间是二层连通的.就是node1和node2在一个局域网.通过arp协议可以访问到.

那么如果node1和node2不在一个局域网咋整,那么就需要通过几个子网来达到通信的目的.就是多个路由器.

那么还有一个保证,这几个路由器的转发表也要有相应的配置,不能去的时候可以去,回不来了就尴尬了



##### udp模式

<img src="/images/img/image-20220508120957904.png" alt="image-20220508120957904" style="zoom:200%;" />

###### flannel0

该方案使用时会在各个 Work 节点上运行一个Flannel 进程，同时创建一个 flannel0 设备 ，而这个 flannel0 它是一个 TUN 设备（Tunnel 设备）。

在 Linux 中，TUN 设备是一种工作在三层（Network Layer）的虚拟网络设备。TUN 设备的功能非常简单，即：**在操作系统内核和用户应用程序之间传递 IP 包**。

当操作系统将一个 IP 包发送给 flannel0 设备之后，flannel0 就会把这个 IP 包，交给创建这个设备的应用程序，也就是 Flannel 进程。

> 这是一个从内核态向用户态的流动方向。

反之，如果 Flannel 进程向 flannel0 设备发送了一个 IP 包，那么这个 IP 包就会出现在宿主机网络栈中，然后根据宿主机的路由表进行下一步处理。

> 这是一个从用户态向内核态的流动方向。

###### Subnet

子网（Subnet) 是 Flannel 项目里一个非常重要的概念。

事实上，在由 Flannel 管理的容器网络里，一台宿主机上的所有容器，都属于该宿主机被分配的一个“子网”。

> 在我们的例子中，Node 1 的子网是 100.96.1.0/24，container-1 的 IP 地址是 100.96.1.2。Node 2 的子网是 100.96.2.0/24，container-2 的 IP 地址是 100.96.2.3。

而这些子网与宿主机的对应关系，正是保存在 Etcd 当中，如下所示：

```shell
$ etcdctl ls /coreos.com/network/subnets /coreos.com/network/subnets/100.96.1.0-24 /coreos.com/network/subnets/100.96.2.0-24 /coreos.com/network/subnets/100.96.3.0-24 
```

所以，flanneld 进程在处理由 flannel0 传入的 IP 包时，就可以根据目的 IP 的地址（比如 100.96.2.3），匹配到对应的子网（比如 100.96.2.0/24），然后从 Etcd 中找到这个子网对应的宿主机的 IP 地址，如下所示：

```shell
$ etcdctl get /coreos.com/network/subnets/100.96.2.0-24 {"PublicIP":"10.168.0.3"}
```

> 即根据容器IP确定子网，根据子网确定目标宿主机IP。

###### 具体步骤

**step 1：容器到宿主机**

container-1 容器里的进程发起的 IP 包，其源地址就是 100.96.1.2，目的地址就是 100.96.2.3。

由于目的地址 100.96.2.3 并不在 Node 1 的 docker0 网桥的网段里，所以这个 IP 包会被交给默认路由规则，通过容器的网关进入 docker0 网桥（如果是同一台宿主机上的容器间通信，走的是直连规则），从而出现在宿主机上。

**step 2：宿主机路由到 flannel0 设备**

这时候，这个 IP 包的下一个目的地，就取决于宿主机上的路由规则了。

> Flannel 已经在宿主机上创建出了一系列的路由规则。

以 Node 1 为例，如下所示：

```shell
`# 在Node 1上 
$ ip route 
default via 10.168.0.1 dev eth0 
100.96.0.0/16 dev flannel0  proto kernel  scope link  src 
100.96.1.0 100.96.1.0/24 dev docker0  proto kernel  scope link  src 
100.96.1.1 10.168.0.0/24 dev eth0  proto kernel  scope link  src 10.168.0.2 
```

由于我们的 IP 包的目的地址是 100.96.2.3，只能匹配到第二条、也就是 100.96.0.0/16 对应的这条路由规则，从而进入到一个叫作 flannel0 的设备中。

**step 3：flanneld 进程转发给 Node2**

flannel0 设备收到 IP 包后转给 flanned 进程。然后，flanneld 根据这个 IP 包的目的地址，是 100.96.2.3，去 etcd 中查询到对应的宿主机IP，就是 Node2，因此会把它发送给了 Node 2 宿主机，不过发送之前会对该 IP包 进行封装。

**step 4：封装UDP包**

flanneld 进程会把这个 IP 包直接封装在一个 UDP 包里，然后发送给 Node 2。不难理解，这个 UDP 包的源地址，就是 flanneld 所在的 Node 1 的地址，而目的地址，则是 container-2 所在的宿主机 Node 2 的地址。

> 由于 flanneld 进程监听的是 8285 端口，所以会发送给 Node2 的 8285 端口。

**step 5：Node2 解析并处理UDP包**

Node2 上的 flanneld 进程收到这个 UDP 包之后就可以从里面解析出container-1 发来的原 IP 包。

解析后将其发送给 flannel0 设备，flannel0 则会将其转发给操作系统内核。

**step 6：内核处理IP包**

Linux 收到这个IP包之后，Linux 内核网络栈就会负责处理这个 IP 包。具体的处理方法，就是通过本机的路由表来寻找这个 IP 包的下一步流向。

> 该路由规则同样由 Flannel 维护。

而 Node 2 上的路由表，跟 Node 1 非常类似，如下所示：

```shell
# 在Node 2上
$ ip route
default via 10.168.0.1 dev eth0
100.96.0.0/16 dev flannel0  proto kernel  scope link  src 100.96.2.0
100.96.2.0/24 dev docker0  proto kernel  scope link  src 100.96.2.1
10.168.0.0/24 dev eth0  proto kernel  scope link  src 10.168.0.3
```

由于这个 IP 包的目的地址是 100.96.2.3，它跟第三条、也就是 100.96.2.0/24 网段对应的路由规则匹配更加精确。所以，Linux 内核就会按照这条路由规则，把这个 IP 包转发给 docker0 网桥。

**step 7：容器网络**

IP 包到 docker0 网桥后的流程就属于容器网络了。

###### 分析

实际上，相比于两台宿主机之间的直接通信，基于 Flannel UDP 模式的容器通信多了一个额外的步骤，即 flanneld 的处理过程。

而这个过程，由于使用到了 flannel0 这个 TUN 设备，仅在发出 IP 包的过程中，就**需要经过三次用户态与内核态之间的数据拷贝**，如下所示：

![flannel-udp-tun](https://github.com/lixd/blog/raw/master/images/kubernetes/flannel/flannel-udp-tun.jpg)

- 1）第一次，用户态的容器进程发出的 IP 包经过 docker0 网桥进入内核态；
- 2）第二次，IP 包根据路由表进入 TUN（flannel0）设备，从而回到用户态的 flanneld 进程；
- 3）第三次，flanneld 进行 UDP 封包之后重新进入内核态，将 UDP 包通过宿主机的 eth0 发出去。

此外，我们还可以看到，Flannel 进行 UDP 封装（Encapsulation）和解封装（Decapsulation）的过程，也都是在用户态完成的。在 Linux 操作系统中，上述这些上下文切换和用户态操作的代价其实是比较高的，这也正是造成 Flannel UDP 模式性能不好的主要原因。

所以说，**我们在进行系统级编程的时候，有一个非常重要的优化原则，就是要减少用户态到内核态的切换次数，并且把核心的处理逻辑都放在内核态进行**。这也是为什么，Flannel 后来支持的VXLAN 模式，逐渐成为了主流的容器网络方案的原因。

 

##### VXLAN模式:

<img src="/images/img/image-20220508122244887.png" alt="image-20220508122244887" style="zoom:200%;" />

由于udp模式需要从用户态到内核态切换3次.很慢.是基于三层三层网络(封装的ip包)

1. VTEP是一个既有ip,同时也有mac地址.vxlan的目的就是不通过内核态-用户态切换.直接在VTEP间进行传输.

2. 容器1->容器2,根据route表(启动时添加的规则,大体就是请求10.1.16.xx的时候,要走本机的flannel.1设备),将包发送给flannel.1设备.

3. flannel.1设备接收到后,

```
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
10.1.16.0       10.1.16.0       255.255.255.0   UG    0      0        0 flannel.1
```

4. flannel.1设备得想法把包直接发给另一个flannel.1设备.

目标ip知道吗?知道就是这个gateway

目标mac地址呢?node2启动的时候,flannelID进程就会把mac地址写入arp缓存了.

源ip,源mac都有.

但是还有个问题.这个目标mac地址是有,但是node2可能都不接受这个包,因为node2只知道eth0的.那咋增呢?

还是要包装一层.封装成一个普通的udp报文,发送给宿主机.宿主机的网络栈拆包的时候,识别到VXLAN Header就可以把包转发给VTEP,VTEP再转发给docker0



### 4.2 flannel 网络特点

Flannel 网络特点有：

1. 使集群中的不同 Node 主机创建的Docker容器都具有**全集群唯一**的虚拟 IP 地址。
2. 建立一**个覆盖网络（overlay network）**，通过这个覆盖网络，将数据包原封不动的传递到目标容器。覆盖网络是建立在另一个网络之上并由其基础设施支持的虚拟网络。覆盖网络通过将一个分组封装在另一个分组内来将网络服务与底层基础设施分离。在将封装的数据包转发到端点后，将其解封装。
3. 创建一个**新的虚拟网卡 flannel0** 接收 docker0 网桥的数据，通过维护路由表，对接收到的数据进行封包和转发（vxlan）。
4. **etcd** 保证了所有 node 上 flanneld 所看到的配置是一致的。同时每个node上的 flanneld 监听 etcd 上的数据变化，实时感知集群中 node 的变化。

### 4.3 flannel 网络原理及通信流程

Flannel是 CoreOS 团队针对 Kubernetes 设计的一个**覆盖网络（Overlay Network）**工具：

- **目的**在于帮助每一个使用 Kuberentes 的 CoreOS 主机拥有一个完整的子网。
- **功能**是让集群中的不同节点主机创建的 Docker 容器都具有全集群唯一的虚拟IP地址。让所有的容器相当于在同一个直连的网络，底层通过 UDP/VxLAN等进行报文的封装和转发。

其中，**Overlay Network**：**是覆盖网络，在基础网络的基础上叠加的一种虚拟网络技术模式，该网络的主机通过虚拟链路连接起来。**

VXLAN：**将原数据包包装在 UDP 中，并使用基础网络的 IP/MAC 作为外层报文头进行封装，然后在以太网上传输，到达目的地后由隧道端点解封并将数据发送给目标地址**。其中，VXLAN网络结构如下：

<img src="https://www.guoshaohe.com/wp-content/uploads/2021/10/VXLAN%E7%BD%91%E7%BB%9C%E7%BB%93%E6%9E%84.png" alt="VXLAN网络结构" style="zoom:200%;" />

Flannel：**是 Overlay Network 的一种，也是将原数据包封装在另一种网络包里面进行的路由转发和通信，目前已经支持 UDP、VXLAN、AWS VPC和GCE路由数据转发方式**。

具体的通信流程，不同物理节点上的容器**通信流程**：

1. node1 节点的 容器 container1 产生数据，根据 容器的路由表，将数据发送给 cni0
   - **Cni0** : 网桥设备，也就是 docker0 网桥，每创建一个 容器 都会创建一对 veth pair。其中一端是 容器 中的 eth0，另一端是 docker0 网桥中的端口（网卡）。容器 中从网卡 eth0 发出的流量都会发送到 docker0 网桥设备的端口（网卡）上。
2. Cni0 根据节点的路由表，将数据发送到隧道设备 flannel0
   - **Flannel0**: overlay 网络的设备，用来进行 vxlan 或其他方式 报文的处理（封包和解包）。不同 node 之间的 pod 数据流量都从 overlay 设备以隧道的形式发送到对端。
3. Flannel0 查看数据包的目的 ip，从 flanneld 获得对端隧道设备的必要信息，封装数据包。
   - **Flanneld**：flannel 在每个主机中运行 flanneld 作为 agent，它会为所在主机从集群的网络地址空间中，获取一个小的网段subnet，本主机内所有容器的 IP 地址都将从中分配。同时 Flanneld 监听 etcd 数据库，为 Flannel0 设备提供封装数据时必要的 mac，ip 等网络数据信息。
4. Flanneld 将数据包通过本机网卡发送到对端设备。对端节点的网卡接收到数据包，发现数据包为 overlay 数据包，解开外层封装，并发送内层封装到flanneld 设备。
5. Flannel0 设备查看数据包，根据路由表匹配，将数据发送给 Cni0 设备。
6. Cni0 匹配路由表，发送数据给网桥上对应的端口。

注意：

- Flannel 配置第三层 IPv4 Overlay 网络，会创建一个大型内部网络，并且跨越集群中的每个节点
- 在 Overlay 网络中，每个节点都有个子网 subnet ，用于给容器分配ip地址
- 同一主机中的 容器可以使用 docker0 直接通信，而不同主机上的容器需要使用 flanneld 进行封装在传输。

其中，一个具体的例子，Flannel 网络结构如下：

<img src="https://www.guoshaohe.com/wp-content/uploads/2021/10/Flannel%E7%BD%91%E7%BB%9C%E7%BB%93%E6%9E%84-1200x593.png" alt="Flannel网络结构" style="zoom:200%;" />

上图表示：

- 数据从源容器 cache1 container 中（IP地址为10.1.15.2）发出后，经由所在主机的 Docker0 虚拟网卡转发到 flannel0 虚拟网卡，这是个 P2P 的虚拟网卡，flanneld 服务监听在网卡的另外一端。
- Flannel 通过 Etcd 服务维护了一张节点间的路由。
- 源主机的 flanneld 服务将原本的数据内容 UDP 封装后根据自己的路由表投递给目的节点的 flanneld 服务，数据到达以后被解包，然后直 接进入目的节点的 flannel0 虚拟网卡，然后被转发到目的主机的 Docker0 虚拟网卡，最后就像本机容器通信一下的有 Docker0 路由到达目标容器 backend1 container （IP地址为10.1.20.3）。

### 4.4 Flannel 网络的安装配置

可以直接使用 flannel 的 yaml 文件，在 kubernentes 中一键安装。

## 5. Calico 网络原理

### 5.1 Calico 简介

**calico 是完全利用路由规则实现动态组网，通过 BGP 协议通告路由。**

calico 的好处是 endpoints 组成的网络是单纯的三层网络，报文的流向完全通过路由规则控制，没有 overlay 等额外开销。

#### 5.1.1 calico 模式

##### **BGP工作模式：**

<img src="/images/img/image-20220508122532145.png" alt="image-20220508122532145" style="zoom:200%;" />

不会有任何网桥!!!! 需要为每一个容器设置一个Vethpear,设置到宿主机上.

1.目的mac地址是node2的mac地址;目的ip地址是容器ip10.233.2.3.

2.node2收到后,mac地址是自己,那么接受包;  目的ip不是自己,那么去查询route表,应该发给容器4.ending!!

从流程也能看出Flannel host-gw和Calico要求集群宿主机之间是二层连通的

如果node1和node2不在一个子网,那node1根本没办法把包发送给node2.
———————————————

bgp工作模式和flannel的host-gw模式几乎一样；

也就是说，Calico 也会在每台宿主机上，添加一个格式如下所示的路由规则：

```
<目的容器IP地址段> via <网关的IP地址> dev eth0
```

其中，网关的 IP 地址，正是目的容器所在宿主机的 IP 地址。

而正如前所述，这个三层网络方案得以正常工作的核心，是为每个容器的 IP 地址，找到它所对应的、“下一跳”的**网关**。

不过，**不同于 Flannel 通过 Etcd 和宿主机上的 flanneld 来维护路由信息的做法，Calico 项目使用了一个“重型武器”来自动地在整个集群中分发路由信息**。

**所谓 BGP，就是在大规模网络中实现节点路由信息共享的一种协议**。用来维护集群路由信息.如果我们不用BGP,我们就需要手动去维护路由表,以达到网络互通.不太现实,而BGP ,每个边界网关上都会运行着一个小程序，它们会将各自的路由表信息，通过 TCP 传输给其他的边界网关。而其他边界网关上的这个小程序，则会对收到的这些数据进行分析，然后将需要的信息添加到自己的路由表里.

BGP 在每个边界网关（路由器）上运行，彼此之间通信更新路由表信息。而 BGP 的这个能力，正好可以取代 Flannel 维护主机上路由表的功能。

**除了对路由信息的维护方式之外，Calico 项目与 Flannel 的 host-gw 模式的另一个不同之处，就是它不会在宿主机上创建任何网桥设备**。

Calico 的 CNI 插件会为每个容器设置一个 Veth Pair 设备，然后把其中的一端放置在宿主机上（它的名字以 cali 前缀开头）。

此外，由于 Calico 没有使用 CNI 的网桥模式，Calico 的 CNI 插件还需要在宿主机上为每个容器的 Veth Pair 设备配置一条路由规则，用于接收传入的 IP 包。比如，宿主机 Node 2 上的 Container 4 对应的路由规则，如下所示：

即：发往 10.233.2.3 的 IP 包，应该进入 cali5863f3 设备。

```
10.233.2.3 dev cali5863f3 scope link
```

有了这样的 Veth Pair 设备之后，容器发出的 IP 包就会经过 Veth Pair 设备出现在宿主机上。然后，宿主机网络栈就会根据路由规则的下一跳 IP 地址，把它们转发给正确的网关。接下来的流程就跟 Flannel host-gw 模式完全一致了。

其中，**这里最核心的“下一跳”路由规则，就是由 Calico 的 Felix 进程负责维护的**。这些路由规则信息，则是通过 BGP Client 也就是 BIRD 组件，使用 BGP 协议传输而来的。

**Calico 项目实际上将集群里的所有节点，都当作是边界路由器来处理，它们一起组成了一个全连通的网络，互相之间通过 BGP 协议交换路由规则。这些节点，我们称为 BGP Peer**。

需要注意的是，**Calico 维护的网络在默认配置下，是一个被称为“Node-to-Node Mesh”的模式**。这时候，每台宿主机上的 BGP Client 都需要跟其他所有节点的 BGP Client 进行通信以便交换路由信息。但是，随着节点数量 N 的增加，这些连接的数量就会以 N²的规模快速增长，从而给集群本身的网络带来巨大的压力。

所以，Node-to-Node Mesh 模式一般推荐用在少于 100 个节点的集群里。而在更大规模的集群中，你需要用到的是一个叫作 Route Reflector 的模式。

在这种模式下，Calico 会指定一个或者几个专门的节点，来负责跟所有节点建立 BGP 连接从而学习到全局的路由规则。而其他节点，只需要跟这几个专门的节点交换路由信息，就可以获得整个集群的路由规则信息了。

bird是bgd的客户端，与集群中其它节点的bird进行通信，以便于交换各自的路由信息；

随着节点数量N的增加，这些路由规则将会以指数级的规模快速增长，给集群本身网络带来巨大压力，官方建议小于100个节点；

限制：**和flannel的host-gw限制一样，要求物理机在二层是连能的，不能跨网段；因此要求集群内的机器是同一个网段下的**



##### **IPIP模式：**

<img src="/images/img/image-20220508122424599.png" alt="image-20220508122424599" style="zoom:200%;" />

场景：**用在跨网段通信的情况下**，**bgp模式在跨网段的场景将不能工作**；

tunl0：创建的虚拟网卡设备，此时的作用就和flannel的VxLAN工作模式类似（此处的tunl0不是flannel的UDP模式中的tun0）

举个例子，假如我们有两台处于不同子网的宿主机 Node 1 和 Node 2，对应的 IP 地址分别是 `192.168.1.2` 和 `192.168.2.2`。需要注意的是，**这两台机器通过路由器实现了三层转发**，所以这两个 IP 地址之间是可以相互通信的。

而我们现在的需求，还是 Container 1 要访问 Container 4。

Felix 进程在 Node 1 上添加的路由规则， Node 1 上添加如下所示的一条路由规则：

```
10.233.2.0/16 via 192.168.2.2 eth0 
```

上面这条规则里的下一跳地址是 192.168.2.2，可是它对应的 Node 2 跟 Node 1 却根本不在一个子网里，没办法通过二层网络把 IP 包发送到下一跳地址。

**在这种情况下，你就需要为 Calico 打开 IPIP 模式**。

在 Calico 的 IPIP 模式下，Felix 进程在 Node 1 上添加的路由规则，会稍微不同，如下所示：

```
10.233.2.0/24 via 192.168.2.2 tunl0
```

这一次，要负责将 IP 包发出去的设备，变成了 tunl0，Calico 使用的这个 tunl0 设备，是一个 IP 隧道（IP tunnel）设备。

IP 包进入 IP 隧道设备之后，就会被 Linux 内核的 IPIP 驱动接管。IPIP 驱动会将这个 IP 包直接封装在一个宿主机网络的 IP 包中，如下所示：

其中，经过封装后的新的 IP 包的目的地址（图中的 Outer IP Header 部分），正是原 IP 包的下一跳地址，即 Node 2 的 IP 地址：192.168.2.2。

而原 IP 包本身，则会被直接封装成新 IP 包的 Payload。

**这样，原先从容器到 Node 2 的 IP 包，就被伪装成了一个从 Node 1 到 Node 2 的 IP 包**。

由于宿主机之间已经使用路由器配置了三层转发，也就是设置了宿主机之间的“下一跳”。所以这个 IP 包在离开 Node 1 之后，就可以经过路由器，最终“跳”到 Node 2 上。

这时，Node 2 的网络内核栈会使用 IPIP 驱动进行解包，从而拿到原始的 IP 包。然后，原始 IP 包就会经过路由规则和 Veth Pair 设备到达目的容器内部。



- 也就是说： 不同网段，二层不通，但是三层能通，这个时候不能使用bgp，因为二层不通，对端网段路由表都加不上的，就要使用三层的 IPIP 模式

### 5.2 Calico 原理

#### 5.2.1 Calico 原理：

- **Calico 在每一个计算节点利用 Linux Kernel 实现了一个高效的 vRouter 来负责数据转发**
- 每个 vRouter 通过 BGP 协议负责把自己上运行的 workload 的路由信息向整个 Calico 网络内传播——小规模部署可以直接互联，大规模下可通过指定的 BGP route reflector 来完成。
- 保证最终所有的 workload 之间的数据流量都是通过 IP 路由的方式完成互联的。
- Calico 节点组网可以直接利用数据中心的网络结构（无论是 L2 或者 L3），不需要额外的 NAT，隧道或者 Overlay Network。
- Calico 基于 iptables 还提供了丰富而灵活的网络 Policy，保证通过各个节点上的 ACLs 来提供 Workload 的多租户隔离、安全组以及其他可达性限制等功能。

#### 5.2.2 Calico 组成：

- 1）Calico 的 CNI 插件。这就是 Calico 与 Kubernetes 对接的部分。
- 2）Felix。它是一个 DaemonSet，负责在宿主机上插入路由规则（即：写入 Linux 内核的 FIB 转发信息库），以及维护 Calico 所需的网络设备等工作。
- 3）BIRD。它就是 BGP 的客户端，专门负责在集群里分发路由规则信息。

### 5.3 Calico 组网

组网示意图：

```
+--------------------+              +--------------------+ 
|   +------------+   |              |   +------------+   | 
|   |            |   |              |   |            |   | 
|   |    ConA    |   |              |   |    ConB    |   | 
|   |            |   |              |   |            |   | 
|   +-----+------+   |              |   +-----+------+   | 
|         |veth      |              |         |veth      | 
|       wl-A         |              |       wl-B         | 
|         |          |              |         |          |
+-------node-A-------+              +-------node-B-------+ 
        |    |                               |    |
        |    |   type1.  in the same lan     |    |
        |    +-------------------------------+    |
        |                                         |
        |      type2. in different network        |
        |             +-------------+             |
        |             |             |             |
        +-------------+   Routers   |-------------+
                      |             |
                      +-------------+
```



注意：

- calico 组网的核心原理就是 IP 路由，每个容器或者虚拟机会分配一个 workload-endpoint(wl)。
  - endpoint : 接入到 calico 网络中的网卡称为 endpoint
  - workload-Endpoint : 虚拟机、容器使用的 endpoint
- 从容器 ConA 中发送给 ConB 的报文被 nodeA 的 wl-A 接收，根据 nodeA 上的路由规则，经过各种 iptables 规则后，转发到nodeB。
- 如果 nodeA 和 nodeB 在同一个二层网段，下一条地址直接就是 node-B，经过二层交换机即可到达。
- 如果 nodeA 和 nodeB 在不同的网段，报文被路由到下一跳，经过三层交换或路由器，一步步跳转到 node-B。
- nodeA 怎样得知下一跳的地址？答案是 node 之间通过 BGP 协议交换路由信息。

### 5.4 Calico 安装配置

#### 5.4.1 搭建 etcd 集群

1. 安装 etcd

- - 两台 etcd 集群，分别装有 docker
  - `yum install -y etcd`

1. 配置 /etc/etcd/etcd.conf 配置文件， 根据节点不同修改 ip 地址

```
ETCD_DATA_DIR="/var/lib/etcd/cluster.etcd"
ETCD_LISTEN_PEER_URLS="http://192.168.26.91:2380,http://localhost:2380"
ETCD_LISTEN_CLIENT_URLS="http://192.168.26.91:2379,http://localhost:2379"
ETCD_NAME="etcd-91"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.26.91:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://localhost:2379,http://192.168.26.91:2379"
ETCD_INITIAL_CLUSTER="etcd-91=http://192.168.26.91:2380,etcd-92=http://192.168.26.92:238
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
```

1. 启动 etcd 集群

- - `systemctl start etcd`

1. 配置 docker 的存储为 etcd , 在两个节点上 修改 docker 启动选项，添加：（配置为自己的etcd节点）

```
// 节点一：
--cluster-store=etcd://192.168.26.91:2379
// 节点二：
--cluster-store=etcd://192.168.26.92:2379
```

#### 5.4.2 安装配置 calico

1. 创建 calico 容器，用来进行两个物理节点上容器的互通, 在所有节点上执行

- 下载安装 calicoctl， 地址：`curl -O -L https://github.com/projectcalico/calicoctl/releases/download/v3.16.3/calicoctl`
- 添加可执行权限：`chmod +x calicoctl`
- 建立 calico node 的容器，容器镜像为：quay.io/calico/node

```
calicoctl node run --node-image=quay.io/calico/node -c /etc/calico/calicoctl.cfg
```

- 可以看到对方的主机的信息

```
calicoctl node status
```

1. 创建 docker 网络，在任意主机上创建网络（可以同步给远端）

- `--driver calico` 指定使用calico 的 libnetwork CNM driver。
- `--ipam-driver calico-ipam` 指定使用 calico 的IPAM driver 管理IP。
- calico 为 global 网络，etcd 会将 calnet1 同步到所有主机。

```
docker network create --driver calico --ipam-driver calico-ipam calnet1
```

1. 每在主机上创建一个容器，则会在物理机上创建一张虚拟网卡出来
   - 在 vms91 上创建一个容器 : `docker run --name c91 --net calnet1 -itd busybox`
   - 在 vms92 上创建一个容器 : `docker run --name c92 --net calnet1 -itd busybox`
   - 两个容器就可以互相通信了

### 5.5 calico 网卡分析

1. 创建好 calico node 后，在没有创建容器时，只有 本地网卡和docker0网桥，没有其他网卡。
2. 当创建一个busybox 容器后，在容器中 和 在 物理机中都多了一张网卡
   - 在容器中的网卡名称为：6: cali0@if7
   - 在物理机中的网卡名称为：7: calif6391d136be@if6
   - 可以看到：
     - 物理机网卡的序号 7 与 容器网卡的 @if7 对应
     - 物理机网卡的 @if6 与 物理机网卡的 序号6 对应
     - 建立了 veth pair 关系，详细：https://www.cnblogs.com/bakari/p/10613710.html
3. 在容器中的路由表 route -n，可以看到，所有的数据包的默认网关是 cali0 网卡
4. 在物理集中查看路由表 route -n， 可以看到，
   - 凡是去往本机容器中的地址的数据包，默认网卡是 calif6391d136be
   - 凡是去往其他物理机中容器地址的数据包，默认网卡都是 本机的网卡出去
   - 也就是说，在calico的节点上，通过egb协议，相互学习路由
5. 每台主机都知道不同的容器在哪台主机上，所以会动态的设置路由。



## 6 总结


Kubernetes通过一个叫做CNI的接口，维护了一个单独的网桥来代替docker0。这个网桥的名字就叫作：CNI网桥，它在宿主机上的设备名称默认是：cni0。

容器“跨主通信”的三种主流实现方法：UDP、host-gw、VXLAN。 之前介绍了UDP和VXLAN，它们都属于隧道模式，需要封装和解封装。接下来介绍一种纯三层网络方案，host-gw模式和Calico项目

Host-gw模式通过在宿主机上添加一个路由规则：

    <目的容器IP地址段> via <网关的IP地址> dev eth0

IP包在封装成帧发出去的时候，会使用路由表里的“下一跳”来设置目的MAC地址。这样，它就会通过二层网络到达目的宿主机。
这个三层网络方案得以正常工作的核心，是为每个容器的IP地址，找到它所对应的，“下一跳”的网关。所以说，Flannel host-gw模式必须要求集群宿主机之间是二层连通的，如果宿主机分布在了不同的VLAN里（三层连通），由于需要经过的中间的路由器不一定有相关的路由配置（出于安全考虑，公有云环境下，宿主机之间的网关，肯定不会允许用户进行干预和设置），部分节点就无法找到容器IP的“下一条”网关了，host-gw就无法工作了。

Calico项目提供的网络解决方案，与Flannel的host-gw模式几乎一样，也会在宿主机上添加一个路由规则：

    <目的容器IP地址段> via <网关的IP地址> dev eth0

其中，网关的IP地址，正是目的容器所在宿主机的IP地址，而正如前面所述，这个三层网络方案得以正常工作的核心，是为每个容器的IP地址，找到它所对应的，“下一跳”的网关。区别是如何维护路由信息：

- Host-gw : Flannel通过Etcd和宿主机上的flanneld来维护路由信息
- Calico: 通过BGP（边界网关协议）来实现路由自治，所谓BGP，就是在大规模网络中实现节点路由信息共享的一种协议。

**三层网络主要通过维护路由规则，将数据包直接转发到对应的宿主机上**。

- Flannel host-gw 主要通过 etcd 中的子网信息来维护路由规则；

- Calico 则通过 BGP 协议收集路由信息，由 Felix进程来维护；

- 隧道模式主要通过在 IP 包外再封装一层 MAC 包头来实现。

- > 额外的封包和解包工作会导致集群网络性能下降，在实际测试中，Calico IPIP 模式与 Flannel VXLAN 模式的性能大致相当。

技术分类：

- 隧道技术（需要封装包和解包，因为需要伪装成宿主机的IP包，需要三层链通）：Flannel UDP / VXLAN / Calico IPIP

- 三层网络（不需要封包和解封包，需要二层链通）：Flannel host-gw / Calico 普通模式

### 三层和隧道的异同：

- 相同之处是都实现了跨主机容器的三层互通，而且都是通过对目的 MAC 地址的操作来实现的；不同之处是三层通过配置下一条主机的路由规则来实现互通，隧道则是通过通过在 IP 包外再封装一层 MAC 包头来实现。
- 三层的优点：少了封包和解包的过程，性能肯定是更高的。
- 三层的缺点：需要自己想办法维护路由规则。
- 隧道的优点：简单，原因是大部分工作都是由 Linux 内核的模块实现了，应用层面工作量较少。
- 隧道的缺点：主要的问题就是性能低。

### **使用场景**

在大规模集群里，三层网络方案在宿主机上的路由规则可能会非常多，这会导致错误排查变得困难。此外，在系统故障的时候，路由规则出现重叠冲突的概率也会变大。

- 1）如果是在`公有云`上，由于宿主机网络本身比较“直白”，一般推荐更加简单的 Flannel host-gw 模式。
- 2）但不难看到，在`私有部署环境`里，Calico 项目才能够覆盖更多的场景，并为你提供更加可靠的组网方案和架构思路。

