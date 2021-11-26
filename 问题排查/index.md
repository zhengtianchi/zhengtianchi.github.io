# istio - 问题排查


# istio 问题排查



⚠️ 用例部署的 yaml 会在文末给出



## 前言

问题排查分类如下：

- `istio CR` 错误的配置如何排查
- 如何使用 `istioctl` 检测和防止错误的资源配置
- 如何使用 `istioctl` 检查 `envoy` 配置
- 如何理解 `envoy` 的日志以帮助了解`envoy`的行为
- 利用收集的遥测技术深入了解应用程序

istio可以有效地帮助你了解网络通讯，诸如：超时、重试和断路等服务弹性功能，以便应用程序能够自动响应网络问题，但是，如果 `envoy`本身的意外行为会发生什么呢？

![image-20211125133145806](/images/img/image-20211125133145806.png)

上图中展示了`istio`中路由请求参与的组件：

- `istiod` 确保数据面同步到所期望的状态
- `Ingress Gateway` 允许流量进入集群的入口网关
- `istio-proxy`提供访问控制并处理从 `downstream`到 `Application`的流量
- `Application` 服务于请求的应用程序本身。应用程序可能会请求另一个服务，继续连接到另一个 `UpStream`服务

当检查 `istio`中问题的时候，可能会与上述组件中的任意组件有关。调试每个组件可能会花费大量的时间。因此，本文将讲述如何使用一些useful工具通过检查`proxy`及其相关`config`来排除一些问题与错误。

## 最常见的问题：数据面 misconfigured

istio 中 有许多的`CR`，诸如：`VirtualService`、`DestinationRule` 等等。这些资源会被转换成`Envoy`配置并应用于数据面中。如果在应用新资源后，数据面的行为和我们预期的不一致，最常见的问题就是我们可能错误地配置了数据面。

为了展示如何在 `misconfigured`数据面后解决故障，我们将设置如下：使用 `Gateway`资源允许流量通过istio入口网关和`VirtualService` 将 `20%` 请求路由到 `subset catalog vesion-v1` 其余 `80%` 的请求路由到`subset catalog version-v2`，如下图所示。

```shell
$  kubectl apply -f ./catalog.yaml                 # yaml 放在文末给出
$  kubectl apply -f ./catalog-deployment-v2.yaml   # yaml 放在文末给出
$  kubectl apply -f ./catalog-gateway.yaml         # 创建 Gateway 来配置入口网关以允许 HTTP 流量
$  kubectl apply -f ./catalog-virtualservice-subsets-v1-v2.yaml # 创建 VirtualService 将流量路由到 catalog
```

```yaml
# catalog-gateway.yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: catalog-gateway
  namespace: istioinaction
spec:
  selector:
    istio: ingressgateway
  servers:
  - hosts:
    - "catalog.istioinaction.io"
    port:
      number: 80
      name: http
      protocol: HTTP
```

```yaml
# catalog-virtualservice-subsets-v1-v2.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: catalog-v1-v2
  namespace: istioinaction
spec:
  hosts:
  - "catalog.istioinaction.io"
  gateways:
  - "catalog-gateway"
  http:
  - route:
    - destination:
        host: catalog.istioinaction.svc.cluster.local
        subset: version-v1
        port:
          number: 80
      weight: 20
    - destination:
        host: catalog.istioinaction.svc.cluster.local
        subset: version-v2
        port:
          number: 80
      weight: 80
```

![image-20211125135137695](/images/img/image-20211125135137695.png)

但是，你会发现` subset` 并没有定义，那么 `subset`是在哪里定义的呢？ 答案：在 `DestinationRule`中

因为上述并没有通过 `DestinationRule`定义 `subset`，入口网关就没有针对 `version-v1`和 `version-v2`子集的集群定义，因此按照上述的部署模式下，所有的请求都会失败。

```shell
# 创建好资源后，打开一个新的终端并执行一个持续运行的测试来生成“catalog”workload的流量。

$  ./bin/query-catalog.sh get-items-cont
# curl -s -H "Host: catalog.istioinaction.io"
#-w "\nStatus Code %{http_code}" localhost/items

Status Code 000
# infinite loop CTRL-C to terminate
```

在输出中，我们看到由于缺少子集，响应代码是“503 Service Unavailable”。

**如何修复:question:** 配置上 `DestinationRule` 即可

```yaml
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: catalog
  namespace: istioinaction
spec:
  host: catalog.istioinaction.svc.cluster.local
  subsets:
  - name: version-v1
    labels:
      version: v1
  - name: version-v2
    labels:
      version: v2
```

## 数据面和控制面配置同步问题

在与控制面发生同步之前，环境（`services、endpoints、health`）以及配置的更改不会立即反馈到 数据面。

控制面会将特定服务的每个 `endpoint IP` 发送到数据面（就是`service`每个 `POD IP`），如果这些 `endpoints`中任何一个变得不健康，`kubernetes`需要一段时间来识别并标记 `pod unhealth`。在某一时刻，控制面也会识别出这一点（控制面通过 api-server 实现 `service` 和 `endpoint` 的配置发现和更新），并将 `endpoint`从数据面中删除。由此让数据面和控制面保持一致。

![image-20211125141841747](/images/img/image-20211125141841747.png)

​       对于工作负载和事件数量增加的较大集群，数据平面同步所需的时间会按比例增加。我之后会单独起一篇文章阐述如何“优化控制平面的性能”。

### istioctl proxy-status

```shell
# 让我们检查数据平面是否与最新配置同步
$ istioctl proxy-status

NAME                                      CDS      LDS      RDS
catalog.<...>.istioinaction               SYNCED   SYNCED   SYNCED
catalog.<...>.istioinaction               SYNCED   SYNCED   SYNCED
catalog.<...>.istioinaction               SYNCED   SYNCED   SYNCED
istio-egressgateway.<...>.istio-system    SYNCED   SYNCED   NOT SENT
istio-ingressgateway.<...>.istio-system   SYNCED   SYNCED   SYNCED
```

输出列出了所有 `workload` 的每个 `xDS API` 的同步状态 。有如下状态：

- SYNCED : Envoy已经确认了控制平面发送的最后一个配置。
- NOT SENT : 控制面没有发送任何东西给 Envoy。这通常是因为控制平面没有什么可发送的。例如`RDS`的` istio-egressgateway `。(出口网关不会在服务网格内路由请求，因此不需要路由配置)
- STALE : `istiod` 控制平面已经发送了一个更新，但它没有被确认。这表示以下情况之一: 控制平面过载; envoy 与控制平面之间缺乏或失去连接; 或者可能是Istio本身的一个bug。

## 可视化问题排查 kiali 

使用 Kiali 可以快速发现配置错误的服务。

```shell
$ istioctl dashboard kiali
http://localhost:20001/kiali
```

![image-20211125144020411](/images/img/image-20211125144020411.png)

这里我以外文原文的图做演示：

![image-20211125144222922](/images/img/image-20211125144222922.png)

仪表板在`istioinaction`名称空间中显示一个警告。单击它会将您重定向到`Istio Config`视图，该视图列出了在所选名称空间中应用的所有Istio配置。

![image-20211125144325175](/images/img/image-20211125144325175.png)

提示了 `Virtualservice` 有问题，点击后，所有Istio配置都被列出，并伴随着错误配置的通知，就像`catalog-v1-v2 `` VirtualService `的情况一样。单击警告图标会将您重定向到`virtualService`的`YAML`视图，在该视图中，嵌入式编辑器突出显示了配置错误的部分。

![image-20211125144636436](/images/img/image-20211125144636436.png)

将鼠标悬停在警告图标上，会显示警告消息“KIA1107子集未找到”。有关此警告的更多信息，请查看Kiali文档的[Kiali validation](https://kiali.io/documentation/latest/validations/) (https://kiali.io/documentation/latest/validations/)页面。此页面提供了已识别错误的描述、严重性和解决方案。例如，下面是我们面临的“KIA1107” `warning`部分:

- 修正指向不存在的子集的路由。它可能是修复子集名称中的拼写错误，或者在`DestinationRule `中定义缺失的子集。

- 该描述帮助我们识别并修复问题，因为它正确地指出了哪些配置错误。在这种情况下，子集是缺失的，我们应该创建一个`DestinationRule `来定义它们。



## 使用 istioctl 发现错误配置

最有用的两个指令：

- `istioctl analyze`
- `istioctl describe`

### istioctl analyze (可以分析 yaml 和 namespace)

`istioctl analyze `命令是一个强大的分析Istio配置的诊断工具。它可以在已经遇到问题的集群上运行，甚至可以在将这些问题应用到集群之前验证配置，以便首先防止错误配置资源.

“analyze”命令运行一组分析器，其中每个分析器都专门用于检测特定的一组问题。

让我们 `analyze`下 `istioinaction` namespace :

```shell
$  istioctl analyze -n istioinaction

Error [IST0101] (VirtualService catalog-v1-v2.istioinaction) Referenced
  host+subset in destinationrule not found:
  "catalog.istioinaction.svc.cluster.local+version-v1"
Error [IST0101] (VirtualService catalog-v1-v2.istioinaction) Referenced
  host+subset in destinationrule not found:
  "catalog.istioinaction.svc.cluster.local+version-v2"
Error: Analyzers found issues when analyzing namespace: istioinaction.

See https://istio.io/v1.10/docs/reference/config/analysis for more
  information about causes and resolutions.
```

输出显示没有找到子集，除了错误消息`Referenced host+subset in destinationrule not found`外，它还给我们提供了错误代码“IST0101”，我们可以在Istio的[文档](https://istio.io/latest/docs/reference/config/analysis/) (https://istio.io/latest/docs/reference/config/analysis/)中找到关于这个问题的更多细节。

### istioctl describe (workload-specific 的配置分析)

`describe`命令用于描述特定于工作负载的配置。它分析直接或间接影响一个工作负载的Istio配置并打印摘要。这个摘要回答了关于工作负载的问题，例如:它是服务网格的一部分吗?应用于它的虚拟服务和目标规则是什么?它是否需要相互验证的请求等。

选择任何`catalog`工作负载的名称，并执行下面的描述命令:

```shell
$ istioctl x describe pod catalog-68666d4988-vqhmb

Pod: catalog-68666d4988-q6w42
   Pod Ports: 3000 (catalog), 15090 (istio-proxy)
---------------
Service: catalog
   Port: http 80/HTTP targets pod port 3000

Exposed on Ingress Gateway http://13.91.21.16
VirtualService: catalog-v1-v2
  WARNING: No destinations match pod subsets (checked 1 HTTP routes)

    Warning: Route to subset version-v1 but NO DESTINATION RULE defining
    subsets!

    Warning: Route to subset version-v2 but NO DESTINATION RULE defining
    subsets!
```

`description`命令的输出显示警告消息`Route to subset version-v1 but NO DESTINATION RULE defining subset`。这意味着路由是为不存在的子集配置的。为了完整起见，如果正确配置了工作负载，将显示`istioctl describe `输出如下：

```shell
Pod: catalog-68666d4988-q6w42
   Pod Ports: 3000 (catalog), 15090 (istio-proxy)
---------------
Service: catalog
   Port: http 80/HTTP targets pod port 3000
DestinationRule: catalog for "catalog.istioinaction.svc.cluster.local"
   Matching subsets: version-v1
      (Non-matching subsets version-v2)
   No Traffic Policy

Exposed on Ingress Gateway http://
VirtualService: catalog-v1-v2
   Weight 20%
```

`analyze`和`describe`子命令都有助于识别配置中的常见错误，通常足以提出修复建议。对于这些命令无法解决的问题，或者对于如何解决这些问题没有提供足够的指导，您需要深入挖掘!这就是我们下一节要做的。



## 通过拉取 envoy 配置查找错误配置

当前面提到的一些自动分析器达不到要求时，可以使用手动调查 envoy 配置的方式查找问题。

### envoy dashboard

envoy 的 dashboard 公开envoy 配置和其他功能，以修改 envoy 的某些方面比如提高日志记录级别。

```shell
$ istioctl dashboard envoy deploy/catalog -n istioinaction

http://localhost:15000
```

我们将使用`config_dump`在代理中打印当前加载的Envoy配置

![7.envoy admin dashboard](https://drek4537l1klr.cloudfront.net/posta2/v-14/Figures/7.envoy-admin-dashboard.png)

:warning: config_dump 出来的结果 数据量非常大

以为，`istioctl `提供了将输出过滤成更小块的工具，这有助于提高可读性和理解性。那就是 `istioctl proxy-config`

### istioctl proxy-config

`istioctl proxy-config `命令使我们能够基于`Envoy xDS api`检索和过滤工作负载的代理配置，其中每个子命令都被适当命名为:

- **cluster** -获取集群配置
- **endpoint** -获取端点配置
- **listener** -检索侦听器配置
- **route** -获取路由配置信息
- **secret** -获取秘密配置

![image-20211125160336175](/images/img/image-20211125160336175.png)

1. `Envoy Listeners`定义一个网络配置，例如`IP Address`和`Port`，它允许`downstream`流量进入`envoy`。为允许的连接创建`HTTP Filter chain`。链中最重要的过滤器是路由器过滤器，它执行高级路由任务。
2. `Envoy Routes`是一组将`virtual hots`匹配到集群的规则。路由按列出的顺序处理。第一个匹配的用于将流量路由到工作负载集群。路由可以静态配置，但Istio使用路由发现服务动态配置。
3. `Envoy Clusters`是一组集群，其中每个集群都有一组指向类似工作负载的`endpoints`。`subsets`用于在集群中进一步划分工作负载，从而支持细粒度的流量管理。
4. `Envoy Endpoints`是一组`endpoints`，它们代表服务请求的工作负载的IP地址。

#### 查询 envoy listener 配置

​       首先确保允许到达本地主机端口80上的入口网关的流量进入集群。如前所述，接收流量是envoy监听器(`Envoy Listeners`)的职责，它们在Istio中使用`Gateway`资源配置。让我们查询一下网关的监听器配置，并验证流量是否被80端口所允许:

```shell
# 查询ingress pod的侦听器配置

$  istioctl proxy-config listeners deploy/istio-ingressgateway -n istio-system

ADDRESS PORT  MATCH DESTINATION
0.0.0.0 8080  ALL   Route: http.80         # 在 8080 端口上配置了 listener，流量根据该侦听器名为“http.80”的路由进行路由
0.0.0.0 15021 ALL   Inline Route: /healthz/ready*
0.0.0.0 15090 ALL   Inline Route: /stats/prometheus*
```

```shell
# Ports of the istio-ingressgateway service printed in yaml format

$  kubectl -n istio-system get svc istio-ingressgateway -o yaml \
| grep "ports:" -A 10
  ports:
  - name: status-port
    nodePort: 30618
    port: 15021
    protocol: TCP
    targetPort: 15021
  - name: http2
    nodePort: 32589
    port: 80       # Traffic in port 80 targets pods on port 8080
    protocol: TCP       |
    targetPort: 8080 <--+
```

因此，我们验证了流量到达端口 8080 并且存在一个侦听器以允许它进入入口网关。 此外，我们看到此侦听器的路由是由路由“http.80”完成的，这是我们的下一个检查点。

#### 查询 envoy Route 配置

envoy Route 配置定义了一组规则，这些规则决定将通信路由到哪个集群。Istio使用`VirtualService `资源配置envoy Routes，同时，`clusters`要么自动发现，要么使用`DestinationRule `资源定义。

```shell
# 要找出“http.80”路由的流量路由到哪些集群，让我们查询其配置：

$  istioctl pc routes deploy/istio-ingressgateway -n istio-system --name http.80

NOTE: This output only contains routes loaded via RDS.
NAME        DOMAINS                          MATCH                 VIRTUAL SERVICE
http.80     *                                /productpage          bookinfo.default
http.80     *                                /static*              bookinfo.default
http.80     *                                /login                bookinfo.default
http.80     *                                /logout               bookinfo.default
http.80     *                                /api/v1/products*     bookinfo.default
http.80     bookinfo.10.20.144.36.nip.io     /*                    productpage.bookinfo
http.80     catalog.istioinaction.io         /*                    catalog.istioinaction

# 其 URL 与路径前缀 `/*` 匹配的 host `catalog.istioinaction.io` 的流量被路由到 catalog `VirtualService`，该 catalog 服务 位于 `istioinaction` 命名空间中的 `catalog` service 中。
```

当路由配置以 JSON 格式打印时，会显示有关 `catalog.istioinaction` `VirtualService` 后面的集群的详细信息：

```shell
# 打印路由配置

$  istioctl pc routes deploy/istio-ingressgateway -n istio-system \
      --name http.80 -o json

<ommitted>
"routes": [
  {
    "match": {
      "prefix": "/"    # Route rule that has to match
    },
    "route": {
      "weightedClusters": {
         "clusters": [     # Clusters to which traffic is routed when the rule is matched
            {
            "name": "outbound|80|version-v2|catalog.istioinaction.svc.cluster.local",
            "weight": 80
            },
            {
            "name": "outbound|80|version-v1|catalog.istioinaction.svc.cluster.local",
            "weight": 20
            }
         ]
      },
<ommitted>
}
```

由输出可知，路由匹配时，两个`clusters`接收流量，分别为:

- `outbound|80|version-v2|catalog.istioinaction.svc.cluster.local`
- `outbound|80|version-v1|catalog.istioinaction.svc.cluster.local`

让我们研究每个管道分离部分的含义，并研究如何将工作负载作为成员分配给这些集群。



#### 查询 envoy Clutsers 配置

`Envoy Clusters`配置定义了请求可以路由到的后端服务。 跨实例或端点的集群负载平衡。 这些端点（通常是 IP 地址）代表为最终用户流量提供服务的各个工作负载实例。

使用 `istioctl` 我们可以查询入口网关知道的集群，但是，有很多集群，因为每个后端可路由服务都配置了一个集群。

我们可以使用以下 `istioctl proxy-config clusters` 配合 `flags`: `direction`、`fqdn`、`port` 和 `subset`打印我们感兴趣的集群.

所有`flags`的信息都包含在我们在前面部分检索到的集群名称中：

![9.cluster components](https://drek4537l1klr.cloudfront.net/posta2/v-14/Figures/9.cluster-components.png)

让我们查询其中一个集群，例如配置了入口网关的 `version-v1`子集的集群。 我们可以通过在查询中指定所有集群属性。

```shell
$  istioctl proxy-config clusters \
     deploy/istio-ingressgateway.istio-system \
     --fqdn catalog.istioinaction.svc.cluster.local  \
     --port 80 \
     --subset version-v1

SERVICE FQDN   PORT   SUBSET   DIRECTION   TYPE   DESTINATION RULE
```

会发现，子集`version-v1`没有集群!  如果没有这些子集的集群，请求将失败，因为 virtual service 路由到了不存在的集群。

显然，这是一个配置错误的情况，我们可以通过创建一个定义这些子集的集群的`DestinationRule`来解决这个问题。

```yaml
# catalog-destinationrule-v1-v2.yaml
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: catalog
  namespace: istioinaction
spec:
  host: catalog.istioinaction.svc.cluster.local
  subsets:
  - name: version-v1
    labels:
      version: v1
  - name: version-v2
    labels:
      version: v2
```

但是在将它应用到集群之前，让我们使用 `istioctl analysis` 子命令来验证此配置：

```shell
# 分析目标规则应用于网格时的影响

istioctl analyze ./catalog-destinationrule-v1-v2.yaml \
    -n istioinaction

✔ No validation issues found when analyzing ch10/catalog-destinationrule-v1-v2.yaml.

# 在模拟应用资源的影响时，输出显示集群中没有验证错误。 这意味着应用目标规则会修复集群配置。 让我们这样做吧！

$  kubectl apply -f ./catalog-destinationrule-v1-v2.yaml

destinationrule.networking.istio.io/catalog created
```

再次查询：

```shell
$  istioctl pc clusters deploy/istio-ingressgateway -n istio-system --fqdn catalog.istioinaction.svc.cluster.local --port 80

SERVICE FQDN        PORT  SUBSET      DIRECTION  TYPE  DESTINATION RULE
catalog.<...>.local 80    -           outbound   EDS   catalog.<...>
catalog.<...>.local 80    version-v1  outbound   EDS   catalog.<...>
catalog.<...>.local 80    version-v2  outbound   EDS   catalog.<...>
```

如果想看更详细的信息，可以通过如下指令查看：

```shell
$  istioctl pc clusters deploy/istio-ingressgateway -n istio-system \
--fqdn catalog.istioinaction.svc.cluster.local --port 80 \
--subset version-v1 -o json

# Output is truncated
"name": "outbound|80|version-v1|catalog.istioinaction.svc.cluster.local",
"type": "EDS",
"edsClusterConfig": {
    "edsConfig": {
        "ads": {},
        "resourceApiVersion": "V3"
    },
    "serviceName": "outbound|80|version-v1|catalog.istioinaction.svc.cluster.local"
},


# 输出显示' edsClusterConfig '被配置为使用聚合发现服务来查询端点
# 服务名' outbound|80|version-v1|catalog. istioaction .svc.cluster。local '被用作从ADS查询的端点的过滤器。
```



#### 查询 envoy cluster endpoints

现在我们知道`envoy proxy`被配置为使用服务名查询ADS，我们可以使用这个信息手动查询入口网关中这个集群的`endpoints`，使用` istioctl proxy-config endpoint `命令:

```shell
$ istioctl pc endpoints deploy/istio-ingressgateway -n istio-system \
--cluster "outbound|80|version-v1|catalog.istioinaction.svc.cluster.local"

ENDPOINT         STATUS   OUTLIER CHECK  CLUSTER
10.1.0.60:3000   HEALTHY  OK             outbound|80|version-v1|catalog...
```

输出列出了这个集群后面的唯一工作负载的端点。让我们使用这个IP查询pod，并验证其背后是否有实际的工作负载。

```shell
$ kubectl get pods -n istioinaction \
    --field-selector status.podIP=10.1.0.60

NAME                       READY   STATUS    RESTARTS   AGE
catalog-5b56677c4c-v7hkj   2/2     Running   0          3h47m
```

## 使用日志查找问题

对于基于微服务的应用程序，服务代理生成的日志和指标有助于解决许多问题，如发现导致性能瓶颈的服务、识别经常失败的端点、检测性能下降等。我们将使用Envoy访问日志和度量来排除应用程序弹性问题。

```shell
# 首先，更新我们的服务，以解决我们可以解决的问题。

# 设置超时的间歇慢工作负载，目录工作负载可以配置为间歇性地返回慢响应，使用以下脚本:

$ ./query-catalog.sh delayed-responses

blowups=[object Object]

------------------------------------------------------------------------------------------------------

# 现在让我们继续并将catalog-v1-v2 VirtualService配置为当请求需要超过半秒才能得到服务时超时:

$  kubectl patch vs catalog-v1-v2 -n istioinaction --type json \
      -p '[{"op": "add", "path": "/spec/http/0/timeout", "value": "0.5s"}]'

# 配置的可视化结果如下所示：
```

![image-20211126103702917](/images/img/image-20211126103702917.png)

让我们在一个单独的终端中生成到“目录”工作负载的连续通信。这将产生我们在接下来的章节中所需要的日志和遥测技术。

```shell
$  ./bin/query-catalog.sh get-items-cont
```

在连续触发的请求中，我们看到一些请求被路由到较慢的工作负载，结果那些请求以超时结束。输出如下:

```shell
upstream request timeout
Status Code 504
```

状态码“504网关超时”是我们可以用来查询Envoy访问日志的一条信息。

Envoy访问日志记录了Envoy代理处理的所有请求，这有助于调试和排除故障。默认情况下，Istio将代理配置为使用`TEXT`格式的日志。这是简明但难以阅读，如下所示:

```shell
# 只查询和打印状态码为504的日志

$ kubectl -n istio-system logs deploy/istio-ingressgateway \
 | grep 504

# output is truncated to a single failing request
[2020-08-22T16:20:20.049Z] "GET /items HTTP/1.1" 504 UT "-" "-" 0 24
501 - "192.168.65.3" "curl/7.64.1" "6f780bed-9996-9c95-a899-a5e293cd9fe4"
"catalog.istioinaction.io" "10.1.0.68:3000"
outbound|80|version-v2|catalog.istioinaction.svc.cluster.local
10.1.0.69:34488 10.1.0.69:8080 192.168.65.3:55962 - -
```

为了易于读取日志，将服务代理配置为使用`JSON`格式：

```shell
# 服务代理访问日志是可配置的，默认情况下，只有Istio的“demo”安装配置文件将访问日志打印到标准输出。如果您正在使用任何其他配置文件，那么您需要设置以下属性' meshConfig.accessLogFile="/dev/stdout" '在Istio安装过程中。如下所示:

$ istioctl install --set meshConfig.accessLogFile="/dev/stdout" --set meshConfig.accessLogEncoding="JSON"
```

上述的部署方式是对整个网格日志做了修改，如果要对特定的`workload`启动访问日志记录，可以使用`EnvoyFilter`：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: webapp-access-logging
  namespace: istioinaction
spec:
  workloadSelector:
    labels:
      app: webapp   # 这个配置允许对“webapp”工作负载进行访问日志记录
  configPatches:
  - applyTo: NETWORK_FILTER
    match:
      context: ANY
      listener:
        filterChain:
          filter:
            name: "envoy.filters.network.http_connection_manager"
    patch:
      operation: MERGE
      value:
        typed_config:
          "@type": "type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager"
          access_log:
          - name: envoy.access_loggers.file
            typed_config:
              "@type": "type.googleapis.com/envoy.extensions.access_loggers.file.v3.FileAccessLog"
              path: /dev/stdout
              format: "[%START_TIME%] \"%REQ(:METHOD)% %REQ(X-ENVOY-ORIGINAL-PATH?:PATH)% %PROTOCOL%\" %RESPONSE_CODE% %RESPONSE_FLAGS% \"%UPSTREAM_TRANSPORT_FAILURE_REASON%\" %BYTES_RECEIVED% %BYTES_SENT% %DURATION% %RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)% \"%REQ(X-FORWARDED-FOR)%\" \"%REQ(USER-AGENT)%\" \"%REQ(X-REQUEST-ID)%\" \"%REQ(:AUTHORITY)%\" \"%UPSTREAM_HOST%\" %UPSTREAM_CLUSTER% %UPSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_REMOTE_ADDRESS% %REQUESTED_SERVER_NAME% %ROUTE_NAME%\n"
```

Istio安装更新后，我们可以再次查询访问日志，这一次考虑到它是`JSON`，我们可以将输出管道到**jq**，以提高可读性。

```shell
# 查询最近的日志并使用jq格式化它

$ kubectl -n istio-system logs deploy/istio-ingressgateway \
| grep 504 | tail -n 1 | jq
{
  "user_agent":"curl/7.64.1",
  "Response_code":"504",
  "response_flags":"UT",       # Envoy response flag   'UT'表示“上游请求超时”
  "start_time":"2020-08-22T16:35:27.125Z",
  "method":"GET",
  "request_id":"e65a3ea0-60dd-9f9c-8ef5-42611138ba07",
  "upstream_host":"10.1.0.68:3000", # Upstream host receiving the request  表示处理请求的工作负载的实际IP地址
  "x_forwarded_for":"192.168.65.3",
  "requested_server_name":"-",
  "bytes_received":"0",
  "istio_policy_status":"-",
  "bytes_sent":"24",
  "upstream_cluster":
    "outbound|80|version-v2|catalog.istioinaction.svc.cluster.local",
  "downstream_remote_address":"192.168.65.3:41260",
  "authority":"catalog.istioinaction.io",
  "path":"/items",
  "protocol":"HTTP/1.1",
  "upstream_service_time":"-",
  "upstream_local_address":"10.1.0.69:48016",
  "duration":"503",   # Exceeding duration of 500 milliseconds
  "upstream_transport_failure_reason":"-",
  "route_name":"-",
  "downstream_local_address":"10.1.0.69:8080"
}
```

将`response_flags`UT与此请求关联是很重要的，因为它使我们能够区分超时决策是由代理而不是应用程序做出的。您将经常面对的一些响应标志是:

- UH:没有正常的上游，即集群没有工作负载
- NR:未配置路由
- UC:上行连接终止
- DC:下行连接端子

The entire list can be found in the Envoy [documentation](https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage) (https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage).

`upstream_host`可以帮助我们找到有问题的`POD`

```shell
# 记录下 对应的 POD，以供后面使用
$  SLOW_POD_IP=$(kubectl -n istio-system logs deploy/istio-ingressgateway \
| grep 504 | tail -n 1 | jq -r .upstream_host | cut -d ":" -f1)
$  SLOW_POD=$(kubectl get pods -n istioinaction \
     --field-selector status.podIP=$SLOW_POD_IP \
     -o jsonpath={.items..metadata.name})
```

#### 日志等级

**查找**

```shell
# ' istioctl '为我们提供了读取和更改Envoy代理的日志级别的工具。当前日志级别打印如下图所示:

$  istioctl proxy-config log deploy/istio-ingressgateway -n istio-system

active loggers:
  connection: warning
  conn_handler: warning
  filter: warning
  http: warning
  http2: warning
  jwt: warning
  pool: warning
  router: warning
  stats: warning
# output is truncated

# 让我们从输出中详细说明“connection: warning”的含义。键' connection '表示日志记录范围。同时，值' warning '表示此范围的日志级别，这意味着只有日志级别为warning的日志才会打印与连接相关的日志。
```

日志级别：

- none
- error
- warning
- info
- debug

在我们的例子中，我们可以在这些范围中找到有用的日志:

- connection -与第4层(传输)相关的日志;TCP连接细节
- HTTP - 7层相关的日志(应用程序);HTTP的细节
- router与HTTP请求路由相关的日志
- pool -连接池获取或丢弃上游主机连接的相关日志

**更改日志级别**

```shell
# 增加连接、http和路由器日志记录级别，以便我们对envoy行为有更多的了解:

$  istioctl proxy-config log deploy/istio-ingressgateway \
   -n istio-system \
   --level http:debug,router:debug,connection:debug,pool:debug
```

```shell
# 现在继续打印入口网关的日志。为简单起见，将输出重定向到一个临时文件，如下所示:

$  kubectl logs -n istio-system deploy/istio-ingressgateway \
> /tmp/ingress-logs.txt
```

打开`txt`文件后 搜索关键词 `HTTP 504` ：

```shell
2020-08-29T13:59:47.678259Z    debug   envoy http
[C198][S86652966017378412] encoding headers via codec (end_stream=false):
':status', '504'
'content-length', '24'
'content-type', 'text/plain'
'date', 'Sat, 29 Aug 2020 13:59:47 GMT'
'server', 'istio-envoy'
```

可以看到上述中包含了 连接ID （`C198`），我们可以根据`C198`关键词 查询与该连接相关的所有日志:

```shell
2020-08-29T13:59:47.178478Z debug   envoy http
[C198] new stream     # A new connection stream is created
2020-08-29T13:59:47.178714Z     debug   envoy http
[C198][S86652966017378412] request headers complete (end_stream=true):
':authority', 'catalog.istioinaction.io'
':path', '/items'

2020-08-29T13:59:47.178739Z     debug   envoy http
[C198][S86652966017378412] request end stream
2020-08-29T13:59:47.178926Z     debug   envoy router
[C198][S86652966017378412] cluster
'outbound|80|version-v2|catalog.istioinaction.svc.cluster.local'
match for URL '/items'   # The cluster to route traffic to is matched
2020-08-29T13:59:47.179003Z     debug   envoy router
[C198][S86652966017378412] router decoding headers:
':authority', 'catalog.istioinaction.io'
':path', '/items'
':method', 'GET'
':scheme', 'https'
```

## 通过抓包查找问题

抓包用到两个工具：

-  `ksniff`        一个kubectl插件，可以通过tcpdump抓取pod的网络流量，并将其重定向到Wireshark**
-  `Wireshark` 网络报文分析工具

### Installing Krew, Ksniff, and Wireshark

要安装`ksniff ` , 我们需要Krew的kubectl插件管理器。`Krew`的安装过程在其官方文件[documentation](https://krew.sigs.k8s.io/docs/user-guide/setup/install/) (https://krew.sigs.k8s.io/docs/user-guide/setup/install/)中进行了记录。

安装完`Krew`后，再安装`Ksniff` ：

```shell
$  kubectl krew install sniff
```

`Wireshark`安装直接参考网上别的教程。

### Inspecting network traffic

```shell
# 捕获slow pod的本地主机网络接口上的流量

$  kubectl sniff -n istioinaction $SLOW_POD -i lo

# 在一个成功的连接上，' ksniff'将使用tcpdump从本地主机网络接口捕获网络流量，并将输出重定向到本地的Wireshark实例以进行可视化。如果您仍然运行生成流量的脚本，那么在短时间内将捕获足够的流量。如果不是，那么在一个单独的终端窗口中执行该脚本。

$ ./query-catalog.sh get-items-cont
```

停止捕获额外的网络数据包

![11.stop capturing packets](https://drek4537l1klr.cloudfront.net/posta2/v-14/Figures/11.stop-capturing-packets.png)

为了更好地了解，让我们只显示路径为' /items '的' get '方法的HTTP协议数据包。这可以使用Wireshark显示过滤器，使用查询' http contains "GET /items"，如下图所示。

![12.filtering with display filter](https://drek4537l1klr.cloudfront.net/posta2/v-14/Figures/12.filtering-with-display-filter.png)

​       这减少了输出到我们感兴趣的请求，我们可以得到更多关于*TCP连接*的开始和它被取消的时间的细节，通过跟踪它的*TCP流*。右键单击第一行，选择菜单项“Follow”，然后选择“TCP Stream”。这将打开“跟随TCP流窗口”，它显示了一个易于理解的TCP流格式。请随意关闭此窗口，因为在主Wireshark窗口中过滤的输出就足够了，如下图所示。

![13.stream of packets annotated](https://drek4537l1klr.cloudfront.net/posta2/v-14/Figures/13.stream-of-packets-annotated.png)

- **Point 1:** The TCP three-way handshake was performed to set up a TCP connection. Indicated by the TCP flags `[SYN]`, `[SYN, ACK]`, and `[ACK]`
- **Point 2:** After the connection is set up we can see that the connection is reused for multiple requests from the client, and all of those are successfully served.
- **Point 3:** Another request comes in from the client, which is acknowledged by the server, but the response takes longer than half a second. This can be seen from the time difference from the packet no. 95 to the packet no. 102.
- **Point 4:** The client initiates a TCP Connection Termination by sending a FIN flag, due to the request taking too long, this is acknowledged by the server-side, and the connection is terminated.

## 总结

本章向您展示了如何排除数据平面故障，并让您练习使用调试工具，并告诉您如何:

- 使用`istioctl`命令了解服务网格和服务代理，例如:
  - `proxy-status`   用于查看数据平面的同步状态
  - `analyze`            用于分析服务网格配置
  - `description`     用于获取摘要和验证服务代理配置
  - `proxy-config`   用于查询和修改业务代理配置

- 在应用到集群之前，使用`istioctl analyze`来验证配置
- 使用Kiali及其提供的验证功能来检测常见的配置错误
- 使用`ksniff`捕获受影响 pod 的网络流量
- 解释`envoy logs`的含义，以及如何使用`istioctl`提高日志级别

## 附录

catalog.yaml

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: catalog
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: catalog
  name: catalog
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 3000
  selector:
    app: catalog
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: catalog
    version: v1
  name: catalog
spec:
  replicas: 1
  selector:
    matchLabels:
      app: catalog
      version: v1
  template:
    metadata:
      labels:
        app: catalog
        version: v1
    spec: 
      serviceAccountName: catalog
      containers:
      - env:
        - name: KUBERNETES_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        image: istioinaction/catalog:latest
        imagePullPolicy: IfNotPresent
        name: catalog
        ports:
        - containerPort: 3000
          name: http
          protocol: TCP
        securityContext:
          privileged: false
```

Catalog-deployment-v2.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: catalog
    version: v2
  name: catalog-v2
spec:
  replicas: 2
  selector:
    matchLabels:
      app: catalog
      version: v2
  template:
    metadata:
      labels:
        app: catalog
        version: v2
      # annotations:
        # sidecar.istio.io/extraStatTags: destination_ip,source_ip,source_port
        # readiness.status.sidecar.istio.io/applicationPorts: ""
    spec:
      serviceAccountName: catalog
      containers:
      - env:
        - name: KUBERNETES_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        image: istioinaction/catalog:latency
        imagePullPolicy: IfNotPresent
        name: catalog
        ports:
        - containerPort: 3000
          name: http
          protocol: TCP
        securityContext:
          privileged: false
```

query-catalog.sh

```shell
#!/usr/bin/env bash

help () {
        cat <<EOF
        This script is a collection of request that are used in the book. 
        Below is a list of arguments and the requests that those make:
            - 'get-items' Continuous requests that print the response status code 
            - 'random-agent' Adds either chrome or firefox in the request header.

        Usage: ./bin/query-catalog.sh status-code
EOF
    exit 1
}

TYPE=$1

case ${TYPE} in
    get-items-cont)
        echo "#" curl -s -H \"Host: catalog.istioinaction.io\" -w \"\\nStatus Code %{http_code}\" localhost/items
        echo 
        sleep 2
        while :
        do
            curl -s -H "Host: catalog.istioinaction.io" -w "\nStatus Code %{http_code}\n\n" localhost/items
            sleep .5
        done
        ;;
    get-items)
        echo "#" curl -s -H \"Host: catalog.istioinaction.io\" -w \"\\nStatus Code %{http_code}\" localhost/items
        echo 
        curl -s -H "Host: catalog.istioinaction.io" -w "\nStatus Code %{http_code}" localhost/items
        ;;
    random-agent)
        echo "== REQUEST EXECUTED =="
        echo curl -s -H "Host: catalog.istioinaction.io" -H "User-Agent: RANDOM_AGENT" -w "\nStatus Code %{http_code}\n\n" localhost/items
        echo 
        while :
        do
          useragents=(chrome firefox)
          agent=${useragents[ ($RANDOM % 2) ]}
          curl -s -H "Host: catalog.istioinaction.io" -H "User-Agent: $agent" -w "\nStatus Code %{http_code}\n\n" localhost/items
          sleep .5
        done
        ;;
    delayed-responses)
        CATALOG_POD=$(kubectl get pods -l version=v2 -n istioinaction -o jsonpath={.items..metadata.name} | cut -d ' ' -f1)
        if [ -z "$CATALOG_POD" ]; then
            echo "No pods found with the following query:"
            echo "-> kubectl get pods -l version=v2 -n istioinaction"
            exit 1
        fi

        kubectl -n istioinaction exec -c catalog $CATALOG_POD \
            -- curl -s -X POST -H "Content-Type: application/json" \
            -d '{"active": true,  "type": "latency", "latencyMs": 1000, "volatile": true}' \
            localhost:3000/blowup
        ;;
    *)
        help
        ;;
esac
```


