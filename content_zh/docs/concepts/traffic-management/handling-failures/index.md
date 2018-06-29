---
title: 故障处理
description: Envoy 故障处理能力简介，这一功能在无需修改应用的情况下，提高了应用的健壮性并防止级联故障的产生。
weight: 30
keywords: [traffic-management,handling-failures]
---

Envoy 提供了一套开箱即用，**可选的**的故障恢复功能，对应用中的服务大有裨益。这些功能包括：

1. 超时
1. 具备超时预算，并能够在重试之间进行可变抖动（间隔）的有限重试功能
1. 并发连接数和上游服务请求数限制
1. 对负载均衡池中的每个成员进行主动（定期）运行健康检查
1. 细粒度熔断器（被动健康检查）- 适用于负载均衡池中的每个实例

这些功能可以使用 [Istio 的流量管理规则](/docs/concepts/traffic-management/rules-configuration/)在运行时进行动态配置。

对超载的上游服务来说，重试之间的抖动极大的降低了重试造成的影响，而超时预算确保调用方服务在可预测的时间范围内获得响应（成功/失败）。

主动和被动健康检查（上述 4 和 5 ）的组合最大限度地减少了在负载平衡池中访问不健康实例的机会。当与平台级健康检查（例如由 Kubernetes 或 Mesos 支持的检查）相结合时，应用程序可以确保将不健康的 Pod/容器/虚拟机 快速地从服务网格中去除，从而最小化请求失败和延迟产生影响。

总之，这些功能使得服务网格能够耐受故障节点，并防止本地故障导致的其他节点的稳定性下降。

## 微调

Istio 的流量管理规则允许运维人员为每个服务/版本设置故障恢复的全局默认值。然而，服务的消费者也可以通过特殊的 HTTP 头提供的请求级别值覆盖[超时](/docs/reference/config/istio.routing.v1alpha1/#HTTPTimeout)和[重试](/docs/reference/config/istio.routing.v1alpha1/#HTTPRetry)的默认值。在 Envoy 代理的实现中，对应的 Header 分别是 `x-envoy-upstream-rq-timeout-ms` 和 `x-envoy-max-retries`。

## FAQ

Q: *在 Istio 中运行的应用程序是否仍需要处理故障？*

是的。Istio可以提高网格中服务的可靠性和可用性。但是，**应用程序仍然需要处理故障（错误）并采取适当的回退操作**。例如，当负载均衡池中的所有实例都失败时，Envoy 将返回 HTTP 503。，应用程序有责任实现必要的逻辑，对这种来自上游服务的 HTTP 503 错误做出合适的响应。

Q: *已经使用容错库（例如 [Hystrix](https://github.com/Netflix/Hystrix)）的应用程序，是否会因为 Envoy 的故障恢复功能受到破坏？*

不会。Envoy对应用程序是完全透明的。在进行服务调用时，由 Envoy 返回的故障响应与上游服务返回的故障响应不会被区分开来。

Q: *同时使用应用级库和 Envoy 时，怎样处理故障？*

假如对同一个目的服务给出两个故障恢复策略（例如，两次超时设置——一个在 Envoy 中设置，另一个在应用程序库中设置），**当故障发生时，两个限制都将被触发**。例如，如果应用程序为服务的 API 调用设置了 5 秒的超时时间，而运维人员配置了 10 秒的超时时间，那么应用程序的超时将会首先启动。同样，如果 Envoy 的熔断器在应用熔断器之前触发，对该服务的 API 调用将从 Envoy 收到 503 错误。