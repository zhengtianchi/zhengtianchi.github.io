# istio - 多集群管控入门


# 多云管理1



- **Multi-cluster service mesh**  多集群服务网格，配置工作负载来路由跨集群的流量。所有这些都符合应用Istio配置，如' VirtualServices '， ' DestinationRules '， ' Sidecar '等。
- **Mesh Federation** 网格联邦，表示公开并支持两个独立服务网格的工作负载的通信



对于网格联邦，您可以查看Istio文档[multiple mesh](https://istio.io/latest/docs/ops/deployment/deployment-models/#multiple-meshes) 或一个名为[Gloo mesh](https://docs.solo.io/gloo-mesh/latest/)的开源项目，以帮助自动化配置以支持它。



## 多集群服务网格



![1.multi cluster service mesh reqs](https://drek4537l1klr.cloudfront.net/posta2/v-14/Figures/1.multi-cluster-service-mesh-reqs.png)

多集群服务网格需要跨集群发现、连接和共同信任

- **跨集群工作负载发现** -控制平面必须发现对等集群中的工作负载，以便配置服务代理 (例如，集群的`API server `必须可以访问对面集群中的Istio控制平面)。

- **跨集群工作负载连接性** -工作负载之间必须具有连接性。除非您可以初始化到工作负载端点的连接，否则对工作负载端点的感知是没有用的。

- **集群之间的共同信任** -跨集群的工作负载必须相互认证以使Istio的安全特性成为可能的`PeerAuthentication`和` AuthorizationPolicy `。



## 多集群部署模型



集群分类：

* **主集群** （Primary Cluster）  :  安装Istio控制平面的集群
* **远端集群**（Remote Cluster）: 安装控制平面的远端集群

部署模型：

- Primary-Remote(共享控制平面)
- Primary-Primary(复用控制平面)
- External控制平面



### Primary-Remote 部署模型

![2.primary remote](https://drek4537l1klr.cloudfront.net/posta2/v-14/Figures/2.primary-remote.png)

**Primary-Remote**部署模型有一个管理网格的单一控制平面，因此，它经常被称为单一控制平面或共享控制平面部署模型。
这种模型使用更少的资源，然而，主集群的中断会影响整个网格，因此它的可用性很低。



### Primary-Primary部署模型

![3.primary primary](https://drek4537l1klr.cloudfront.net/posta2/v-14/Figures/3.primary-primary.png)

**Primary-Primary** 部署模型有多个控制平面，这确保了更高的可用性，因为中断的范围仅限于发生中断的集群，但也需要更多的资源。我们将此模型称为复制控制平面部署模型。



### External控制平面

![4.external control plane](https://drek4537l1klr.cloudfront.net/posta2/v-14/Figures/4.external-control-plane.png)

**External控制平面** 为所有集群远程到控制平面的部署模型。这种部署模型使云提供商能够将Istio作为托管服务提供。



## 多集群下的服务发现

Istio的控制平面需要与`Kubernetes API`服务器通信，以收集相关信息来配置服务代理，比如 `services` 和 `endpoints`。

但是对 kubernetes 的 `api server` 的访问，存在安全的问题，因为您可以查找资源细节、查询敏感信息，并更新或删除资源，从而使集群处于糟糕且不可逆的状态。

可以参考 RBAC 来解决这个问题



为`istiod` 提供` remote cluster `的` service account token` , `istiod`使用 这个`token` 对 remote cluster 进行身份验证，并发现运行在其中的工作负载。

![6.remote cluster token](https://drek4537l1klr.cloudfront.net/posta2/v-14/Figures/6.remote-cluster-token.png)

## 跨集群工作负载连接

两种情况：

- 集群处于同一网络平面，工作负载使用 IP 连接，天生满足条件。
- 集群处于不同网络，必须使用位于 网格边缘的特殊 istio 入口网关来代理跨集群网络

在“多网络”网格中桥接集群的入口网关 称为 **“东西向网关”**， 东西向网关将 作为一个 **反向代理**，将请求发送到个字集群中的工作负载。

![7.east west gateways](https://drek4537l1klr.cloudfront.net/posta2/v-14/Figures/7.east-west-gateways.png)



## 集群之间的相互信任

拥有公共信任可以确保不同集群的工作负载可以相互进行身份验证。

实现集群之间相互信任的方法：

- 第一种方法是使用我们称为**Plug-In CA Certificates**的东西，它是由一个公共根证书颁发机构颁发的用户定义的证书。
- 第二种方法是集成一个外部证书颁发机构，两个集群都使用它来签署证书。

### Plug-in CA Certificates

默认情况下，Istio CA生成自签名根证书和密钥，并使用它们对工作负载证书进行签名。要保护根CA密匙，应该使用在安全机器上脱机运行的根CA，并使用根CA向在每个集群中运行的Istio CA颁发中间证书。

Istio CA可以使用管理员指定的证书和密钥对工作负载证书进行签名，并将管理员指定的根证书作为信任根分发到工作负载。

因为，不是让 istio 生成中间证书颁发机构，而是通过在Istio安装名称空间上提供秘密证书来指定要使用的证书（就是创建一个secret 持久化到 etcd 中）。您可以对两个集群都这样做，并使用由公共根CA签名的中间CA。

![8.common trust](https://drek4537l1klr.cloudfront.net/posta2/v-14/Figures/8.common-trust.png)

上图为 使用由同一根签名的中间CA证书

但是上述方法存在安全风险：

如果暴露中间证书颁发机构，攻击者可以使用这些签名来签署证书，然后这些证书将被信任，直到检测到暴露和中间CA的证书被撤销。

### External certificate authority integration

- Cert-manager (待补充，见原文)
- Custom development （待补充，见原文）

# 多集群、多网络、多控制平面服务网格概述

- **West-cluster:** Kubernetes cluster，该cluster在us-west区域拥有私有网络。我们将运行`webapp`服务。
- **East-cluster:** Kubernetes cluster，它在us-east区域拥有自己的私有网络。在那里我们将运行`catalog`服务。

![9.multi cluster mesh](https://drek4537l1klr.cloudfront.net/posta2/v-14/Figures/9.multi-cluster-mesh.png)

本文将建立一个一个多集群、多网络、多控制平面的服务网格，使用东西网关连接网络，并使用`primary-primary`部署模型



## 第一步 创建集群

创建两个 `kubernetes` 集群，每个集群都位于不同的网络上。

## 第二步 配置 plug-in CA certificates 建立相互信任

Istio 默认会生成一个在 Istio-system namespace 下的 `istio-ca-secret`的 secret。

通过插入我们自己的证书颁发机构，可以覆盖这个 `secret`。

![10.cacerts](https://drek4537l1klr.cloudfront.net/posta2/v-14/Figures/10.cacerts.png)

​                                       cacert密钥由根CA的公钥和中间CA的公钥和私钥组成。根CA的私钥在集群外“安全”存储

创建一个`cacerts` secret，包含如下输入文件`ca-cert.pem`, `ca-key.pem`, `root-cert.pem` 和`cert-chain.pem`。需要注意的是创建出来的secret的名称必须是`cacerts`，这样才能被istio正确挂载。

- ca-cert.pem - 中间CA的证书
- ca-key.pem - 中间CA的私钥
- root-cert.pem - 颁发中间CA的根CA的证书，用于验证由它颁发的任何中间CA颁发的证书
- cert-chain.pem - 中间CA证书和根CA证书的连接，形成信任链

```shell
# 通过创建命名空间' istio-system '，然后将证书作为名为' cacerts '的秘密文件应用，在每个集群中配置中间CA。

# setting up certificates for the west-cluster （在 west-cluster 下操作）
$  kubectl create namespace istio-system
$  kubectl create secret generic cacerts -n istio-system \
       --from-file=ch12/certs/west-cluster/ca-cert.pem \
       --from-file=ch12/certs/west-cluster/ca-key.pem \
       --from-file=ch12/certs/root-cert.pem \
       --from-file=ch12/certs/west-cluster/cert-chain.pem

# setting up certificates for the east-cluster (在 east-cluster 下操作)
$  kubectl create namespace istio-system
$  kubectl create secret generic cacerts -n istio-system \
       --from-file=ch12/certs/east-cluster/ca-cert.pem \
       --from-file=ch12/certs/east-cluster/ca-key.pem \
       --from-file=ch12/certs/root-cert.pem \
       --from-file=ch12/certs/east-cluster/cert-chain.pem
```



## 第三步 安装各个集群的控制面

继续安装Istio控制面，该控制面将挑选插件CA证书(即用户定义的中间证书)来签署工作负载证书。



**标记集群的网络拓扑**

```shell
# 标记 west-cluster的 网络拓扑
$  kubectl label namespace istio-system topology.istio.io/network=west-network
# 标记 east-cluster的 网络拓扑
$  kubectl label namespace istio-system topology.istio.io/network=east-network
```

通过这些标签，Istio形成了对网络拓扑的理解，并使用它来决定如何配置工作负载。



**部署控制面**

部署 west 集群

``` shell
# 方式一
$ istioctl install --set profile=demo \
  --set values.global.meshID=usmesh \
  --set values.global.multiCluster.clusterName=west-cluster \
  --set values.global.network=west-network
  
# 方式二 使用 istioOperator 部署
apiVersion: install.istio.io/v1alpha1
metadata:
 name: istio-controlplane
 namespace: istio-system
kind: IstioOperator
spec:
  profile: demo
  components:
    egressGateways:
    - name: istio-egressgateway
      enabled: false
 values:
   global:
     meshID: usmesh    # 属性' meshID '使我们能够识别这个安装属于哪个网格。 Istio提供了在集群中安装多个网格的选项，允许团队分别管理他们网格的操作。
     multiCluster:
       clusterName: west-cluster
     network: west-network
     
# 上述 yaml 使用如下命令安装：
$ istioctl install -f xxx.yaml
```

使用同样的方式部署 east 集群， 和 west 集群的不同在于 clusterName 和 network 不同

```yaml
apiVersion: install.istio.io/v1alpha1
metadata:
  name: istio-controlplane
  namespace: istio-system
kind: IstioOperator
spec:
  profile: demo
  components:
    egressGateways:
    - name: istio-egressgateway
      enabled: false
  values:
    global:
      meshID: usmesh
      multiCluster:
        clusterName: east-cluster
      network: east-network
```



在 west 和 east 集群 安装完控制面后，我们有了两个独立的网格，两个网格都运行着 istiod ，但是只发现本地的服务。当前网格的状态如下图所示：

![11.disjointed meshes](https://drek4537l1klr.cloudfront.net/posta2/v-14/Figures/11.disjointed-meshes.png)

当前网格缺乏跨集群工作负载发现和连接，会在 第四步 和 第五步 讨论。

此时可以在集群中创建一些工作负载，分别在 west 和 east 集群中，运行如下：

```shell
# 在 west 集群安装应用和 网关
$  kubectl create ns istioinaction
$  kubectl label namespace istioinaction istio-injection=enabled
$  kubectl -n istioinaction apply -f ./webapp-deployment-svc.yaml
$  kubectl -n istioinaction apply -f ./webapp-gw-vs.yaml
$  kubectl -n istioinaction apply -f ./catalog-svc.yaml

# 在 east 集群安装 catalog
$  kubectl create ns istioinaction
$  kubectl label namespace istioinaction istio-injection=enabled
$  kubectl -n istioinaction apply -f ./catalog.yaml
```

![image-20211120214227189](img/image-20211120214227189.png)

测试用用例如上所示。

## 第四步 启动跨集群的服务发现

为了从远程集群中查询信息，istio 需要使用 `service account `进行身份验证。

istio 在安装时会创建一个名为 `istio-reader-service-account` 的 `service account`，具有可被另一个控制平面使用的最小权限集。

```shell
$ kubectl get sa -n istio-system

NAME                                   SECRETS   AGE
default                                1         4d20h
istio-egressgateway-service-account    1         4d20h
istio-ingressgateway-service-account   1         4d20h
istio-reader-service-account           1         4d20h
istiod-service-account                 1         4d20h
```

但是，我们需要使服务帐户令牌对对端的集群可用，以及证书来启动到远程集群的安全连接。



**创建对端集群访问的secret**

```shell
# 在 east-cluster上执行如下：  (创建时，务必指明集群名称 --name)
$  istioctl x create-remote-secret --name="east-cluster"
# 得到如下结果

# This file is autogenerated, do not edit.
apiVersion: v1
kind: Secret
metadata:
  annotations:
    networking.istio.io/cluster: east-cluster
  creationTimestamp: null
  labels:
    istio/multiCluster: "true"
  name: istio-remote-secret-east-cluster
  namespace: istio-system
stringData:
  east-cluster: |
    apiVersion: v1
    clusters:   # 包含集群地址和验证API服务器提供的连接的证书颁发机构数据的集群列表。
    - cluster:
        certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUN5RENDQWJDZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkF...
        server: https://10.20.144.83:6443
      name: east-cluster
    contexts:  # 上下文列表，每组一个用户和一个集群，这简化了切换集群(与我们的用例无关)。
    - context:
        cluster: east-cluster
        user: east-cluster
      name: east-cluster
    current-context: east-cluster
    kind: Config
    preferences: {}
    users:     # 一个列表，定义了包含要对API服务器进行身份验证的令牌的用户
    - name: east-cluster
      user:
        token: eyJhbGciOiJSUzI1NiIsImtpZCI6....
---

# 该命令使用默认的 istio-reader-service-account service account 为远程集群访问创建secret。
# 上述的 secret 就是“kubectl”需要启动到Kubernetes API服务器的安全连接并对其进行身份验证的全部内容。

# 需要将 上述内容 手动 执行 kubectl apply -f xxx.yaml 应到到 west-cluster
```

:warning:  注意

- 上述的内容，先在一个集群中(east-cluster)生成 secret 的 yaml ，然后将生成的 yaml 放到 remote 集群中(west-cluster) 中 执行 apply

一旦在 remote 集群中创建了secret，`istiod`就会获取它，并在新添加的 remote 集群中查询工作负载。通过查看日志可以更详细看到细节：

```shell
# 在 remote 集群中(west-cluster) 查看 istiod 的日志
$ kubectl logs deploy/istiod -n istio-system | grep 'Adding cluster'

2021-10-08T08:47:32.408052Z     info    Adding cluster_id=east-cluster from
  secret=istio-system/istio-remote-secret-east-cluster
  
# 通过日志可以验证集群是否初始化完成，并且west-cluster的控制面可以发现 east-cluster 的 workload
```

为了配置成 `primary-primary `部署模式，同样要对 对端集群做同样的操作，使得 east-cluster 的控制面可以发现 west-cluster 的 workload。

```shell
# 在west-cluster 中执行如下：
$ kubectl x create-remote-secret --name="west-cluster"
# 将生成的secret 放到 east-cluster中apply,配置 east-cluster查看west-cluster
$ kubectl apply -f 刚刚生成的secret.yaml

secret/istio-remote-secret-west-cluster created
```

:heart: 现在，双方的控制面都可以查询到对端集群的 workload，接下来要去设置跨集群连接。



## 第五步 建立跨集群连接

不同内部网络(在我们的实例中是集群网络)之间的流量称为“东西流量”

将内部服务通过网关定向到内部网络的流量称为“南北向流量”

- 南北向流量管理：使用Ingress将Kubernetes中的应用暴露成对外提供的服务，针对这个对外暴露的服务可以实现灰度发布、A/B测试、全量发布、流量管理等。我们把这种流量管理称为南北向流量管理，也就是入口请求到集群服务的流量管理。
- 东西向流量管理：Istio还有侧重于集群服务网格之间的流量管理，我们把这类管理称为东西向流量管理。

![12.east west traffic](https://drek4537l1klr.cloudfront.net/posta2/v-14/Figures/12.east-west-traffic.png)

对于不同云服务商的集群，或者网络并非对等连接的地方，istio 提供了东西向网关作为解决方案。

**Istio 东西向网关**

东西向网关的目标初了成为跨集群东西向流量的入口点之外，还在于使这对运营服务的团队透明。为了实现这一目标，它必须:

- 开启跨集群的细粒度流量管理
- 通过路由加密流量，实现负载之间的双向认证

istio 有两个特性：

- SNI cluster
- SNI Auto Passthrough

**东西向网关配置 SNI cluster**

```yaml
# cluster-east-eastwest-gateway.yaml

apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: istio-eastwestgateway  # 一定不能和前面的 istioOperator 资源相同，如果相同，将会覆盖之前的istioOpeator
  namespace: istio-system
spec:
  profile: empty
  components:
    ingressGateways:
    - name: istio-eastwestgateway
      label:
        istio: eastwestgateway
        app: istio-eastwestgateway
      enabled: true
      k8s:
        env:
          - name: ISTIO_META_ROUTER_MODE    # SNI集群的配置是一个可选特性
            value: "sni-dnat"               # 网关路由器模式设置为' snii -dnat '来启用, 自动配置SNI集群
          - name: ISTIO_META_REQUESTED_NETWORK_VIEW # The network to which traffic is routed
            value: east-network
        service:
          ports:
            # redacted for brevity
  values:
    global:
      meshID: usmesh
      multiCluster:
        clusterName: east-cluster    # 注意：这是 east-cluster
      network: east-network
```

```shell
# 在 east 集群中执行如下，apply 上述的yaml
$ istioctl install -y -f ./cluster-east-eastwest-gateway.yaml

✔ Ingress gateways installed
✔ Installation complete
```

安装了东西向网关后，可以查询网关的集群代理配置并将输出过滤为仅包含“catalog”文本的行，来验证SNI集群是否配置了。

```shell
# 在 east 集群中执行如下：
$  istioctl pc clusters deploy/istio-eastwestgateway.istio-system  \
    | grep catalog | awk '{printf "CLUSTER: %s\n", $1}'

CLUSTER: catalog.istioinaction.svc.cluster.local
CLUSTER: outbound_.80_._.catalog.istioinaction.svc.cluster.local
```



**如何使用SNI Auto Passthrough路由跨集群流量**

SNI Auto Passthrough，顾名思义，不需要手动创建“VirtualServices”来路由允许的流量。

这是使用SNI集群完成的，这些集群使用“snii -dnat”路由器模式在东西网关中自动配置。

![image-20211120180746067](img/image-20211120180746067.png)

SNI Auto Passthrough 模式 使用 Istio Gateway 配置

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: cross-network-gateway
  namespace: istio-system
spec:
  selector:
    istio: eastwestgateway
  servers:
    - port:
        number: 15443
        name: tls
        protocol: TLS
      tls:
        mode: AUTO_PASSTHROUGH
      hosts:
        - "*.local"
```

```shell
# 把east-cluster的workload 暴露给 west-cluster
# 在 east-cluster 上操作
$ kubectl apply -n istio-system -f 上述.yaml

gateway.networking.istio.io/cross-network-gateway created
```

同样，需要在west-cluster上进行相似的操作

```shell
# west-cluster 环境中

# 首先部署 istioOperator 创建东西向网关， kubectl apply -f 下述yaml

apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: istio-eastwestgateway
  namespace: istio-system
spec:
  profile: empty
  components:
    ingressGateways:
    - name: istio-eastwestgateway
      label:
        istio: eastwestgateway
        app: istio-eastwestgateway
        topology.istio.io/network: west-network
      enabled: true
      k8s:
        env:
        - name: ISTIO_META_ROUTER_MODE
          value: "sni-dnat"
        # The network to which traffic is routed
        - name: ISTIO_META_REQUESTED_NETWORK_VIEW
          value: west-network
        service:
          ports:
          - name: status-port
            port: 15021
            targetPort: 15021
          - name: mtls
            port: 15443
            targetPort: 15443
          - name: tcp-istiod
            port: 15012
            targetPort: 15012
          - name: tcp-webhook
            port: 15017
            targetPort: 15017  
  values:
    global:
      meshID: usmesh
      multiCluster:
        clusterName: west-cluster   # 注意：这是 west-cluster
      network: west-network
  
# 然后将集群中的服务暴露给 east-cluster, kubectl apply -f 下述yaml (其实这个和在east-cluster中配置的yaml是一样)

apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: cross-network-gateway
  namespace: istio-system
spec:
  selector:
    istio: eastwestgateway
  servers:
    - port:
        number: 15443
        name: tls
        protocol: TLS
      tls:
        mode: AUTO_PASSTHROUGH
      hosts:
        - "*.local"
```

使用 SNI 自动直通配置网关后，网关上的传入流量将使用 SNI 集群路由到预期的工作负载。 Istio 的控制平面侦听这些资源的创建并发现*现在*存在路由跨集群流量的路径。 因此，它使用新发现的端点更新所有工作负载。



## 第六步 验证跨集群 workload discovery

之前的步骤中，我们已经准备好了测试用例，现在可以派上用场了。

![image-20211120214336727](img/image-20211120214336727.png)

测试用例描述：

- 在 west-cluster 中，部署了 webapp ，它有 deployment 和 service ，同时部署了 catalog，但是catalog只部署了service
- 在easy-cluster中，部署了catalog，它有 deployment 和 service 

经过之前的部署，现在 east-cluster 的 workloads 已经暴露于 west-cluster ，我们希望 west-cluster 中的 webapp 的 envoy cluster 配置中，存在一个 catalog 的 endpoints， 因为我们并没有在 west-cluster 中部署 catalog 的deployment，而是在 east-cluster 中部署了 catalog 的 deployment，如果集群之间没有打通，那么在 west-cluster 中 是不会存在 catalog 的 endpoints。

并且，这个 endpoints 应该指向 东西向网关的地址，该地址将请求代理到其网络中的 catalog workload

```shell
# 首先获取 east-cluster 的 east-west gateway 的 地址 (在 east-cluster 下操作)
$ kubectl -n istio-system get svc istio-eastwestgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}'

40.114.190.251

# 将其与 west-cluster 中的 workload 在跨集群通讯时使用的地址进行比较 (在 west-cluster 下操作)
$ istioctl pc endpoints deploy/webapp.istioinaction | grep catalog

40.114.190.251:15443  HEALTHY  OK  outbound|80||catalog.istioinaction.svc.cluster.local
^-----------^^---^
         |       |
         |

# 如果 catalog 的 endpoint 与 东西向网关的地址匹配，那么就可以发现 workload ,实现跨集群通讯。

# 让我们在west-cluster中手动发起一个到catalog 的请求         
$  EXT_IP=$(kubectl -n istio-system get svc istio-ingressgateway \
     -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

$  curl http://$EXT_IP/api/catalog -H "Host: webapp.istioinaction.io"

[
  {
    "id": 0,
    "color": "teal",
    "department": "Clothing",
    "name": "Small Metal Shoes",
    "price": "232.00"
  }
]

# 可以看到，对入口网关的请求被路由到了 west-cluster 中的 webapp 中， 然后被分配到了 east-cluster 中的 caltalog workload，并最终服务于请求。
```



至此，我们已经验证集群、多网络、多控制平面服务网格的建立，并发现了跨集群的工作负载，它们可以使用东西网关作为通道启动相互验证的连接。

让我们回顾一下建立多集群服务网格需要做些什么:

1. **跨集群工作负载发现**通过使用包含服务帐户令牌和证书的* kubecconfig *为每个控制平面提供对对等集群的访问，这个过程使用了' istioctl '，我们只将它应用到相反的集群。
2. **跨集群工作负载连接性**通过配置东西网关在不同集群(驻留在不同网络中)的工作负载之间路由流量，并为每个集群标记网络信息，以便Istio知道网络负载驻留。
3. 通过使用颁发相反集群的中间证书的公共信任根来配置集群之间的**信任**。



## 第七步 集群间的负载均衡

这部分放在 **负载均衡** 部分，和本地负载均衡一起介绍。


