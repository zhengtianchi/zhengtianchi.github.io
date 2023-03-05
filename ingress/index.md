# k8s-ingress 原理和使用




## ingress是啥东东

上篇文章介绍service时有说了暴露了service的三种方式ClusterIP、NodePort与LoadBalance，这几种方式都是在service的维度提供的，service的作用体现在两个方面，对集群内部，它不断跟踪pod的变化，更新endpoint中对应pod的对象，提供了ip不断变化的pod的服务发现机制，对集群外部，他类似负载均衡器，可以在集群内外部对pod进行访问。但是，单独用service暴露服务的方式，在实际生产环境中不太合适：
ClusterIP的方式只能在集群内部访问。
NodePort方式的话，测试环境使用还行，当有几十上百的服务在集群中运行时，NodePort的端口管理是灾难。
LoadBalance方式受限于云平台，且通常在云平台部署ELB还需要额外的费用。

所幸k8s还提供了一种集群维度暴露服务的方式，也就是ingress。ingress可以简单理解为service的service，他通过独立的ingress对象来制定请求转发的规则，把请求路由到一个或多个service中。这样就把服务与请求规则解耦了，可以从业务维度统一考虑业务的暴露，而不用为每个service单独考虑。
举个例子，现在集群有api、文件存储、前端3个service，可以通过一个ingress对象来实现图中的请求转发：

![clipboard.png](https://segmentfault.com/img/bVbvcFX?w=726&h=456)

ingress规则是很灵活的，可以根据不同域名、不同path转发请求到不同的service，并且支持https/http。

## ingress与ingress-controller

要理解ingress，需要区分两个概念，ingress和ingress-controller：

- ingress对象：
  指的是k8s中的一个api对象，一般用yaml配置。作用是定义请求如何转发到service的规则，可以理解为配置模板。
- ingress-controller：
  具体实现反向代理及负载均衡的程序，对ingress定义的规则进行解析，根据配置的规则来实现请求转发。

简单来说，ingress-controller才是负责具体转发的组件，通过各种方式将它暴露在集群入口，外部对集群的请求流量会先到ingress-controller，而ingress对象是用来告诉ingress-controller该如何转发请求，比如哪些域名哪些path要转发到哪些服务等等。

### ingress-controller

ingress-controller并不是k8s自带的组件，实际上ingress-controller只是一个统称，用户可以选择不同的ingress-controller实现，目前，由k8s维护的ingress-controller只有google云的GCE与ingress-nginx两个，其他还有很多第三方维护的ingress-controller，具体可以参考[官方文档](https://link.segmentfault.com/?enc=gOwi4b0GRczVszW6sLhxiw%3D%3D.wriAE8eamBB4ZMaXP3BcO3LEAJX9W0kduETZ1M7nkwUkzT3bpzHErWMWJwQ2IxpXV9gVz0ohwyNRYssv7I%2F%2B3WcUHr8C4We8SdwBr53h5dpNpz%2FNnsBQyIPPjkAC%2Fgjov%2Fuzd6lE9GBdvG0fwF6zGw%3D%3D)。但是不管哪一种ingress-controller，实现的机制都大同小异，只是在具体配置上有差异。一般来说，ingress-controller的形式都是一个pod，里面跑着daemon程序和反向代理程序。daemon负责不断监控集群的变化，根据ingress对象生成配置并应用新配置到反向代理，比如nginx-ingress就是动态生成nginx配置，动态更新upstream，并在需要的时候reload程序应用新配置。为了方便，后面的例子都以k8s官方维护的nginx-ingress为例。

### ingress

ingress是一个API对象，和其他对象一样，通过yaml文件来配置。ingress通过http或https暴露集群内部service，给service提供外部URL、负载均衡、SSL/TLS能力以及基于host的方向代理。ingress要依靠ingress-controller来具体实现以上功能。前一小节的图如果用ingress来表示，大概就是如下配置：

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: abc-ingress
  annotations: 
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  tls:
  - hosts:
    - api.abc.com
    secretName: abc-tls
  rules:
  - host: api.abc.com
    http:
      paths:
      - backend:
          serviceName: apiserver
          servicePort: 80
  - host: www.abc.com
    http:
      paths:
      - path: /image/*
        backend:
          serviceName: fileserver
          servicePort: 80
  - host: www.abc.com
    http:
      paths:
      - backend:
          serviceName: feserver
          servicePort: 8080
```

与其他k8s对象一样，ingress配置也包含了apiVersion、kind、metadata、spec等关键字段。有几个关注的在spec字段中，tls用于定义https密钥、证书。rule用于指定请求路由规则。这里值得关注的是metadata.annotations字段。在ingress配置中，annotations很重要。前面有说ingress-controller有很多不同的实现，而不同的ingress-controller就可以根据"kubernetes.io/ingress.class:"来判断要使用哪些ingress配置，同时，不同的ingress-controller也有对应的annotations配置，用于自定义一些参数。列如上面配置的'nginx.ingress.kubernetes.io/use-regex: "true"',最终是在生成nginx配置中，会采用location ~来表示正则匹配。

## ingress的部署

ingress的部署，需要考虑两个方面：

1. ingress-controller是作为pod来运行的，以什么方式部署比较好
2. ingress解决了把如何请求路由到集群内部，那它自己怎么暴露给外部比较好

下面列举一些目前常见的部署和暴露方式，具体使用哪种方式还是得根据实际需求来考虑决定。

### Deployment+LoadBalancer模式的Service

如果要把ingress部署在公有云，那用这种方式比较合适。用Deployment部署ingress-controller，创建一个type为LoadBalancer的service关联这组pod。大部分公有云，都会为LoadBalancer的service自动创建一个负载均衡器，通常还绑定了公网地址。只要把域名解析指向该地址，就实现了集群服务的对外暴露。

### Deployment+NodePort模式的Service

同样用deployment模式部署ingress-controller，并创建对应的服务，但是type为NodePort。这样，ingress就会暴露在集群节点ip的特定端口上。由于nodeport暴露的端口是随机端口，一般会在前面再搭建一套负载均衡器来转发请求。该方式一般用于宿主机是相对固定的环境ip地址不变的场景。
NodePort方式暴露ingress虽然简单方便，但是NodePort多了一层NAT，在请求量级很大时可能对性能会有一定影响。

### DaemonSet+HostNetwork+nodeSelector

用DaemonSet结合nodeselector来部署ingress-controller到特定的node上，然后使用HostNetwork直接把该pod与宿主机node的网络打通，直接使用宿主机的80/433端口就能访问服务。这时，ingress-controller所在的node机器就很类似传统架构的边缘节点，比如机房入口的nginx服务器。该方式整个请求链路最简单，性能相对NodePort模式更好。缺点是由于直接利用宿主机节点的网络和端口，一个node只能部署一个ingress-controller pod。比较适合大并发的生产环境使用。

## ingress测试

我们来实际部署和简单测试一下ingress。测试集群中已经部署有2个服务gowebhost与gowebip，每次请求能返回容器hostname与ip。测试搭建一个ingress来实现通过域名的不同path来访问这两个服务：

![clipboard.png](https://segmentfault.com/img/bVbvvaL?w=516&h=390)

测试ingress使用k8s社区的[ingress-nginx](https://link.segmentfault.com/?enc=nKydMD0zrAP8lu5Slcfzlw%3D%3D.StuvIALLtxFDuPvDD4qhINfupR%2B%2FTibG4qzXIKOjO2rbJLWIbDQ6uv6S8xs0GFWg)，部署方式用DaemonSet+HostNetwork。

### 部署ingress-controller

#### 部署ingress-controller pod及相关资源

官方文档中，部署只要简单的执行一个yaml

```awk
https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/mandatory.yaml
```

mandatory.yaml这一个yaml中包含了很多资源的创建，包括namespace、ConfigMap、role，ServiceAccount等等所有部署ingress-controller需要的资源，配置太多就不粘出来了，我们重点看下deployment部分：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/part-of: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
      annotations:
        prometheus.io/port: "10254"
        prometheus.io/scrape: "true"
    spec:
      serviceAccountName: nginx-ingress-serviceaccount
      containers:
        - name: nginx-ingress-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.25.0
          args:
            - /nginx-ingress-controller
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
            - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
            - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
            - --publish-service=$(POD_NAMESPACE)/ingress-nginx
            - --annotations-prefix=nginx.ingress.kubernetes.io
          securityContext:
            allowPrivilegeEscalation: true
            capabilities:
              drop:
                - ALL
              add:
                - NET_BIND_SERVICE
            # www-data -> 33
            runAsUser: 33
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - name: http
              containerPort: 80
            - name: https
              containerPort: 443
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 10
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 10
```

可以看到主要使用了“quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.25.0”这个镜像，指定了一些启动参数。同时开放了80与443两个端口，并在10254端口做了健康检查。
我们需要使用daemonset部署到特定node，需要修改部分配置：先给要部署nginx-ingress的node打上特定标签,这里测试部署在"node-1"这个节点。

```crmsh
$ kubectl  label   node  node-1 isIngress="true"
```

然后修改上面mandatory.yaml的deployment部分配置为：

```yaml
# 修改api版本及kind
# apiVersion: apps/v1
# kind: Deployment
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
# 删除Replicas
# replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/part-of: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
      annotations:
        prometheus.io/port: "10254"
        prometheus.io/scrape: "true"
    spec:
      serviceAccountName: nginx-ingress-serviceaccount
      # 选择对应标签的node
      nodeSelector:
        isIngress: "true"
      # 使用hostNetwork暴露服务
      hostNetwork: true
      containers:
        - name: nginx-ingress-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.25.0
          args:
            - /nginx-ingress-controller
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
            - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
            - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
            - --publish-service=$(POD_NAMESPACE)/ingress-nginx
            - --annotations-prefix=nginx.ingress.kubernetes.io
          securityContext:
            allowPrivilegeEscalation: true
            capabilities:
              drop:
                - ALL
              add:
                - NET_BIND_SERVICE
            # www-data -> 33
            runAsUser: 33
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - name: http
              containerPort: 80
            - name: https
              containerPort: 443
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 10
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 10
```

修改完后执行apply,并检查服务

```bash
$ kubectl apply -f mandatory.yaml

namespace/ingress-nginx created
configmap/nginx-configuration created
configmap/tcp-services created
configmap/udp-services created
serviceaccount/nginx-ingress-serviceaccount created
clusterrole.rbac.authorization.k8s.io/nginx-ingress-clusterrole created
role.rbac.authorization.k8s.io/nginx-ingress-role created
rolebinding.rbac.authorization.k8s.io/nginx-ingress-role-nisa-binding created
clusterrolebinding.rbac.authorization.k8s.io/nginx-ingress-clusterrole-nisa-binding created
daemonset.extensions/nginx-ingress-controller created
# 检查部署情况
$ kubectl get daemonset -n ingress-nginx
NAME                       DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR    AGE
nginx-ingress-controller   1         1         1       1            1           isIngress=true   101s
$ kubectl get po -n ingress-nginx -o wide
NAME                             READY   STATUS    RESTARTS   AGE    IP               NODE     NOMINATED NODE   READINESS GATES
nginx-ingress-controller-fxx68   1/1     Running   0          117s   172.16.201.108   node-1   <none>           <none>
```

可以看到，nginx-controller的pod已经部署在在node-1上了。

#### 暴露nginx-controller

到node-1上看下本地端口：

```bash
[root@node-1 ~]#  netstat -lntup | grep nginx
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      2654/nginx: master  
tcp        0      0 0.0.0.0:8181            0.0.0.0:*               LISTEN      2654/nginx: master  
tcp        0      0 0.0.0.0:443             0.0.0.0:*               LISTEN      2654/nginx: master  
tcp6       0      0 :::10254                :::*                    LISTEN      2632/nginx-ingress- 
tcp6       0      0 :::80                   :::*                    LISTEN      2654/nginx: master  
tcp6       0      0 :::8181                 :::*                    LISTEN      2654/nginx: master  
tcp6       0      0 :::443                  :::*                    LISTEN      2654/nginx: master
```

由于配置了hostnetwork，nginx已经在node主机本地监听80/443/8181端口。其中8181是nginx-controller默认配置的一个default backend。这样，只要访问node主机有公网IP，就可以直接映射域名来对外网暴露服务了。如果要nginx高可用的话，可以在多个node
上部署，并在前面再搭建一套LVS+keepalive做负载均衡。用hostnetwork的另一个好处是，如果lvs用DR模式的话，是不支持端口映射的，这时候如果用nodeport，暴露非标准的端口，管理起来会很麻烦。

### 配置ingress资源

部署完ingress-controller，接下来就按照测试的需求来创建ingress资源。

```yaml
# ingresstest.yaml

apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-test
  annotations:
    kubernetes.io/ingress.class: "nginx"
    # 开启use-regex，启用path的正则匹配 
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  rules:
    # 定义域名
    - host: test.ingress.com
      http:
        paths:
        # 不同path转发到不同端口
          - path: /ip
            backend:
              serviceName: gowebip-svc
              servicePort: 80
          - path: /host
            backend:
              serviceName: gowebhost-svc
              servicePort: 80
```

部署资源

```powershell
$ kubectl apply -f ingresstest.yaml
```

#### 测试访问

部署好以后，做一条本地host来模拟解析test.ingress.com到node的ip地址。测试访问

![clipboard.png](https://segmentfault.com/img/bVbvHpS?w=418&h=141)

![clipboard.png](https://segmentfault.com/img/bVbvHpE?w=420&h=156)

可以看到，请求不同的path已经按照需求请求到不同服务了。
由于没有配置默认后端，所以访问其他path会提示404：

![clipboard.png](https://segmentfault.com/img/bVbvHaE?w=492&h=284)

#### 关于ingress-nginx

关于ingress-nginx多说几句，上面测试的例子是非常简单的，实际ingress-nginx的有非常多的配置，都可以单独开几篇文章来讨论了。但本文主要想说明ingress，所以不过多涉及。具体可以参考ingress-nginx的[官方文档](https://link.segmentfault.com/?enc=491vMUtqCTLPv3jHhG%2BENQ%3D%3D.sQ7SaI4BS8uABThcHwpDNKock68qjAIJ1R7Fx8XZITaAwQOFvvZfN25MPYrWz36n)。同时，在生产环境使用ingress-nginx还有很多要考虑的地方，[这篇文章](https://link.segmentfault.com/?enc=vRm%2FOp2NbtAdiBRXrKogHA%3D%3D.cEX3HvsINeE8ErTuHZguKOuFwyduX%2FeOY%2BTZhc91sx5zmBhqDv3u2JB776IQwVU2HdvwsyeV87H52AveHXbu%2Bw%3D%3D)写得很好，总结了不少最佳实践，值得参考。

## 最后

- ingress是k8s集群的请求入口，可以理解为对多个service的再次抽象
- 通常说的ingress一般包括ingress资源对象及ingress-controller两部分组成
- ingress-controller有多种实现，社区原生的是ingress-nginx，根据具体需求选择
- ingress自身的暴露有多种方式，需要根据基础环境及业务类型选择合适的方式

## 参考

[Kubernetes Document](https://link.segmentfault.com/?enc=pIjJKML%2F%2Feh7k3LMSWXBvw%3D%3D.m9IsNnRQCE0FdW4a2mwRZYepUPQwaSZE26zSEBqiOJOQS4Xln9w8P2xRxNip9oUOYBpAkGL76It%2FZQD2gAe46vK%2FcsP%2F7TgunsAtWOS2Eag%3D)
[NGINX Ingress Controller Document](https://link.segmentfault.com/?enc=k10u2%2FSaxTLlA6LT4OH%2F1g%3D%3D.%2F4VwK47aECSxFSx7ZYxQ5tx7DkdilR7z9RczCydoBI2sndWjbSVYcEkHQ0urHCTo)
[Kubernetes Ingress Controller的使用介绍及高可用落地](https://link.segmentfault.com/?enc=FqQ1RUiw1OQJ06Bk7Sj6jw%3D%3D.QYlVVNnViueEGdjUzCynS50U9ucVoUQPUEn2LcValsnhviK%2BAGrPutOkkRCETUffPRsPuaBFvHm11k6NdPr9ouFlZE8T0OJSLTQy3T1dlL2CIRpDv5sCwQtHVskM%2B5N5)
[通俗理解Kubernetes中Service、Ingress与Ingress Controller的作用与关系](https://link.segmentfault.com/?enc=%2F3xeKSBFyuVR9cxTPIGIiA%3D%3D.cpynWj4e%2F0h6io9J3srqeE1lKYCjSr2%2Fh96IJfpZaKd1Uwy20NN9RjyBvGJHbZQJu5sH8AI5pKqMpobK%2FqIxZJ9gGk2w9%2Fe3Iy1WFs48I%2F%2FE27%2FfTYn5u9ZHxcNwyUgm)

