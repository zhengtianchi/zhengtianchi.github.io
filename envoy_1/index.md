# Envoy入门介绍




# 介绍[¶](https://www.qikqiak.com/envoy-book/#_1)

[Envoy](https://www.envoyproxy.io/) 是一个面向服务架构的 L7 代理和通信总线而设计的，这个项目诞生是出于以下目标：

> 对于应用程序而言，网络应该是透明的，当发生网络和应用程序故障时，能够很容易定位出问题的根源。

![Envoy logo](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200409172220.png)

Envoy 的核心功能：

- 非侵入的架构：Envoy 是和应用服务并行运行的，透明地代理应用服务发出/接收的流量。应用服务只需要和 Envoy 通信，无需知道其他微服务应用在哪里。
- 基于 Modern C++11 实现，性能优异。
- L3/L4 过滤器架构：Envoy 的核心是一个 L3/L4 代理，然后通过插件式的过滤器(network filters)链条来执行 TCP/UDP 的相关任务，例如 TCP 转发，TLS 认证等工作。
- HTTP L7 过滤器架构：HTTP 在现代应用体系里是地位非常特殊的应用层协议，所以 Envoy 内置了一个非常核心的过滤器: `http_connection_manager`。`http_connection_manager` 本身是如此特殊和复杂，支持丰富的配置，以及本身也是过滤器架构，可以通过一系列 http 过滤器来实现 http 协议层面的任务，例如：http 路由，重定向，CORS 支持等等。
- HTTP/2 作为第一公民：Envoy 支持 HTTP/1.1 和 HTTP/2，推荐使用 HTTP/2。
- gRPC 支持：因为对 HTTP/2 的良好支持，Envoy 可以方便的支持 gRPC，特别是在负载和代理上。
- 服务发现： 支持包括 DNS, EDS 在内的多种服务发现方案。
- 健康检查：内置健康检查子系统。
- 高级的负载均衡方案：除了一般的负载均衡，Envoy 还支持基于 rate limit 服务的多种高级负载均衡方案，包括： `automatic retries`、`circuit breaking`、`global rate limiting` 等。
- Tracing：方便集成 Open Tracing 系统，追踪请求
- 统计与监控：内置 stats 模块，方便集成诸如 prometheus/statsd 等监控方案
- 动态配置：通过`“动态配置API”`实现配置的动态调整，而无需重启 Envoy 服务的。



# Envoy 入门[¶](https://www.qikqiak.com/envoy-book/getting-started/#envoy)

[Envoy](https://www.envoyproxy.io/) 是一个开源的边缘服务代理，也是 Istio Service Mesh 默认的数据平面，专为云原生应用程序设计。

下面我们通过一个简单的示例来介绍 Envoy 的基本使用。

## 1. 配置[¶](https://www.qikqiak.com/envoy-book/getting-started/#1)

### 创建代理配置[¶](https://www.qikqiak.com/envoy-book/getting-started/#_1)

Envoy 使用 YAML 配置文件来控制代理的行为。在下面的步骤中，我们将使用静态配置接口来构建配置，也意味着所有设置都是预定义在配置文件中的。此外 Envoy 也支持动态配置，这样可以通过外部一些源来自动发现进行设置。

### 资源[¶](https://www.qikqiak.com/envoy-book/getting-started/#_2)

Envoy 配置的第一行定义了正在使用的接口配置，在这里我们将配置静态 API，因此第一行应为 `static_resources`：

```yaml
static_resources:
```



### 监听器[¶](https://www.qikqiak.com/envoy-book/getting-started/#_3)

在配置的开始定义了监听器（Listeners）。监听器是 Envoy 监听请求的网络配置，例如 IP 地址和端口。我们这里的 Envoy 在 Docker 容器内运行，因此它需要监听 IP 地址 `0.0.0.0`，在这种情况下，Envoy 将在端口 `10000` 上进行监听。

下面是定义监听器的配置：

```yaml
static_resources:
  listeners:
  - name: listener_0
    address:
      socket_address: { address: 0.0.0.0, port_value: 10000 }
```



### 过滤器[¶](https://www.qikqiak.com/envoy-book/getting-started/#_4)

通过 Envoy 监听传入的流量，下一步是定义如何处理这些请求。每个监听器都有一组过滤器，并且不同的监听器可以具有一组不同的过滤器。

在我们这个示例中，我们将所有流量代理到 `baidu.com`，配置完成后我们应该能够通过请求 Envoy 的端点就可以直接看到百度的主页了，而无需更改 URL 地址。

过滤器是通过 `filter_chains` 来定义的，每个过滤器的目的是找到传入请求的匹配项，以使其与目标地址进行匹配：

``` yaml
static_resources:
  listeners:
  - name: listener_0
    address:
      socket_address: { address: 0.0.0.0, port_value: 10000 }

    filter_chains:
    - filters:
      - name: envoy.http_connection_manager
        config:
          stat_prefix: ingress_http
          route_config:
            name: local_route
            virtual_hosts:
            - name: local_service
              domains: ["*"]    # 匹配的主机域名，满足才会路由转发
              routes:           # 路由转发
              - match: { prefix: "/" }  # 请求匹配前缀
                route: { host_rewrite: www.baidu.com, cluster: service_baidu } # 更改requeset header 中的host，以及处理请求的cluster
          http_filters:
          - name: envoy.router
```



该过滤器使用了 `envoy.http_connection_manager`，这是为 HTTP 连接设计的一个内置过滤器:

- `stat_prefix`：为连接管理器发出统计信息时使用的一个前缀。
  - `route_config`：路由配置，如果虚拟主机 `virtual_hosts `匹配上了则检查路由。在我们这里的配置中，无论请求的主机域名是什么，`route_config` 都匹配所有传入的 HTTP 请求，因为`domains` 是 *，所以匹配所有的请求。
- `routes`：如果 URL 前缀匹配，则一组路由规则定义了下一步将发生的状况。`/` 表示匹配根路由。
- `host_rewrite`：更改 HTTP 请求的入站 Host 头信息。
- `cluster`: 将要处理请求的集群名称，下面会有相应的实现。
- `http_filters`: 该过滤器允许 Envoy 在处理请求时去适应和修改请求。

### 集群[¶](https://www.qikqiak.com/envoy-book/getting-started/#_5)

当请求于过滤器匹配时，该请求将会传递到集群`cluster`。下面的配置就是将主机定义为访问 HTTPS 的 baidu.com 域名，如果定义了多个主机，则 Envoy 将执行轮询（Round Robin）策略。配置如下所示：

```yaml
static_resources:
  listeners:
  - name: listener_0
    address:
      socket_address: { address: 0.0.0.0, port_value: 10000 }

    filter_chains:
    - filters:
      - name: envoy.http_connection_manager
        config:
          stat_prefix: ingress_http
          route_config:
            name: local_route
            virtual_hosts:
            - name: local_service
              domains: ["*"]
              routes:
              - match: { prefix: "/" }
                route: { host_rewrite: www.baidu.com, cluster: service_baidu }
          http_filters:
          - name: envoy.router

  clusters:  # 集群定义
  - name: service_baidu
    connect_timeout: 0.25s   # 连接超时
    type: LOGICAL_DNS
    dns_lookup_family: V4_ONLY
    lb_policy: ROUND_ROBIN
    hosts: [{ socket_address: { address: www.baidu.com, port_value: 443 }}]
    tls_context: { sni: baidu.com }
```



### 管理[¶](https://www.qikqiak.com/envoy-book/getting-started/#_6)

最后，还需要配置一个管理模块：

```yaml
admin:
  access_log_path: /tmp/admin_access.log
  address:
    socket_address: { address: 0.0.0.0, port_value: 9901 }
```



上面的配置定义了 Envoy 的静态配置模板，监听器定义了 Envoy 的端口和 IP 地址，监听器具有一组过滤器来匹配传入的请求，匹配请求后，将请求转发到集群。

## 2. 开启代理[¶](https://www.qikqiak.com/envoy-book/getting-started/#2)

配置完成后，可以通过 Docker 容器来启动 Envoy，将上面的配置文件通过 Volume 挂载到容器中的 `/etc/envoy/envoy.yaml` 文件。

然后使用以下命令启动绑定到端口 80 的 Envoy 容器：

```shell
$ docker run --name=envoy -d \
  -p 80:10000 \
  -v $(pwd)/manifests/envoy.yaml:/etc/envoy/envoy.yaml \
  envoyproxy/envoy:latest
```



启动后，我们可以在本地的 80 端口上去访问应用 `curl localhost` 来测试代理是否成功。同样我们也可以通过在本地浏览器中访问 `localhost` 来查看：

![envoy proxy baidu](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200402202659.png)

可以看到请求被代理到了 `baidu.com`，而且应该也可以看到 URL 地址没有变化，还是 `localhost`。

## 3. 管理视图[¶](https://www.qikqiak.com/envoy-book/getting-started/#3)

Envoy 提供了一个管理视图，可以让我们去查看配置、统计信息、日志以及其他 Envoy 内部的一些数据。

我们可以通过添加其他的资源定义来配置 admin，其中也可以定义管理视图的端口，不过需要注意该端口不要和其他监听器配置冲突。

```yaml
admin:
  access_log_path: /tmp/admin_access.log
  address:
    socket_address: { address: 0.0.0.0, port_value: 9901 }
```



当然我们也可以通过 Docker 容器将管理端口暴露给外部用户。上面的配置就会将管理页面暴露给外部用户，当然我们这里仅仅用于演示是可以的，如果你是用于线上环境还需要做好一些安全保护措施，可以查看 [Envoy 的相关文档](https://www.envoyproxy.io/docs/envoy/latest/operations/admin) 了解更多安全配置。

要将管理页面也暴露给外部用户，我们使用如下命令运行另外一个容器：

```shell
$ docker run --name=envoy-with-admin -d \
    -p 9901:9901 \
    -p 10000:10000 \
    -v $(pwd)/manifests/envoy.yaml:/etc/envoy/envoy.yaml \
    envoyproxy/envoy:latest
```



运行成功后，现在我们可以在浏览器里面输入 `localhost:9901` 来访问 Envoy 的管理页面：

![envoy admin view](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200402204612.png)



# 迁移 NGINX 到 Envoy[¶](https://www.qikqiak.com/envoy-book/migrate-from-nginx-to-envoy/#nginx-envoy)

大部分的应用可能还是使用的比较传统的 Nginx 来做服务代理，本文我们将介绍如何将 Nginx 的配置迁移到 Envoy 上来。我们将学到：

- 如何设置 Envoy 代理配置
- 配置 Envoy 代理转发请求到外部服务
- 配置访问和错误日志

最后我们还会了解到 Envoy 代理的核心功能，以及如何将现有的 Nginx 配置迁移到 Envoy 上来。

## 1. Nginx 示例[¶](https://www.qikqiak.com/envoy-book/migrate-from-nginx-to-envoy/#1-nginx)

这里我们使用 [Nginx 官方 Wiki 的完整示例](https://www.nginx.com/resources/wiki/start/topics/examples/fullexample2/)来进行说明，完整的 `nginx.conf` 配置如下所示：

```conf
user  www www;
pid /var/run/nginx.pid;
worker_processes  2;

events {
  worker_connections   2000;
}

http {
  gzip on;
  gzip_min_length  1100;
  gzip_buffers     4 8k;
  gzip_types       text/plain;

  log_format main      '$remote_addr - $remote_user [$time_local]  '
    '"$request" $status $bytes_sent '
    '"$http_referer" "$http_user_agent" '
    '"$gzip_ratio"';

  log_format download  '$remote_addr - $remote_user [$time_local]  '
    '"$request" $status $bytes_sent '
    '"$http_referer" "$http_user_agent" '
    '"$http_range" "$sent_http_content_range"';

  upstream targetCluster {
    172.17.0.3:80;
    172.17.0.4:80;
  }

  server {
    listen        8080;
    server_name   one.example.com  www.one.example.com;

    access_log   /var/log/nginx.access_log  main;
    error_log  /var/log/nginx.error_log  info;

    location / {
      proxy_pass         http://targetCluster/;
      proxy_redirect     off;

      proxy_set_header   Host             $host;
      proxy_set_header   X-Real-IP        $remote_addr;
    }
  }
}
```



上面的 Nginx 配置有3个核心配置：

- 配置 Nginx 服务、日志结构和 Gzip 功能
- 配置 Nginx 在端口 8080 上接受对 `one.example.com` 域名的请求
- 根据不同的路径配置将流量转发给目标服务

并不是所有的 Nginx 的配置都适用于 Envoy，有些方面的配置我们可以不用关系。Envoy 代理主要有4种主要的配置类型，它们是支持 Nginx 提供的核心基础结构的：

- Listeners（监听器）：他们定义 Envoy 代理如何接收传入的网络请求，建立连接后，它会传递到一组过滤器进行处理
- Filters（过滤器）：过滤器是处理传入和传出请求的管道结构的一部分，比如可以开启类似于 Gzip 之类的过滤器，该过滤器就会在将数据发送到客户端之前进行压缩
- Routers（路由器）：这些路由器负责将流量转发到定义的目的集群去
- Clusters（集群）：集群定义了流量的目标端点和相关配置。

我们将使用这4个组件来创建 Envoy 代理配置，去匹配 Nginx 中的配置。Envoy 的重点一直是在 API 和动态配置上，但是我们这里需要使用静态配置。

## 2. Nginx 配置[¶](https://www.qikqiak.com/envoy-book/migrate-from-nginx-to-envoy/#2-nginx)

现在我们来仔细看下上面的 `nginx.conf` 配置文件的内容。

### 工作连接[¶](https://www.qikqiak.com/envoy-book/migrate-from-nginx-to-envoy/#_1)

首先是 Worker 连接数配置，主要是用于定义工作进程和连接的数量，用来配置 Nginx 扩展请求：

```
worker_processes  2;

events {
  worker_connections   2000;
}
```



Envoy 代理用另外一种不同的方式来管理工作进程和连接。Envoy 使用**单进程-多线程**的架构模型。一个 master 线程管理各种琐碎的任务，而一些 worker 线程则负责执行监听、过滤和转发。当监听器接收到一个连接请求时，该连接将其生命周期绑定到一个单独的 worker 线程。这使得 Envoy 主要使用大量单线程处理工作，并且只有少量的复杂代码用于实现 worker 线程之间的协调工作。通常情况下，Envoy 实现了100%的非阻塞。对于大多数工作负载，我们建议将 worker 线程数配置为物理机器的线程数。关于 Envoy 线程模型的更多信息，可以查看 Envoy 官方博客介绍：[Envoy threading model](https://blog.envoyproxy.io/envoy-threading-model-a8d44b922310?gi=8880ecbe27c5)

![Threading overview](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200408105944.png)

### HTTP 配置[¶](https://www.qikqiak.com/envoy-book/migrate-from-nginx-to-envoy/#http)

然后 Nginx 配置的下一个部分是定义 HTTP 配置，比如：

- 定义支持哪些 MIME 类型
- 默认的超时时间
- Gzip 配置

我们可以通过 Envoy 代理中的过滤器来配置这些内容。在 HTTP 配置部分，Nginx 配置指定了监听的端口 8080，并响应域名 `one.example.com` 和 `www.one.example.com` 的传入请求：

```
server {
    listen        8080;
    server_name   one.example.com  www.one.example.com;
    ......
}
```



在 Envoy 中，这部分就是监听器来管理的。开始一个 Envoy 代理最重要的方面就是定义监听器，我们需要创建一个配置文件来描述我们如何去运行 Envoy 实例。

下面的配置将创建一个新的监听器并将其绑定到 8080 端口上，该配置指示了 Envoy 代理用于接收网络请求的端口。Envoy 配置使用的是 YAML 文件，如果你对 YAML 文件格式语法不是很熟悉的，可以[点此先查看官方对应的介绍](https://yaml.org/spec/1.2/spec.html)。

```yaml
static_resources:
  listeners:
  - name: listener_0
    address:
      socket_address: { address: 0.0.0.0, port_value: 8080 }
```



需要注意的是我们没有在监听器部分定义 `server_name`，我们会在过滤器部分进行处理。

### Location 配置[¶](https://www.qikqiak.com/envoy-book/migrate-from-nginx-to-envoy/#location)

当请求进入 Nginx 时，`location` 部分定义了如何处理流量以及在什么地方转发流量。在下面的配置中，站点的所有传入流量都将被代理到一个名为 `targetCluster` 的上游（upstream）集群，上游集群定义了处理请求的节点。

```
location / {
    proxy_pass         http://targetCluster/;
    proxy_redirect     off;

    proxy_set_header   Host             $host;
    proxy_set_header   X-Real-IP        $remote_addr;
}
```



在 Envoy 中，这部分将由过滤器来进行配置管理。在静态配置中，过滤器定义了如何处理传入的请求，在我们这里，将配置一个过滤器去匹配上一步中的 `server_names`，当接收到与定义的域名和路由匹配的传入请求时，流量将转发到集群，集群和 Nginx 配置中的 upstream 是一致的。

```yaml
filter_chains:
- filters:
  - name: envoy.http_connection_manager
    config:
      codec_type: auto
      stat_prefix: ingress_http
      route_config:
        name: local_route
        virtual_hosts:
        - name: backend
          domains:
          - "one.example.com"
          - "www.one.example.com"
          routes:
          - match:
              prefix: "/"
            route:
              cluster: targetCluster
      http_filters:
      - name: envoy.router
```



其中 `envoy.http_connection_manager` 是 Envoy 内置的一个过滤器，用于处理 HTTP 连接的，除此之外，还有其他的一些内置的过滤器，比如 Redis、Mongo、TCP。

### upstream 配置[¶](https://www.qikqiak.com/envoy-book/migrate-from-nginx-to-envoy/#upstream)

在 Nginx 中，upstream（上游）配置定义了处理请求的目标服务器集群，在我们这里的示例中，分配了两个集群。

```
upstream targetCluster {
  172.17.0.3:80;
  172.17.0.4:80;
}
```



在 Envoy 代理中，这部分是通过集群进行配置管理的。upstream 等同与 Envoy 中的 clusters 定义，我们这里通过集群定义了主机被访问的方式，还可以配置超时和负载均衡等方面更精细的控制。

```yaml
clusters:
- name: targetCluster
  connect_timeout: 0.25s   # 负载均衡
  type: STRICT_DNS
  dns_lookup_family: V4_ONLY
  lb_policy: ROUND_ROBIN   # 负载均衡
  hosts: [
    { socket_address: { address: 172.17.0.3, port_value: 80 }},
    { socket_address: { address: 172.17.0.4, port_value: 80 }}
  ]
```



上面我们配置了 `STRICT_DNS` 类型的服务发现，Envoy 会持续异步地解析指定的 DNS 目标。DNS 解析结果返回的每个 IP 地址都将被视为上游集群的主机。所以如果产线返回两个 IP 地址，则 Envoy 将认为集群由两个主机，并且两个主机都应进行负载均衡，如果从结果中删除了一个主机，则 Envoy 会从现有的连接池中将其剔出掉。

### 日志配置[¶](https://www.qikqiak.com/envoy-book/migrate-from-nginx-to-envoy/#_2)

最后需要配置的日志部分，Envoy 采用云原生的方式，将应用程序日志都输出到 stdout 和 stderr，而不是将错误日志输出到磁盘。

当用户发起一个请求时，访问日志默认是被禁用的，我们可以手动开启。要为 HTTP 请求开启访问日志，需要在 HTTP 连接管理器中包含一个 `access_log` 的配置，该路径可以是设备，比如 stdout，也可以是磁盘上的某个文件，这依赖于我们自己的实际情况。

下面过滤器中的配置就会将所有访问日志通过管理传输到 stdout：

```yaml
- name: envoy.http_connection_manager
  config:
    codec_type: auto
    stat_prefix: ingress_http
    access_log:
    - name: envoy.file_access_log
      config:
        path: "/dev/stdout"
    route_config:
    ......
```



默认情况下，Envoy 访问日志格式包含整个 HTTP 请求的详细信息：

```shell
[%START_TIME%] "%REQ(:METHOD)% %REQ(X-ENVOY-ORIGINAL-PATH?:PATH)% %PROTOCOL%"
%RESPONSE_CODE% %RESPONSE_FLAGS% %BYTES_RECEIVED% %BYTES_SENT% %DURATION%
%RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)% "%REQ(X-FORWARDED-FOR)%" "%REQ(USER-AGENT)%"
"%REQ(X-REQUEST-ID)%" "%REQ(:AUTHORITY)%" "%UPSTREAM_HOST%"\n
```



输出结果格式化后如下所示：

```shell
[2020-04-08T04:51:00.281Z] "GET / HTTP/1.1" 200 - 0 58 4 1 "-" "curl/7.47.0" "f21ebd42-6770-4aa5-88d4-e56118165a7d" "one.example.com" "172.18.0.4:80"
```



我们也可以通过设置 `format` 字段来自定义输出日志的格式，例如：

```yaml
access_log:
- name: envoy.file_access_log
  config:
    path: "/dev/stdout"
    format: "[%START_TIME%] "%REQ(:METHOD)% %REQ(X-ENVOY-ORIGINAL-PATH?:PATH)% %PROTOCOL%" %RESPONSE_CODE% %RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)% "%REQ(X-REQUEST-ID)%" "%REQ(:AUTHORITY)%" "%UPSTREAM_HOST%"\n"
```



此外我们也可以通过设置 `json_format` 字段来将日志作为 JSON 格式输出，例如：

```yaml
access_log:
- name: envoy.file_access_log
  config:
    path: "/dev/stdout"
    json_format: {"protocol": "%PROTOCOL%", "duration": "%DURATION%", "request_method": "%REQ(:METHOD)%"}
```



要注意的是，访问日志会在未设置、或者空值的位置加入一个字符：`-`。不同类型的访问日志（例如 HTTP 和 TCP）共用同样的格式字符串。不同类型的日志中，某些字段可能会有不同的含义。有关 Envoy 日志的更多信息，可以[查看官方文档对应的说明](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/observability/access_logging#arch-overview-access-logs)。当然日志并不是 Envoy 代理获得请求可见性的唯一方法，Envoy 还内置了高级跟踪和指标功能，我们会在后续章节中慢慢接触到。

## 3. 测试[¶](https://www.qikqiak.com/envoy-book/migrate-from-nginx-to-envoy/#3)

现在我们已经将 Nginx 配置转换为了 Envoy 代理，接下来我们可以来启动 Envoy 代理进行测试验证。

在 Nginx 配置的顶部，有一行配置 `user www www;`，表示用非 root 用户来运行 Nginx 以提高安全性。而 Envoy 代理采用云原生的方法来管理使用这，我们通过容器启动 Envoy 代理的时候，可以指定一个低特权的用户。

下面的命令将通过 Docker 容器来启动一个 Envoy 实例，该命令使 Envoy 可以监听 80 端口上的流量请求，但是我们在 Envoy 的监听器配置中指定的是 8080 端口，所以我们用一个低特权用户身份来运行：

```shell
$ docker run --name proxy1 -p 80:8080 --user 1000:1000 -v $(pwd)/manifests/envoy.yaml:/etc/envoy/envoy.yaml envoyproxy/envoy
```



启动代理后，就可以开始测试了，下面我们用 `curl` 命令使用代理配置的 host 头发起一个网络请求：

```shell
$ curl -H "Host: one.example.com" localhost -i
HTTP/1.1 503 Service Unavailable
content-length: 91
content-type: text/plain
date: Wed, 08 Apr 2020 04:25:59 GMT
server: envoy

upstream connect error or disconnect/reset before headers. reset reason: connection failure%
```



我们可以看到会出现 503 错误，这是因为我们配置的上游集群主机根本就没有运行，所以 Envoy 代理请求到不可用的主机上去了，就出现了这样的错误。我们可以使用下面的命令启动两个 HTTP 服务，用来表示上游主机：

```shell
$ docker run -d cnych/docker-http-server; docker run -d cnych/docker-http-server;
$ docker ps
CONTAINER ID        IMAGE                      COMMAND                  CREATED             STATUS              PORTS                             NAMES
fd3018535a52        cnych/docker-http-server   "/app"                   2 minutes ago       Up 2 minutes        80/tcp                            practical_golick
ace25541d654        cnych/docker-http-server   "/app"                   2 minutes ago       Up 2 minutes        80/tcp                            lucid_hugle
3c83dfb9392f        envoyproxy/envoy           "/docker-entrypoint.…"   2 minutes ago       Up 2 minutes        10000/tcp, 0.0.0.0:80->8080/tcp   proxy1
```



当上面两个服务启动成功后，现在我们再通过 Envoy 去访问目标服务就正常了：

```shell
$ curl -H "Host: one.example.com" localhost -i
HTTP/1.1 200 OK
date: Wed, 08 Apr 2020 04:32:01 GMT
content-length: 58
content-type: text/html; charset=utf-8
x-envoy-upstream-service-time: 3
server: envoy

<h1>This request was processed by host: fd3018535a52</h1>
$ curl -H "Host: one.example.com" localhost -i
HTTP/1.1 200 OK
date: Wed, 08 Apr 2020 04:32:05 GMT
content-length: 58
content-type: text/html; charset=utf-8
x-envoy-upstream-service-time: 0
server: envoy

<h1>This request was processed by host: ace25541d654</h1>
```



当访问请求的时候，我们可以看到是哪个容器处理了请求，在 Envoy 代理容器中，也可以看到请求的日志输出：

```shell
[2020-04-08T04:32:06.201Z] "GET / HTTP/1.1" 200 - 0 58 1 0 "-" "curl/7.54.0" "ac61099b-f100-46a9-9c08-c323c5ac2320" "one.example.com" "172.17.0.3:80"
[2020-04-08T04:32:08.168Z] "GET / HTTP/1.1" 200 - 0 58 0 0 "-" "curl/7.54.0" "15ee6ca9-b161-4630-a51c-c641d0760cd0" "one.example.com" "172.17.0.4:80"
```



最后转换过后的完整的 Envoy 配置如下：

```yaml
static_resources:
  listeners:
  - name: listener_0
    address:
      socket_address: { address: 0.0.0.0, port_value: 8080 }

    filter_chains:
    - filters:
      - name: envoy.http_connection_manager
        config:
          codec_type: auto
          stat_prefix: ingress_http
          access_log:
          - name: envoy.file_access_log
            config:
              path: "/dev/stdout"
          route_config:
            name: local_route
            virtual_hosts:
            - name: backend
              domains:
                - "one.example.com"
                - "www.one.example.com"
              routes:
              - match:
                  prefix: "/"
                route:
                  cluster: targetCluster
          http_filters:
          - name: envoy.router

  clusters:
  - name: targetCluster
    connect_timeout: 0.25s
    type: STRICT_DNS
    dns_lookup_family: V4_ONLY
    lb_policy: ROUND_ROBIN
    hosts: [
      { socket_address: { address: 172.17.0.3, port_value: 80 }},
      { socket_address: { address: 172.17.0.4, port_value: 80 }}
    ]
```



# 使用 SSL/TLS 保护流量[¶](https://www.qikqiak.com/envoy-book/secure-traffic-with-https/#ssltls)

本节我们将演示如何使用 Envoy 保护 HTTP 网络请求。确保 HTTP 流量安全对于保护用户隐私和数据是至关重要的。下面我们来了解下如何在 Envoy 中配置 SSL 证书。

## 1. SSL 证书[¶](https://www.qikqiak.com/envoy-book/secure-traffic-with-https/#1-ssl)

这里我们将为 `example.com` 域名生成一个自签名的证书，当然如果在生产环境时候，需要使用正规 CA 机构购买的证书，或者 `Let's Encrypt` 的免费证书服务。

下面的命令会在目录 `certs/` 中创建一个新的证书和密钥：

```shell
$ mkdir certs; cd certs;
$ openssl req -nodes -new -x509 \
  -keyout example-com.key -out example-com.crt \
  -days 365 \
  -subj '/CN=example.com/O=youdianzhishi./C=CN';
  Generating a RSA private key
..+++++
.................................................................................................................+++++
writing new private key to 'example-com.key'
-----
$ cd -
```



## 2. 流量保护[¶](https://www.qikqiak.com/envoy-book/secure-traffic-with-https/#2)

在 Envoy 中保护 HTTP 流量，需要通过添加 `tls_context` 过滤器，TLS 上下文提供了为 Envoy 代理中配置的域名指定证书的功能，请求 HTTPS 请求时候，就使用匹配的证书。我们这里直接使用上一步中生成的自签名证书即可。

我们这里的 Envoy 配置文件中包含了所需的 HTTPS 支持的配置，我们添加了两个监听器，一个监听器在 8080 端口上用于 HTTP 通信，另外一个监听器在 8443 端口上用于 HTTPS 通信。

在 HTTPS 监听器中定义了 HTTP 连接管理器，该代理将代理 `/service/1` 和 `/service/2` 这两个端点的传入请求，这里我们需要通过 `tls_context` 配置相关证书，如下所示：

```yaml
tls_context:
  common_tls_context:
    tls_certificates:
    - certificate_chain:
        filename: "/etc/envoy/certs/example-com.crt"
      private_key:
        filename: "/etc/envoy/certs/example-com.key"
```



在 TLS 上下文中定义了生成的证书和密钥，如果我们有多个域名，每个域名都有自己的证书，则需要通过 `tls_certificates` 定义多个证书链。

## 3. 自动跳转[¶](https://www.qikqiak.com/envoy-book/secure-traffic-with-https/#3)

定义了 TLS 上下文后，该站点将能够通过 HTTPS 提供流量了，但是如果用户是通过 HTTP 来访问的服务，为了确保安全，我们也可以将其重定向到 HTTPS 版本服务上去。

在 HTTP 配置中，我们将 `https_redirect: true` 的标志添加到过滤器的配置中即可实现跳转功能。

```yaml
route_config:
  virtual_hosts:
  - name: backend
    domains:
    - "example.com"
    routes:
    - match:
        prefix: "/"
      redirect:
        path_redirect: "/"
        https_redirect: true   # 实现https跳转
```



当用户访问网站的 HTTP 版本时，Envoy 代理将根据过滤器配置来匹配域名和路径，匹配到过后将请求重定向到站点的 HTTPS 版本去。完整的 Envoy 配置如下所示：

```yaml
static_resources:
  listeners:
  - name: listener_http
    address:
      socket_address: { address: 0.0.0.0, port_value: 8080 }
    filter_chains:
    - filters:
      - name: envoy.http_connection_manager
        config:
          codec_type: auto
          stat_prefix: ingress_http
          route_config:
            virtual_hosts:
            - name: backend
              domains:
              - "example.com"
              routes:
              - match:
                  prefix: "/"
                redirect:
                  path_redirect: "/"
                  https_redirect: true
          http_filters:
          - name: envoy.router
            config: {}
  - name: listener_https
    address:
      socket_address: { address: 0.0.0.0, port_value: 8443 }
    filter_chains:
    - filters:
      - name: envoy.http_connection_manager
        config:
          codec_type: auto
          stat_prefix: ingress_http
          route_config:
            name: local_route
            virtual_hosts:
            - name: backend
              domains:
              - "example.com"
              routes:
              - match:
                  prefix: "/service/1"
                route:
                  cluster: service1
              - match:
                  prefix: "/service/2"
                route:
                  cluster: service2
          http_filters:
          - name: envoy.router
            config: {}
      tls_context:
        common_tls_context:
          tls_certificates:
            - certificate_chain:
                filename: "/etc/envoy/certs/example-com.crt"
              private_key:
                filename: "/etc/envoy/certs/example-com.key"
  clusters:
  - name: service1
    connect_timeout: 0.25s
    type: strict_dns
    lb_policy: round_robin
    hosts:
    - socket_address:
        address: 172.17.0.3
        port_value: 80
  - name: service2
    connect_timeout: 0.25s
    type: strict_dns
    lb_policy: round_robin
    hosts:
    - socket_address:
        address: 172.17.0.4
        port_value: 80

admin:
  access_log_path: /tmp/admin_access.log
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 8001
```



## 4. 测试[¶](https://www.qikqiak.com/envoy-book/secure-traffic-with-https/#4)

现在配置已经完成了，我们就可以启动 Envoy 实例来进行测试了。在我们这个示例中，Envoy 暴露 80 端口来处理 HTTP 请求，暴露 443 端口来处理 HTTPS 请求，此外还在 8001 端口上暴露了管理页面，我们可以通过管理页面查看有关证书的信息。

使用如下命令启动 Envoy 代理：

```shell
$ docker run -it --name proxy1 -p 80:8080 -p 443:8443 -p 8001:8001 -v $(pwd):/etc/envoy/ envoyproxy/envoy
```



启动完成后所有的 HTTPS 和 TLS 校验都是通过 Envoy 来进行处理的，所以我们不需要去修改应该程序。同样我们启动两个 HTTP 服务来处理传入的请求：

```shell
$ docker run -d cnych/docker-http-server; docker run -d cnych/docker-http-server;
145738e12c174606f9e6e085ad2ec0ae9bf15a75d372b2bec8929e5d5df96be3
8f9a9355333d91b06a14d2bccc1a0d4a9afd20b258df561278fb94f01cdcd881
$ docker ps
CONTAINER ID        IMAGE                      COMMAND                  CREATED              STATUS              PORTS                                                                            NAMES
8f9a9355333d        cnych/docker-http-server   "/app"                   5 seconds ago        Up 4 seconds        80/tcp                                                                           eager_kapitsa
145738e12c17        cnych/docker-http-server   "/app"                   6 seconds ago        Up 5 seconds        80/tcp                                                                           beautiful_hermann
a499a8ccaedc        envoyproxy/envoy           "/docker-entrypoint.…"   About a minute ago   Up About a minute   0.0.0.0:8001->8001/tcp, 10000/tcp, 0.0.0.0:80->8080/tcp, 0.0.0.0:443->8443/tcp   proxy1
```



上面的几个容器启动完成后，就可以进行测试了，首先我们请求 HTTP 的服务，由于配置了自动跳转，所以应该会被重定向到 HTTPS 的版本上去：

```shell
$ curl -H "Host: example.com" http://localhost -i
HTTP/1.1 301 Moved Permanently   # 重定向响应
location: https://example.com/
date: Wed, 08 Apr 2020 06:53:51 GMT
server: envoy
content-length: 0
```



我们可以看到上面有 `HTTP/1.1 301 Moved Permanently` 这样的重定向响应信息。然后我们尝试直接请求 HTTPS 的服务：

```shell
$ curl -k -H "Host: example.com" https://localhost/service/1 -i
HTTP/1.1 200 OK
date: Wed, 08 Apr 2020 06:55:27 GMT
content-length: 58
content-type: text/html; charset=utf-8
x-envoy-upstream-service-time: 0
server: envoy

<h1>This request was processed by host: 145738e12c17</h1>
$ curl -k -H "Host: example.com" https://localhost/service/2 -i
HTTP/1.1 200 OK
date: Wed, 08 Apr 2020 06:55:49 GMT
content-length: 58
content-type: text/html; charset=utf-8
x-envoy-upstream-service-time: 0
server: envoy

<h1>This request was processed by host: 8f9a9355333d</h1>
```



我们可以看到通过 HTTPS 进行访问可以正常得到对应的响应，需要注意的是由于我们这里使用的是自签名的证书，所以需要加上 `-k` 参数来忽略证书校验，如果没有这个参数则在请求的时候会报错：

```shell
$ curl -H "Host: example.com" https://localhost/service/2 -i
curl: (60) SSL certificate problem: self signed certificate
More details here: https://curl.haxx.se/docs/sslcerts.html

curl performs SSL certificate verification by default, using a "bundle"
 of Certificate Authority (CA) public keys (CA certs). If the default
 bundle file isn't adequate, you can specify an alternate file
 using the --cacert option.
If this HTTPS server uses a certificate signed by a CA represented in
 the bundle, the certificate verification probably failed due to a
 problem with the certificate (it might be expired, or the name might
 not match the domain name in the URL).
If you'd like to turn off curl's verification of the certificate, use
 the -k (or --insecure) option.
HTTPS-proxy has similar options --proxy-cacert and --proxy-insecure.
```



我们也可以通过管理页面去查看证书相关的信息，上面我们启动容器的时候绑定了宿主机的 8001 端口，所以我们可以通过访问 `http://localhost:8001/certs` 来获取到证书相关的信息：

![admin certs info](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200408150129.png)

# 基于文件的动态配置[¶](https://www.qikqiak.com/envoy-book/file-based-dynamic-config/#_1)

Envoy 除了支持静态配置之外，还支持动态配置，而且动态配置也是 Envoy 重点关注的功能，本节我们将学习如何将 Envoy 静态配置转换为动态配置，从而允许 Envoy 自动更新。

## 1. Envoy 动态配置[¶](https://www.qikqiak.com/envoy-book/file-based-dynamic-config/#1-envoy)

前面的章节中，我们都是直接使用的静态配置，但是当我们需要更改配置的时候就比较麻烦了，需要重启 Envoy 代理才会生效。要解决这个问题，我们可以将静态配置更改成动态配置，当我们使用动态配置的时候，更改了配置，Envoy 将会自动去重新加载配置。

Envoy 支持不同的模块进行动态配置，可配置的有如下几个 API：

- `EDS`：端点发现服务（EDS）可以让 Envoy 自动发现上游集群的成员，这使得我们可以动态添加或者删除处理流量请求的服务。
- `CDS`：集群发现服务（CDS）可以让 Envoy 通过该机制自动发现在路由过程中使用的上游集群。
- `RDS`：路由发现服务（RDS）可以让 Envoy 在运行时自动发现 HTTP 连接管理过滤器的整个路由配置，这可以让我们来完成诸如动态更改流量分配或者蓝绿发布之类的功能。
- `VHDS`：虚拟主机发现服务（VHDS）允许根据需要与路由配置本身分开请求属于路由配置的虚拟主机。该 API 通常用于路由配置中有大量虚拟主机的部署中。
- `SRDS`：作用域路由发现服务（SRDS）允许将路由表分解为多个部分。该 API 通常用于具有大量路由表的 HTTP 路由部署中。
- `LDS`：监听器发现服务（LDS）可以让 Envoy 在运行时自动发现整个监听器。
- `SDS`：密钥发现服务（SDS）可以让 Envoy 自动发现监听器的加密密钥（证书、私钥等）以及证书校验逻辑（受信任的根证书、吊销等）。

可以使用普通的文件来进行动态配置，也可以通过 REST-JSON 或者 gRPC 端点来提供。我们可以在 [xDS 配置概述文档](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/operations/dynamic_configuration) 中找到更多相关 API 的介绍。

在接下来的步骤中，我们将先更改配置来使用 `EDS`，让 Envoy 根据配置文件的数据来动态添加节点。

### Cluster ID[¶](https://www.qikqiak.com/envoy-book/file-based-dynamic-config/#cluster-id)

首先我们这里定义了一个基本的 Envoy 配置文件，如下所示：(envoy.yaml)

```yaml
admin:
  access_log_path: "/dev/null"
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 9901

static_resources:
  listeners:
  - name: listener_0
    address:
      socket_address: { address: 0.0.0.0, port_value: 10000 }
    filter_chains:
    - filters:
      - name: envoy.http_connection_manager
        config:
          codec_type: auto
          stat_prefix: ingress_http
          route_config:
            name: local_route
            virtual_hosts:
            - name: backend
              domains:
                - "*"
              routes:
              - match:
                  prefix: "/"
                route:
                  cluster: targetCluster
          http_filters:
          - name: envoy.router
```



我们可以看到现在还没有配置 clusters 集群部分，这是因为我们要通过使用 `EDS` 来进行自动发现。

首先我们需要添加一个节点让 Envoy 来识别并应用这一个唯一的配置，将下面的配置放置在 `envoy.yaml` 文件的顶部区域：

```yaml
node:
  id: id_1
  cluster: test
```



除了 `id` 和 `cluster` 之外，我们还可以配置基于区域的一些位置信息来进行声明，比如 `region`、`zone`、`sub_zone`。

## 2. EDS 配置[¶](https://www.qikqiak.com/envoy-book/file-based-dynamic-config/#2-eds)

接下来我们就可以来定义 `EDS` 配置了，可以来动态控制上游集群数据。在前面章节中，这部分的静态配置是这样的：

```yaml
clusters:
- name: targetCluster
  connect_timeout: 0.25s
  type: STRICT_DNS
  dns_lookup_family: V4_ONLY
  lb_policy: ROUND_ROBIN
  hosts: [
    { socket_address: { address: 172.17.0.3, port_value: 80 }},
    { socket_address: { address: 172.17.0.4, port_value: 80 }}
  ]
```



现在我们将上面的静态配置转换成动态配置，首先需要转换为基于 `EDS` 的 `eds_cluster_config` 属性，并将类型更改为 `EDS`，将下面的集群配置添加到 Envoy 配置的末尾：

```yaml
clusters:
- name: targetCluster
  connect_timeout: 0.25s
  lb_policy: ROUND_ROBIN
  type: EDS
  eds_cluster_config:
    service_name: localservices  # 可选，代替集群的名称，提供给 EDS 服务
    eds_config:  # 集群的 EDS 更新源配置
      path: '/etc/envoy/eds.yaml'
```



上游的服务器 `172.17.0.3` 和 `172.17.0.4` 就将来自于 `/etc/envoy/eds.yaml` 文件，创建一个 `eds.yaml` 文件，内容如下所示：

```yaml
version_info: "0"
resources: 
- "@type": "type.googleapis.com/envoy.api.v2.ClusterLoadAssignment"
  cluster_name: "localservices"
  endpoints:
  - lb_endpoints:
    - endpoint:
        address:
          socket_address:
            address: "172.17.0.3"
            port_value: 80
```



上面我们只定义了 `172.17.0.3` 这一个端点。

现在配置完成后，我们可以启动 Envoy 代理来进行测试。执行下面的命令启动 Envoy 容器：

```shell
$ docker run --name=proxy-eds -d \
    -p 9901:9901 \
    -p 80:10000 \
    -v $(pwd)/manifests:/etc/envoy \
    envoyproxy/envoy:latest
```



然后同样和前面一样运行两个 HTTP 服务来作为上游服务器：

```shell
$ docker run -d cnych/docker-http-server; docker run -d cnych/docker-http-server;
$ docker ps
CONTAINER ID        IMAGE                      COMMAND                  CREATED              STATUS              PORTS                                           NAMES
b9f1955e2704        envoyproxy/envoy:latest    "/docker-entrypoint.…"   4 seconds ago        Up 2 seconds        0.0.0.0:9901->9901/tcp, 0.0.0.0:80->10000/tcp   proxy-eds
4fd8eb3bd415        cnych/docker-http-server   "/app"                   About a minute ago   Up 59 seconds       80/tcp                                          wonderful_hoover
73b616391920        cnych/docker-http-server   "/app"                   About a minute ago   Up About a minute   80/tcp                                          pedantic_moser
```



根据上面的 `EDS` 配置，Envoy 将把所有的流量都发送到 `172.17.0.3` 这一个节点上去，我们可以使用 `curl localhost` 来测试下：

```shell
$ curl localhost
<h1>This request was processed by host: 73b616391920</h1>
$ curl localhost
<h1>This request was processed by host: 73b616391920</h1>
```



接下来我们来尝试更新上面的 `EDS` 配置添加上另外的一个节点，观察 Envoy 代理是否会自动生效。

由于我们这里使用的是 `EDS` 动态配置，所以当我们要扩展上游服务的时候，只需要将新的端点添加到上面我们指定的 `eds.yaml` 配置文件中即可，然后 Envoy 就会自动将新添加的端点包含进来。用上面同样的方式添加 `172.17.0.4` 这个端点，`eds.yaml` 内容如下所示：

```yaml
version_info: "0"
resources: 
- "@type": "type.googleapis.com/envoy.api.v2.ClusterLoadAssignment"
  cluster_name: "localservices"
  endpoints:
  - lb_endpoints:
    - endpoint:
        address:
          socket_address:
            address: "172.17.0.3"
            port_value: 80
    - endpoint:
        address:
          socket_address:
            address: "172.17.0.4"
            port_value: 80
```



由于我们这里是使用的 Docker 容器将配置文件挂载到容器中的，如果直接更改宿主机的配置文件，有时候可能不会立即触发文件变更，我们可以使用如下所示的命令来强制变更：

```shell
$ mv manifests/eds.yaml tmp; mv tmp manifests/eds.yaml
```



这个时候正常情况下 Envoy 就会自动重新加载配置并将新的端点添加到负载均衡中去，这个时候我们再来访问代理：

```shell
$ curl localhost
<h1>This request was processed by host: 73b616391920</h1>
$ curl localhost
<h1>This request was processed by host: 4fd8eb3bd415</h1>
```



可以看到已经可以自动访问到另外的端点去了。

注意

我在测试阶段发现在 Mac 系统下面没有自动热加载，在 Linux 系统下面是可以正常重新加载的。

## 3. CDS 配置[¶](https://www.qikqiak.com/envoy-book/file-based-dynamic-config/#3-cds)

现在已经配置好了 `EDS`，接下来我们就可以去扩大上游集群的规模了，如果我们想要能够动态添加新的域名和集群，就需要实现集群发现服务（CDS）API，在下面的示例中，我们将配置集群发现服务（CDS）和监听器发现服务（LDS）来进行动态配置。

创建一个名为 `cds.yaml` 的文件来配置集群服务发现的数据，文件内容如下所示：

```yaml
version_info: "0"
resources:
- "@type": "type.googleapis.com/envoy.api.v2.Cluster"
  name: targetCluster
  connect_timeout: 0.25s
  lb_policy: ROUND_ROBIN
  type: EDS
  eds_cluster_config:
    service_name: localservices
    eds_config:
      path: /etc/envoy/eds.yaml
```



此外，还需要创建一个名为 `lds.yaml` 的文件来放置监听器的配置，文件内容如下所示：

```yaml
version_info: "0"
resources:
- "@type": "type.googleapis.com/envoy.api.v2.Listener"
  name: listener_0
  address:
    socket_address: { address: 0.0.0.0, port_value: 10000 }
  filter_chanins:
  - filters:
    - name: envoy.http_connection_manager
      config: 
        stat_prefix: ingress_http
        codec_type: AUTO
        route_config:
          name: local_route
          virtual_hosts:
          - name: local_service
            domains: ["*"]
            routes:
            - match:
                prefix: "/"
              route:
                cluster: targetCluster
        http_filters:
        - name: envoy.router
```



仔细观察可以发现 `cds.yaml` 和 `lds.yaml` 配置文件的内容基本上和上面的静态配置文件一致的。我们这里只是将集群和监听器拆分到外部文件中去，这个时候我们需要修改 Envoy 的配置来引用这些文件，我们可以通过将 `static_resources` 更改为 `dynamic_resources` 来进行配置。

重新新建一个 Envoy 配置文件，命名为 `envoy1.yaml`，内容如下所示：

```yaml
node:
  id: id_1
  cluster: test
admin:
  access_log_path: "/dev/null"
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 9901
# 动态配置
dynamic_resources:
  lds_config:
    path: "/etc/envoy/lds.yaml"
  cds_config:
    path: "/etc/envoy/cds.yaml"
```



然后使用上面的配置文件重新启动一个新的 Envoy 代理，命令如下所示：

```shell
$ docker run --name=proxy-xds -d \
    -p 9902:9901 \
    -p 81:10000 \
    -v $(pwd)/manifests:/etc/envoy \
    -v $(pwd)/manifests/envoy1.yaml:/etc/envoy/envoy.yaml \
    envoyproxy/envoy:latest
```



注意

为了避免和前面的 Envoy 实例端口冲突，这里我都修改了和宿主机上绑定的端口。

启动完成后，同样可以访问 Envoy 代理来测试是否生效了：

```shell
$ curl localhost:81
<h1>This request was processed by host: 4fd8eb3bd415</h1>
$ curl localhost:81
<h1>This request was processed by host: 73b616391920</h1>
```



现在我们基于上面配置的 `CDS`、`LDS`、`EDS` 的配置来动态添加一个新的集群。现在我们添加一个名为 `newTargetCluster` 的集群，内容如下所示：

```yaml
version_info: "0"
resources:
- "@type": "type.googleapis.com/envoy.api.v2.Cluster"
  name: targetCluster
  connect_timeout: 0.25s
  lb_policy: ROUND_ROBIN
  type: EDS
  eds_cluster_config:
    service_name: localservices
    eds_config:
      path: /etc/envoy/eds.yaml
- "@type": "type.googleapis.com/envoy.api.v2.Cluster"
  name: newTargetCluster
  connect_timeout: 0.25s
  lb_policy: ROUND_ROBIN
  type: EDS
  eds_cluster_config:
    service_name: localservices
    eds_config:
      path: /etc/envoy/eds1.yaml
```



上面我们新增了一个新的集群，对应的 `eds_config` 配置文件是 `eds1.yaml`，所以我们同样需要去创建该文件去配置新的端点服务数据，内容如下所示：

```yaml
version_info: "0"
resources: 
- "@type": "type.googleapis.com/envoy.api.v2.ClusterLoadAssignment"
  cluster_name: "localservices"
  endpoints:
  - lb_endpoints:
    - endpoint:
        address:
          socket_address:
            address: "172.17.0.6"
            port_value: 80
    - endpoint:
        address:
          socket_address:
            address: "172.17.0.7"
            port_value: 80
```



这个时候新的集群添加上了，但是还没有任何路由来使用这个新集群，我们可以在 `lds.yaml` 中去配置，将之前配置的 `targetCluster` 替换成 `newTargetCluster`。

当然同样我们这里还需要运行两个简单的 HTTP 服务来作为上游服务提供服务，执行如下所示的命令：

```shell
$ docker run -d cnych/docker-http-server; docker run -d cnych/docker-http-server;
```



上面的配置完成后，我们可以执行如下所示的命令来强制动态配置文件更新：

```shell
$ mv manifests/cds.yaml tmp; mv tmp manifests/cds.yaml; mv manifests/lds.yaml tmp; mv tmp manifests/lds.yaml
```



这个时候 Envoy 应该就会自动重新加载并添加新的集群，我们同样可以执行 `curl localhost:81` 命令来验证：

```shell
$ curl localhost:81
<h1>This request was processed by host: f92b16426da5</h1>
$ curl localhost:81
<h1>This request was processed by host: d89d590082dc</h1>
```



可以看到已经变成了新的两个端点数据了。



# 基于 API 的动态端点发现[¶](https://www.qikqiak.com/envoy-book/api-based-dynamic-route-config/#api)

当在 Envoy 配置中定义了上游集群后，Envoy 需要知道如何解析集群成员，这就是服务发现。端点发现服务（EDS）是 Envoy 基于 gRPC 或者用来获取集群成员的 REST-JSON API 服务的 xDS 管理服务。在本节我们将学习如何使用 REST-JSOn API 来配置端点的自动发现。

## 1. 介绍[¶](https://www.qikqiak.com/envoy-book/api-based-dynamic-route-config/#1)

在前面的章节中，我们使用文件来定义了静态和动态配置，在这里我们将介绍另外一种方式来进行动态配置：API 动态配置。

端点发现服务（EDS）是 Envoy 基于 gRPC 或者用来获取集群成员的 REST-JSON API 服务的 xDS 管理服务，集群成员在 Envoy 术语中成为端点，对于每个集群，Envoy 都从发现服务中获取端点。其中 `EDS` 就是最常用的服务发现机制，因为下面几个原因：

- Envoy 对每个上游主机都有一定的了解（相对于通过 DNS 解析的负载均衡器进行路由），可以做出更加智能的负载均衡策略。
- 发现 API 返回的每个主机的一些属性会将主机的负载均衡权重、金丝雀状态、区域等等告知 Envoy，这个额外的属性在负载均衡、统计数据收集等会被 Envoy 网格在全局中使用到
- Envoy 项目在 [Java](https://github.com/envoyproxy/java-control-plane) 和 [Golang](https://github.com/envoyproxy/go-control-plane) 中都提供了 `EDS` 和其他服务发现的 gRPC 实现参考

接下来我们将更改配置来使用 `EDS`，从而允许基于来自 REST-JSON API 服务的数据进行动态添加节点。

## 2. EDS 配置[¶](https://www.qikqiak.com/envoy-book/api-based-dynamic-route-config/#2-eds)

下面是提供的一个 Envoy 配置的初始配置 `envoy.yaml`，文件内容如下所示：

```yaml
admin:
  access_log_path: /dev/null
  address:
    socket_address:
      address: 127.0.0.1
      port_value: 9000

node:
  cluster: mycluster
  id: test-id

static_resources:
  listeners:
  - name: listener_0
    address:
      socket_address: { address: 0.0.0.0, port_value: 10000 }
    filter_chains:
    - filters:
      - name: envoy.http_connection_manager
        config:
          stat_prefix: ingress_http
          codec_type: AUTO
          route_config:
            name: local_route
            virtual_hosts:
            - name: local_service
              domains: ["*"]
              routes:
              - match: { prefix: "/" }
                route: { cluster: targetCluster }
          http_filters:
          - name: envoy.router
```



接下来需要添加一个 `EDS` 类型的集群配置，并在 `eds_config` 中配置使用 REST API：

```yaml
clusters:
- name: targetCluster
  type: EDS
  connect_timeout: 0.25s
  eds_cluster_config:
    service_name: myservice
    eds_config:
      api_config_source:
        api_type: REST 
        cluster_names: [eds_cluster]
        refresh_delay: 5s
```



然后需要定义 `eds_cluster` 的解析方式，这里我们可以使用静态配置：

```yaml
- name: eds_cluster
  type: STATIC
  connect_timeout: 0.25s
  hosts: [{ socket_address: { address: 172.17.0.4, port_value: 8080 }}]
```



然后同样启动一个 Envoy 代理实例来进行测试：

```shell
$ docker run --name=api-eds -d \
    -p 9901:9901 \
    -p 80:10000 \
    -v $(pwd)/manifests:/etc/envoy \
    envoyproxy/envoy:latest
```



然后启动一个如下所示的上游端点服务：

```shell
$ docker run -p 8081:8081 -d -e EDS_SERVER_PORT='8081' cnych/docker-http-server:v4
```



启动完成后我们可以使用如下命令来测试上游的端点服务：

```shell
$ curl http://localhost:8081 -i
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 36
Server: Werkzeug/0.15.4 Python/2.7.16
Date: Tue, 14 Apr 2020 06:32:56 GMT

355d92db-9295-4a22-8b2c-fc0e5956ecf6
```



现在我们启动了 Envoy 代理和上游的服务集群，但是由于我们这里启动的服务并不是 `eds_cluster` 中配置的服务，所以还没有连接它们。这个时候我们去查看 Envoy 代理得日志，可以看到如下所示的一些错误：

```shell
$ docker logs -f api-eds
[2020-04-14 06:50:07.334][1][warning][config] [source/common/config/http_subscription_impl.cc:110] REST update for /v2/discovery:endpoints failed
......
```



## 3. 启动 EDS[¶](https://www.qikqiak.com/envoy-book/api-based-dynamic-route-config/#3-eds)

为了让 Envoy 获取端点服务，我们需要启动 `eds_cluster`，我们这里将使用 python 实现的一个示例 [eds_server](https://github.com/salrashid123/envoy_discovery)。

使用如下所示的命令来启动 `eds_server` 服务：

```shell
$ docker run -p 8080:8080 -d cnych/eds_server
```



服务启动后，可以在服务日志中查看到如下所示的日志信息，表明一个 Envoy 发现请求成功：

```shell
* Serving Flask app "main" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
 * Running on http://0.0.0.0:8080/ (Press CTRL+C to quit)
 * Restarting with stat
 * Debugger is active!
 * Debugger PIN: 185-412-562
172.17.0.2 - - [14/Apr/2020 07:12:00] "POST /v2/discovery:endpoints HTTP/1.1" 200 -
 Inbound v2 request for discovery.  POST payload: {u'node': {u'user_agent_name': u'envoy', u'cluster': u'mycluster', u'extensions': [{......}], u'user_agent_build_version': {u'version': {u'minor_number': 14, u'major_number': 1, u'patch': 1}, u'metadata': {u'ssl.version': u'BoringSSL', u'build.type': u'RELEASE', u'revision.status': u'Clean', u'revision.sha': u'3504d40f752eb5c20bc2883053547717bcb92fd8'}}, u'build_version': u'3504d40f752eb5c20bc2883053547717bcb92fd8/1.14.1/Clean/RELEASE/BoringSSL', u'id': u'test-id'}, u'type_url': u'type.googleapis.com/envoy.api.v2.ClusterLoadAssignment', u'resource_names': [u'myservice'], u'version_info': u'v1'}
172.17.0.2 - - [14/Apr/2020 07:12:08] "POST /v2/discovery:endpoints HTTP/1.1" 200 -
```



现在我们就可以将上游的服务配置添加到 EDS 服务中去了，这样可以让 Envoy 来自动发现上游服务。

我们在 Envoy 配置中将服务定义为了 `myservice`，所以我们需要针对该服务注册一个端点：

```shell
$ curl -X POST --header 'Content-Type: application/json' --header 'Accept: application/json' -d '{
  "hosts": [
    {
      "ip_address": "172.17.0.3",
      "port": 8081,
      "tags": {
        "az": "cn-beijing-a",
        "canary": false,
        "load_balancing_weight": 50
      }
    }
  ]
}' http://localhost:8080/edsservice/myservice
```



由于我们已经启动了上面注册的上游服务，所以现在我们可以通过 Envoy 代理访问到它了：

```shell
$ curl -i http://localhost
HTTP/1.1 200 OK
content-type: text/html; charset=utf-8
content-length: 36
server: envoy
date: Tue, 14 Apr 2020 07:33:04 GMT
x-envoy-upstream-service-time: 4

355d92db-9295-4a22-8b2c-fc0e5956ecf6
```



接下来我们在上游集群中运行更多的节点，并调用 API 来进行动态注册，使用如下所示的命令来向上游集群再添加4个节点：

```shell
for i in 8082 8083 8084 8085
  do
    docker run -d -e EDS_SERVER_PORT=$i cnych/docker-http-server:v4;
    sleep .5
done
```



然后将上面的4个节点注册到 EDS 服务上面去，同样使用如下所示的 API 接口调用：

```shell
$ curl -X PUT --header 'Content-Type: application/json' --header 'Accept: application/json' -d '{
    "hosts": [
        {
        "ip_address": "172.17.0.3",
        "port": 8081,
        "tags": {
            "az": "cn-beijing-a",
            "canary": false,
            "load_balancing_weight": 50
        }
        },
        {
        "ip_address": "172.17.0.5",
        "port": 8082,
        "tags": {
            "az": "cn-beijing-a",
            "canary": false,
            "load_balancing_weight": 50
        }
        },
        {
        "ip_address": "172.17.0.6",
        "port": 8083,
        "tags": {
            "az": "cn-beijing-a",
            "canary": false,
            "load_balancing_weight": 50
        }
        },
        {
        "ip_address": "172.17.0.7",
        "port": 8084,
        "tags": {
            "az": "cn-beijing-a",
            "canary": false,
            "load_balancing_weight": 50
        }
        },
        {
        "ip_address": "172.17.0.8",
        "port": 8085,
        "tags": {
            "az": "cn-beijing-a",
            "canary": false,
            "load_balancing_weight": 50
        }
        }
    ]
    }' http://localhost:8080/edsservice/myservice
```



注册成功后，我们可以通过如下所示的命令来验证网络请求是否与注册的节点之间是均衡的：

```shell
$ while true; do curl http://localhost; sleep .5; printf '\n'; done
d671262d-39b5-4150-9e25-94fb4f733959
dd1519ef-e03a-4708-bcd1-71890d38e40c
b0c218f0-99f4-43e4-87fc-8989d49fccec
355d92db-9295-4a22-8b2c-fc0e5956ecf6
d671262d-39b5-4150-9e25-94fb4f733959
34690963-0887-4d36-8776-c35cf37fa901
......
```



根据上面的输出结果可以看到每次请求的服务是不同的响应，我们一共注册了5个端点服务。

现在我们来通过 API 删除 EDS 服务上面注册的主机来测试下，执行如下所示的命令清空 `hosts`：

```shell
$ curl -X PUT --header 'Content-Type: application/json' --header 'Accept: application/json' -d '{
  "hosts": []
}' http://localhost:8080/edsservice/myservice
```



现在如果我们尝试向 Envoy 发送请求，我们将会看到如下所示的不健康的日志信息：

```shell
$ curl -v http://localhost
* Rebuilt URL to: http://localhost/
*   Trying ::1...
* TCP_NODELAY set
* Connected to localhost (::1) port 80 (#0)
> GET / HTTP/1.1
> Host: localhost
> User-Agent: curl/7.54.0
> Accept: */*
>
< HTTP/1.1 503 Service Unavailable
< content-length: 19
< content-type: text/plain
< date: Tue, 14 Apr 2020 07:50:06 GMT
< server: envoy
<
* Connection #0 to host localhost left intact
no healthy upstream
```



这是因为我们将端点服务的节点清空了，所以没有服务来接收 Envoy 的代理请求了。

接下来我们再来测试下 Envoy 和 EDS 服务器的连接断掉了会是一种什么样的情况。首先还是将前面的上游服务节点恢复：

```shell
$ curl -X PUT --header 'Content-Type: application/json' --header 'Accept: application/json' -d '{
    "hosts": [
        {
        "ip_address": "172.17.0.3",
        "port": 8081,
        "tags": {
            "az": "cn-beijing-a",
            "canary": false,
            "load_balancing_weight": 50
        }
        },
        {
        "ip_address": "172.17.0.5",
        "port": 8082,
        "tags": {
            "az": "cn-beijing-a",
            "canary": false,
            "load_balancing_weight": 50
        }
        },
        {
        "ip_address": "172.17.0.6",
        "port": 8083,
        "tags": {
            "az": "cn-beijing-a",
            "canary": false,
            "load_balancing_weight": 50
        }
        },
        {
        "ip_address": "172.17.0.7",
        "port": 8084,
        "tags": {
            "az": "cn-beijing-a",
            "canary": false,
            "load_balancing_weight": 50
        }
        },
        {
        "ip_address": "172.17.0.8",
        "port": 8085,
        "tags": {
            "az": "cn-beijing-a",
            "canary": false,
            "load_balancing_weight": 50
        }
        }
    ]
    }' http://localhost:8080/edsservice/myservice
```



然后同样用如下命令来验证节点是否正确响应：

```shell
$ while true; do curl http://localhost; sleep .5; printf '\n'; done
```



然后我们使用如下所示的命令来停止并删除 EDS 服务的容器：

```shell
$ docker ps -a | awk '{ print $1,$2 }' | grep cnych/eds_server  | awk '{print $1 }' | xargs -I {} docker stop {}
$ docker ps -a | awk '{ print $1,$2 }' | grep cnych/eds_server  | awk '{print $1 }' | xargs -I {} docker rm {}
```



这个时候我们可以看到上面的验证命令还是正常的收到相应，这就证明即使 Envoy 和 EDS 服务器断开了链接，也不会影响已经发现的集群节点。

# 健康检查[¶](https://www.qikqiak.com/envoy-book/detect-service-health-with-healthchecks/#_1)

本章节我们将学习如何添加一个健康检查，来检查集群中的服务是否可用于接收流量。启用健康检查后，如果服务崩溃了，则 Envoy 将停止发送流量。

## 1. 代理配置[¶](https://www.qikqiak.com/envoy-book/detect-service-health-with-healthchecks/#1)

首先创建一个 Envoy 配置文件 `envoy.yaml`，配置将任何域名的请求都代理到 `172.17.0.3` 和 `172.17.0.4` 这两个上游服务去。完整的配置如下所示：

```yaml
static_resources:
  listeners:
  - name: listener_0
    address:
      socket_address: { address: 0.0.0.0, port_value: 8080 }
    filter_chains:
    - filters:
      - name: envoy.http_connection_manager
        config:
          codec_type: auto
          stat_prefix: ingress_http
          route_config:
            name: local_route
            virtual_hosts:
            - name: backend
              domains:
                - "*"
              routes:
              - match:
                  prefix: "/"
                route:
                  cluster: targetCluster
          http_filters:
          - name: envoy.router
  clusters:
  - name: targetCluster
    connect_timeout: 0.25s
    type: STRICT_DNS
    dns_lookup_family: V4_ONLY
    lb_policy: ROUND_ROBIN
    hosts: [
      { socket_address: { address: 172.17.0.3, port_value: 80 }},
      { socket_address: { address: 172.17.0.4, port_value: 80 }}
    ]
```



假设目前 `172.17.0.3` 这个上游服务出现了故障，现在的 Envoy 代理还是会继续向该服务转发流量过来的，这样当用户访问服务的时候就会遇到不可用的情况。对于这种情况我们更希望的是 Envoy 能够检测到服务不可用的时候自动将其从节点中移除掉，这其实就可以通过向集群中添加健康检查来完成。

## 2. 添加健康检查[¶](https://www.qikqiak.com/envoy-book/detect-service-health-with-healthchecks/#2)

健康检查可以添加到 Envoy 的集群配置中，如下所示的配置将在定义的每个节点内使用 `/health` 端点来进行健康检查，Envoy 会根据端点返回的 HTTP 状态来确定其是否健康。

```yaml
health_checks:
- timeout: 1s
  interval: 10s
  interval_jitter: 1s
  unhealthy_threshold: 6
  healthy_threshold: 1
  http_health_check:
    path: "/health"
```



这里我们简单对上面配置的健康检查的关键字段进行下说明：

- `interval`：执行一次健康检查的时间间隔
- `unhealthy_threshold`：将主机标记为不健康状态之前需要进行的不健康状态检查数量（相当于就是检测到几次不健康就认为是不健康的）
- `healthy_threshold`：将主机标记为健康状态之前需要进行的健康状态检查数量（相当于就是检测到几次健康就认为是健康的）
- `http_health_check.path`：用于健康检查请求的路径

关于健康检查的更多字段介绍可以查看官方的文档说明：https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/core/health_check.proto

## 3. 启动代理[¶](https://www.qikqiak.com/envoy-book/detect-service-health-with-healthchecks/#3)

添加了健康检查之后，Envoy 将检查集群中定义的每个节点的运行状况。同样使用如下所示的命令启动 Envoy 代理：

```shell
$ docker run -d --name proxy1 -p 80:8080 -v $(pwd)/manifests:/etc/envoy envoyproxy/envoy:latest
```



然后启动两个节点，都处于正常运行状态：

```shell
$ docker run -d cnych/docker-http-server:healthy; docker run -d cnych/docker-http-server:healthy;
```



启动完成后，我们可以向 Envoy 发送请求，正常都可以从上面的两个上游服务中返回正常的请求：

```shell
$ curl localhost -i
HTTP/1.1 200 OK
date: Wed, 15 Apr 2020 04:13:01 GMT
content-length: 63
content-type: text/html; charset=utf-8
x-envoy-upstream-service-time: 0
server: envoy

<h1>A healthy request was processed by host: b6336e79951d</h1>
$ curl localhost -i
HTTP/1.1 200 OK
date: Wed, 15 Apr 2020 04:13:02 GMT
content-length: 63
content-type: text/html; charset=utf-8
x-envoy-upstream-service-time: 0
server: envoy

<h1>A healthy request was processed by host: 9a4c07cc4306</h1>
```



## 4. 测试[¶](https://www.qikqiak.com/envoy-book/detect-service-health-with-healthchecks/#4)

接下来我们来测试下 Envoy 是如何处理不正常的节点的。在一个独立的命令行终端中，启动一个循环来发送请求，可以让我们来观察状态变化：

```shell
$ while true; do curl localhost; sleep .5; done
......
```



然后使用如下命令，我们可以来确定哪个 Docker 容器的 IP 为 `172.17.0.3`，然后将这个节点变成不健康的，然后 Envoy 就会自动将其从负载均衡中移除掉。

```shell
$ docker ps -q | xargs -n 1 docker inspect --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}} {{ .Config.Hostname }}' | sed 's/ \// /'
172.17.0.4 b6336e79951d
172.17.0.3 9a4c07cc4306
172.17.0.2 6928df882c4f
```



要让该一个节点变成不健康的状态，我们可以直接请求 `unhealthy` 的端点：

```shell
$ curl 172.18.0.3/unhealthy
```



这个时候可以看到另外一个终端中循环请求的日志信息中就出现了 `unhealthy` 的相关信息：

```shell
......
<h1>A unhealthy request was processed by host: 9a4c07cc4306</h1>
<h1>A unhealthy request was processed by host: 9a4c07cc4306</h1>
<h1>A healthy request was processed by host: b6336e79951d</h1>
<h1>A unhealthy request was processed by host: 9a4c07cc4306</h1>
......
```



这个时候访问该容器就会返回 500 状态码了：

```shell
$ curl 172.18.0.3 -i
HTTP/1.1 500 Internal Server Error
Date: Wed, 15 Apr 2020 04:23:59 GMT
Content-Length: 65
Content-Type: text/html; charset=utf-8

<h1>A unhealthy request was processed by host: 9a4c07cc4306</h1>
```



在这段时间内，Envoy会将请求发送到健康检查的端点。如果健康检查的端点发生了故障，它将继续向该服务发送流量，直到达到 `unhealthy_threshold` 这么多次不健康的请求，此时，Envoy 将从负载均衡器中将其删除。这个时候可以看到另外一个终端中循环请求的日志信息中就只有一个容器的信息了：

```shell
......
<h1>A healthy request was processed by host: b6336e79951d</h1>
<h1>A healthy request was processed by host: b6336e79951d</h1>
<h1>A healthy request was processed by host: b6336e79951d</h1>
<h1>A healthy request was processed by host: b6336e79951d</h1>
......
```

与此同时，Envoy 还会继续检查健康状态的端点，来查看它是否再次变得可用了，一旦可用，它将又会被添加回到 Envoy 的上游服务器集群中去。

我们可以访问下上面不健康容器的 `healthy` 端点让其变成正常运行状态：

```shell
$ curl 172.17.0.3/healthy
```

我们健康检查的间隔是10s，`healthy_threshold` 阈值是1，所以检测到成功后 Envoy 就会将该容器再次添加回来。这个时候可以看到另外一个终端中循环请求的日志信息中就又出现了两个容器的信息：

```shell
......
<h1>A healthy request was processed by host: 9a4c07cc4306</h1>
<h1>A healthy request was processed by host: 9a4c07cc4306</h1>
<h1>A healthy request was processed by host: b6336e79951d</h1>
<h1>A healthy request was processed by host: b6336e79951d</h1>
......
```

接下来我们再来测试下所有服务均不可用时发生的情况。目前已经有两个运行正常的上游服务器，Envoy 代理会在它们之间进行负载均衡。

和上面方法一样，对两个上游服务访问 `unhealthy` 端点，这样就可以将两个服务变成不健康的状态：

```shell
$ curl 172.18.0.3/unhealthy
$ curl 172.18.0.4/unhealthy
```

现在两个上游服务都已经不健康了，所以当我们请求 Envoy 时，将得到如下所示的信息：

```shell
$ curl localhost -i
HTTP/1.1 500 Internal Server Error
date: Wed, 15 Apr 2020 06:19:01 GMT
content-length: 65
content-type: text/html; charset=utf-8
x-envoy-upstream-service-time: 0
server: envoy

<h1>A unhealthy request was processed by host: b6336e79951d</h1>
```



## 5. 被动健康检查[¶](https://www.qikqiak.com/envoy-book/detect-service-health-with-healthchecks/#5)

和前面的主动健康检查不同，被动健康检查从真实的请求响应来确定端点是否健康。一旦端点被删除后，Envoy 将使用基于超时的方法进行重新插入，使用该方法可以通过配置 interval 将不正常的主机重新添加到集群中去，后续的每次删除都会增加一定的时间间隔，这样的话不健康的端点对用户的流量影响就会尽可能小。

和前面的主动健康检查一样，被动健康检查也需要针对每个集群进行配置。如下所示的配置表示返回3个连续的`5xx`错误时，该配置会将主机删除30s：

```yaml
outlier_detection:
  consecutive_5xx: "3"
  base_ejection_time: "30s"
```



- `consecutive_5xx`：表示上游主机返回一定数量的连续 `5xx` 状态，则将其移除。需要注意的是在这种情况下，`5xx`表示实际的`5xx`响应码值，或者是一个导致 HTTP 路由器返回一个上游的事件行为（比如重置、连接失败等）
- `base_ejection_time`：表示移除主机的基准时间。真实的时间等于基准时间乘以主机移除的次数，默认为 30000ms 或 30s。

当启用被动健康检查过后，Envoy 会根据实际的请求响应来删除主机。同样首先我们先运行两个新的上游节点:

```shell
$ docker run -d cnych/docker-http-server:healthy; docker run -d cnych/docker-http-server:healthy;
```

然后启动一个新的 Envoy 代理，对应的配置文件为 `envoy1.yaml`，内容如下所示：

```yaml
static_resources:
  listeners:
  - name: listener_0
    address:
      socket_address: { address: 0.0.0.0, port_value: 8080 }
    filter_chains:
    - filters:
      - name: envoy.http_connection_manager
        config:
          codec_type: auto
          stat_prefix: ingress_http
          route_config:
            name: local_route
            virtual_hosts:
            - name: backend
              domains:
                - "*"
              routes:
              - match:
                  prefix: "/"
                route:
                  cluster: targetCluster
          http_filters:
          - name: envoy.router
  clusters:
  - name: targetCluster
    connect_timeout: 0.25s
    type: STRICT_DNS
    dns_lookup_family: V4_ONLY
    lb_policy: ROUND_ROBIN
    hosts: [
      { socket_address: { address: 172.17.0.5, port_value: 80 }},
      { socket_address: { address: 172.17.0.6, port_value: 80 }}
    ]
    outlier_detection:
        consecutive_5xx: "3"
        base_ejection_time: "30s"
```



然后执行如下命令启动 Envoy 代理：

```shell
$ docker run -d --name proxy2 -p 81:8080 \
    -v $(pwd)/manifests/envoy1.yaml:/etc/envoy/envoy.yaml \
    envoyproxy/envoy
```

启动完成后，在单独的一个命令行终端中，执行下面的命令来循环发送请求观察状态的变化：

```shell
$ while true; do curl localhost:81; sleep .5; done
```



然后我们将 `172.17.0.5` 这个端点变成不健康的状态：

```shell
$ curl 172.17.0.5/unhealthy
```

该命令会将该端点的所有请求变成 500 错误：

```shell
$ curl 172.17.0.5 -i
HTTP/1.1 500 Internal Server Error
Date: Wed, 15 Apr 2020 06:55:02 GMT
Content-Length: 65
Content-Type: text/html; charset=utf-8

<h1>A unhealthy request was processed by host: 55e0950029b8</h1>
```

然后我们会在循环的终端中看到会收到3个不健康的请求，然后 Envoy 就会将该上游服务给移除掉：

```shell
......
<h1>A healthy request was processed by host: 5749edc61125</h1>
<h1>A healthy request was processed by host: 55e0950029b8</h1>
<h1>A healthy request was processed by host: 5749edc61125</h1>
<h1>A unhealthy request was processed by host: 55e0950029b8</h1>
<h1>A unhealthy request was processed by host: 55e0950029b8</h1>
<h1>A healthy request was processed by host: 5749edc61125</h1>
<h1>A unhealthy request was processed by host: 55e0950029b8</h1>
<h1>A healthy request was processed by host: 5749edc61125</h1>
<h1>A healthy request was processed by host: 5749edc61125</h1>
<h1>A healthy request was processed by host: 5749edc61125</h1>
......
```

然后我们也可以再次将 `172.17.0.5` 标记为健康，执行如下命令即可：

```shell
$ curl 172.17.0.5/healthy
```

然后差不多 30s 过后，我们查看 Envoy 又将该端点添加回来参与负载均衡了：

```shell
......
<h1>A healthy request was processed by host: 5749edc61125</h1>
<h1>A healthy request was processed by host: 5749edc61125</h1>
<h1>A healthy request was processed by host: 5749edc61125</h1>
<h1>A healthy request was processed by host: 55e0950029b8</h1>
<h1>A healthy request was processed by host: 5749edc61125</h1>
<h1>A healthy request was processed by host: 5749edc61125</h1>
<h1>A healthy request was processed by host: 55e0950029b8</h1>
<h1>A healthy request was processed by host: 5749edc61125</h1>
......
```

到这里我们就完成了在 Envoy 中的健康检查相关的配置。

