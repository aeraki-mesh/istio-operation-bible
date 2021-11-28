在实际项目中，除了同步调用之外，异步消息也是微服务架构中常见的一种通信方式。在本篇文章中，我将继续利用 eshop demo 程序来探讨如何通过 OpenTracing 将 Kafka 异步消息也纳入到 Istio 的分布式调用跟踪中。

# eshop 示例程序结构

如下图所示，demo 程序中增加了发送和接收 Kafka 消息的代码。eshop 微服务在调用 inventory，billing，delivery 服务后，发送了一个 kafka 消息通知，consumer 接收到通知后调用 notification 服务的 REST 接口向用户发送购买成功的邮件通知。
![ 'eshop-demo.jpg'](image/eshop-demo-1.jpg)

# 将 Kafka 消息处理加入调用链跟踪

## 植入 Kafka OpenTracing 代码
首先从 github 下载代码。

```bash
git clone git@github.com:aeraki-framework/method-level-tracing-with-istio.git
```

可以直接使用该代码，但建议跟随下面的步骤查看相关的代码，以了解各个步骤背后的原理。

根目录下分为了 rest-service 和 kafka-consumer 两个目录，rest-service 下包含了各个 REST 服务的代码，kafka-consumer 下是 Kafka 消息消费者的代码。

首先需要将 spring kafka 和 OpenTracing kafka 的依赖加入到两个目录下的 pom 文件中。

```xml
<dependency>
	<groupId>org.springframework.kafka</groupId>
	<artifactId>spring-kafka</artifactId>
</dependency>
 <dependency>
	<groupId>io.opentracing.contrib</groupId>
	<artifactId>opentracing-kafka-client</artifactId>
	<version>${version.opentracing.kafka-client}</version>
</dependency>
```

在 rest-service 目录中的 KafkaConfig.java 中配置消息 Producer 端的 OpenTracing Instrument。TracingProducerInterceptor 会在发送 Kafka 消息时生成发送端的 Span。

```java
@Bean
public ProducerFactory<String, String> producerFactory() {
    Map<String, Object> configProps = new HashMap<>();
    configProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapAddress);
    configProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
    configProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
    configProps.put(ProducerConfig.INTERCEPTOR_CLASSES_CONFIG, TracingProducerInterceptor.class.getName());
    return new DefaultKafkaProducerFactory<>(configProps);
}
```

在 kafka-consumer 目录中的 KafkaConfig.java 中配置消息 Consumer 端的 OpenTracing Instrument。TracingConsumerInterceptor 会在接收到 Kafka 消息是生成接收端的 Span。

```java
@Bean
public ConsumerFactory<String, String> consumerFactory() {
    Map<String, Object> props = new HashMap<>();
    props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapAddress);
    props.put(ConsumerConfig.GROUP_ID_CONFIG, groupId);
    props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
    props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
    props.put(ConsumerConfig.INTERCEPTOR_CLASSES_CONFIG, TracingConsumerInterceptor.class.getName());
    return new DefaultKafkaConsumerFactory<>(props);
}
```
只需要这两步即可完成 Spring 程序的 Kafka OpenTracing 代码植入。下面安装并运行示例程序查看效果。

## 安装 Kafka 集群

示例程序中使用到了 Kafka 消息，因此我们在 TKE 集群中部署一个简单的 Kafka 实例：

```bash
cd method-level-tracing-with-istio
kubectl apply -f k8s/kafka.yaml
```

## 部署 demo 应用

修改 Kubernetes yaml 部署文件 k8s/eshop.yaml，设置 Kafka bootstrap server，以用于 demo 程序连接到 Kafka 集群中。

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: delivery
  ......
    spec:
      containers:
      - name: eshop
        image: aeraki/istio-opentracing-demo:latest
        ports:
        - containerPort: 8080
        env:
          ....
          //在这里加入 Kafka server 地址
          - name: KAFKA_BOOTSTRAP_SERVERS
            value: "kafka-service:9092"

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kafka-consumer
  ......
    spec:
      containers:
      - name: kafka-consumer
        image: aeraki/istio-opentracing-demo-kafka-consumer:latest
        env:
          ....
          //在这里加入 Kafka server 地址
          - name: KAFKA_BOOTSTRAP_SERVERS
            value: "kafka-service:9092"
```

然后部署应用程序，相关的镜像可以直接从 dockerhub 下载，也可以通过源码编译生成。

```bash
kubectl apply -f k8s/eshop.yaml
```

在浏览器中打开地址：http://${INGRESS_EXTERNAL_IP}/checkout ，以触发调用 eshop 示例程序的 REST 接口。然后打开 TCM 的界面查看生成的分布式调用跟踪信息。
![ 'Screen Shot 2021-04-01 at 2.43.06 PM.png'](image/trace-screenshot-5.png)

从图中可以看到，在调用链中增加了两个 Span，分布对应于Kafka消息发送和接收的两个操作。由于Kafka消息的处理是异步的，消息发送端不直接依赖接收端的处理。根据 OpenTracing 对引用关系的定义，From_eshop_topic Span 对 To_eshop_topic Span 的引用关系是 FOLLOWS_FROM 而不是 CHILD_OF 关系。

# 将调用跟踪上下文从Kafka传递到REST服务

现在 eshop 代码中已经加入了 REST 和 Kafka 的 OpenTracing Instrumentation，可以在进行 REST 调用和发送 Kafka 消息时生成调用跟踪信息。但如果需要从 Kafka 的消息消费者的处理方法中调用一个 REST 接口呢？

我们会发现在 eshop 示例程序中，缺省生成的调用链里面并不会把 Kafka 消费者的 Span 和其发起的调用 notification 服务的 REST 请求的 Span 关联在同一个 Trace 中。

要分析导致该问题的原因，我们首先需要了解[“Active Span”](https://opentracing.io/docs/overview/scopes-and-threading/)的概念。在 OpenTracing 中，一个线程可以有一个 Active Span，该 Active Span 代表了目前该线程正在执行的工作。在调用 Tracer.buildSpan() 方法创建新的 Span 时，如果 Tracer 目前存在一个 Active Span，则会将该 Active Span 缺省作为新创建的 Span 的 Parent Span。

Tracer.buildSpan 方法的说明如下：

```java
Tracer.SpanBuilder buildSpan(String operationName)
Return a new SpanBuilder for a Span with the given `operationName`.
You can override the operationName later via BaseSpan.setOperationName(String).

A contrived example:


   Tracer tracer = ...

   // Note: if there is a `tracer.activeSpan()`, it will be used as the target of an implicit CHILD_OF
   // Reference for "workSpan" when `startActive()` is invoked.
   // 如果存在 active span，则其创建的新 Span 会隐式地创建一个 CHILD_OF 引用到该 active span
   try (ActiveSpan workSpan = tracer.buildSpan("DoWork").startActive()) {
       workSpan.setTag("...", "...");
       // etc, etc
   }

   // 也可以通过 asChildOf 方法指定新创建的 Span 的 Parent Span
   // It's also possible to create Spans manually, bypassing the ActiveSpanSource activation.
   Span http = tracer.buildSpan("HandleHTTPRequest")
                     .asChildOf(rpcSpanContext)  // an explicit parent
                     .withTag("user_agent", req.UserAgent)
                     .withTag("lucky_number", 42)
                     .startManual();
```

分析 Kafka OpenTracing Instrumentation 的代码，会发现 TracingConsumerInterceptor 在调用 Kafka 消费者的处理方法之前已经把消费者的 Span 结束了，因此发起 REST 调用时 tracer 没有 active span，不会将 Kafka 消费者的 Span 作为后面 REST 调用的 parent span。

```java
public static <K, V> void buildAndFinishChildSpan(ConsumerRecord<K, V> record, Tracer tracer,
      BiFunction<String, ConsumerRecord, String> consumerSpanNameProvider) {
    SpanContext parentContext = TracingKafkaUtils.extractSpanContext(record.headers(), tracer);

    String consumerOper =
        FROM_PREFIX + record.topic(); // <====== It provides better readability in the UI
    Tracer.SpanBuilder spanBuilder = tracer
        .buildSpan(consumerSpanNameProvider.apply(consumerOper, record))
        .withTag(Tags.SPAN_KIND.getKey(), Tags.SPAN_KIND_CONSUMER);

    if (parentContext != null) {
      spanBuilder.addReference(References.FOLLOWS_FROM, parentContext);
    }

    Span span = spanBuilder.start();
    SpanDecorator.onResponse(record, span);

    //在调用消费者的处理方法之前，该 Span 已经被结束。
    span.finish();

    // Inject created span context into record headers for extraction by client to continue span chain
    //这个 Span 被放到了 Kafka 消息的 header 中
    TracingKafkaUtils.inject(span.context(), record.headers(), tracer);
  }
```

此时 TracingConsumerInterceptor 已经将 Kafka 消费者的 Span 放到了 Kafka 消息的 header 中，因此从 Kafka 消息头中取出该 Span，显示地将 Kafka 消费者的 Span 作为 REST 调用的 Parent Span 即可。

为MessageConsumer.java使用的RestTemplate设置一个TracingKafka2RestTemplateInterceptor。

```java
@KafkaListener(topics = "eshop-topic")
public void receiveMessage(ConsumerRecord<String, String> record) {
    restTemplate
            .setInterceptors(Collections.singletonList(new TracingKafka2RestTemplateInterceptor(record.headers())));
    restTemplate.getForEntity("http://notification:8080/sendEmail", String.class);
}
```

TracingKafka2RestTemplateInterceptor 是基于 Spring OpenTracing Instrumentation 的 TracingRestTemplateInterceptor 修改的，将从 Kafka header 中取出的 Span 设置为出向请求的 Span 的 Parent Span。

```java
@Override
public ClientHttpResponse intercept(HttpRequest httpRequest, byte[] body, ClientHttpRequestExecution xecution)
        throws IOException {
    ClientHttpResponse httpResponse;
    SpanContext parentSpanContext = TracingKafkaUtils.extractSpanContext(headers, tracer);
    Span span = tracer.buildSpan(httpRequest.getMethod().toString()).asChildOf(parentSpanContext)
            .withTag(Tags.SPAN_KIND.getKey(), Tags.SPAN_KIND_CLIENT).start();
    ......
}
```

在浏览器中打开地址：http://${INGRESS_EXTERNAL_IP}/checkout  ，以触发调用 eshop 示例程序的 REST 接口。然后打开 TCM 的界面查看生成的分布式调用跟踪信息。
![ 'WeChatWorkScreenshot_487c2202-4960-48be-b6f6-33fbec457cf8 copy.png'](image/trace-screenshot-5.png)

从上图可以看到，调用链中出现了 Kafka 消费者调用 notification 服务的 sendEmail REST 接口的 Span。从图中可以看到，由于调用链经过了 Kafka 消息，sendEmail Span 的时间没有包含在 checkout Span 中。

# 总结

Istio 服务网格通过分布式调用跟踪来提高微服务应用的可见性，这需要在应用程序中通过 HTTP header 传递调用跟踪的上下文。对于 JAVA 应用程序，我们可以使用 OpenTracing Instrumentation 来代替应用编码传递分布式跟踪的相关 http header，以减少对业务代码的影响；我们还可以将方法级的调用跟踪和 Kafka 消息的调用跟踪加入到 Istio 生成的调用跟踪链中，以为应用程序的故障定位提供更为丰富详细的调用跟踪信息。

# 参考资料

1. [本文中 eshop 示例程序的源代码](https://github.com/aeraki-framework/method-level-tracing-with-istio)

