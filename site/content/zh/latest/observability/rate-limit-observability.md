---
title: "RateLimit Observability"
---

Envoy Gateway 为 RateLimit 实例提供可观察性。
本指南将会讲述如何配置 RateLimit 的可观察性, 以及 traces 部分。

## 前提条件

按照 [快速开始指南](../quickstart) 中的步骤安装 Envoy Gateway 和 HTTPRoute 示例 manifest。
在进行下一步之前，应能使用 HTTP 查询示例 backend。按照 [Global Rate Limit](../traffic/global-rate-limit) 中的步骤安装 RateLimit。

[OpenTelemetry Collector](https://opentelemetry.io/docs/collector/)提供了与供应商无关的接收、处理和输出 telemetry 数据的方法。

安装 OTel-Collector：

```shell
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update
helm upgrade --install otel-collector open-telemetry/opentelemetry-collector -f https://raw.githubusercontent.com/envoyproxy/gateway/latest/examples/otel-collector/helm-values.yaml -n monitoring --create-namespace --version 0.60.0
```

## Traces

默认情况下，Envoy Gateway 不配置 RateLimit 以向 OpenTelemetry Sink 发送 traces。
你可以在 `EnvoyGateway`CRD 的 `rateLimit.telemetry.tracing` 中配置收集器。

RateLimit 使用 OpenTelemetry Exporter 向收集器输出跟踪。
你可以配置支持 OTLP 协议的收集器，其中包括但不限于 OpenTelemetry Collector、Jaeger、Zipkin 等。

***注意：***

* 默认情况下，Envoy Gateway 会为 RateLimit 配置 `100%` 的采样率，这可能会导致性能问题。
* Envoy Gateway 使用 `BackendObjectReference` 构建 Kubernetes FQDN，作为 RateLimit trace 收集器的目标端点。`BackendObjectReference` "是通过收集器服务配置的。请注意，收集器服务的配置目前不支持使用 `Service.type=ExternalName` 配置收集器服务。

假设 OpenTelemetry 采集器运行在 `observability` 命名空间，它有一个名为 `otel-svc` 的服务，
我们只想对 trace 数据的 `50%` 进行采样，配置如下：

```shell
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: envoy-gateway-config
  namespace: envoy-gateway-system
data:
  envoy-gateway.yaml: |
    apiVersion: gateway.envoyproxy.io/v1alpha1
    kind: EnvoyGateway
    provider:
      type: Kubernetes
    gateway:
      controllerName: gateway.envoyproxy.io/gatewayclass-controller
    rateLimit:
      backend:
        type: Redis
        redis:
          url: redis-service.default.svc.cluster.local:6379
      telemetry:
        tracing:
          sampleRate: 50
          provider:
            url: otel-svc.observability
EOF
```

更新 ConfigMap 后，需要重新启动 envoy-gateway 部署，以使配置生效：

```shell
kubectl rollout restart deployment envoy-gateway -n envoy-gateway-system
```
