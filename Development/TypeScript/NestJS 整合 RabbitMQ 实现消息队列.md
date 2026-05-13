---
title: "NestJS 整合 RabbitMQ 实现消息队列"
date: 2026-05-13
author: Ryan Zeng
tags:
  - RabbitMQ
  - NestJS
  - 消息队列
  - TypeScript
categories:
  - Development
draft: false
---

## 什么是 RabbitMQ

在分布式系统中，服务之间如何可靠地传递数据是一个核心问题。当系统从单体架构演进到微服务架构时，直接的服务间调用（HTTP/RPC）会遇到一系列挑战：耦合度过高、无法应对流量突增、调用失败导致连锁崩溃。消息队列（Message Queue）正是为了解决这些问题而诞生的中间件。

消息队列的核心思想很简单——**生产者不直接把消息发给消费者，而是发给一个中间代理（Broker），由 Broker 负责存储和转发**。这种间接带来的好处是：生产者和消费者互不感知对方的存在，它们只需要和 Broker 打交道。

### 同步调用 vs 异步消息

考虑一个用户注册的场景：

- **同步调用**：用户注册服务在保存数据后，直接调用邮件服务发送欢迎邮件、调用积分服务发放新人积分。如果邮件服务宕机，注册流程也会失败。调用链越长，整体可用性越低。
- **异步消息**：用户注册服务将「用户已注册」这个事件发送到消息队列，立即返回成功。邮件服务和积分服务各自从队列中消费消息，互不影响。即使邮件服务暂时不可用，消息也会留在队列中等待后续处理。

```mermaid
graph LR
    subgraph 同步调用
        A1[注册服务] -->|HTTP| B1[邮件服务]
        A1 -->|HTTP| C1[积分服务]
    end

    subgraph 异步消息
        A2[注册服务] -->|发布消息| D2[RabbitMQ]
        D2 -->|消费| B2[邮件服务]
        D2 -->|消费| C2[积分服务]
    end
```

异步消息带来的关键优势：

- **解耦**：生产者不需要知道消费者的存在，新增消费者无需修改生产者代码。
- **削峰**：当瞬时流量远超系统处理能力时，消息先堆积在队列中，消费者按自己的节奏消费。
- **容错**：消费者暂时不可用时，消息不会丢失，恢复后继续处理。

### 为什么是 RabbitMQ

RabbitMQ 是一个开源的消息代理软件，最初由 Rabbit Technologies Ltd 开发，目前由 VMware（原 Pivotal）维护。它实现了 AMQP（Advanced Message Queuing Protocol）协议，使用 Erlang 语言编写，天生具备高并发和容错能力。

选择 RabbitMQ 的理由：

- **协议标准化**：基于 AMQP 0-9-1 开放标准，任何语言的 AMQP 客户端都可以对接，不存在厂商锁定。
- **路由灵活**：支持 Direct、Fanout、Topic、Headers 四种交换器类型，可以覆盖从简单到复杂的路由需求。
- **可靠性高**：支持消息确认（ACK）、消息持久化、发布确认等机制，确保消息不丢失。
- **管理界面**：内置 Web 管理插件，可以直观地查看队列状态、连接数、消息吞吐量。
- **社区成熟**：2007 年开源，被广泛应用于生产环境，文档和社区支持完善。

当然，RabbitMQ 也有局限：它的吞吐量（万级/秒）不如 Kafka（百万级/秒），不适合大数据日志采集场景；消息堆积过多时性能会下降；Erlang 的学习曲线对排查问题有一定门槛。

> 参考：[RabbitMQ 官方文档](https://www.rabbitmq.com/docs)，[AMQP 0-9-1 规范](https://www.rabbitmq.com/resources/specs/amqp0-9-1.pdf)

### 消息队列横向对比

选择消息队列时，RabbitMQ 不是唯一选项。以下是主流消息队列的核心差异：

| 维度 | RabbitMQ | Kafka | Redis Pub/Sub | NATS |
|------|----------|-------|---------------|------|
| **定位** | 通用消息代理 | 分布式事件流平台 | 轻量级发布-订阅 | 高性能消息系统 |
| **协议** | AMQP 0-9-1 | 自定义 TCP 协议 | Redis 协议 | NATS 协议 |
| **吞吐量** | 万级/秒 | 百万级/秒 | 十万级/秒 | 百万级/秒 |
| **消息持久化** | 支持（磁盘+可选镜像队列） | 支持（追加日志） | 不支持（纯内存） | 可选（JetStream） |
| **消息顺序** | 单队列有序 | 分区内有序 | 无保证 | 流内有序 |
| **路由灵活度** | 高（4 种交换器类型） | 低（基于 Topic 分区） | 低（频道匹配） | 中（Subject 通配符） |
| **消息回溯** | 不支持 | 支持（按偏移量重放） | 不支持 | JetStream 支持 |
| **适用场景** | 业务消息、任务队列、RPC | 日志采集、事件流、大数据 | 实时通知、缓存失效广播 | 微服务通信、IoT、请求-回复 |

**RabbitMQ** 的核心优势在于路由灵活性和消息可靠性保证。它适合需要复杂路由规则（如根据消息头部路由、多级分发）和严格消息确认机制的业务场景。典型用例：订单处理、支付通知、异步任务分发。

**Kafka** 的核心优势在于吞吐量和消息回溯能力。它将消息追加到持久化日志中，消费者可以按偏移量重放任意时间段的消息。适合需要处理海量事件流、做实时数据分析的场景。典型用例：日志采集、用户行为追踪、流处理。

**Redis Pub/Sub** 是最轻量的选择，但消息不持久化——如果消费者离线，消息就丢了。它适合对可靠性要求不高但需要极低延迟的场景，如实时配置推送、WebSocket 消息广播。

**NATS** 追求极致性能和极低延迟，核心服务器用 Go 编写，资源占用极小。JetStream 模块提供了持久化和流处理能力。适合微服务间的高频通信、IoT 设备消息分发等对延迟敏感的场景。

选择建议：需要复杂路由和消息确认 → RabbitMQ；需要高吞吐和消息回溯 → Kafka；需要极简实时通知 → Redis Pub/Sub；需要极低延迟的微服务通信 → NATS。

> 参考：[RabbitMQ vs Kafka 比较](https://www.rabbitmq.com/blog/2024/08/14/rabbitmq-vs-kafka)，[NATS 文档](https://nats.io/docs/)

## RabbitMQ 核心概念

理解 RabbitMQ 的关键在于理解 AMQP 协议定义的消息模型。这个模型由几个核心组件构成，它们之间的关系决定了消息如何流动。

### 整体架构

```mermaid
graph TB
    P[生产者 Producer] -->|发送消息| E[交换器 Exchange]
    E -->|通过 Binding 路由| Q1[队列 Queue 1]
    E -->|通过 Binding 路由| Q2[队列 Queue 2]
    Q1 -->|推送消息| C1[消费者 Consumer 1]
    Q2 -->|推送消息| C2[消费者 Consumer 2]
    C1 -->|ACK 确认| Q1
    C2 -->|ACK 确认| Q2
```

一条消息从生产到消费的完整路径：**Producer → Exchange → Binding → Queue → Consumer**。下面逐个拆解这些组件。

### 交换器（Exchange）

交换器是消息进入 RabbitMQ 的第一站。生产者不会直接把消息发给队列，而是发给交换器，由交换器根据路由规则决定消息应该去往哪些队列。这种间接性正是 RabbitMQ 路由灵活性的来源。

RabbitMQ 定义了四种交换器类型：

**Direct Exchange**：精确匹配 Routing Key。当消息的 Routing Key 与某个 Binding 的 Routing Key 完全一致时，消息被路由到对应的队列。这是最简单的路由模式，适用于点对点通信。

**Fanout Exchange**：广播模式，无视 Routing Key。交换器将收到的消息发送到所有绑定的队列。适用于需要通知多个服务的场景，例如配置变更广播。

**Topic Exchange**：模式匹配 Routing Key。Routing Key 使用 `.` 分隔的多个单词（如 `order.created.email`），Binding 的 Routing Key 可以包含通配符 `*`（匹配一个单词）和 `#`（匹配零个或多个单词）。这是最灵活的路由模式，适用于复杂的发布-订阅场景。

**Headers Exchange**：基于消息 Header 属性路由，不使用 Routing Key。通过消息头中的键值对匹配，可以指定 `x-match: all`（全部匹配）或 `x-match: any`（任意匹配）。性能不如其他类型，不常用。

```mermaid
graph TB
    subgraph Direct
        DE[Direct Exchange] -->|routingKey=order| DQ1[Queue A]
        DE -->|routingKey=user| DQ2[Queue B]
    end
    subgraph Fanout
        FE[Fanout Exchange] -->|全部消息| FQ1[Queue A]
        FE -->|全部消息| FQ2[Queue B]
        FE -->|全部消息| FQ3[Queue C]
    end
    subgraph Topic
        TE[Topic Exchange] -->|order.* 匹配| TQ1[Queue A]
        TE -->|*.created 匹配| TQ2[Queue B]
    end
```

### 队列（Queue）

队列是 RabbitMQ 中存储消息的缓冲区。消息进入队列后，等待消费者取出。队列具有以下关键属性：

- **Durable（持久化）**：设置为 `true` 时，队列的元数据会在 Broker 重启后保留。注意这只是队列本身持久化，消息持久化需要单独设置 `persistent` 属性。
- **Exclusive（排他）**：仅被一个连接使用，连接关闭时队列自动删除。
- **Auto-delete（自动删除）**：最后一个消费者断开连接后，队列自动删除。

### 绑定（Binding）

绑定是连接交换器和队列的规则。一个绑定包含三个要素：目标队列、源交换器、Routing Key。Binding 本身不是组件，而是交换器和队列之间的"连线"。一个队列可以绑定到多个交换器，一个交换器也可以绑定多个队列。

### Routing Key

Routing Key 是生产者发送消息时附带的一个字符串标签，交换器用它来决定消息的路由。在 Direct Exchange 中做精确匹配，在 Topic Exchange 中做模式匹配，在 Fanout Exchange 中被忽略。

### 通道（Channel）与连接（Connection）

Connection 是应用程序与 RabbitMQ Broker 之间的 TCP 连接。每个 Connection 可以创建多个 Channel，消息的发送和接收都在 Channel 上进行。

为什么不直接在 Connection 上操作？因为频繁创建和销毁 TCP 连接开销很大，而 Channel 是轻量级的逻辑连接，可以在同一个 TCP 连接上复用。一个 Connection 上可以同时存在数百个 Channel，每个 Channel 有自己独立的确认机制和 QoS 设置。这相当于一个物理连接被虚拟化为多个逻辑通道，各自独立工作。

### Virtual Host

Virtual Host（vhost）是 RabbitMQ 中的逻辑隔离单元。每个 vhost 拥有独立的交换器、队列、绑定和权限系统，不同 vhost 之间的资源完全隔离。这允许在一台 RabbitMQ 服务器上为多个应用提供隔离的消息环境，而不需要部署多个实例。

### 消息确认（Message Acknowledgment）

RabbitMQ 默认在消费者收到消息后立即将消息从队列中删除。但这种方式可能丢失消息——如果消费者在处理过程中崩溃，消息已经丢了。为了解决这个问题，RabbitMQ 提供了消息确认机制：

- **自动确认（autoAck: true）**：消费者收到消息即确认，RabbitMQ 立即删除消息。性能好但不可靠。
- **手动确认（autoAck: false）**：消费者处理完业务逻辑后显式调用 `channel.ack(msg)`，RabbitMQ 才删除消息。如果消费者在 ACK 前崩溃，消息会自动重新入队，由其他消费者处理。

```mermaid
sequenceDiagram
    participant P as Producer
    participant B as RabbitMQ Broker
    participant C as Consumer

    P->>B: publish(message)
    B->>C: deliver(message)
    Note over C: 处理业务逻辑...
    C->>B: ack(message)
    Note over B: 删除消息
```

如果消费者无法处理消息（如数据格式错误），可以调用 `channel.nack(msg, false, false)` 拒绝消息，第二个参数 `false` 表示不重新入队。配合死信交换器，被拒绝的消息会被路由到死信队列进行人工处理。

> 参考：[RabbitMQ AMQP Concepts](https://www.rabbitmq.com/tutorials/amqp-concepts)，[RabbitMQ Exchanges and Queues](https://www.rabbitmq.com/tutorials)

## Express 原生实战：用 amqplib 理解底层机制

在进入 NestJS 封装之前，先用 Node.js 原生客户端 [amqplib](https://www.npmjs.com/package/amqplib) 手动构建生产者和消费者。这能帮助你理解底层 API，后续 NestJS 的装饰器本质上就是对这些 API 的封装。

### 初始化项目

```bash
mkdir express-rabbitmq-demo && cd express-rabbitmq-demo
npm init -y
npm install amqplib express
npm install -D typescript @types/node @types/express tsx
```

创建 `tsconfig.json`：

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true
  }
}
```

确保本地 RabbitMQ 已启动（默认端口 5672）。如果使用 Docker：

```bash
docker run -d --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:3.13-management
```

### 生产者：发送消息

生产者负责连接 RabbitMQ、声明交换器和队列、建立绑定关系、发送消息。

```typescript
// src/producer.ts
import amqplib from 'amqplib';

const EXCHANGE_NAME = 'order.exchange';
const QUEUE_NAME = 'order.created.queue';
const ROUTING_KEY = 'order.created';

async function startProducer() {
  // 1. 建立 TCP 连接
  const connection = await amqplib.connect('amqp://localhost:5672');

  // 2. 创建 Channel（轻量级逻辑连接）
  const channel = await connection.createChannel();

  // 3. 声明交换器：type 为 topic，durable 表示持久化
  await channel.assertExchange(EXCHANGE_NAME, 'topic', { durable: true });

  // 4. 声明队列：durable 表示队列持久化
  await channel.assertQueue(QUEUE_NAME, { durable: true });

  // 5. 绑定队列到交换器，指定 Routing Key
  await channel.bindQueue(QUEUE_NAME, EXCHANGE_NAME, ROUTING_KEY);

  // 6. 发送消息：persistent 表示消息持久化到磁盘
  const message = JSON.stringify({
    orderId: 'ORD-001',
    userId: 42,
    amount: 99.9,
    createdAt: new Date().toISOString(),
  });

  channel.publish(EXCHANGE_NAME, ROUTING_KEY, Buffer.from(message), {
    persistent: true,
    contentType: 'application/json',
    messageId: 'msg-001',
  });

  console.log(`[Producer] 消息已发送: ${message}`);

  // 7. 关闭连接
  setTimeout(() => {
    channel.close();
    connection.close();
  }, 500);
}

startProducer().catch(console.error);
```

### 消费者：接收消息

消费者连接到同一个交换器和队列，监听并处理消息。

```typescript
// src/consumer.ts
import amqplib from 'amqplib';

const EXCHANGE_NAME = 'order.exchange';
const QUEUE_NAME = 'order.created.queue';
const ROUTING_KEY = 'order.created';

async function startConsumer() {
  const connection = await amqplib.connect('amqp://localhost:5672');
  const channel = await connection.createChannel();

  await channel.assertExchange(EXCHANGE_NAME, 'topic', { durable: true });
  await channel.assertQueue(QUEUE_NAME, { durable: true });
  await channel.bindQueue(QUEUE_NAME, EXCHANGE_NAME, ROUTING_KEY);

  // prefetch: 每个消费者同时最多处理 1 条消息（公平调度）
  channel.prefetch(1);

  console.log('[Consumer] 等待消息...');

  // 消费消息：autoAck: false 表示手动确认
  channel.consume(
    QUEUE_NAME,
    (msg) => {
      if (!msg) return;

      const content = JSON.parse(msg.content.toString());
      console.log(`[Consumer] 收到订单: ${content.orderId}, 金额: ${content.amount}`);

      // 模拟业务处理
      setTimeout(() => {
        // 处理成功后手动确认
        channel.ack(msg);
        console.log(`[Consumer] 订单 ${content.orderId} 处理完成`);
      }, 1000);
    },
    { noAck: false }, // noAck: false = 手动确认模式
  );
}

startConsumer().catch(console.error);
```

### 运行验证

```bash
# 终端 1：启动消费者
npx tsx src/consumer.ts

# 终端 2：发送消息
npx tsx src/producer.ts
```

消费者终端会输出：

```
[Consumer] 等待消息...
[Consumer] 收到订单: ORD-001, 金额: 99.9
[Consumer] 订单 ORD-001 处理完成
```

### 关键点总结

这个示例展示了 RabbitMQ 最基础的消息流转。回顾整个过程：

1. **连接与通道**：`amqplib.connect()` 建立 TCP 连接，`connection.createChannel()` 创建通道。一个连接可以复用多个通道。
2. **声明资源**：`assertExchange()`、`assertQueue()`、`bindQueue()` 是幂等操作——如果资源已存在则不会重复创建。生产者和消费者都声明是为了确保无论谁先启动，资源都能正确建立。
3. **消息持久化**：交换器 `durable: true`、队列 `durable: true`、消息 `persistent: true` 三者缺一不可。缺任何一项，Broker 重启后消息都可能丢失。
4. **手动确认**：`noAck: false` 配合 `channel.ack(msg)` 确保消息被成功处理后才从队列中删除。如果消费者在处理过程中崩溃，消息会自动重新入队，由其他消费者接管。

> 参考：[amqplib 文档](https://amqp-node.github.io/amqplib/channel_api.html)，[RabbitMQ Tutorial - Work Queues](https://www.rabbitmq.com/tutorials/tutorial-two-javascript)

## NestJS 整合 RabbitMQ

NestJS 官方的微服务模块支持 RabbitMQ 作为传输层，但它为了兼容多种传输协议（Redis、Kafka、MQTT 等）而牺牲了 RabbitMQ 特有的能力。[@golevelup/nestjs-rabbitmq](https://www.npmjs.com/package/@golevelup/nestjs-rabbitmq) 是一个社区维护的专用包，提供了更贴近 RabbitMQ 的装饰器驱动开发体验，底层基于 `amqp-connection-manager` 实现连接恢复能力。

### 架构概览

```mermaid
graph TB
    subgraph NestJS Application
        RM[RabbitMQModule.forRoot]
        AC[AmqpConnection]
        PS[PublishService]
        subgraph Handlers
            SH[RabbitSubscribe Handler]
            RH[RabbitRPC Handler]
        end
    end

    PS -->|publish| AC
    AC -->|request| RH
    SH -->|consume| AC
    RH -->|reply| AC

    AC -->|Channel| RB[RabbitMQ Broker]
    RB -->|Exchange| Q1[Queue]
    RB -->|Exchange| Q2[Queue]
```

`RabbitMQModule.forRoot()` 在应用启动时建立与 RabbitMQ Broker 的连接，并自动创建配置中声明的交换器。`AmqpConnection` 是核心服务，注入到任何需要发送消息的 Service 或 Controller 中，提供 `publish()` 和 `request()` 方法。`@RabbitSubscribe` 和 `@RabbitRPC` 装饰器将 Service 方法注册为消息处理器，框架自动处理队列绑定和消息确认。

### 初始化 NestJS 项目

```bash
nest new nestjs-rabbitmq-demo
cd nestjs-rabbitmq-demo
npm install @golevelup/nestjs-rabbitmq
```

### 配置 RabbitMQModule

在 `AppModule` 中导入 `RabbitMQModule.forRoot()` 进行全局配置：

```typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { RabbitMQModule } from '@golevelup/nestjs-rabbitmq';
import { OrderModule } from './order/order.module';

@Module({
  imports: [
    RabbitMQModule.forRoot({
      uri: 'amqp://localhost:5672',
      // 声明交换器，模块初始化时自动创建
      exchanges: [
        { name: 'order.exchange', type: 'topic' },
        { name: 'order.dlx.exchange', type: 'topic' },
      ],
      // 全局默认发布选项
      defaultPublishOptions: { persistent: true },
      // 连接初始化选项：不阻塞应用启动，自动重连
      connectionInitOptions: { wait: false },
    }),
    OrderModule,
  ],
})
export class AppModule {}
```

关键配置说明：

- **`uri`**：RabbitMQ 连接地址，格式为 `amqp://user:password@host:port/vhost`。生产环境建议使用数组配置多个节点实现高可用。
- **`exchanges`**：应用启动时自动 `assertExchange`。如果交换器已存在且类型匹配则跳过，类型冲突则启动失败。
- **`defaultPublishOptions`**：全局默认的发布选项，所有 `publish()` 调用都会合并这些选项。这里设置 `persistent: true` 确保所有消息默认持久化。
- **`connectionInitOptions.wait: false`**：允许应用在 RabbitMQ 不可用时正常启动，连接建立后自动恢复。适合容器化环境中 RabbitMQ 可能后于应用启动的场景。

### 定义消息结构

```typescript
// src/order/interfaces/order-message.interface.ts
export interface OrderCreatedMessage {
  orderId: string;
  userId: number;
  amount: number;
  items: Array<{ productId: string; quantity: number }>;
  createdAt: string;
}

export interface OrderProcessedResult {
  orderId: string;
  status: 'confirmed' | 'rejected';
  processedAt: string;
}
```

### 生产者：发布订单创建消息

生产者通过注入 `AmqpConnection` 来发送消息：

```typescript
// src/order/order.service.ts
import { Injectable } from '@nestjs/common';
import { AmqpConnection } from '@golevelup/nestjs-rabbitmq';
import { OrderCreatedMessage } from './interfaces/order-message.interface';

@Injectable()
export class OrderService {
  constructor(private readonly amqpConnection: AmqpConnection) {}

  async createOrder(userId: number, items: Array<{ productId: string; quantity: number }>) {
    // 1. 业务逻辑：保存订单到数据库（此处省略）
    const orderId = `ORD-${Date.now()}`;
    const amount = items.reduce((sum, item) => sum + item.quantity * 10, 0);

    // 2. 构建消息体
    const message: OrderCreatedMessage = {
      orderId,
      userId,
      amount,
      items,
      createdAt: new Date().toISOString(),
    };

    // 3. 发布消息到交换器
    this.amqpConnection.publish(
      'order.exchange',   // exchange
      'order.created',    // routing key
      message,            // payload（自动 JSON.stringify）
    );

    console.log(`[OrderService] 订单 ${orderId} 已发布到消息队列`);
    return { orderId, status: 'pending' };
  }
}
```

`AmqpConnection.publish()` 是即发即忘（fire-and-forget）模式，不等待消费者处理结果。如果需要同步等待消费者响应，使用 `request()` 方法（RPC 模式）。

### 消费者：处理订单创建消息

消费者使用 `@RabbitSubscribe` 装饰器声明：

```typescript
// src/order/order.consumer.ts
import { Injectable } from '@nestjs/common';
import { RabbitSubscribe, Nack } from '@golevelup/nestjs-rabbitmq';
import { ConsumeMessage } from 'amqplib';
import { OrderCreatedMessage } from './interfaces/order-message.interface';

@Injectable()
export class OrderConsumer {
  @RabbitSubscribe({
    exchange: 'order.exchange',
    routingKey: 'order.created',
    queue: 'order.created.queue',
    queueOptions: {
      durable: true,
      deadLetterExchange: 'order.dlx.exchange',
      deadLetterRoutingKey: 'order.created.dlq',
    },
  })
  async handleOrderCreated(msg: OrderCreatedMessage, amqpMsg: ConsumeMessage) {
    console.log(`[OrderConsumer] 收到订单: ${msg.orderId}, 金额: ${msg.amount}`);

    try {
      // 模拟业务处理：库存扣减、支付等
      await this.processOrder(msg);

      console.log(`[OrderConsumer] 订单 ${msg.orderId} 处理成功`);
      // 返回 void/undefined，框架自动 ACK
    } catch (error) {
      console.error(`[OrderConsumer] 订单 ${msg.orderId} 处理失败:`, error);
      // 返回 Nack(false)：不重新入队，消息进入死信队列
      return new Nack(false);
    }
  }

  private async processOrder(msg: OrderCreatedMessage): Promise<void> {
    // 实际业务逻辑：验证库存、创建支付单等
    await new Promise((resolve) => setTimeout(resolve, 500));
  }
}
```

`@RabbitSubscribe` 装饰器参数说明：

- **`exchange`** 和 **`routingKey`**：定义了此处理器订阅的消息来源。框架会自动声明队列并绑定到指定的交换器和路由键。
- **`queue`**：指定队列名称。多个实例使用相同队列名称时会形成竞争消费者模式——每条消息只被一个实例处理。
- **`queueOptions.durable`**：队列持久化，Broker 重启后队列不丢失。
- **`queueOptions.deadLetterExchange`**：死信交换器，处理失败的消息将被路由到此处。
- **`queueOptions.deadLetterRoutingKey`**：死信路由键，指定消息进入死信交换器后使用的路由键。

处理器方法签名中，第一个参数 `msg` 是已自动反序列化的消息体（默认 `JSON.parse`），第二个参数 `amqpMsg` 是 `amqplib.ConsumeMessage` 原始对象，包含 `properties`（headers、correlationId 等）和 `fields`（exchange、routingKey 等）元数据。

### 注册模块

```typescript
// src/order/order.module.ts
import { Module } from '@nestjs/common';
import { OrderService } from './order.service';
import { OrderConsumer } from './order.consumer';
import { OrderController } from './order.controller';

@Module({
  controllers: [OrderController],
  providers: [OrderService, OrderConsumer],
})
export class OrderModule {}
```

```typescript
// src/order/order.controller.ts
import { Controller, Post, Body } from '@nestjs/common';
import { OrderService } from './order.service';

@Controller('orders')
export class OrderController {
  constructor(private readonly orderService: OrderService) {}

  @Post()
  async createOrder(@Body() body: { userId: number; items: Array<{ productId: string; quantity: number }> }) {
    return this.orderService.createOrder(body.userId, body.items);
  }
}
```

### 完整消息流转

```mermaid
sequenceDiagram
    participant C as HTTP Client
    participant OC as OrderController
    participant OS as OrderService
    participant AC as AmqpConnection
    participant RB as RabbitMQ Broker
    participant OCon as OrderConsumer

    C->>OC: POST /orders
    OC->>OS: createOrder(userId, items)
    OS-->>OS: 保存订单到数据库
    OS->>AC: publish('order.exchange', 'order.created', message)
    AC->>RB: 发送消息
    OS-->>OC: { orderId, status: 'pending' }
    OC-->>C: 201 Created

    RB->>OCon: deliver(message)
    OCon->>OCon: processOrder(message)
    OCon->>RB: ack(message)
```

### RPC 模式：请求/响应

除了发布-订阅，`@golevelup/nestjs-rabbitmq` 还支持 RPC（Remote Procedure Call）模式。生产者发送消息后等待消费者返回结果。

**RPC 处理器**：

```typescript
// src/order/order.rpc-handler.ts
import { Injectable } from '@nestjs/common';
import { RabbitRPC } from '@golevelup/nestjs-rabbitmq';
import { OrderCreatedMessage, OrderProcessedResult } from './interfaces/order-message.interface';

@Injectable()
export class OrderRpcHandler {
  @RabbitRPC({
    exchange: 'order.exchange',
    routingKey: 'order.process',
    queue: 'order.process.queue',
  })
  async processOrder(msg: OrderCreatedMessage): Promise<OrderProcessedResult> {
    // 处理订单并返回结果
    const isValid = msg.amount > 0 && msg.items.length > 0;

    return {
      orderId: msg.orderId,
      status: isValid ? 'confirmed' : 'rejected',
      processedAt: new Date().toISOString(),
    };
  }
}
```

**RPC 调用方**：

```typescript
// src/order/order.service.ts 中的 RPC 调用示例
async createOrderWithRpc(userId: number, items: Array<{ productId: string; quantity: number }>) {
  const message: OrderCreatedMessage = {
    orderId: `ORD-${Date.now()}`,
    userId,
    amount: items.reduce((sum, item) => sum + item.quantity * 10, 0),
    items,
    createdAt: new Date().toISOString(),
  };

  // request() 会等待 RPC 处理器返回结果
  const result = await this.amqpConnection.request<OrderProcessedResult>({
    exchange: 'order.exchange',
    routingKey: 'order.process',
    payload: message,
    timeout: 10000, // 超时时间（毫秒）
  });

  console.log(`[OrderService] RPC 结果: ${result.status}`);
  return result;
}
```

RPC 模式适用于需要同步确认的场景（如支付验证），但它引入了调用方对响应方的依赖，超时和故障处理需要仔细设计。

> 参考：[@golevelup/nestjs-rabbitmq README](https://github.com/golevelup/nestjs/blob/master/packages/rabbitmq/README.md)，[RabbitMQ Direct Reply-To](https://www.rabbitmq.com/direct-reply-to.html)

## 进阶：死信队列与错误处理

在消息队列的实际使用中，消息处理失败是不可避免的。网络超时、数据格式错误、业务规则校验不通过——这些情况都需要一个明确的处理策略，而不是让失败的消息无限重试或静默丢失。死信队列（Dead Letter Queue, DLQ）就是 RabbitMQ 提供的兜底机制。

### 什么是死信

一条消息变成「死信」（Dead Letter）有以下三种情况：

1. **消费者 NACK 且不重新入队**：消费者显式调用 `channel.nack(msg, false, false)`，或在 `@golevelup/nestjs-rabbitmq` 中返回 `new Nack(false)`。
2. **消息 TTL 过期**：消息在队列中的存活时间超过了 `x-message-ttl` 设置的阈值。
3. **队列达到最大长度**：队列设置了 `x-max-length`，超出部分的消息被丢弃。

当上述任一情况发生时，如果队列配置了死信交换器（Dead Letter Exchange, DLX），RabbitMQ 会将死信路由到 DLX，再由 DLX 根据路由键转发到死信队列。

### 死信流转架构

```mermaid
graph LR
    P[生产者] -->|消息| EQ[order.created.queue]
    EQ -->|正常消费| C[消费者]
    C -->|NACK 不重入队| EQ
    EQ -->|死信路由| DLX[order.dlx.exchange]
    DLX -->|deadLetterRoutingKey| DLQ[order.created.dlq]
    DLQ -->|人工处理| DLH[死信消费者]

    style DLX fill:#ff6b6b,color:#fff
    style DLQ fill:#ff6b6b,color:#fff
```

### 配置死信队列

在 `RabbitMQModule.forRoot()` 中声明死信交换器，并在业务队列的 `queueOptions` 中指定 DLX 和死信路由键：

```typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { RabbitMQModule } from '@golevelup/nestjs-rabbitmq';
import { OrderModule } from './order/order.module';

@Module({
  imports: [
    RabbitMQModule.forRoot({
      uri: 'amqp://localhost:5672',
      exchanges: [
        // 业务交换器
        { name: 'order.exchange', type: 'topic' },
        // 死信交换器
        { name: 'order.dlx.exchange', type: 'topic' },
        // 重试交换器
        { name: 'order.retry.exchange', type: 'topic' },
      ],
      defaultPublishOptions: { persistent: true },
      connectionInitOptions: { wait: false },
    }),
    OrderModule,
  ],
})
export class AppModule {}
```

消费者中配置队列的死信路由：

```typescript
// src/order/order.consumer.ts
import { Injectable } from '@nestjs/common';
import { RabbitSubscribe, Nack } from '@golevelup/nestjs-rabbitmq';
import { ConsumeMessage } from 'amqplib';
import { OrderCreatedMessage } from './interfaces/order-message.interface';

@Injectable()
export class OrderConsumer {
  @RabbitSubscribe({
    exchange: 'order.exchange',
    routingKey: 'order.created',
    queue: 'order.created.queue',
    queueOptions: {
      durable: true,
      // 指定死信交换器和路由键
      deadLetterExchange: 'order.dlx.exchange',
      deadLetterRoutingKey: 'order.created.dlq',
    },
  })
  async handleOrderCreated(msg: OrderCreatedMessage, amqpMsg: ConsumeMessage) {
    try {
      await this.processOrder(msg);
      // 返回 void，框架自动 ACK
    } catch (error) {
      console.error(`订单 ${msg.orderId} 处理失败:`, error);
      // Nack(false)：不重新入队，消息路由到死信队列
      return new Nack(false);
    }
  }

  private async processOrder(msg: OrderCreatedMessage): Promise<void> {
    if (msg.amount <= 0) {
      throw new Error('订单金额必须大于零');
    }
    // 业务处理...
  }
}
```

### 死信消费者

死信消费者负责接收和记录无法处理的消息，通常用于告警和人工介入：

```typescript
// src/order/dead-letter.consumer.ts
import { Injectable, Logger } from '@nestjs/common';
import { RabbitSubscribe } from '@golevelup/nestjs-rabbitmq';
import { ConsumeMessage } from 'amqplib';

@Injectable()
export class DeadLetterConsumer {
  private readonly logger = new Logger(DeadLetterConsumer.name);

  @RabbitSubscribe({
    exchange: 'order.dlx.exchange',
    routingKey: 'order.created.dlq',
    queue: 'order.created.dlq',
    queueOptions: { durable: true },
  })
  async handleDeadLetter(msg: unknown, amqpMsg: ConsumeMessage) {
    const deathInfo = amqpMsg.properties.headers?.['x-death']?.[0];
    const originalQueue = deathInfo?.queue ?? 'unknown';
    const deathCount = deathInfo?.count ?? 0;

    this.logger.error(
      `死信消息收到 - 原始队列: ${originalQueue}, 死亡次数: ${deathCount}, 消息体: ${JSON.stringify(msg)}`,
    );

    // 记录到数据库或发送告警通知
    // 实际生产中：写入死信表、发送钉钉/飞书告警、或人工审核
  }
}
```

RabbitMQ 在消息变为死信时会自动添加 `x-death` Header，其中包含原始队列名、死亡时间、死亡次数等信息，方便追踪问题。

### 带延迟的重试策略

直接 Nack 到死信队列意味着只有一次处理机会。在实际场景中，很多失败是暂时性的（如数据库连接中断），需要延迟重试。可以利用 TTL + DLX 组合实现延迟重试：

```mermaid
graph LR
    P[生产者] -->|消息| EQ[order.created.queue]
    EQ -->|正常消费| C[消费者]
    C -->|NACK 重入队| EQ
    EQ -->|重试次数超限| DLX[order.dlx.exchange]
    DLX -->|路由| RQ[order.created.retry.queue]
    RQ -->|TTL 过期后| RX[order.retry.exchange]
    RX -->|重新入队| EQ
    EQ -->|最终失败| FDLX[order.dlx.exchange]
    FDLX -->|路由| DLQ[order.created.dlq]

    style RQ fill:#ffd93d
    style RX fill:#ffd93d
```

实现带延迟的重试消费者：

```typescript
// src/order/order.consumer.ts
import { Injectable, Logger } from '@nestjs/common';
import { RabbitSubscribe, Nack, AmqpConnection } from '@golevelup/nestjs-rabbitmq';
import { ConsumeMessage } from 'amqplib';
import { OrderCreatedMessage } from './interfaces/order-message.interface';

const MAX_RETRY = 3;

@Injectable()
export class OrderConsumer {
  private readonly logger = new Logger(OrderConsumer.name);

  constructor(private readonly amqpConnection: AmqpConnection) {}

  @RabbitSubscribe({
    exchange: 'order.exchange',
    routingKey: 'order.created',
    queue: 'order.created.queue',
    queueOptions: {
      durable: true,
      deadLetterExchange: 'order.dlx.exchange',
      deadLetterRoutingKey: 'order.created.dlq',
    },
  })
  async handleOrderCreated(msg: OrderCreatedMessage, amqpMsg: ConsumeMessage) {
    const retryCount = (amqpMsg.properties.headers?.['x-retry-count'] as number) ?? 0;

    try {
      await this.processOrder(msg);
      // 成功，自动 ACK
    } catch (error) {
      this.logger.error(`订单 ${msg.orderId} 处理失败 (第 ${retryCount + 1} 次): ${error.message}`);

      if (retryCount < MAX_RETRY) {
        // 延迟重试：发布到重试交换器，带 TTL
        this.amqpConnection.publish('order.retry.exchange', 'order.created.retry', msg, {
          persistent: true,
          expiration: 5000, // 5 秒后重试
          headers: { 'x-retry-count': retryCount + 1 },
        });
        // ACK 当前消息（已重新发布，不算丢失）
      } else {
        // 超过重试上限，Nack 不重入队 → 进入死信队列
        this.logger.error(`订单 ${msg.orderId} 重试 ${MAX_RETRY} 次后仍失败，进入死信队列`);
        return new Nack(false);
      }
    }
  }

  private async processOrder(msg: OrderCreatedMessage): Promise<void> {
    if (msg.amount <= 0) {
      throw new Error('订单金额必须大于零');
    }
    // 业务处理...
  }
}
```

重试队列需要在 `RabbitMQModule.forRoot()` 中声明：

```typescript
exchanges: [
  { name: 'order.exchange', type: 'topic' },
  { name: 'order.dlx.exchange', type: 'topic' },
  { name: 'order.retry.exchange', type: 'topic' },
],
queues: [
  {
    name: 'order.created.retry.queue',
    durable: true,
    // 重试队列：TTL 5 秒，过期后路由回业务交换器
    options: {
      messageTtl: 5000,
      deadLetterExchange: 'order.exchange',
      deadLetterRoutingKey: 'order.created',
    },
  },
],
```

注意重试队列的 `deadLetterRoutingKey` 指向业务交换器而非死信交换器，这样 TTL 过期后消息会重新回到业务队列进行重试。而业务队列的 DLX 指向死信交换器，确保最终失败的消息进入死信队列。

### 错误处理策略对比

| 策略 | 适用场景 | 实现方式 |
|------|---------|---------|
| 自动 ACK | 消息丢失可接受，追求最高吞吐量 | 处理器返回 `void` |
| Nack 不重入队 | 永久性错误（数据格式错误），需要人工介入 | `return new Nack(false)` |
| Nack 重入队 | 瞬时性错误（网络抖动），但需防止无限循环 | `return new Nack(true)` |
| 延迟重试 + 死信 | 生产环境推荐方案，有限重试 + 兜底 | TTL + DLX 组合 |

> 参考：[RabbitMQ Dead Letter Exchanges](https://www.rabbitmq.com/docs/dlx)，[RabbitMQ TTL](https://www.rabbitmq.com/docs/ttl)

## 最佳实践与常见问题

### 连接管理

在生产环境中，网络波动和 RabbitMQ 重启是常态。`@golevelup/nestjs-rabbitmq` 底层使用 `amqp-connection-manager`，默认支持自动重连。关键配置：

```typescript
RabbitMQModule.forRoot({
  uri: [
    'amqp://node1:5672',
    'amqp://node2:5672',
    'amqp://node3:5672',
  ],
  connectionInitOptions: {
    wait: false,      // 不阻塞应用启动
    timeout: 10000,   // 连接超时 10 秒
    reject: false,    // 连接失败不抛异常
  },
  connectionManagerOptions: {
    reconnectTimeInSeconds: 2,    // 重连间隔
    heartbeatIntervalInSeconds: 30, // 心跳间隔
  },
  // ...
})
```

生产环境建议使用 URI 数组配置多个 RabbitMQ 节点，配合 `connectionInitOptions.wait: false` 确保应用在 RabbitMQ 不可用时也能正常启动。

### 消息持久化三要素

消息持久化需要三个条件同时满足，缺一不可：

1. **交换器持久化**：`{ durable: true }`
2. **队列持久化**：`{ durable: true }`
3. **消息持久化**：`{ persistent: true }`

如果只设置了队列持久化而消息不是 `persistent`，Broker 重启后队列还在但消息消失了。如果队列不是持久化的，Broker 重启后队列本身就不存在了。

在 `@golevelup/nestjs-rabbitmq` 中，推荐在 `forRoot` 中设置全局 `defaultPublishOptions: { persistent: true }`，避免每次 `publish()` 都手动指定。

### 幂等性

消息队列不保证 Exactly-Once 语义——同一消息可能被投递多次（At-Least-Once）。消费者必须具备幂等性：多次处理同一条消息的结果与处理一次相同。

实现幂等的常见方式：

- **业务唯一键**：在消息体中携带唯一 ID（如 `orderId`），消费者处理前先检查该 ID 是否已处理过。可以在数据库中记录处理状态或使用 Redis 的 SETNX。
- **数据库唯一约束**：利用数据库的唯一索引防止重复写入。

```typescript
async handleOrderCreated(msg: OrderCreatedMessage, amqpMsg: ConsumeMessage) {
  // 幂等检查：如果该订单已处理，直接 ACK 跳过
  const existing = await this.orderRepository.findById(msg.orderId);
  if (existing?.status !== 'pending') {
    return; // 已处理，自动 ACK
  }

  // 处理逻辑...
}
```

### 公平调度

默认情况下，RabbitMQ 按轮询方式将消息分发给消费者，不考虑每个消费者的当前负载。如果某些消息处理耗时较长，会导致负载不均。

通过设置 `prefetch` 实现公平调度：每个消费者同时最多处理 N 条消息，处理完才能接收新消息。在 `@golevelup/nestjs-rabbitmq` 中通过 Channel 配置：

```typescript
RabbitMQModule.forRoot({
  uri: 'amqp://localhost:5672',
  prefetchCount: 1, // 全局默认
  channels: {
    'order-channel': {
      prefetchCount: 5, // 该 Channel 同时最多处理 5 条消息
      default: true,
    },
  },
})
```

`prefetchCount` 的取值取决于每条消息的处理耗时和系统吞吐量要求。处理耗时长（秒级）且需要均匀分配时设为 1；处理耗时短（毫秒级）且追求吞吐量时设为较大值。

### 消费者限流

除了 `prefetchCount`，还可以通过 Channel 级别的 `prefetchCount` 为不同类型的消费者设置不同的处理速度：

```typescript
channels: {
  'fast-channel': { prefetchCount: 50 },   // 轻量处理
  'slow-channel': { prefetchCount: 2 },    // 重量处理
}
```

在 `@RabbitSubscribe` 中通过 `queueOptions.channel` 指定使用哪个 Channel。

### 监控与运维

RabbitMQ 内置管理插件（默认端口 15672）提供了 Web 管理界面，可以查看队列深度、消息速率、消费者数量等指标。

生产环境需要关注的关键指标：

- **队列深度（Queue Depth）**：消息堆积数量。持续增长说明消费者处理速度跟不上生产者。
- **消费者利用率（Consumer Utilization）**：消费者实际处理消息的时间比例。过低可能意味着网络延迟或消费者性能瓶颈。
- **未确认消息数（Unacked Messages）**：已投递但尚未 ACK 的消息数量。持续偏高说明消费者处理过慢。
- **死信队列消息数**：任何大于零的值都需要立即关注，排查原因。

可以通过 RabbitMQ 的 HTTP API 或 Prometheus + Grafana 进行指标采集和可视化。

### 常见问题排查

**消息丢失了？** 检查三个持久化是否都开启。检查消费者是否在 ACK 前崩溃（消息会自动重新入队，不会丢失）。检查队列是否有 TTL 设置导致消息过期。

**消息重复消费？** 消息队列的 At-Least-Once 语义决定了重复消费是正常现象。确保消费者幂等。

**消费者不消费？** 检查交换器类型与 Routing Key 是否匹配。Topic Exchange 中 `order.created` 不会匹配 Binding `order.*.email`。检查队列是否被排他连接占用。

**连接频繁断开？** 检查心跳间隔配置。检查防火墙或负载均衡器是否有空闲连接超时。适当降低 `heartbeatIntervalInSeconds`。

> 参考：[RabbitMQ Production Checklist](https://www.rabbitmq.com/docs/production-checklist)，[RabbitMQ Monitoring](https://www.rabbitmq.com/docs/monitoring)，[RabbitMQ Reliability Guide](https://www.rabbitmq.com/docs/reliability)