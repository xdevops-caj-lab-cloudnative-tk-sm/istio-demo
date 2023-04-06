# Test BookInfo application in Istio

## About Bookinfo application

References:
- https://istio.io/latest/docs/examples/bookinfo/

The Bookinfo application is broken into four separate microservices:

- productpage. The productpage microservice calls the details and reviews microservices to populate the page.
- details. The details microservice contains book information.
- reviews. The reviews microservice contains book reviews. It also calls the ratings microservice.
- ratings. The ratings microservice contains book ranking information that accompanies a book review.

There are 3 versions of the reviews microservice:

- Version v1 doesn’t call the ratings service. （看不到星星）
- Version v2 calls the ratings service, and displays each rating as 1 to 5 black stars. （1-5颗黑色星星）
- Version v3 calls the ratings service, and displays each rating as 1 to 5 red stars. （1-5颗红色星星）

![architecture](https://istio.io/latest/docs/examples/bookinfo/withistio.svg)

## Istio Tasks

### 流量管理 - 请求路由

References:
- https://istio.io/latest/docs/tasks/traffic-management/request-routing/

访问 http://$GATEWAY_URL/productpage ，在Book Reviews区域：
- 有时看不到星星 （reviews-v1)
- 有时能看到黑色星星 reviews-v2)
- 有时能看到红色星星 (reviews-v3)

因为默认没有明确配置productpage应该访问哪个reviews版本，所以会随机访问reviews-v1、reviews-v2、reviews-v3。

#### 定义服务的版本

```bash
kubectl apply -f samples/bookinfo/networking/destination-rule-all.yaml
```

定义服务可用的版本（subset）：
- productpage: v1
- details: v1
- reviews: v1, v2, v3
- ratings: v1

检查创建的destinationrule:
```bash
kubectl get destinationrules
```

也可以在Kiali上查看Istio Config。

#### 将流量全部指向v1版本

```bash
kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

这将创建以下几个virtualservices:
- productpage
- reviews
- ratings
- details

且这几个virtualservice只有一个版本（subset)为v1。

检查创建的virtualservices:

```bash
kubectl get virtualservices
```

也可以在Kiali上查看Istio Config。

再次在浏览器上访问productpage，无论刷新多少次，Book Reviews区域都只能看到reviews-v1 （没有星星）。


#### 根据HTTP header分发流量

productpage服务向reviews服务的所有出站HTTP请求添加自定义`end-user` header。

修改reviews的VirtualService，将带有`end-user: jason`的请求路由到reviews-v2，其他请求路由到reviews-v1。

```bash
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml
```

重新打开productpage页面：
- 用jason用户登录，总是看到黑色星星，即路由都请求到了reviews-v2。
- 用其他用户登录或匿名访问，总是看不到星星，即路由都请求到了reviews-v1。


### 流量管理 - 流量切分

#### 按权重比例分发流量

先修改VirtualService，将全部流量都路由到v1版本。

```bash
kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

然后修改reviews的VirtualService，将50%的流量分别分发给reviews-v1和reviews-v3。

```bash
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-50-v3.yaml
```

重新打开productpage页面（需要刷新页面多次直到Envovy生效）：
- 有一半的机会看不到星星（reviews-v1)
- 有一半的机会看到红色星星（reviews-v3)

最后修改reviews的VirtualService，将100%的流量都分发给reviews-v3。

```bash
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-v3.yaml
```

重新打开productpage页面（需要刷新页面多次直到Envovy生效），此时只能看到红色星星（reviews-v3)。


使用该方式可以来实现蓝绿部署，即将流量从旧版本（reviews-v1）逐渐切换到新版本（reviews-v3）。

### 流量管理 - 故障注入

References:
- https://istio.io/latest/docs/tasks/traffic-management/fault-injection/


#### 注入延迟故障

先修改VirtualService，将全部流量都路由到v1版本。

```bash
kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

再修改reviews的VirtualService，将带有`end-user: jason`的请求路由到reviews-v2，其他请求路由到reviews-v1。

```bash
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml
```

修改ratings的VirtualService，对带有`end-user: jason`的请求注入7s延迟：

```bash
kubectl apply -f samples/bookinfo/networking/virtual-service-ratings-test-delay.yaml
```

请注意，reviews:v2 服务对ratings服务的调用有 10 秒的硬编码连接超时。 
即使您引入了7秒的延迟，您仍然希望端到端流程继续而不会出现任何错误。

重新打开productpage页面（需要刷新页面多次直到Envovy生效）：
- 使用jason用户登录，报错：
```bash
Error fetching product reviews!
Sorry, product reviews are currently unavailable for this book.
```

出错原因：
- productpage服务调用reviews服务的超时为3秒加1次重试（一共6秒左右）
- reviews-v2服务调用ratings服务的超时为10秒
- 所以productpage -> reviews-v2 -> ratings的整个调用链路超时为6秒左右 （小于注入的7秒延迟），所以程序出错

解决办法：
- 将reviews服务调用ratings服务的超时减少为2.5秒 （reviews-v3已经实现）
- 将全部路由给reviews-v3

```bash
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-v3.yaml
```

再次测试
- 在Kiali中，打开ratings的VirtualService，将注入的延迟改为2s

重新打开productpage页面，用jason用户登录，页面显示正常，红色星星，表示请求都到了reviews-v3。

打开浏览器的Developer Tools，查看Network，找到productpage请求，查看Timing。
可以看到productpage的请求耗时为2s多一点。（2s为注入的延迟）

退出用户登录，再次查看productpage请求，可以看到productpage的请求耗时为50ms左右。（此时没有注入的延迟）

#### 注入错误故障

先修改VirtualService，将全部流量都路由到v1版本。

```bash
kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

再修改reviews的VirtualService，将带有`end-user: jason`的请求路由到reviews-v2，其他请求路由到reviews-v1。

```bash
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml
```

修改ratings的VirtualService，对带有`end-user: jason`的请求注入HTTP 500错误：

```bash
kubectl apply -f samples/bookinfo/networking/virtual-service-ratings-test-abort.yaml
```

重新打开productpage页面，用jason用户登录，请求都到了reviews-v2，但是没有星星，报错：Ratings service is currently unavailable


退出用户登录，再次查看productpage请求，请求都到了reviews-v1，显示正常，没有星星。

## 流量管理 - 超时

Istio提供了超时配置，可以用来控制服务之间的调用超时时间。
- https://istio.io/latest/docs/tasks/traffic-management/request-timeouts/

当然，你仍然可以在应用程序中配置请求的超时时间。

## 流量管理 - 断路器

Istio提供了断路器配置：
- https://istio.io/latest/docs/tasks/traffic-management/circuit-breaking/

当然，你仍然可以在应用程序使用第三方库（比如Hystrix或Resilience4j）来实现断路器，或者你可以在API网关来实现断路器。

## 可观测性 - 使用Kiali可视化管理服务网格

打开Kiali dashboard：
```bash
# run in a sperate terminal
istioctl dashboard kiali
```

先删除掉之前创建的路由规则：

```bash
kubectl delete -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

测试向Bookinfo应用发送请求：
```bash
hey -n 100 -c 2 "http://$GATEWAY_URL/productpage"
```

Kiali 主要功能：
- 查看Overview
- 查看Graph
  - App graph 将应用程序的所有版本聚合到单个图节点中
  - Versioned app graph 图表类型为应用程序的每个版本显示一个节点，但特定应用程序的所有版本都组合在一起
  - Workload graph 显示服务网格中每个工作负载的节点
- 流量切分
- 验证Istio配置
- 查看和编辑Istio配置

## 可观测性 - 使用Prometheus和Grafana可视化监控指标

References:
- https://istio.io/latest/docs/tasks/observability/metrics/using-istio-dashboard/

打开Grafana dashboard：
```bash
# run in a spearate terminal
istioctl dashboard grafana
```

打开Manage / Dashboards，打开Istio目录，包括以下Dashboard：
- Istio Control Plane Dashboard
- Istio Mesh Dashboard
- Istio Performance Dashboard
- Istio Service Dashboard
- Istio Wasm Extension Dashboard
- Istio Workload Dashboard

打开其中的Istio Mesh Dashboard，这提供了网格的全局视图以及网格中的服务和工作负载。 

测试向Bookinfo应用发送请求：
```bash
hey -n 1000 -c 5 "http://$GATEWAY_URL/productpage"
```

切换到Istio Service Dashboard，这提供了有关服务指标的详细信息，然后是该服务的客户端工作负载（调用此服务的工作负载）和服务工作负载（提供此服务的工作负载）。


重新测试向Bookinfo应用发送请求。

切换到Istio Workload Dashboard，提供了有关每个工作负载的指标的详细信息，然后是该工作负载的入站工作负载（向此工作负载发送请求的工作负载）和出站服务（此工作负载向其发送请求的服务）。


重新测试向Bookinfo应用发送请求。


## 使用Jaeger作分布式追踪

References:
- https://istio.io/latest/docs/tasks/observability/distributed-tracing/jaeger/

打开Jaeger dahboard：
```bash
# run in a spearate terminal
istioctl dashboard jaeger
```

测试向Bookinfo应用发送请求：
```bash
hey -n 1000 -c 5 "http://$GATEWAY_URL/productpage"
```

选择Service为productpage.bookinfo，然后点击Find Traces。

打开最近的一个trace。

trace 由一组 span 组成，其中每个 span 对应一个 Bookinfo 服务，在执行 /productpage 请求或内部 Istio 组件时调用。

这个例子中的服务调用链包括：
- istio-ingressgateway -> productpage  -> details
- istio-ingressgateway -> productpage  -> reviews -> ratings


