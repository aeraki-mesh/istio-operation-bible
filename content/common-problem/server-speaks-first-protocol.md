# Server Speaks First 协议访问失败

## 故障现象

Istio 网格开启 allow any 访问模式，在一个注入了 sidecar 的 pod 内，mysql 客户端访问 mysql-ip-1:3306 成功，访问 mysql-ip-2:10000 没有响应：

```
# mysql -h55.135.153.1 -utest -pxxxx -P3306
Welcome to the MariaDB monitor.  Commands end with ; or \g.

# mysql -h55.108.108.2 -utest -pxxxx -P10000
(no response)
```

## 故障分析

查看日志，把 access log 设置为 debug、trace 均没有发现有用信息。

分析发现，网格内有一个 http server，也使用了和 mysql-ip-2 相同的端口 10000：

```
apiVersion: v1
kind: Service
metadata:
  name: irrelevant-svc
......
spec:
  ports:
  - name: http
    nodePort: 31025
    port: 10000     # 端口相同
    protocol: TCP
    targetPort: 8080
```

我们尝试把该服务端口改成 10001，访问 mysql-ip-2:10000 成功，推测和端口冲突相关:

```
# mysql -h55.108.108.2 -utest -pxxxx -P10000
Welcome to the MariaDB monitor.  Commands end with ; or \g.
```

我们再尝试对 mysql-ip-1 复现故障：在网格内创建了一个包括 3306 端口的 http 服务，mysql 请求无响应，问题复现。

另外我们还尝试过，如果把冲突端口的协议定义为 tcp（通过 port name），该问题不存在：

```
apiVersion: v1
kind: Service
metadata:
  name: irrelevant-svc
......
spec:
  ports:
  - name: tcp        # 如果是 tcp 则不会出问题
    nodePort: 31025
    port: 10000
    protocol: TCP
    targetPort: 8080
```

## 故障原因

### Server Speaks First

Mysql 协议是一种 **Server Speaks First** 协议，也就是说 client 和 server 完成三次握手后，是 server 会先发起会话, 简要过程：

```
S: 服务端首先会发一个握手包到客户端
C: 客户端向服务端发送认证信息 ( 用户名，密码等 )
S: 服务端收到认证包后，会检查用户名与密码是否合法，并发送包告知客户端认证信息。
```

除了 Mysql，常见的 Server Speaks First 协议还包括 SMTP，DNS，MongoDB 等。下面是一个 SMTP 交互流程：

```
S: 220 smtp.example.com ESMTP Postfi
C: HELO relay.example.com
S: 250 smtp.example.com, I am glad to meet you
C: MAIL FROM:<bob@example.com>
S: 250 Ok
C: RCPT TO:<alice@example.com>
S: 250 Ok
C: RCPT TO:<theboss@example.com>
S: 250 Ok
C: DATA
S: 354 End data with <CR><LF>.<CR><LF>
C: From: "Bob Example" <bob@example.com>
C: To: Alice Example <alice@example.com>
C: Cc: theboss@example.com
C: Date: Tue, 15 Jan 2008 16:02:43 -0500
C: Subject: Test message
C:
C: Hello Alice.
C: This is a test message with 5 header fields and 4 lines in the message body.
C: Your friend,
C: Bob
C: .
S: 250 Ok: queued as 12345
C: QUIT
S: 221 Bye
{The server closes the connection}
```

### istio 不是完全透明

当前 istio 的某些特性，不能做到**透明**兼容 Server Speaks First 协议，这些特性包括：

* 协议嗅探
* PERMISSIVE mTLS
* Authorization Policy

这些特性都希望 client 能先发起会话，以协议嗅探为例，envoy 是通过分析 client 发出的初始若干字节来推测协议类型。

对于 Server Speaks First 协议，比如 mysql，三次握手后，这时候 mysql client 在等待 mysql server 发起初次会话，而 client 端的 envoy 尝试做协议嗅探，也在等 mysql client 发出数据，这类似一个死锁，最终超时。


## 解决方案

以下是一些可行的方案：

1. 为 Server Speaks First 协议服务创建一个 ServiceEntry，并指定协议为 TCP。
2. 避免 Server Speaks First 协议服务端口和网格内服务端口重叠，这样请求可以直接走 passthrough。
3. 把 Server Speaks First 服务 ip 放到 excludeIPRanges，这样请求不经过 envoy 处理，适用于 DB 服务不需要网格治理的情况。


## 参考资料

* [Server First Protocols](https://istio.io/latest/docs/ops/deployment/requirements/#server-first-protocols)
* [Server-first TCP protocols are not supported](https://istio.io/latest/docs/ops/best-practices/security/#server-first-tcp-protocols-are-not-supported)
* [Istio Envoy passthrough goes wrong when port 80 are used for SMTP protocol instead of standard ports](https://www.linkedin.com/pulse/istio-envoy-passthrough-goes-wrong-when-port-80-used-smtp-liu-)
* [Server-Speaks-First 有点坑](https://www.cnblogs.com/hacker-linner/p/15122404.html)

