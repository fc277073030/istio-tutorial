## Istio 简介
   Istio 是 Google、IBM 和 Lyft 联合开源的服务网格（Service Mesh）框架，旨在解决大量微服务的发现、连接、管理、监控以及安全等问题。Istio 对应用是透明的，不需要改动任何服务代码就可以实现透明的服务治理。
   
Istio 的主要特性包括：
* HTTP、gRPC、WebSocket 和 TCP 网络流量的自动负载均衡
* 细粒度的网络流量行为控制， 包括丰富的路由规则、重试、故障转移和故障注入等
* 可选策略层和配置 API 支持访问控制、速率限制以及配额管理
* 自动度量、日志记录和跟踪所有进出的流量
* 强大的身份认证和授权机制实现服务间的安全通信

## Istio 原理
Istio 从逻辑上可以分为数据平面和控制平面：

* **数据平面** 主要由一系列的智能代理（默认为 Envoy）组成，管理微服务之间的网络通信
* **控制平面** 负责管理和配置代理来路由流量，并配置 Mixer 以进行策略部署和遥测数据收集

Istio 架构如图：
![ac7b55982e28d7bebaa1aadc098c2a6c.png](evernotecid://481E08E3-B0BC-4AF0-BD1D-F5A91D581CF3/appyinxiangcom/15709100/ENResource/p947)

它主要由以下组件构成：

* Envoy：Lyft 开源的高性能代理，用于调解服务网格中所有服务的入站和出站流量。它支持动态服务发现、负载均衡、TLS 终止、HTTP/2 和 gPRC 代理、熔断、健康检查、故障注入和性能测量等丰富的功能。Envoy 以 sidecar 的方式部署在相关的服务的 Pod 中，从而无需重新构建或重写代码。
* Mixer：负责访问控制、执行策略并从 Envoy 代理中收集遥测数据。Mixer 支持灵活的插件模型，方便扩展（支持 GCP、AWS、Prometheus、Heapster 等多种后端）。
* Pilot：动态管理 Envoy 实例的生命周期，提供服务发现、智能路由和弹性流量管理（如超时、重试）等功能。它将流量管理策略转化为 Envoy 数据平面配置，并传播到 sidecar 中。
* Pilot 为 Envoy sidecar 提供服务发现功能，为智能路由（例如 A/B 测试、金丝雀部署等）和弹性（超时、重试、熔断器等）提供流量管理功能。它将控制流量行为的高级路由规则转换为特定于 Envoy 的配置，并在运行时将它们传播到 sidecar。Pilot 将服务发现机制抽象为符合 Envoy 数据平面 API 的标准格式，以便支持在多种环境下运行并保持流量管理的相同操作接口。
* Citadel 通过内置身份和凭证管理提供服务间和最终用户的身份认证。支持基于角色的访问控制、基于服务标识的策略执行等。

![0c8b90bdce4fbdbb9485da7c717fea21.png](evernotecid://481E08E3-B0BC-4AF0-BD1D-F5A91D581CF3/appyinxiangcom/15709100/ENResource/p948)

在数据平面上，除了 Envoy，还可以选择使用 nginxmesh、linkerd 等作为网络代理。比如，使用 nginxmesh 时，Istio 的控制平面（Pilot、Mixer、Auth）保持不变，但用 Nginx Sidecar 取代 Envoy。


## 安装

Istio 的安装部署详见[官方文档](https://istio.io/zh/docs/setup/getting-started/)
当使用 kubectl apply 来部署应用时，如果 pod 启动在标有 istio-injection=enabled 的命名空间中，那么，Istio sidecar 注入器将自动注入 Envoy 容器到应用的 pod 中：
```sh
$ kubectl label namespace <namespace> istio-injection=enabled
```
在没有 istio-injection 标记的命名空间中，在部署前可以使用 istioctl kube-inject 命令将 Envoy 容器手动注入到应用的 pod 中：
```sh
$ istioctl kube-inject -f <your-app-spec>.yaml | kubectl apply -f 
```

## 流量管理
#### VirtualService
示例一：配置请求路由
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:   # 列举虚拟服务的主机，即用户指定的目标或是路由规则设定的目标。
  - reviews
  http:  # 路由规则
  - match:  # 匹配条件
    - headers:
        end-user:
          exact: jason
    route: # 如果 headers 满足匹配则路由到后端服务 reviews 的 v2 版本
    - destination:
        host: reviews
        subset: v2
  - route:   # 其它的都路由到 reviews 的 v3 版本
    - destination:
        host: reviews
        subset: v3
```
示例二：
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
    - bookinfo.com
  http:
  - match:
    - uri:
        prefix: /reviews
    route:
    - destination:
        host: reviews
      weight: 75
  - match:
    - uri:
        prefix: /ratings
    route:
    - destination:
        host: ratings
      weight: 25
```

#### DestinationRule
目标规则示例：
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: my-destination-rule
spec:
  host: my-svc
  trafficPolicy:
    loadBalancer:
      simple: RANDOM
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
  - name: v3
    labels:
      version: v3
```
每个子集都是基于一个或多个 labels 定义的，在 Kubernetes 中它是附加到像 Pod 这种对象上的键/值对。这些标签应用于 Kubernetes 服务的 Deployment 并作为 metadata 来识别不同的版本。

除了定义子集之外，目标规则对于所有子集都有默认的流量策略，而对于该子集，则有特定于子集的策略覆盖它。定义在 subsets 上的默认策略，为 v1 和 v3 子集设置了一个简单的随机负载均衡器。在 v2 策略中，轮询负载均衡器被指定在相应的子集字段上。

#### Gateway 示例
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: ext-host-gwy
spec:
  selector:
    app: my-gateway-controller
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - ext-host.example.com
    tls:
      mode: SIMPLE
      serverCertificate: /tmp/tls.crt
      privateKey: /tmp/tls.key
```
这个网关配置让 HTTPS 流量从 ext-host.example.com 通过 443 端口流入网格，但没有为请求指定任何路由规则,想要为工作的网关指定路由，必须把网关绑定到virtualservice上，如下示例：
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: virtual-svc
spec:
  hosts:
  - ext-host.example.com
  gateways:
    - ext-host-gwy
  http:
  - route:
    - destination:
        host: productpage
        subset: v1
```

#### 服务入口-ServiceEntry
使用服务入口（Service Entry） 来添加一个入口到 Istio 内部维护的服务注册中心。添加了服务入口后，Envoy 代理可以向服务发送流量，就好像它是网格内部的服务一样。配置服务入口允许您管理运行在网格外的服务的流量，它包括以下几种能力：

* 为外部目标 redirect 和转发请求，例如来自 web 端的 API 调用，或者流向遗留老系统的服务。
* 为外部目标定义重试、超时和故障注入策略。
* 添加一个运行在虚拟机的服务来扩展您的网格。
* 从逻辑上添加来自不同集群的服务到网格，在 Kubernetes 上实现一个多集群 Istio 网格。

你可以不需要为网格服务要使用的每个外部服务都添加服务入口。默认情况下，Istio 配置 Envoy 代理将请求传递给未知服务。但是，您不能使用 Istio 的特性来控制没有在网格中注册的目标流量

ServiceEntry 示例
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: svc-entry
spec:
  hosts:
  - ext-svc.example.com
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  location: MESH_EXTERNAL
  resolution: DNS
```

#### Sidecar 
默认情况下，Istio 让每个 Envoy 代理都可以访问来自和它关联的工作负载的所有端口的请求，然后转发到对应的工作负载。您可以使用 sidecar 配置去做下面的事情：

* 微调 Envoy 代理接受的端口和协议集。
* 限制 Envoy 代理可以访问的服务集合。

您可以指定将 sidecar 配置应用于特定命名空间中的所有工作负载，或者使用 workloadSelector 选择特定的工作负载。例如，下面的 sidecar 配置将 bookinfo 命名空间中的所有服务配置为仅能访问运行在相同命名空间和 Istio 控制平面中的服务（目前需要使用 Istio 的策略和遥测功能）：
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Sidecar
metadata:
  name: default
  namespace: bookinfo
spec:
  egress:
  - hosts:
    - "./*"
    - "istio-system/*"
```

#### 超时
超时是 Envoy 代理等待来自给定服务的答复的时间量，以确保服务不会因为等待答复而无限期的挂起，并在可预测的时间范围内调用成功或失败。HTTP 请求的默认超时时间是 15 秒，这意味着如果服务在 15 秒内没有响应，调用将失败。
对于某些应用程序和服务，Istio 的缺省超时可能不合适，Istio 允许您使用虚拟服务按服务轻松地动态调整超时，而不必修改您的业务代码。
超时示例：
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - route:
    - destination:
        host: ratings
        subset: v1
    timeout: 10s
```

#### 重试
重试设置指定如果初始调用失败，Envoy 代理尝试连接服务的最大次数。通过确保调用不会因为临时过载的服务或网络等问题而永久失败，重试可以提高服务可用性和应用程序的性能。重试之间的间隔（25ms+）是可变的，并由 Istio 自动确定，从而防止被调用服务被请求淹没。默认情况下，在第一次失败后，Envoy 代理不会重新尝试连接服务。
与超时一样，Istio 默认的重试行为在延迟方面可能不适合您的应用程序需求（对失败的服务进行过多的重试会降低速度）或可用性。您可以在虚拟服务中按服务调整重试设置，而不必修改业务代码。
下面的示例配置了在初始调用失败后最多重试 3 次来连接到服务子集，每个重试都有 2 秒的超时。
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - route:
    - destination:
        host: ratings
        subset: v1
    retries:
      attempts: 3   # 最多重试3次
      perTryTimeout: 2s  #每次重试有2秒的超时
```

#### 熔断器
熔断器是 Istio 为创建具有弹性的微服务应用提供的另一个有用的机制。在熔断器中，设置一个对服务中的单个主机调用的限制，例如并发连接的数量或对该主机调用失败的次数。一旦限制被触发，熔断器就会“跳闸”并停止连接到该主机。使用熔断模式可以快速失败而不必让客户端尝试连接到过载或有故障的主机。

示例
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: forecast-dr
spec:
  host: forecast
  subsets:
    - labels:
        version: v1
      name: v1
    - labels:
        version: v2
      name: v2
  trafficPolicy:
    connectionPool:  # 连接池
      tcp:
        maxConnections: 3   #最大连接数
      http:
        http1MaxPendingRequests: 5  #最大待处理请求
        maxRequestsPerConnection: 1   # 到后端的每个连接的最大请求数。将此参数设置为1将禁用保持活动状态。默认值0，表示“无限制”，最大2 ^ 29。
    outlierDetection:
      consecutiveErrors: 2  # 连续返回错误次数2次
      interval: 10s   # 每10秒扫描一次
      baseEjectionTime: 2m  #移出连接池2分钟
      maxEjectionPercent: 100   #最大移出实例比例
```

* connectionPool：表示如果对forecast服务发起超过3个 HTTP/1.1 的连接，并且存在5个及以上的待处理请求，就会触发熔断机制 (maxConnections：到目标主机的HTTP1/TCP最大连接数量，只作用于http1.1，不作用于http2，因为后者只建立一次连接)
* outlierDetection：表示每10秒扫描一次forecast服务的后端实例，在连续返回两次网关错误的实例中有40%（本例是两个）会被移出连接池两分钟

#### 故障注入
在配置了网络，包括故障恢复策略之后，可以使用 Istio 的故障注入机制来为整个应用程序测试故障恢复能力。故障注入是一种将错误引入系统以确保系统能够承受并从错误条件中恢复的测试方法。使用故障注入特别有用，能确保故障恢复策略不至于不兼容或者太严格，这会导致关键服务不可用。

与其他错误注入机制（如延迟数据包或在网络层杀掉 Pod）不同，Istio 允许在应用层注入错误。这使您可以注入更多相关的故障，例如 HTTP 错误码，以获得更多相关的结果。

您可以注入两种故障，它们都使用虚拟服务配置：

* 延迟：延迟是时间故障。它们模拟增加的网络延迟或一个超载的上游服务。
* 终止：终止是崩溃失败。他们模仿上游服务的失败。终止通常以 HTTP 错误码或 TCP 连接失败的形式出现。
例如，下面的虚拟服务为千分之一的访问 ratings 服务的请求配置了一个 5 秒的延迟：
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - fault:
      delay:
        percentage:
          value: 0.1
        fixedDelay: 5s
    route:
    - destination:
        host: ratings
        subset: v1
```