# 应用程序启动失败/启动时无法访问网络

## 故障现象

该问题的表现是安装了 sidecar proxy 的应用在启动后的一小段时间内无法通过网络访问 pod 外面的服务。应用在启动时通常会从一些外部服务中获取数据，并采用这些数据对自身进行初始化。例如从配置中心读取程序配置，从数据库中初始化程序用户信息等。而安装了 sidecar proxy 的应用在启动后的一小段时间内网络是不通的。如果应用代码中没有合适的容错和重试逻辑，该问题常常会导致应用启动失败。

## 故障原因

如下图所示，Envoy 启动后会通过 xDS 协议向 pilot 请求服务和路由配置信息，Pilot 收到请求后会根据 Envoy 所在的节点（pod或者VM）组装配置信息，包括 Listener、Route、Cluster等，然后再通过 xDS 协议下发给 Envoy。根据 Mesh 的规模和网络情况，该配置下发过程需要数秒到数十秒的时间。在这段时间内，由于初始化容器已经在 pod 中创建了 Iptables rule 规则，因此应用向外发送的网络流量会被重定向到 Envoy ，而此时 Envoy 中尚没有对这些网络请求进行处理的监听器和路由规则，无法对此进行处理，导致网络请求失败。（关于 Envoy sidecar 初始化过程和 Istio 流量管理原理的更多内容，可以参考这篇文章 [Istio流量管理实现机制深度解析](https://zhaohuabing.com/post/2018-09-25-istio-traffic-management-impl-intro/)）。

![](image/envoy-initialize.png)

## 解决方案

参见：[最佳实践-在 Sidecar 初始化完成后再启动应用容器](../best-practice/startup-dependence.md)