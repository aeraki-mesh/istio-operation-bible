# 在 Istio 中指定 HTTP Header 大小写

## 问题背景

Envoy 缺省会把 HTTP Header 的 key 转换为小写，例如有一个 HTTP Header Test-Upper-Case-Header: some-value，经过 Envoy 代理后会变成 test-upper-case-header: some-value。这个在正常情况下没问题，RFC 2616 规范也说明了处理 HTTP Header 应该是大小写不敏感的。

部分场景下，业务请求对某些 Header 字段有大小写要求，此时被 Envoy 转换成为小些会导致请求出现问题。

## 解决方案

Envoy 支持几种不同的 Header 规则：
- 全小写（默认规则）
- 首字母大写

Envoy 1.8 之后新增支持：
- 保留请求原本样式

基于以上能力，为了解决 Header 默认改为小写的问题在 Istio 1.8 及之前可配置成为首字母大写形式，Istio 1.10 及以后可以配置保留 Header 原有样式。

## 配置方法

Istio 1.8 之前可添加如下 EnvoyFilter 配置：
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: http-header-proper-case-words
  namespace: istio-system
spec:
  configPatches:
    - applyTo: CLUSTER
      match:
        context: SIDECAR_OUTBOUND
        cluster:
          # 集群名称可通过 ConfigDump 查询
          name: "outbound|3000||test2.default.svc.cluster.local"
      patch:
        operation: MERGE
        value:
          http_protocol_options:
            header_key_format:
              proper_case_words: {}
```
在需要依赖大写 Header 的服务对应的集群中添加规则，将 Header 全部转为首字母大写的形式。

Istio 1.10 及之后可以添加如下 EnvoyFilter 配置：
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: http-header-proper-case-words
  namespace: istio-system
spec:
  configPatches:
  - applyTo: NETWORK_FILTER
    match:
      listener:
        filterChain:
          filter:
            name: "envoy.http_connection_manager"
    patch:
      operation: MERGE
      value:
        name: "envoy.http_connection_manager"
        typed_config:
          "@type": "type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager"
          http_protocol_options:
            header_key_format:
             stateful_formatter:
               name: preserve_case
               typed_config:
                 "@type": type.googleapis.com/envoy.extensions.http.header_formatters.preserve_case.v3.PreserveCaseFormatterConfig
```
通过此配置可以让 Envoy 保持 Header 原有大小写形式。