# [使用 C# 和 NBomber 对 Kafka 管道进行负载测试](https://antondevtips.com/blog/load-testing-kafka-pipelines-with-charp-and-nbomber)

现代分布式系统依赖于微服务之间的消息驱动通信。在这些系统中，Apache Kafka 通常作为核心系统——每天处理在生产者和消费者之间流动的数百万条消息。

当你的系统增长时，流量也会随之增长。支付、交易、警报或任何基于事件的消息都可能迅速堆积。

如果你的 Kafka 管道未在真实负载下进行适当测试，小问题可能迅速演变为重大的生产系统中断。而这些生产环境中的性能问题可能会让你同时损失金钱和信任。

NBomber 是一个强大的工具，用于在 .NET 应用程序中模拟针对 Kafka 的真实负载。它允许你使用 C# 或 F# 编写的场景来测试生产者、消费者和端到端流程。

NBomber 在设计上是协议无关的。与许多工具不同，它不依赖于特定协议的外部包，这使其足够灵活，可以测试从 HTTP 和 WebSockets 到 gRPC、Kafka、NoSQL 数据库或自定义协议的任何内容。

今天，我将向你展示如何使用 NBomber 测试基于 Kafka 的微服务系统。我们将建立一个欺诈检测管道，并模拟真实流量，以便在问题出现在生产环境之前发现它们。

今天，我们将探讨：

- 为什么对 Kafka 管道进行负载测试很重要
- 欺诈检测 Kafka 管道概述
- 如何使用 NBomber 进行 Kafka 负载测试
- 跨管道的端到端负载测试
- 你应该跟踪的自定义指标

闲话少说，让我们开始吧！

## 为何对 Kafka 管道进行负载测试很重要

Kafka 被设计为可扩展、容错且持久的系统。它能够以低延迟和高吞吐量处理大量数据。然而，这并不意味着 Kafka 能够免受性能问题或故障的影响。

根据你的用例、数据量、数据格式、网络条件、硬件规格、配置设置和代码质量，在使用 Kafka 时可能会遇到各种挑战，例如：

- 消息丢失或重复
- 代理服务器过载或崩溃
- 安全或合规性违规

但 Kafka 并非孤立存在。它通常用于构建 ETL 或流处理管道，其中包含多个服务或工作进程，这些服务或工作进程能够实时发布、消费和处理消息。

对于许多企业而言，Kafka 管道的速度可能至关重要——特别是当它影响到其服务级别协议（SLA）时。在定义服务级别目标（SLO）或服务级别协议（SLA）时，特定流程的延迟需要保持一致，这为高效且足够快速地运行此类管道带来了额外的挑战。

基于 Kafka 的管道在流量较小的开发环境中可能运行顺畅。

但在生产环境中，意外的流量峰值可能会暴露出一些隐藏的问题，例如：

- 并发问题。
- 处理背压的不当设置。
- 消息处理中的 I/O 或序列化瓶颈。
- 分区配置不当导致消费者间负载不均衡。
- 针对预期负载的消费者扩展不当。
- 有状态或无状态消息处理中的并发问题导致的高延迟峰值。
- 一个包含多个 ETL 工作进程的长管道，这导致了较高的端到端延迟
- Kafka 事务的不当使用，这需要协调
- 所有节点的提交确认延迟
- 提交偏移量的策略不当，导致高延迟

使用 NBomber，你可以：

- 测量生产与消费消息之间的端到端延迟。
- 跟踪消息吞吐量在不断增加的负载下的变化情况。
- 检测消费者何时开始落后于生产者。

## 欺诈检测 Kafka 管道概述

在深入进行负载测试之前，让我们仔细了解一下我们将要测试的基于 Kafka 的欺诈检测微服务系统。

我们将测试负责注册支付和执行欺诈检测的核心 Kafka 管道。

管道中涉及 2 个微服务：

- PaymentService - 负责处理和保存支付信息
- FraudDetectionService - 负责欺诈检测和审批

以下是系统的完整事件流程：

![流程图](https://antondevtips.com/media/code_screenshots/architecture/kafka-load-testing-nbomber/img_11.png)

步骤 1：支付创建请求

流程始于向 create-payment Kafka 主题发布 CreatePaymentEvent 时。

步骤 2：PaymentService 接收并存储支付信息

PaymentService 有一个后台消费者正在监听 create-payment 主题。当它接收到事件时，它会做两件事：

1. 在其数据库中创建一条状态为"处理中"的支付记录。
2. 向 payment-registered 主题发布一条 PaymentRegisteredEvent 。

步骤 3：FraudDetectionService 分析支付

FraudDetectionService 订阅 payment-registered 主题。

FraudDetectionService 将支付信息通过其欺诈检测引擎进行处理。该引擎基于多个因素计算风险分数——高风险国家、可疑金额、IP 模式、交易时间和卡 BIN 声誉。

分析支付后，它会做出决定：允许、审核或阻止。然后它将一个 FraudDecisionEvent 发布到 fraud-decision 主题。

步骤 4：PaymentService 接收欺诈决策

PaymentService 有另一个后台消费者正在监听 fraud-decision 主题。

当它收到欺诈决策时，它会更新其数据库中的支付状态：

- "允许"决策 → 支付状态变为"已确认"
- "审核"决策 → 支付状态变为"审核中"
- "阻止"决策 → 支付状态变为"已拒绝"

更新状态后，它会向 payment-processed 主题发布一条最终的 PaymentProcessedEvent 。

以下是系统的完整事件流程：

CreatePaymentEvent → PaymentRegisteredEvent → FraudDecisionEvent → PaymentProcessedEvent

以下是在 Docker 中使用 UI 的 Kafka 设置：

```yaml
services:
  kafka:
    image: apache/kafka:latest
    container_name: kafka
    restart: always
    ports:
      - "9094:9094"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: INTERNAL://kafka:9092,EXTERNAL://0.0.0.0:9094,CONTROLLER://kafka:9093
      KAFKA_ADVERTISED_LISTENERS: INTERNAL://kafka:9092,EXTERNAL://localhost:9094
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: INTERNAL:PLAINTEXT,EXTERNAL:PLAINTEXT,CONTROLLER:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: INTERNAL
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
    volumes:
      - ./kafka-data:/var/lib/kafka/data
    networks:
      - docker-web

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: kafka-ui
    restart: always
    ports:
      - "8080:8080"
    environment:
      - KAFKA_CLUSTERS_0_NAME=local
      - KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS=kafka:9092
    depends_on:
      - kafka
    networks:
      - docker-web

volumes:
  kafka_data:
    driver: local

networks:
  docker-web:
    driver: bridge
```

现在让我们探索如何使用 NBomber 测试这个管道。

## 如何使用 NBomber 进行 Kafka 负载测试

要创建负载测试，你需要定义一个场景。场景代表你需要测试的某种用户行为。

关于对基于 Kafka 的系统进行负载测试，你首先需要了解的是，你测试的不是一个请求-响应 API。

你正在测试一个异步消息流，其中生产者和消费者独立运行。

你不需要将所有内容组合到一个测试场景中：发布消息、等待响应并测量时间。这种方法行不通。

它无法反映你的系统在生产环境中的实际行为，并且会给你带来误导性的性能数据。

你需要同时运行两个独立的场景——一个模拟生产者发布消息，另一个模拟消费者读取消息。这种分离至关重要，因为在生产环境中，你的生产者和消费者并不同步。

生产者会持续发布消息，而不考虑消费者跟上的速度；消费者也会持续轮询，而不考虑生产者发送的速度。

以下是在 NBomber 中如何进行设置：

```csharp
var fraudDetectionScenario = new FraudDetectionScenario();

var publishScenario = fraudDetectionScenario.CreatePublishScenario("localhost:9094");
var consumeScenario = fraudDetectionScenario.CreateConsumeScenario("localhost:9094");

NBomberRunner
    .RegisterScenarios(publishScenario, consumeScenario)
    .Run(args);
```

注意我是如何注册两个场景并一起运行它们的。

NBomber 会并发执行这些场景，这意味着当一组虚拟用户正在发布支付事件时，另一组虚拟用户正在消费已处理的支付事件。这模拟了真实的生产环境流量。

这是 NBomber 文档中推荐的测试事件驱动系统的方法。他们展示了使用 MQTT 的示例，但同样的原则也适用于 Kafka。

有两种负载测试微服务的方法：

- 跨管道的端到端负载测试
- 隔离测试单个微服务

让我们先探讨如何隔离测试 FraudDetectionService 。

## 生产者场景

生产者场景模拟进入 FraudDetectionService 的 payment-registered 主题。

每次迭代会创建一个新的 payment-registered 事件，添加一个用于延迟跟踪的时间戳头，并将其发布到 Kafka：

```csharp
public ScenarioProps CreateProducerScenario(string kafkaBootstrapServers)
{
    var producerConfig = new ProducerConfig
    {
        BootstrapServers = kafkaBootstrapServers,
        Acks = Acks.All,
        EnableIdempotence = true,
        LingerMs = 5,
        CompressionType = CompressionType.Lz4,
        BatchSize = 128 * 1024
    };

    var producer = new ProducerBuilder<string, string>(producerConfig).Build();

    var scenario = Scenario.Create("fraud_detection_publish_scenario", async context =>
    {
        var transactionId = Guid.NewGuid().ToString();
        var timestamp = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds();

        var paymentRegisteredEvent = CreatePaymentRegisteredEvent(transactionId, context);

        var message = CreateKafkaMessage(paymentRegisteredEvent, timestamp, transactionId);

        var deliveryResult = await producer.ProduceAsync("payment-registered", message, context.ScenarioCancellationToken);

        return deliveryResult.Status == PersistenceStatus.Persisted
            ? Response.Ok()
            : Response.Fail(statusCode: "500", message: "Failed to persist message");
    })
    .WithClean(_ =>
    {
        producer?.Dispose();
        return Task.CompletedTask;
    })
    .WithLoadSimulations(
        Simulation.KeepConstant(1, TimeSpan.FromSeconds(30))
    );

    return scenario;
}
```

此场景使用一个虚拟用户运行 30 秒，持续发布支付事件。

这里的关键见解是向 Kafka 消息传递一个 timestamp 头信息。我稍后会解释为什么这很重要。

## 消费者场景

消费者场景模拟等待欺诈决策结果的客户端。它会轮询 fraud-decision 主题，并测量获取欺诈决策结果所需的时间：

```csharp
public ScenarioProps CreateConsumeScenario(string kafkaBootstrapServers)
{
    var consumerConfig = new ConsumerConfig
    {
        BootstrapServers = kafkaBootstrapServers,
        GroupId = "nbomber-fraud-test",
        AutoOffsetReset = AutoOffsetReset.Latest,
        EnableAutoCommit = true,
        EnablePartitionEof = true
    };

    var consumer = new ConsumerBuilder<string, string>(consumerConfig).Build();
    consumer.Subscribe("fraud-decision");

    var scenario = Scenario.Create("fraud_decision_consume_scenario", async context =>
    {
        var consumeResult = consumer.Consume(TimeSpan.FromMilliseconds(100));
        if (consumeResult?.Message is null)
        {
            return Response.Ok(statusCode: "204");
        }

        var fraudDecisionEvent = JsonSerializer.Deserialize<FraudDecisionEvent>(consumeResult.Message.Value);
        if (fraudDecisionEvent == null)
        {
            return Response.Fail(statusCode: "400", message: "Failed to deserialize message");
        }

        var timestampMs = ExtractTimestampFromHeaders(consumeResult);
        if (timestampMs <= 0)
        {
            return Response.Fail(statusCode: "500", payload: fraudDecisionEvent, customLatencyMs: 0);   
        }
            
        var currentTimeMs = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds();
        var latency = currentTimeMs - timestampMs;

        return Response.Ok(payload: fraudDecisionEvent, customLatencyMs: latency);
    })
    .WithClean(_ =>
    {
        consumer.Close();
        consumer.Dispose();
        return Task.CompletedTask;
    })
    .WithLoadSimulations(
        Simulation.KeepConstant(copies: 1, during: TimeSpan.FromSeconds(30))
    );

    return scenario;
}
```

## 使用自定义 Kafka 标头跟踪消息延迟

当我们发布支付事件时，我们会捕获当前时间戳并将其添加到消息头中：

```csharp
private static Message<string, string> CreateKafkaMessage(CreatePaymentEvent paymentRegisteredEvent, long timestamp, string transactionId)
{
    var messageValue = JsonSerializer.Serialize(paymentRegisteredEvent);
    var timestampBytes = JsonSerializer.SerializeToUtf8Bytes(new TimestampContainer(timestamp));
        
    var message = new Message<string, string>
    {
        Key = transactionId,
        Value = messageValue,
        Headers = new Headers
        {
            { "timestamp", timestampBytes }
        }
    };
    return message;
}
```

此标头会传播到整个管道。在消费者端，我们提取时间戳并计算延迟：

```csharp
var consumeResult = consumer.Consume(TimeSpan.FromMilliseconds(100));

var timestampMs = ExtractTimestampFromHeaders(consumeResult);
if (timestampMs <= 0)
{
    return Response.Fail(statusCode: "500", payload: fraudDecisionEvent, customLatencyMs: 0);   
}
    
var currentTimeMs = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds();
var latency = currentTimeMs - timestampMs;

return Response.Ok(payload: paymentProcessedEvent, customLatencyMs: latency);
```

ExtractTimestampFromHeaders 方法很简单：

```csharp
private static long ExtractTimestampFromHeaders(ConsumeResult<string, string> consumeResult)
{
    var timestampHeader = consumeResult.Message.Headers.FirstOrDefault(h => h.Key == "timestamp");
    if (timestampHeader is null)
    {
        return 0;
    }
    
    return JsonSerializer.Deserialize<TimestampContainer>(timestampHeader.GetValueBytes())?.UnixTimeMilliseconds ?? 0;
}
```

我们使用 customLatencyMs 参数将延迟值传递给 NBomber 的 Response。这一点至关重要。

NBomber 的默认延迟测量仅跟踪你的场景步骤需要多长时间——在这种情况下，就是 consumer.Consume() 运行多长时间，通常只有几毫秒。但我们真正关心的是事件流的端到端延迟。

然后 NBomber 会为我们提供关于这种自定义延迟的统计数据：最小值、最大值、平均值和百分位数（p50、p75、p95、p99）。这正是我们理解系统真实性能所需要的。

当我们对 FraudDetection 算法进行更改时，我们可以重新运行负载测试，以确定性能是否发生变化以及是否仍在 SLA（服务级别协议）范围内。

注意：我们为生产者和消费者使用 WithClean 。这很重要，因为它们持有需要适当清理的持久连接。

现在让我们运行测试，看看它们的表现如何：

![测试结果](https://antondevtips.com/media/code_screenshots/architecture/kafka-load-testing-nbomber/img_1.png)

结果显示存在一个严重的性能问题。

FraudDetectionService 在使用 1 个发布者和 2 个消费者的情况下，达到了每秒 35 个请求（RPS）的性能。

虽然吞吐量看起来还算合理，但延迟情况却大不相同：响应时间从 100 毫秒（最小值）到 15 秒（p99）不等。这种广泛的延迟分布表明存在一个关键瓶颈。

问题很明显： FraudDetectionService 无法以足够快的速度消费消息来跟上传入事件的节奏。当生产者发布消息时，它们在 Kafka 中排队堆积的速度超过了服务能够处理它们的速度。这造成了不断增长的积压，队列后面的消息需要等待更长时间才能被处理。

这是基于 Kafka 的系统中的一个常见挑战。当消费者吞吐量落后于生产者吞吐量时，队列中等待的消息的延迟会呈指数级增长。

以下是消费者 BackgroundService 的典型实现：

```csharp
public class PaymentRegisteredConsumerService(IKafkaProducer kafkaProducer)
    : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await Task.Run(() => StartConsumerLoop(stoppingToken), stoppingToken); 
    }
    
    private async Task StartConsumerLoop(CancellationToken cancellationToken)
    {
        var config = new ConsumerConfig { ... };

        using var consumer = new ConsumerBuilder<string, string>(config).Build();
        consumer.Subscribe("payment-registered");

        while (!cancellationToken.IsCancellationRequested)
        {
            await consumer.ConsumeWithInstrumentation(async (result, token) =>
            {
                if (result is not null)
                {
                    await ProcessPaymentRegisteredAsync(result, token);
                }
            }, cancellationToken: cancellationToken);
        }

        consumer.Close();
    }
}
```

在添加更多服务实例之前，我们可以通过运行多个后台工作进程在单个服务内进行水平扩展：

```csharp
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    const int maxNumberOfWorkers = 10;

    var tasks = Enumerable.Range(0, maxNumberOfWorkers)
        .Select(_ => Task.Run(() => StartConsumerLoop(stoppingToken), stoppingToken))
        .ToArray();

    await Task.WhenAll(tasks);
}
```

此更改在同一服务实例中创建了 10 个独立的消费者循环。每个工作线程在自己的线程上运行并并行处理消息。这可以将吞吐量提高多达 10 倍，而无需部署额外的基础设施。

为了最大化多个工作线程的效率，我们需要增加 Kafka 主题中的分区数量。Kafka 会将每个分区分配给消费者组中的一个消费者。

如果我们有 10 个工作线程但只有 1 个分区，那么 9 个工作线程将会闲置。

通过将 payment-registered 主题增加到 10 个分区，我们允许所有 10 个工作线程同时处理消息：

```bash
docker exec -it kafka /opt/kafka/bin/kafka-topics.sh --bootstrap-server kafka:9092 --create --if-not-exists --topic payment-registered --partitions 10 --replication-factor 1
```

让我们重新运行测试，看看它们的表现如何。

优化前：

![优化前](https://antondevtips.com/media/code_screenshots/architecture/kafka-load-testing-nbomber/img_1.png)

优化后：

![优化后](https://antondevtips.com/media/code_screenshots/architecture/kafka-load-testing-nbomber/img_2.png)

在进行了这两项优化后，性能得到了显著提升。延迟最小值为 38 毫秒，p99 值为 123 毫秒。

## 跨管道的端到端负载测试

让我们来探讨如何测试完整的端到端流程。在这里，所有组件协同工作——生产者发布支付请求，整个欺诈检测管道处理这些请求，消费者测量整个流程所需的时间。

支付处理管道由流经多个服务的四个事件组成：

CreatePaymentEvent → PaymentRegisteredEvent → FraudDecisionEvent → PaymentProcessedEvent

以下是事件管道的分布式跟踪：

![分布式跟踪](https://antondevtips.com/media/code_screenshots/architecture/kafka-load-testing-nbomber/img_9.png)

PaymentProcessingScenario 创建了两个独立的测试场景——一个用于生产者，一个用于消费者——它们同时运行：

```csharp
var paymentProcessingScenario = new PaymentProcessingScenario();

var publishScenario = paymentProcessingScenario.CreateProducerScenario("localhost:9094");
var consumeScenario = paymentProcessingScenario.CreateConsumeScenario("localhost:9094");

NBomberRunner
    .RegisterScenarios(publishScenario, consumeScenario)
    .Run(args);
```

生产者场景通过以 1 个虚拟用户的恒定速率发布 create-payment 事件 30 秒来模拟真实的支付流量。这创建了一个持续进入系统的支付流。

消费者场景监听 payment-processed 主题，并测量每笔支付完成整个流程所需的时间：

```csharp
public class PaymentProcessingScenario
{
    private readonly ConcurrentDictionary<string, IConsumer<string, string>> _consumers = new();
    
    public ScenarioProps CreateProducerScenario(string kafkaBootstrapServers) { ... }

    public ScenarioProps CreateConsumeScenario(string kafkaBootstrapServers)
    {
        var consumerConfig = new ConsumerConfig
        {
            BootstrapServers = kafkaBootstrapServers,
            GroupId = "nbomber-load-test",
            AutoOffsetReset = AutoOffsetReset.Latest,
            EnableAutoCommit = true,
            EnablePartitionEof = false
        };
        
        var scenario = Scenario.Create("payment_consume_scenario", async context =>
        {
            var consumer = GetOrAddConsumer(context, consumerConfig);
            
            var consumeResult = consumer.Consume(TimeSpan.FromMilliseconds(100));
            if (consumeResult?.Message is null)
            {
                return Response.Ok(statusCode: "204", customLatencyMs: 0);
            }
    
            var paymentProcessedEvent = JsonSerializer.Deserialize<PaymentProcessedEvent>(consumeResult.Message.Value);
            if (paymentProcessedEvent is null)
            {
                return Response.Fail(statusCode: "400", message: "Failed to deserialize message");
            }
    
            var timestampMs = ExtractTimestampFromHeaders(consumeResult);
            if (timestampMs <= 0)
            {
                return Response.Fail(statusCode: "500", payload: paymentProcessedEvent, customLatencyMs: 0);
            }
    
            var currentTimeMs = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds();
            var latency = currentTimeMs - timestampMs;
    
            context.Logger.Debug("Transaction {TransactionId}: timestamp={Timestamp}, current={Current}, latency={Latency}ms", 
                paymentProcessedEvent.TransactionId, timestampMs, currentTimeMs, latency);
            
            return Response.Ok(payload: paymentProcessedEvent, customLatencyMs: latency);
        })
        .WithClean(_ =>
        {
            foreach (var consumer in _consumers)
            {
                consumer.Value.Close();
                consumer.Value.Dispose();
            }
            
            return Task.CompletedTask;
        })
        .WithLoadSimulations(
            Simulation.KeepConstant(copies: 2, during: TimeSpan.FromSeconds(30))
        );
    
        return scenario;
    }
    
    private IConsumer<string, string> GetOrAddConsumer(IScenarioContext context, ConsumerConfig consumerConfig)
    {
        return _consumers.GetOrAdd(context.ScenarioInfo.InstanceId, _ =>
        {
            var newConsumer = new ConsumerBuilder<string, string>(consumerConfig).Build();
            newConsumer.Subscribe("fraud-decision");

            context.Logger.Debug("Created consumer for thread {ThreadId}", context.ScenarioInfo.InstanceId);

            return newConsumer;
        });
    }
}
```

我们正在运行 2 个并发消费者。

在测试 Kafka 管道时有一个重要的技术细节：根据 Confluent 的文档，Kafka 消费者对象不是线程安全的。这意味着我们不能在多个测试线程之间共享单个消费者实例。

为了解决这个问题，我使用了 GetOrAddConsumer 方法与 ConcurrentDictionary 。这种模式为每个测试线程创建一个唯一的消费者，通过 context.ScenarioInfo.InstanceId 进行标识。

每个虚拟用户都有自己的消费者实例，确保在并发测试期间的线程安全操作。

让我们运行测试并探索结果：

![测试结果](https://antondevtips.com/media/code_screenshots/architecture/kafka-load-testing-nbomber/img_3.png)

初始测试揭示了与孤立欺诈检测场景类似的性能问题。系统难以足够快速地处理支付。

让我们将后台工作线程数量增加到 5：

![测试结果](https://antondevtips.com/media/code_screenshots/architecture/kafka-load-testing-nbomber/img_4.png)

将后台工作线程数量增加到 5 后，性能显著提升。最长迭代时间降至 8 秒——比之前的测试提升了 2 倍。

让我们尝试使用 10 个后台工作线程，并为每个 Kafka 主题配置 10 个分区：

![测试结果](https://antondevtips.com/media/code_screenshots/architecture/kafka-load-testing-nbomber/img_5.png)

结果得到了显著改善。现在最长的迭代仅需 500 毫秒完成——比第一次测试提升了 16 倍。

接下来，让我们在负载测试中增加消费者虚拟用户的数量：

```csharp
.WithLoadSimulations(
    Simulation.KeepConstant(copies: 5, during: TimeSpan.FromSeconds(30))
);
```

![测试结果](https://antondevtips.com/media/code_screenshots/architecture/kafka-load-testing-nbomber/img_6.png)

延迟保持稳定，但吞吐量增加了。我们现在每秒能处理更多支付，且响应时间完全相同。

在对 Kafka 管道进行负载测试时，三个因素必须协同工作才能产生真实的结果：

- 后台工作线程：处理消息的服务实例数量
- Kafka 分区：可用于并行处理的分区数量
- 虚拟消费者：负载测试中模拟的客户端数量

你需要平衡这三个要素。过少的工作线程或分区会造成瓶颈。

在没有足够工作线程的情况下使用过多的虚拟消费者会导致延迟。找到合适的平衡点可以让你获得反映真实生产环境的准确性能数据。

## 你应该跟踪的自定义指标

NBomber 为你提供出色的开箱即用指标 - 吞吐量、延迟百分位数、错误率。

但是当你对欺诈检测管道进行负载测试时，你需要特定领域的指标来告诉你系统内部实际发生的情况。

NBomber 允许你为业务或技术 KPI 创建自定义指标。

两种常见类型是：

- 计数器 – 跟踪运行总数（例如，成功登录的总次数）。
- 仪表盘 – 跟踪最新值（例如，当前内存使用情况）。

### 计数器：跟踪欺诈检测决策

在欺诈检测管道中，最关键的指标是决策的分布情况：有多少付款被允许，有多少被标记为需要审查，以及有多少被拒绝。

以下是我们如何定义计数器：

让我们探讨如何跟踪欺诈检测决策。在消费者场景中，反序列化 payment-processed 事件后，我检查状态：

```csharp
public ScenarioProps CreateConsumeScenario(string kafkaBootstrapServers)
{
    var confirmedCounter = Metric.CreateCounter("payments-confirmed", unitOfMeasure: "payments");
    var reviewingCounter = Metric.CreateCounter("payments-reviewing", unitOfMeasure: "payments");
    var rejectedCounter = Metric.CreateCounter("payments-rejected", unitOfMeasure: "payments");

    // ...

    var scenario = Scenario.Create("payment_consume_scenario", async context =>
    {
        // ...
        
        var paymentProcessedEvent = JsonSerializer.Deserialize<PaymentProcessedEvent>(consumeResult.Message.Value);
        
        switch (paymentProcessedEvent.Status)
        {
            case "Confirmed":
                confirmedCounter.Add(1);
                break;
            case "Reviewing":
                reviewingCounter.Add(1);
                break;
            case "Rejected":
                rejectedCounter.Add(1);
                break;
        }

        // ...
    })
    .WithInit(context =>
    {
        context.RegisterMetric(confirmedCounter);
        context.RegisterMetric(reviewingCounter);
        context.RegisterMetric(rejectedCounter);
        
        WarmUpConsumer(consumer, context);
        return Task.CompletedTask;
    })
    .WithClean(...)
    .WithLoadSimulations(...);

    return scenario;
}
```

关键在于使用 context.RegisterMetric() 在 WithInit 方法中注册这些指标。这会告诉 NBomber 跟踪这些计数器并将它们包含在最终报告中。

### 仪表盘：按决策类型测量处理时间

并非所有支付处理所需的时间都相同。被快速拦截的支付可能比需要复杂欺诈分析的支付延迟更低。我们可以使用仪表来测量这一点：

```csharp
public ScenarioProps CreateConsumeScenario(string kafkaBootstrapServers)
{
    var confirmedLatencyGauge = Metric.CreateGauge("confirmed-latency", unitOfMeasure: "ms");
    var reviewingLatencyGauge = Metric.CreateGauge("reviewing-latency", unitOfMeasure: "ms");
    var rejectedLatencyGauge = Metric.CreateGauge("rejected-latency", unitOfMeasure: "ms");

    var scenario = Scenario.Create("payment_consume_scenario", async context =>
    {
        // ...

        var timestampMs = ExtractTimestampFromHeaders(consumeResult);
                    
        var currentTimeMs = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds();
        var latency = currentTimeMs - timestampMs;

        switch (paymentProcessedEvent.Status)
        {
            case "Confirmed":
                confirmedLatencyGauge.Set(latency);
                break;
            case "Reviewing":
                reviewingLatencyGauge.Set(latency);
                break;
            case "Rejected":
                rejectedLatencyGauge.Set(latency);
                break;
        }

        return Response.Ok(
            payload: paymentProcessedEvent,
            sizeBytes: consumeResult.Message.Value.Length,
            customLatencyMs: latency
        );
    })
    .WithInit(context =>
    {
        context.RegisterMetric(confirmedLatencyGauge);
        context.RegisterMetric(reviewingLatencyGauge);
        context.RegisterMetric(rejectedLatencyGauge);
        
        WarmUpConsumer(consumer, context);
        return Task.CompletedTask;
    })
    .WithClean(...)
    .WithLoadSimulations(...);
}
```

每次我们处理消息时，都会计算端到端延迟，并根据支付状态设置相应的仪表。

![仪表盘](https://antondevtips.com/media/code_screenshots/architecture/kafka-load-testing-nbomber/img_7.png)

### 使用阈值验证性能

NBomber 允许在这些自定义指标上定义阈值，以确保系统满足性能标准。以下是添加阈值来验证欺诈检测行为的方法：

```csharp
var scenario = Scenario.Create("payment_consume_scenario", async context =>
{
    // ...
})
.WithInit(context =>
{
    context.RegisterMetric(confirmedLatencyGauge);
    context.RegisterMetric(reviewingLatencyGauge);
    context.RegisterMetric(rejectedLatencyGauge);
    
    WarmUpConsumer(consumer, context);
    return Task.CompletedTask;
})
.WithThresholds( 
    // Ensure confirmed payment latency stays under 200ms
    Threshold.Create(metric => metric.Gauges.Get("confirmed-latency").Value < 200),
    
    // Ensure rejected payment latency is fast (under 150ms)
    Threshold.Create(metric => metric.Gauges.Get("rejected-latency").Value < 150)
);
```

当我们运行测试时，NBomber 会自动评估这些阈值。如果有任何阈值未通过，测试将被标记为失败，我们可以确切地看到哪个条件未被满足。

![仪表盘](https://antondevtips.com/media/code_screenshots/architecture/kafka-load-testing-nbomber/img_8.png)

## 总结

NBomber 允许你使用纯 C# 或 F# 测试 Kafka 管道，使用与你的微服务相同的代码。

正确测试 Kafka 管道的方法是分别运行生产者和消费者，就像在生产环境中一样。

NBomber 使这一过程变得简单 — 你创建两个场景并同时运行它们。一个发布支付事件，而另一个消费欺诈决策。这会揭示真实的性能问题：缓慢的消费者、错误的分区数量以及负载下的延迟峰值。

你可以跟踪自定义业务指标（如审批率），测量 Kafka 管道的真实端到端延迟，并在生产环境之前发现瓶颈。

使用 NBomber，你可以快速获得可行的洞察。内置的 HTML 报告显示吞吐量、延迟百分位数和错误率。你可以设置阈值，在性能下降时自动使测试失败。

NBomber 供个人使用免费。在组织中使用 NBomber 需要付费许可证，在此处了解更多信息。

NBomber 的定价非常实惠，因为其许可证覆盖整个组织。单个许可证可以在所有团队之间共享，因此无需管理单独的开发者席位——一个许可证即可适用于整个公司。

立即开始使用 NBomber 构建你的负载测试，在问题进入生产环境之前发现它们：

- 我强烈建议从《负载测试微服务》开始。它将为你提供如何通过隔离式和端到端（E2E）负载测试来覆盖你系统的基础知识。
- 之后，你就可以探索他们的演示示例集合了

今天就到这里。希望这些内容对你有帮助。
