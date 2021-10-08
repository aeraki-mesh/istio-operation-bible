# Sidecar 初始化完成后再启动应用程序

## 为什么需要配置 Sidecar 和应用程序的启动顺序？
在安装了 Sidecar Proxy 的 Pod 中，应用发出的外部网络请求会被 Iptables 规则重定向到 Proxy 中。如果应用发出请求时 Proxy 还未初始化完成，则 Proxy 无法对请求进行正确路由，导致请求失败。该问题导致的故障现象参见 [常见问题-应用程序启动失败/启动时无法访问网络](../common-problem/application-start-fail.md)。

## 配置方法 - Istio 1.7 及之后版本
Istio 1.7 及之后的版本中，可以通过下面的方法配置在 Sidecar 初始化完成后再启动应用容器。

全局配置：

在 istio-system/istio ConfigMap 中将 `holdApplicationUntilProxyStarts` 这个全局配置项设置为 true。

```yaml
apiVersion: v1
data:
  mesh: |-
    defaultConfig:
      holdApplicationUntilProxyStarts: true
```

按 Deployment 配置：

如果不希望该配置全局生效，则可以通过下面的 annotation 在 Deployment 级别进行配置。

```yaml
  template:
    metadata:
      annotations:
        proxy.istio.io/config: '{ "holdApplicationUntilProxyStarts": true }'
```

实现原理：在开启 `holdApplicationUntilProxyStarts` 选项后，Istio Sidecar Injector Webhook 会在 Pod 中插入下面的 yaml 片段。该 yaml 片段在 sidecar proxy 的 postStart 生命周期时间中执行了 `pilot-agent wait` 命令。该命令会检测 proxy 的状态，待 proxy 初始化完成后再启动 pod 中的下一个容器。这样，在应用容器启动时，sidecar proxy 已经完成了配置初始化，可以正确代理应用容器的对外网络请求。

```yaml
spec:
  containers:
  - name: istio-proxy
    lifecycle:
      postStart:
        exec:
          command:
          - pilot-agent
          - wait
```

## 配置方法 - Istio 1.7 之前的版本

Istio 1.7 之前的版本没有直接提供配置 Sidecar 和应用容器启动顺序的能力。由于 Istio 新版本中解决了老版本中的很多故障，建议尽量升级到新版本。如果由于特殊原因还要继续使用 Istio 1.7 之前的版本，可以在应用进程启动时判断 Envoy sidecar 的初始化状态，待其初始化完成后再启动应用进程。

Envoy 的健康检查接口 localhost:15020/healthz/ready 会在 xDS 配置初始化完成后才返回 200，否则将返回 503，因此可以根据该接口判断 Envoy 的配置初始化状态，待其完成后再启动应用容器。我们可以在应用容器的启动命令中加入调用 Envoy 健康检查的脚本，如下面的配置片段所示。在其他应用中使用时，将 start-awesome-app-cmd 改为容器中的应用启动命令即可。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: awesome-app-deployment
spec:
  selector:
    matchLabels:
      app: awesome-app
  replicas: 1
  template:
    metadata:
      labels:
        app: awesome-app
    spec:
      containers:
      - name: awesome-app
        image: awesome-app
        ports:
        - containerPort: 80
        command: ["/bin/bash", "-c"]
        args: ["while [[ \"$(curl -s -o /dev/null -w ''%{http_code}'' localhost:15020/healthz/ready)\" != '200' ]]; do echo Waiting for Sidecar;sleep 1; done; echo Sidecar available; start-awesome-app-cmd"]
```

## 解耦应用服务之间的启动依赖关系

以上配置的思路是控制 pod 中容器的启动顺序，在 Envoy sidecar 初始化完成后再启动应用容器，以确保应用容器启动时能够通过网络正常访问其他服务。但即使 pod 中对外的网络访问没有问题，应用容器依赖的其他服务也可能由于尚未启动，或者某些问题而不能在此时正常提供服务。要彻底解决该问题，建议解耦应用服务之间的启动依赖关系，使应用容器的启动不再强依赖其他服务。

在一个微服务系统中，原单体应用中的各个业务模块被拆分为多个独立进程（服务）。这些服务的启动顺序是随机的，并且服务之间通过不可靠的网络进行通信。微服务多进程部署、跨进程网络通信的特定决定了服务之间的调用出现异常是一个常见的情况。为了应对微服务的该特点，微服务的一个基本的设计原则是 “design for failure”，即需要以优雅的方式应对可能出现的各种异常情况。当在微服务进程中不能访问一个依赖的外部服务时，需要通过重试、降级、超时、断路等策略对异常进行容错处理，以尽可能保证系统的正常运行。

Envoy sidecar 初始化期间网络暂时不能访问的情况只是放大了微服务系统未能正确处理服务依赖的问题，即使解决了 Envoy sidecar 的依赖顺序，该问题依然存在。假设应用启动时依赖配置中心，配置中心是一个独立的微服务，当一个依赖配置中心的微服务启动时，配置中心有可能尚未启动，或者尚未初始化完成。在这种情况下，如果在代码中没有对该异常情况进行处理，也会导致依赖配置中心的微服务启动失败。在一个更为复杂的系统中，多个微服务进程之间可能存在网状依赖关系，如果没有按照 “design for failure” 的原则对微服务进行容错处理，那么只是将整个系统启动起来就将是一个巨大的挑战。