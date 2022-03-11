# 在 Istio 中开启内部重定向

## Envoy 内部重定向

Envoy 支持在内部处理 3xx 重定向，捕获可配置的 3xx 重定向响应，合成一个新的请求，将其发送给新路由匹配指定的上游，将重定向的响应作为对原始请求的响应返回。原始请求的 header 和 body 将会发送至新位置。Trailers 尚不支持。

内部重定向可以使用路由配置中的 [internal_redirect_policy](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route_components.proto#envoy-v3-api-field-config-route-v3-routeaction-internal-redirect-policy) 字段来配置。 当重定向处理开启，任何来自上游的 3xx 响应，只要匹配到配置的 [redirect_response_codes](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route_components.proto#envoy-v3-api-field-config-route-v3-internalredirectpolicy-redirect-response-codes) 的响应都将由 Envoy 来处理。

如果 Envoy 内部重定向配置了 303 并且接收到了 303 响应，如果原始请求不是 GET 或者 HEAD，Envoy 将使用没有 body 的 GET 处理重定向。如果原始请求是 GET 或者 HEAD，Envoy 将使用原始的 HTTP Method 处理重定向。更多信息请查看 [RFC 7231 Section 6.4.4](https://datatracker.ietf.org/doc/html/rfc7231#section-6.4.4) 。

要成功地处理重定向，必须通过以下检查：

1. 响应码匹配到配置的 [redirect_response_codes](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route_components.proto#envoy-v3-api-field-config-route-v3-internalredirectpolicy-redirect-response-codes) ，默认是 302， 或者其他的 3xx 状态码（301, 302, 303, 307, 308）。
2. 拥有一个有效的、完全限定的 URL 的 location 头。
3. 该请求必须已被 Envoy 完全处理。
4. 请求必须小于 [per_request_buffer_limit_bytes](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route_components.proto#envoy-v3-api-field-config-route-v3-route-per-request-buffer-limit-bytes) 的限制。
5. [allow_cross_scheme_redirect](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route_components.proto#envoy-v3-api-field-config-route-v3-internalredirectpolicy-allow-cross-scheme-redirect) 是 true（默认是 false）， 或者下游请求的 scheme 和 location 头一致。
6. 给定的下游请求之前处理的内部重定向次数不超过请求或重定向请求命中的路由配置的 [max_internal_redirects](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route_components.proto#envoy-v3-api-field-config-route-v3-internalredirectpolicy-max-internal-redirects) 。
7. 所有 [predicates](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route_components.proto#envoy-v3-api-field-config-route-v3-internalredirectpolicy-predicates) 都接受目标路由。 

任何失败都将导致重定向传递给下游。

由于重定向请求可能会在不同的路由之间传递，重定向链中的任何满足以下条件的路由都将导致重定向被传递给下游。

1. 没有启用内部重定向
2. 或者当重定向链命中的路由的 [max_internal_redirects](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route_components.proto#envoy-v3-api-field-config-route-v3-internalredirectpolicy-max-internal-redirects) 小于等于重定向链的长度。
3. 或者路由被 [predicates](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route_components.proto#envoy-v3-api-field-config-route-v3-internalredirectpolicy-predicates) 拒绝。

[previous_routes](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/internal_redirect/previous_routes/v3/previous_routes_config.proto#envoy-v3-api-msg-extensions-internal-redirect-previous-routes-v3-previousroutesconfig) 和 [allow_listed_routes](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/internal_redirect/allow_listed_routes/v3/allow_listed_routes_config.proto#envoy-v3-api-msg-extensions-internal-redirect-allow-listed-routes-v3-allowlistedroutesconfig) 这两个 predicates 可以创建一个有向无环图 (DAG) 来定义一个过滤器链，具体来说，allow_listed_routes 定义的有向无环图（DAG）中各个节点的边，而 previous_routes 定义了边的“访问”状态，因此如果需要就可以避免循环。

第三个 predicate [safe_cross_scheme](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/internal_redirect/safe_cross_scheme/v3/safe_cross_scheme_config.proto#envoy-v3-api-msg-extensions-internal-redirect-safe-cross-scheme-v3-safecrossschemeconfig) 被用来阻止 HTTP -> HTTPS 的重定向。

一旦重定向通过这些检查，发送到原始上游的请求头将被修改为：

- 将完全限定的原始请求 URL 放到 x-envoy-original-url 头中。
- 使用 Location 头中的值替换 Authority/Host、Scheme、Path 头。

修改后的请求头将选择一个新的路由，通过一个新的过滤器链发送，然后把所有正常的 Envoy 请求都发送到上游进行清理。

请注意，HTTP 连接管理器头清理（例如清除不受信任的标头）仅应用一次。即使原始路由和第二个路由相同，每个路由的头修改也将同时应用于原始路由和第二路由，因此请谨慎配置头修改规则， 以避免重复不必要的请求头值。


一个简单的重定向流如下所示：

1. 客户端发送 GET 请求以获取 http://foo.com/bar
2. 上游 1 发送 302 响应码并携带 “location: http://baz.com/eep”
3. Envoy 被配置为允许原始路由上重定向，并发送新的 GET 请求到上游 2，携带请求头 “x-envoy-original-url: http://foo.com/bar” 获取 http://baz.com/eep
4. Envoy 将 http://baz.com/eep 的响应数据代理到客户端，作为对原始请求的响应。

## 在 Isito 中通过 Envoyfilter 开启内部重定向

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: follow-redirects
  namespace: istio-system
spec:
  workloadSelector:
    labels:
      app: istio-ingressgateway
  configPatches:
  - applyTo: HTTP_ROUTE
    match:
      context: ANY
    patch:
      operation: MERGE
      value:
        route:
          internal_redirect_policy:
            max_internal_redirects: 5
            redirect_response_codes: ["302"]
```

## 测试

开启前

```bash
curl -i '172.16.0.2/redirect-to?url=http://172.16.0.2/status/200'

HTTP/1.1 302 Found
server: istio-envoy
date: Fri, 11 Mar 2022 07:20:38 GMT
content-type: text/html; charset=utf-8
content-length: 0
location: http://172.16.0.2/status/200
access-control-allow-origin: *
access-control-allow-credentials: true
x-envoy-upstream-service-time: 1
```

开启后

```bash
curl -i '172.16.0.2/redirect-to?url=http://172.16.0.2/status/200'

HTTP/1.1 200 OK
server: istio-envoy
date: Fri, 11 Mar 2022 07:21:03 GMT
content-type: text/html; charset=utf-8
access-control-allow-origin: *
access-control-allow-credentials: true
content-length: 0
x-envoy-upstream-service-time: 0
```

注意 location 需返回完整 URL，下面这种情况不会触发内部重定向

```bash
curl -i '172.16.0.2/status/302'

HTTP/1.1 302 Found
server: istio-envoy
date: Fri, 11 Mar 2022 07:30:38 GMT
location: /redirect/1
access-control-allow-origin: *
access-control-allow-credentials: true
content-length: 0
x-envoy-upstream-service-time: 1
```

## 参考资料
* https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/http/http_connection_management#internal-redirects
* https://cloudnative.to/blog/envoy-http-connection-management/
* https://github.com/istio/istio/issues/32673
