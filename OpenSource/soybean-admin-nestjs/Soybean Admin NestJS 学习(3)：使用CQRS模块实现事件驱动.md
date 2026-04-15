---
Title: Soybean Admin NestJS 学习(3)：使用CQRS模块实现事件驱动
Draft: false
tags:
  - ERP
  - 开源项目
  - TypeScript
  - NestJS
Author: Ruby Ceng
---

# Soybean Admin NestJS 学习（3）：使用 CQRS 模块实现事件驱动

## 1. 事件驱动架构简介

### 1.1 事件驱动架构 (EDA)

事件驱动架构 (EDA) 是一种软件设计模式，它将应用程序中的业务逻辑分解为独立的事件，并使用事件总线来协调这些事件的处理。

在传统的请求-响应模式中，一个操作（比如用户登录）的所有相关逻辑都必须在同一个函数调用链中同步完成。例如，登录成功后，你可能需要：

1. 生成 Token
2. 记录登录日志
3. 更新用户最后登录时间
4. 发送欢迎邮件

如果这些都写在 login 服务里，这个函数会变得非常臃肿，而且违反了单一职责原则。任何一个辅助操作（如邮件服务）的失败都可能导致整个登录失败。

事件驱动架构解决了这个问题。它的核心思想是：当一个核心操作完成后，它不直接调用其他业务，而是"发布一个事件"，宣告"某件事已经发生了"。其他关心这件事的模块可以"订阅这个事件"，并在接收到事件后执行它们自己的逻辑。

### 1.2 主要优点

- **解耦 (Decoupling)**：登录逻辑和日志记录逻辑完全分离，可以独立修改和维护
- **单一职责 (Single Responsibility)**：每个模块只做自己的事，代码更清晰
- **可扩展性 (Scalability)**：未来如果需要增加"登录后送积分"的功能，只需增加一个新的事件处理器，完全不用动现有的登录代码
- **异步处理 (Asynchronous)**：耗时的操作（如发送邮件）可以异步执行，不会阻塞主流程，能提升响应速度

## 2. 引入 CQRS 模块

### 2.1 安装 CQRS 模块

```bash
npm install @nestjs/cqrs
```

### 2.2 定义 CQRS 事件

```typescript
// domain/events/user-logged-in.event.ts
export class UserLoggedInEvent implements IEvent {
  constructor(
    public readonly userId: string,
    public readonly username: string,
    public readonly domain: string,
    public readonly ip: string,
    public readonly address: string,
    public readonly userAgent: string,
    public readonly requestId: string,
    public readonly type: string,
    public readonly port?: number | null
  ) {}
}
```

### 2.3 引入 CQRS 聚合根基类

```typescript
// domain/user.ts
export interface IUser {
  verifyPassword(password: string): Promise<boolean>;
  canLogin(): Promise<boolean>;
  loginUser(password: string): Promise<{ success: boolean; message: string }>;
  commit(): void;
}

export class User extends AggregateRoot implements IUser {
  readonly id: string;
  readonly username: string;
  readonly nickName: string;
  readonly password: Password;
  readonly status: Status;
  readonly domain: string;
  readonly avatar: string | null;
  readonly email: string | null;
  readonly phoneNumber: string | null;
  createdAt: Date;
  createdBy: string;

  constructor(
    properties: UserProperties | UserCreateProperties | UserUpdateProperties
  ) {
    super();
    Object.assign(this, properties);

    if ("password" in properties && properties.password) {
      this.password = Password.fromHashed(properties.password);
    }
  }

  // ...各类业务方法
}
```

### 2.4 定义事件处理器

```typescript
// application/event-handlers/user-logged-in.event.handler.ts
@EventsHandler(UserLoggedInEvent)
export class UserLoggedInHandler implements IEventHandler<UserLoggedInEvent> {
  constructor(
    @Inject(LoginLogWriteRepoPortToken)
    private readonly loginLogWriteRepo: LoginLogWriteRepoPort
  ) {}

  async handle(event: UserLoggedInEvent) {
    const loginLog = new LoginLogEntity(
      event.userId,
      event.username,
      event.domain,
      event.ip,
      event.address,
      event.userAgent,
      event.requestId,
      event.type,
      event.userId,
      event.port
    );

    // 调用Repository进行持久化 - Clean Architecture
    return await this.loginLogWriteRepo.save(loginLog);
  }
}
```

### 2.5 业务层中发布领域事件

```typescript
// authentication.service.ts
// ... inside execPasswordLogin method ...

// 1. 创建一个聚合根（Aggregate Root），这是CQRS模式中的一个概念，可以看作是业务实体
const userAggregate = new User(user);

// ... 其他逻辑 ...

// 2. 将事件"应用"到聚合根上。此时事件还未发布，只是被暂存起来。
userAggregate.apply(
  new UserLoggedInEvent(
    user.id,
    user.username
    // ... 其他参数
  )
);

// 也可以 apply 其他事件
userAggregate.apply(
  new TokenGeneratedEvent()
  // ...
);

// 3. 将聚合根与 EventPublisher 关联
this.publisher.mergeObjectContext(userAggregate);

// 4. 提交！所有被 apply 的事件会在这里被一次性发布出去
userAggregate.commit();
```

## 3. CQRS 事件机制详解

### 3.1 关键概念

- **apply**: 用于将一个新事件注册到业务实体上
- **commit**: 真正将所有已注册的事件通过事件总线（Event Bus）广播出去

**解耦效果**: 一次登录，触发了"角色"和"日志审计"两个完全不同业务域的逻辑。

### 3.2 代码执行逻辑

#### 步骤 1: 创建聚合根实例

```typescript
const userAggregate = new UserAggregate(...)
```

- 创建了一个 UserAggregate 的实例
- 这个实例继承自 AggregateRoot，内部有一个私有数组：`private readonly _events: IEvent[] = []`，用来存放事件

#### 步骤 2: 应用事件

```typescript
userAggregate.apply(new UserLoggedInEvent(...))
```

- 当调用 apply 方法时，UserLoggedInEvent 这个事件对象被 push 进了 userAggregate 实例内部的 \_events 数组里
- 此时，`EventPublisher` 对这一切毫不知情，事件完全被封装在 userAggregate 对象内部

#### 步骤 3: 绑定事件发布器

```typescript
this.publisher.mergeObjectContext(userAggregate);
```

这是最关键的一步：

- EventPublisher 服务接收到你的 userAggregate 对象
- 它实际上做的是：将 `EventPublisher` 自己的 `publish` 方法"嫁接"或者说"代理"到你传入的 `userAggregate` 对象的 `commit` 方法上
- 简单来说，它建立了一条从 userAggregate 到 EventPublisher 的通信管道
- EventPublisher 现在"认识"了这个特定的 userAggregate 实例，并准备好为它服务

#### 步骤 4: 提交事件

```typescript
userAggregate.commit();
```

当调用 userAggregate.commit() 时，因为第 3 步的绑定，这个调用实际上触发了 EventPublisher 的内部逻辑。

EventPublisher 会：

- **a.** 获取到它所绑定的 userAggregate 对象
- **b.** 访问该对象内部的 \_events 数组，拿到所有待发布的事件
- **c.** 遍历这个数组，将每一个事件发布到全局的事件总线（Event Bus）
- **d.** 清空 \_events 数组，防止事件被重复发布

### 3.3 事件驱动的优势

通过这种机制，我们实现了：

1. **业务逻辑解耦**：主业务流程和辅助业务完全分离
2. **可扩展性**：新增功能只需添加新的事件处理器
3. **异步处理**：事件处理可以异步执行，提高系统响应速度
4. **单一职责**：每个事件处理器只处理特定的业务逻辑

## 4. 快速开始指南

本节提供一个完整的 CQRS 事件驱动实现示例，帮助您快速上手。

### 4.1 安装依赖

```bash
npm install @nestjs/cqrs
```

### 4.2 创建事件类

首先定义一个自定义事件，实现 `IEvent` 接口：

```typescript
// my-custom.event.ts
import { IEvent } from "@nestjs/cqrs";

export class MyCustomEvent implements IEvent {
  constructor(public readonly data: string, public readonly timestamp: Date) {}
}
```

### 4.3 创建事件处理器

创建事件处理器来响应事件：

```typescript
// my-custom.event.handler.ts
import { EventsHandler, IEventHandler } from "@nestjs/cqrs";
import { MyCustomEvent } from "./my-custom.event";

@EventsHandler(MyCustomEvent)
export class MyCustomEventHandler implements IEventHandler<MyCustomEvent> {
  async handle(event: MyCustomEvent) {
    console.log("处理自定义事件:", event.data);
    console.log("事件时间:", event.timestamp);

    // TODO: 在这里实现你的业务逻辑
    // 例如：发送邮件、记录日志、更新缓存等
  }
}
```

### 4.4 注册到模块

将事件处理器注册到 NestJS 模块中：

```typescript
// my-custom.module.ts
import { Module } from "@nestjs/common";
import { CqrsModule } from "@nestjs/cqrs";
import { MyCustomEventHandler } from "./my-custom.event.handler";

@Module({
  imports: [CqrsModule],
  providers: [MyCustomEventHandler],
})
export class MyCustomModule {}
```

### 4.5 发布事件

在服务中使用 `EventBus` 发布事件：

```typescript
// my-service.ts
import { Injectable } from "@nestjs/common";
import { EventBus } from "@nestjs/cqrs";
import { MyCustomEvent } from "./my-custom.event";

@Injectable()
export class MyService {
  constructor(private readonly eventBus: EventBus) {}

  async doSomething() {
    // 执行主要业务逻辑...
    console.log("执行主要业务逻辑");

    // 发布事件，触发相关的副作用处理
    this.eventBus.publish(new MyCustomEvent("hello world", new Date()));

    console.log("事件已发布");
  }
}
```

### 4.6 完整示例

将所有组件整合到应用模块中：

```typescript
// app.module.ts
import { Module } from "@nestjs/common";
import { CqrsModule } from "@nestjs/cqrs";
import { MyCustomModule } from "./my-custom.module";
import { MyService } from "./my-service";

@Module({
  imports: [CqrsModule, MyCustomModule],
  providers: [MyService],
})
export class AppModule {}
```

**运行效果：**

当调用 `MyService.doSomething()` 方法时，会依次输出：

1. "执行主要业务逻辑"
2. "事件已发布"
3. "处理自定义事件: hello world"
4. "事件时间: [当前时间戳]"

这展示了事件驱动架构的核心特性：**主业务逻辑与副作用处理的完全解耦**。
