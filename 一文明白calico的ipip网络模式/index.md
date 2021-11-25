# calico - 网络流通以及抓包实战




### 前言

本文主要分析k8s中网络组件calico的 `IPIP网络模式`。旨在理解IPIP网络模式下产生的`calixxxx`，`tunl0`等设备以及跨节点网络通信方式。可能看着有点枯燥，但是请花几分钟时间坚持看完，如果看到后面忘了前面，请反复看两遍，这几分钟时间一定你会花的很值。

### 一、calico介绍

Calico是`Kubernetes`生态系统中另一种流行的网络选择。虽然`Flannel`被公认为是最简单的选择，但`Calico`以其性能、灵活性而闻名。`Calico`的功能更为全面，不仅提供主机和pod之间的网络连接，还涉及网络安全和管理。`Calico CNI`插件在CNI框架内封装了Calico的功能。

Calico是一个基于`BGP`的**纯三层**的网络方案，与OpenStack、Kubernetes、AWS、GCE等云平台都能够良好地集成。Calico在每个计算节点(every node)都利用Linux Kernel实现了一个高效的`虚拟路由器vRouter`来负责数据转发。每个`vRouter`都通过`BGP协议`把在本节点上运行的容器的**路由信息**向整个Calico网络**广播**，并自动设置到达其他节点的路由转发规则。Calico保证所有容器之间的数据流量都是通过IP路由的方式完成互联互通的。Calico节点组网时可以直接利用数据中心的网络结构(L2或者L3)，不需要额外的NAT、隧道或者Overlay Network，没有额外的封包解包，能够节约CPU运算，提高网络效率。

此外，Calico基于iptables还提供了丰富的网络策略，实现了Kubernetes的`Network Policy`策略，提供容器间网络可达性限制的功能。

**calico官网：**https://www.projectcalico.org/

### 二、calico架构及核心组件

架构图如下：

[![img](https://s3.51cto.com/oss/202105/06/1f7efa8b5fa9aae52a55d33a35350472.png-wh_600x-s_3459497392.png)](https://s3.51cto.com/oss/202105/06/1f7efa8b5fa9aae52a55d33a35350472.png-wh_600x-s_3459497392.png)

#### calico核心组件：

- **Felix (菲利克斯)**：运行在每个需要运行`workload`的节点上的`agent进程`。主要负责配置路由及 ACLs(访问控制列表) 等信息来确保 endpoint 的连通状态，保证跨主机容器的网络互通;
- **etcd**：强一致性、高可用的键值存储，持久存储calico数据的存储管理系统。主要负责网络元数据一致性，确保Calico网络状态的准确性;
- **BGP Client(BIRD)**：读取Felix设置的内核路由状态，在数据中心分发状态。
- **BGP Route Reflector(BIRD)**：BGP路由反射器，在较大规模部署时使用。如果仅使用`BGP Client`形成mesh全网互联就会导致规模限制，因为所有`BGP client`节点之间两两互联，需要建立N^2个连接，拓扑也会变得复杂。因此使用`reflector`来负责client之间的连接，防止节点两两相连。

### 三、calico工作原理

​		Calico把每个操作系统的协议栈认为是一个路由器，然后把所有的容器认为是连在这个路由器上的网络终端，在路由器之间跑标准的路由协议——BGP的协议，然后让它们自己去学习这个网络拓扑该如何转发。所以Calico方案其实是一个纯三层的方案，也就是说让每台机器的协议栈的三层去确保两个容器，跨主机容器之间的三层连通性。

### 四、calico的两种网络方式

### 1)IPIP

​       把 IP 层封装到 IP 层的一个` tunnel`。它的作用其实基本上就相当于一个基于IP层的网桥!一般来说，普通的网桥是基于mac层的，根本不需 IP，而这个 `ipip` 则是通过两端的路由做一个` tunnel`，把两个本来不通的网络通过点对点连接起来。ipip 的源代码在内核 `net/ipv4/ipip.c` 中可以找到。

### 2)BGP

边界网关协议`(Border Gateway Protocol, BGP)`是互联网上一个核心的去中心化自治路由协议。它通过维护IP路由表或‘前缀’表来实现自治系统(AS)之间的可达性，属于矢量路由协议。`BGP`不使用传统的内部网关协议(IGP)的指标，而使用基于路径、网络策略或规则集来决定路由。因此，它更适合被称为矢量性协议，而不是路由协议。

### 五、IPIP网络模式分析

由于个人环境中使用的是IPIP模式，因此接下来这里分析一下这种模式。

```shell
# kubectl get po -o wide -n paas | grep hello 

demo-hello-perf-d84bffcb8-7fxqj   1/1     Running   0          9d      10.20.105.215   node2.perf  <none>           <none> 
demo-hello-sit-6d5c9f44bc-ncpql   1/1     Running   0          9d      10.20.42.31     node1.sit   <none>           <none> 
```



进行ping测试

这里在`demo-hello-perf`这个pod中`ping demo-hello-sit`这个pod。

```shell
root@demo-hello-perf-d84bffcb8-7fxqj:/# ping 10.20.42.31 

PING 10.20.42.31 (10.20.42.31) 56(84) bytes of data. 
64 bytes from 10.20.42.31: icmp_seq=1 ttl=62 time=5.60 ms 
64 bytes from 10.20.42.31: icmp_seq=2 ttl=62 time=1.66 ms 
64 bytes from 10.20.42.31: icmp_seq=3 ttl=62 time=1.79 ms 
^C 
--- 10.20.42.31 ping statistics --- 
3 packets transmitted, 3 received, 0% packet loss, time 6ms 
rtt min/avg/max/mdev = 1.662/3.015/5.595/1.825 ms 
```

进入`pod demo-hello-perf`中查看这个pod中的路由信息

```shell
root@demo-hello-perf-d84bffcb8-7fxqj:/# route -n 

Kernel IP routing table 
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface 
0.0.0.0         169.254.1.1     0.0.0.0         UG    0      0        0 eth0 
169.254.1.1     0.0.0.0         255.255.255.255 UH    0      0        0 eth0 
```

根据路由信息，ping 10.20.42.31，会匹配到第一条。

**第一条路由的意思是：**去往任何网段的数据包都发往网关`169.254.1.1`，然后从`eth0网卡`发送出去。

`demo-hello-perf`所在的`node node2.perf `宿主机上路由信息如下：

```shell
# node2
# route -n 

Kernel IP routing table 
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface 
0.0.0.0         172.16.36.1     0.0.0.0         UG    100    0        0 eth0 
10.20.42.0      172.16.35.4     255.255.255.192 UG    0      0        0 tunl0 
10.20.105.196   0.0.0.0         255.255.255.255 UH    0      0        0 cali4bb1efe70a2 
169.254.169.254 172.16.36.2     255.255.255.255 UGH   100    0        0 eth0 
172.16.36.0     0.0.0.0         255.255.255.0   U     100    0        0 eth0 
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0 
```

可以看到一条`Destination`为 `10.20.42.0`的路由。

该路由的意思是：去往`10.20.42.0/26`的网段的数据包都发往网关`172.16.35.4`(其实 这个 ip 就是 对端 node 的 ip)。因为`demo-hello-perf`的pod在`172.16.36.5`上，`demo-hello-sit`的pod在`172.16.35.4`上。所以数据包就通过设备`tunl0`发往到node节点上。

`demo-hello-sit`所在的`node node1.sit` 宿主机上路由信息如下：

```shell
# node1
# route -n 
Kernel IP routing table 
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface 
0.0.0.0         172.16.35.1     0.0.0.0         UG    100    0        0 eth0 
10.20.15.64     172.16.36.4     255.255.255.192 UG    0      0        0 tunl0 
10.20.42.31     0.0.0.0         255.255.255.255 UH    0      0        0 cali04736ec14ce 
10.20.105.192   172.16.36.5     255.255.255.192 UG    0      0        0 tunl0 
```

当node节点网卡收到数据包之后，发现发往的目的ip为`10.20.42.31`，于是匹配到`Destination`为`10.20.42.31`的路由。

该路由的意思是：`10.20.42.31`是本机直连设备，去往设备的数据包发往`cali04736ec14ce`

**为什么这么奇怪会有一个名为`cali04736ec14ce`的设备呢?这是个啥玩意儿呢?**

其实这个设备就是`veth pair`的一端。在创建`demo-hello-sit` 时`calico`会给`demo-hello-sit`创建一个`veth pair`设备。一端是`demo-hello-sit` 的网卡，另一端就是我们看到的`cali04736ec14ce`

接着验证一下。我们进入`demo-hello-sit` 的pod，查看到 4 号设备后面的编号是：`122964`

```shell
root@demo-hello-sit--6d5c9f44bc-ncpql:/# ip a 
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00 
    inet 127.0.0.1/8 scope host lo 
       valid_lft forever preferred_lft forever 
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1000 
    link/ipip 0.0.0.0 brd 0.0.0.0 
4: eth0@if122964: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1380 qdisc noqueue state UP group default  
    link/ether 9a:7d:b2:26:9b:17 brd ff:ff:ff:ff:ff:ff link-netnsid 0 
    inet 10.20.42.31/32 brd 10.20.42.31 scope global eth0    # 这个 ip 就是 pod ip 
       valid_lft forever preferred_lft forever 
```

然后我们登录到`demo-hello-sit`这个pod所在的宿主机查看

```shell
# node1 宿主机

# ip a | grep -A 5 "cali04736ec14ce" 
122964: cali04736ec14ce@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1380 qdisc noqueue state UP group default  
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 16 
    inet6 fe80::ecee:eeff:feee:eeee/64 scope link  
       valid_lft forever preferred_lft forever 
120918: calidd1cafcd275@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1380 qdisc noqueue state UP group default  
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 2 
```

发现`pod demo-hello-sit`中 的另一端设备编号和这里在node上看到的`cali04736ec14ce`编号`122964`是一样的

所以，node上的路由，发送`cali04736ec14ce`网卡设备的数据其实就是发送到了`demo-hello-sit`的这个pod中去了。到这里ping包就到了目的地。

注意看 `demo-hello-sit`这个pod所在的宿主机的路由，有一条 `Destination`为`10.20.105.192`的路由

```shell
# node1
# route -n 
Kernel IP routing table 
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface 
... 
0.0.0.0         172.16.35.1     0.0.0.0         UG    100    0        0 eth0 
10.20.105.192   172.16.36.5     255.255.255.192 UG    0      0        0 tunl0 
... 
```

再查看一下`demo-hello-sit`的pod中路由信息，和`demo-hello-perf`的pod中是一样的。

所以综合上述例子来看，IPIP的网络模式就是将IP网络封装了一层。特点就是所有pod的数据流量都从隧道tunl0发送，并且`tunl0`这里增加了一层传输层的封包操作。

![image-20211124192743157](/images/img/image-20211124192743157.png)

### 六、抓包分析



这里我使用了`httpbin` 服务和 `sleep` 服务来演示抓包过程

![image-20211125104231815](/images/img/image-20211125104231815.png)

在  `node1 sleep`这个 pod 中去 `curl` `node2 httpbin` 服务，接着在 `httpbin` 所在`node2` 上`tcpdump`抓包

```shell
# 先在 nodes2 上 启动抓包
$ tcpdump  -i ens160 -nn -w httpbin_ens160.cap 
tcpdump: listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes 
```

```shell
# 然后在 node1 上发起 curl 请求
$ kubectl exec sleep-76c9b748f4-p964r  -c sleep -n zhangji -- curl httpbin.httpbin:8000/headers
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   565  100   565    0     0   137k      0 --:--:-- --:--:-- --:--:--  137k
{
  "headers": {
    "Accept": "*/*", 
    "Content-Length": "0", 
    "Host": "httpbin.httpbin:8000", 
    "User-Agent": "curl/7.76.1-DEV", 
    "X-B3-Parentspanid": "365864bb263d7b45", 
    "X-B3-Sampled": "0", 
    "X-B3-Spanid": "65ee1633796944f9", 
    "X-B3-Traceid": "a36a2e412ab0d22b365864bb263d7b45", 
    "X-Envoy-Attempt-Count": "1", 
    "X-Forwarded-Client-Cert": "By=spiffe://cluster.local/ns/httpbin/sa/httpbin;Hash=cdd5655c5735fa86c8ad8636fdd91027fa288a2c31b6cb18d7feae0d563b009f;Subject=\"\";URI=spiffe://cluster.local/ns/zhangji/sa/default"
  }
}
```

结束抓包后下载`httpbin_ens160.cap`到本地`wireshark`进行抓包分析

能看到该数据包一共5层，其中`IP(Internet Protocol)`所在的网络层有两个，分别是pod之间的网络和主机之间的网络封装。

![image-20211125103214188](/images/img/image-20211125103214188.png)

红色框选的是两个pod所在的宿主机，黄色框选的是两个pod的ip，`src`表示发起ping操作的pod所在的宿主机ip以及发起`curl`操作的pod的ip，`dst`表示被`curl`的pod所在的宿主机ip及被`curl`的pod的ip

封包顺序如下所示：

![image-20211125105310728](/images/img/image-20211125105310728.png)

可以看到每个数据报文共有两个IP网络层,内层是Pod容器之间的IP网络报文,外层是宿主机节点的网络报文(2个node节点)。之所以要这样做是因为`tunl0`是一个隧道端点设备，在数据到达时要加上一层封装，便于发送到对端隧道设备中。

两层封包的具体内容如下：

![image-20211125105723878](/images/img/image-20211125105723878.png)

Pod间的通信经由IPIP的三层隧道转发,相比较VxLAN的二层隧道来说，IPIP隧道的开销较小，但其安全性也更差一些。


