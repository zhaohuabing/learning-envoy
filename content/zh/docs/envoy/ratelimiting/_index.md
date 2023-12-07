---
title: "Rate Limiting"
linkTitle: "Rate Limiting"
weight: 1
description: >
  Rate Limiting
---

为了实现限流配置的最大灵活性，Envoy 并未像路由匹配一样直接采用 HTTP header 来作为限流的匹配条件，而是通过一个 action 来生成 descriptor，以 descriptor 作为限流的匹配条件。因此要理解 Rate Limiting 的运行机制，需要先了解一下这些关键概念。

* descriptor（限流匹配条件）：一组 key/value 键值对，用于匹配 HTTP 请求，匹配上的请求会被进行相应的限流处理。例如请求的源地址，请求的源/目的 cluster，请求 header 等。需要注意的是，descriptor 并不是直接提取的 HTTP header，而是通过 action 配置的规则生成的。
* action（descriptor 生成器）：用于根据 HTTP 请求的属性来生成 descriptor 的配置。例如匹配一组 header 后生成一个指定的 descriptor。
* domain（限流配置分组）：用于对 Rate Limit 配置进行分组，例如不同的应用可以使用不同的 domain。只有在进行全局限流时才需要对来自不同 Listener 的限流请求进行分组，因此 domain 只用于全局限流。

## Global Rate Limiting

如果配置了 Global Rate Limiting Filter，Envoy 在收到一个 HTTP 请求后会向限流服务器发送一个 GRPC 请求。该请求中会带上根据 [Envoy 的 Rate Limit 配置](#envoy-rate-limiting-配置)从 HTTP 请求中提取的 domain 和 descriptor， 该请求结构如下：

```
{
  "domain": ...,
  "descriptors": [],
  "hits_addend": ...
}
```

限流服务器根据自身配置文件对 Envoy 发出的限流请求中的 domain 和 descriptor 进行匹配，然后根据其限流配置判断该请求是否应该被限流，并返回一个 GPRC 响应。

```
  "code": ...,
  "current_limit": {...},
  "limit_remaining": ...,
  "duration_until_reset": {...}
```

响应消息中的 code 代表了限流结果，“OVER_LIMIT” 表示目前符合该条件的请求已经超过了限制，“OK” 表示尚未超过限制。

### Envoy Rate Limiting 配置

在 rate limit filter 中设置 domain 和 Rate Limit Server 的地址。

```yaml
        httpFilters:
        - name: envoy.filters.http.ratelimit
          typedConfig:
            '@type': type.googleapis.com/envoy.extensions.filters.http.ratelimit.v3.RateLimit
            domain: first-listener
            enableXRatelimitHeaders: DRAFT_VERSION_03
            rateLimitService:
              grpcService:
                envoyGrpc:
                  clusterName: ratelimit_cluster
              transportApiVersion: V3
```

在 Route 中提供了 RateLimit 配置，其中的 actions 用于生成前面 GRPC 请求中的 descritptor，也就是用于匹配限流条件的键值对。

备注：Route 中 的 RateLimit 配置是对 Global Rate Limiting 和 Local Rate Limiting 共用的。生成的 descriptor 既可以用于发送到 Rate Limit Server 进行全局限流的判断，也可以直接在 Envoy 的 Local Rate Limit Filter 中用于本地限流的判断。

config.route.v3.RouteAction
```json
{
  "cluster": ...,
  ...
  "rate_limits": [],
}
```

config.route.v3.RateLimit
```json
{
  "stage": {...},
  "disable_key": ...,
  "actions": [],
  "limit": {...}
}
```

config.route.v3.RateLimit.Action
```json
{
  "source_cluster": {...},
  "destination_cluster": {...},
  "request_headers": {...},
  "remote_address": {...},
  "generic_key": {...},
  "header_value_match": {...},
  "dynamic_metadata": {...},
  "metadata": {...},
  "extension": {...},
  "masked_remote_address": {...},
  "query_parameter_value_match": {...}
}
```
Action 是一个 “oneof” 结构，其取值为上面结构中的类型之一，其中较为复杂的几种类型说明如下：

#### request_headers

如果 HTTP 请求中存在该 action，则生成配置中对应的 descriptor_key，则生成一个 descriptor，其 key 为配置中的 descriptor_key，value 则为配置中 header_name 对应的值。

```json
{
  "header_name": ...,
  "descriptor_key": ...,
  "skip_if_absent": ...
}
```

例如下面的配置会在 HTTP 请求中存在一个 x-user-id header 时生成一个 (user, "$header_value") 这个 descriptor。

备注：descriptor 中 “$header_value” 是一个变量，是 HTTP 请求中 x-user-id 这个 header 的值。

```yaml
        - actions:
          - requestHeaders:
              descriptorKey: user
              headerName: x-user-id
```

#### header_value_match

如果 HTTP 请求中的 Header 能够匹配上该 action 中的 headers 匹配条件，则生成一个 descriptor，其 key 和 value 分别为配置中对应的 descriptor_key 和 descriptor_value。

```json
{
  "descriptor_key": ...,
  "descriptor_value": ...,
  "expect_match": {...},
  "headers": []
}
```

例如下面的配置会在 HTTP 请求中存在一个 x-user-id header 并且其值为 one 时，生成 (descriptor-key-foo, descriptor-value-bar) 这个 descriptor。

```yaml
        - actions:
          - headerValueMatch:
              descriptorKey: descriptor-key-foo
              descriptorValue: descriptor-value-bar
              expectMatch: true
              headers:
              - name: x-user-id
                stringMatch:
                  exact: one
              - name: x-org-id
                stringMatch:
                  exact: two
```

#### masked_remote_address

根据请求的 “x-forwarded-for“ header 中的 IP 地址生成一个 descriptor，其 key 为 masked_remote_address，value 是该地址所属的网段。

```json
{
  "v4_prefix_mask_len": {...},
  "v6_prefix_mask_len": {...}
}
```

例如下面的配置会在 HTTP 请求中存在一个 “x-forwarded-for“ header 并且其值为 192.168.1.1 时，生成 (masked_remote_address，192.168.1.0/24) 这个 descriptor。
```yaml
        - actions:
          - maskedRemoteAddress:
              v4PrefixMaskLen: 24
```

#### generic_key

generic_key 用于直接生成一个指定的 （descriptor_key，descriptor_value）descriptor。

{
  "descriptor_key": ...
  "descriptor_value": ...,
}

由于 generic_key 的生成没有其他附加条件，可以用来对一个 route 配置一个全局限流。例如下面的配置，会为所有 test-route 的 HTTP 请求都生成一个（test-foo,test-bar）descriptor。

```yaml
  virtualHosts:
  - domains:
    - '*'
    name: first-listener/*
    routes:
    - match:
        path: test
      name: test-route
      route:
        cluster: test-dest
        rateLimits:
        - actions:
          - genericKey:
              descriptorKey: test-foo
              descriptorValue: test-bar
```

结合下面的 [Rate Limit Server 配置](#rate-limiting-server-配置)，则可以为该 route 设置 500 / sec 的限流。

```yaml
domain: bookstore
descriptors:
  - key: test-foo
    value: test-bar
    rate_limit:
      unit: second
      requests_per_unit: 500
```


其他类型的介绍参见 [Envoy 的相关文档](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route_components.proto#config-route-v3-ratelimit-action)。

### Rate Limiting Server 配置

限流服务器的配置文件结构如下：
```yaml
domain: <unique domain ID>
descriptors:
  - key: <rule key: required>
    value: <rule value: optional>
    rate_limit: (optional block)
      name: (optional)
      replaces: (optional)
       - name: (optional)
      unit: <see below: required>
      requests_per_unit: <see below: required>
    shadow_mode: (optional)
    detailed_metric: (optional)
    descriptors: (optional block)
      - ... (nested repetition of above)
```

下面是一个简单的例子。

```yaml
domain: bookstore
descriptors:
  - key: user
    value: default
    rate_limit:
      unit: second
      requests_per_unit: 500

  - key: user
    value: admin
    rate_limit:
      unit: second
      requests_per_unit: 10
```

该配置会对请求应用如下的限流规则：

```
(user, default): 500 / Sec
(user, admin): 10 / Sec
```

descriptor 支持嵌套，嵌套的 descriptor 之间是 “与” 的关系，可以用于对多个条件进行组合判断。下面是一个嵌套的例子：

```yaml
domain: bookstore
descriptors:
  - key: user
    value: default
    descriptors:
      - key: masked_remote_address
        value: 192.168.0.0/16
        rate_limit:
          requests_per_unit: 5
          unit: second
```

该配置会对请求应用如下的限流规则：

```
(user, default), (masked_remote_address, 192.168.0.0/16): 5 /sec
```

如果不设置 descriptor 的 value ，则每个可能的值会获得一个独立的限流 bucket。例如下面的配置会为 192.168.0.0/16 这个 IP 段内的每个客户端 IP 分配一个 5 / sec 的 bucket。


```yaml
domain: bookstore
descriptors:
  - key: masked_remote_address
    value: 192.168.0.0/24
    descriptors:
      - key: remote_address
        rate_limit:
          requests_per_unit: 5
          unit: second
```

该配置会对请求应用如下的限流规则：

```
(masked_remote_address, 192.168.0.0/24), (remote_address, 192.168.0.1): 5 /sec
(masked_remote_address, 192.168.0.0/24), (remote_address, 192.168.0.2): 5 /sec
(masked_remote_address, 192.168.0.0/24), (remote_address, 192.168.0.3): 5 /sec
...
(masked_remote_address, 192.168.0.0/24), (remote_address, 192.168.0.254): 5 /sec
```

## Local Rate Limiting

Local Rate Limiting 的限流判断是在 Envoy 的 Local Rate Limiting Filter 中进行的，不依赖限流服务器。

其限流配置主要包含 descriptor 生成（action）和限流配置两部分。其中 descriptor 生成的部分和全局限流是一样的，参见 Global Rate Limiting 的[Envoy Rate Limiting 配置](#envoy-rate-limiting-配置) 部分。

限流配置可以在 filter， virtual host，route 三个层次进行。其中 filter， virtual host 两个层次只能配置 listener 或者 VH 级别的一个限流 bucket，不能进行更精确的限流，比较简单。下面主要介绍 route 级别的配置。

Route 的 action 配置中生成 descriptor，然后在 route 的 typed_per_filter_config 中设置 local rate limiting filter 按照 descriptor 进行限流。

```yaml
# 在 Listener 的 HCM Filter Chain 中配置 local ratelimit filter
- name: first-listener
  address:
    socketAddress:
      address: 0.0.0.0
      portValue: 10080
  defaultFilterChain:
    filters:
    - name: envoy.filters.network.http_connection_manager
      typedConfig:
        '@type': type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
        httpFilters:
        - name: envoy.filters.http.local_ratelimit
          typedConfig:
            '@type': type.googleapis.com/envoy.extensions.filters.http.local_ratelimit.v3.LocalRateLimit
            statPrefix: http_local_rate_limiter
        - name: envoy.filters.http.router
          typedConfig:
            '@type': type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
        rds:
          configSource:
            ads: {}
            resourceApiVersion: V3
          routeConfigName: first-listener
...
# 在 Route 上配置 action 和 descriptor
- name: first-listener
  virtualHosts:
  - domains:
    - '*'
    routes:
    - match:
        path: foo/bar
      route:
        cluster: first-route-dest
        rateLimits:
        - actions: # 生成 descriptor，用于限流判断条件
          - headerValueMatch:
              descriptorKey: user
              descriptorValue: foo
              expectMatch: true
              headers:
              - name: x-user-id
                stringMatch:
                  exact: foo
          - headerValueMatch:
              descriptorKey: org
              descriptorValue: bar
              expectMatch: true
              headers:
              - name: x-org-id
                stringMatch:
                  exact: bar
      typedPerFilterConfig:
        envoy.filters.http.local_ratelimit:
          '@type': type.googleapis.com/envoy.extensions.filters.http.local_ratelimit.v3.LocalRateLimit
          descriptors:
          - entries:
            - key: user
              value: one
            - key: org
              value: bar
            tokenBucket:     # 为 (user, one), (org, bar) 的请求应用 10 / 60s 的限流
              fillInterval: 60s
              maxTokens: 10
              tokensPerFill: 10
          filterEnabled:
            defaultValue:
              numerator: 100
          filterEnforced:
            defaultValue:
              numerator: 100
          tokenBucket:      # 为该 route 的所有其他请求应用 100 / 60s 的限流
            fillInterval: 60s
            maxTokens: 100
            tokensPerFill: 100
```


