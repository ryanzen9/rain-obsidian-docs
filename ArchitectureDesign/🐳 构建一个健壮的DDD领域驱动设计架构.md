---
Title: 🐳 构建一个健壮的DDD领域驱动设计架构
Draft: true
tags:
  - ERP
  - DDD领域驱动设计
  - 架构设计
Author: Ruby Ceng
---

## 引言

领域驱动设计（DDD）不仅仅是一种技术，更是一种思想、一种方法论。它旨在通过将软件的核心复杂性与业务领域模型紧密对齐，来构建出易于维护、扩展且能精准反映业务需求的软件系统。传统的 MVC 或三层架构在处理复杂业务时，常常会导致逻辑泄露、Service 层臃肿（Fat Service）和贫血领域模型等问题，使得系统随着业务增长而变得难以驾驭。

本文将通过我们共同构建一个“小超市 ERP 结算系统”的完整历程，深度梳理并总结如何搭建一个真正健壮、清晰的 DDD 架构。我们将从架构蓝图开始，深入探讨每一个核心概念，并用具体的代码示例来展示它们是如何协同工作的。

## 蓝图：DDD 分层架构总览

在动手之前，我们必须有一张清晰的地图。DDD 推崇“分层架构”（Layered Architecture），也与“洋葱架构”、“六边形架构”等思想异曲同工。其核心目标是：**保护领域核心，让依赖关系永远指向内部**。

这是我们最终确定的项目文件结构，它将是我们贯穿全文的参照物：

```
src/
├── modules/
│   └── order/
│       ├─ domain/  # 【领域层】业务核心，与框架无关
│       │   ├─ aggregates/
│       │   │  └─ order.aggregate-root.ts
│       │   ├─ entities/
│       │   │  └─ order-item.entity.ts
│       │   ├─ services/
│       │   │  └─ discount-calculation.service.ts
│       │   ├─ repositories/
│       │   │  └─ order.repository.ts
│       │   └─ events/
│       │      └─ order-paid.event.ts
│       │
│       ├─ application/  # 【应用层】编排用例，连接内外
│       │   ├─ services/
│       │   │  └─ order.service.ts
│       │   ├─ dtos/
│       │   │  └─ create-order.dto.ts
│       │   └─ mappers/
│       │      └─ order.mapper.ts
│       │
│       ├─ infrastructure/  # 【基础设施层】技术实现
│       │   ├─ repositories/
│       │   │  └─ prisma-order.repository.ts
│       │   └─ mappers/
│       │      └─ order.prisma-mapper.ts
│       │
│       ├─ presentation/  # 【表现层】对外暴露的接口
│       │   └─ order.controller.ts
│       │
│       └─ order.module.ts
│
└── shared/ # 跨模块共享的通用代码
    ├─ domain/
    │  ├─ aggregate-root.base.ts
    │  └─ ...
    └─ infrastructure/
       └─ prisma/
          └─ prisma.service.ts
```

### 各层职责说明

- **表现层 (Presentation Layer)**：系统的门户。负责接收外部请求（如 HTTP API）、解析参数、调用应用层服务，并将结果返回给客户端。在我们的项目中，它就是由 NestJS 的`Controller`组成的。
- **应用层 (Application Layer)**：很薄的一层，是业务用例（Use Cases）的编排者。它不包含任何业务规则，只负责：加载领域对象（聚合根）、调用领域对象的方法来执行业务、最后通过仓储持久化结果。我们的`OrderService`就在这一层。
- **领域层 (Domain Layer)**：**项目的心脏与灵魂**。这一层包含了所有纯粹的业务逻辑、规则和状态。它是技术的“绝缘层”，不应该依赖任何外部框架（没有`import { ... } from '@nestjs/common'`）。它是用代码表达的通用语言（Ubiquitous Language）。
- **基础设施层 (Infrastructure Layer)**：技术的具体实现。它负责实现领域层定义的“接口”（如仓储），为系统提供数据库访问、消息队列、缓存、文件系统等能力。我们的`PrismaOrderRepository`就在这里，它用 Prisma 技术实现了`OrderRepository`接口。

---

## 深入心脏：解构领域层 (Domain Layer)

领域层是 DDD 最迷人也最关键的地方。它由一系列构建块（Building Blocks）组成，这些构建块共同构成了能精确表达业务的领域模型。

### 1. 聚合根 (Aggregate Root)

这是 DDD 中最重要的概念。

- **是什么？** 聚合根是一个或多个实体的集合，它被视为一个单一的、原子性的工作单元。它是数据一致性的“守护者”和事务的“边界”。
- **规则**：外部世界只能引用聚合根本身，绝对不能直接访问或修改聚合内部的实体。任何对内部状态的修改都必须通过聚合根上定义的方法来执行。
- **在我们的项目中**：`OrderAggregate`是完美的例子。一个“订单”不仅仅是订单本身，它还包含了一组“订单项”(`OrderItem`)。`OrderAggregate`负责保护这些订单项，确保当一个商品被添加时，订单的总价(`totalAmount`)也必须被同步更新。

**源码解读 (`order.aggregate-root.ts`)：**

```typescript
export class OrderAggregate extends AggregateRoot<OrderProps> {
  // 业务方法：添加商品
  public addItem(product: ProductSnapshot, quantity: number): void {
    // 规则1：检查订单状态
    if (this.props.status !== OrderStatus.DRAFT) {
      throw new Error(
        "Cannot add items to an order that is not in draft status."
      );
    }
    // 规则2：检查商品库存
    if (product.stock < quantity) {
      throw new Error("Insufficient product stock.");
    }

    const existingItem = this.props.items.find(
      (item) => item.productId === product.id
    );
    if (existingItem) {
      existingItem.increaseQuantity(quantity);
    } else {
      const newItem = OrderItem.create(/*...*/);
      this.props.items.push(newItem);
    }

    // 关键！调用私有方法，保证数据一致性
    this.recalculateTotal();
    this.props.updatedAt = new Date();
  }

  // 私有方法，由聚合根自己调用，确保总价永远是正确的
  private recalculateTotal(): void {
    this.props.totalAmount = this.props.items.reduce(
      (sum, item) => sum + item.subTotal,
      0
    );
  }
}
```

`OrderAggregate`就像一位经验丰富的业务经理，它自己管理着所有内部事务，绝不允许外部的“野指针”来扰乱它的账目。

### 2. 实体 (Entity) & 值对象 (Value Object)

- **实体 (Entity)**：拥有唯一标识符（ID）并且其生命周期很重要的对象。在我们的项目中，`OrderItem`就是一个实体。每个订单项都有自己的 ID，但它的生命周期完全依附于`OrderAggregate`。
- **值对象 (Value Object)**：没有唯一标识符，其相等性由它所包含的属性来定义的对象，通常是不可变的。例如，一个`Money`对象（包含`amount`和`currency`）或是一个`ShippingAddress`（包含省、市、区、街道）。

### 3. 仓储 (Repository)

- **是什么？** 仓储是一个**接口**，定义在领域层。它提供了一种类似集合（Collection）的抽象，用于封装持久化聚合根的逻辑。
- **目的**：让领域层能够以一种不关心具体数据库技术的方式，来“保存”和“加载”聚合根。它将领域模型与数据映射逻辑完全隔离。

**源码解读 (`order.repository.ts`)：**

```typescript
import { OrderAggregate } from "../aggregates/order.aggregate-root";

// 这是一个契约，一个接口
export abstract class OrderRepository {
  abstract save(order: OrderAggregate): Promise<void>;
  abstract findById(id: string): Promise<OrderAggregate | null>;
  abstract findDraftByUserId(userId: string): Promise<OrderAggregate | null>;
}
```

注意，这里只有抽象方法，没有任何 Prisma 或 SQL 的痕迹。它的具体实现 `PrismaOrderRepository` 被放在了基础设施层。这正是**依赖倒置原则**的最佳体现。

### <span id="service-vs-event">4. 领域服务 (Domain Service) vs. 领域事件 (Domain Event)</span>

这是 DDD 中两个极易混淆但功能迥异的工具，用于处理跨聚合的交互。

#### 领域服务 (Domain Service)

- **何时使用？** 当一个业务操作逻辑复杂，需要协调**多个不同**的聚合根，并且主流程**必须同步等待**它的计算结果来决定下一步时，就应该使用领域服务。
- **特点**：同步的、通常是无状态的、有返回值的，并且与主流程在同一个事务中。
- **我们的例子**：一个复杂的**“折扣计算”**服务。它需要同时读取`OrderAggregate`（购物车内容）、`UserAggregate`（会员等级）和`PromotionAggregate`（优惠券信息）来计算出一个最终折扣。`OrderService`必须**立即拿到**这个折扣结果，才能更新订单总价，因此必须使用领域服务。

#### 领域事件 (Domain Event)

- **何时使用？** 当一个核心业务操作完成时，它需要通知系统的其他部分，但它本身**不关心**其他部分如何响应。
- **特点**：异步的、“事后”的、单向通知、无返回值的，并且每个事件的处理都在其自己的独立事务中。
- **我们的例子**：`OrderPaidEvent`。当订单被成功支付后，`OrderAggregate`的`markAsPaid`方法并不去亲自扣减库存、增加积分。它只是**宣布（发布）**一个“订单已支付”的事件。
  - **库存模块**会有一个`OrderPaidHandler`监听到这个事件，然后执行扣减库存的逻辑。
  - **积分模块**可以有另一个处理器，监听到同一个事件，然后为用户增加积分。
- **好处**：极致的解耦！`Order`模块完全不知道库存和积分模块的存在。未来要增加新的后续流程（如财务记账、物流通知），只需增加新的事件处理器即可，`Order`模块的代码一行都不用改，完美符合**开闭原则**。

---

## 编排与暴露：应用层与表现层

- **应用层 (`OrderService`)**：扮演着“交通警察”的角色。它接收来自表现层的指令（如`addItemDto`），指挥领域对象工作（加载`OrderAggregate`，调用`order.addItem()`），然后指挥仓储保存结果。它是连接外部世界和领域核心的薄薄的桥梁。

- **表现层 (`OrderController`)**：系统的 API 门户。它只关心 HTTP 协议，如路由、参数验证、序列化等。它将 HTTP 请求翻译成对应用服务的简单调用。

---

## 串联一切：一个完整的故事

让我们以“添加商品到购物车”为例，看看数据和逻辑是如何在各层之间流动的：

1.  **POST `/orders/cart/items`** 请求到达 `OrderController`。
2.  `OrderController` 调用 `OrderService`的`addItemToCart(userId, dto)`方法。
3.  `OrderService`：
    a. 调用 `ProductRepository` 加载 `ProductAggregate`。
    b. 调用 `OrderRepository` 加载或创建用户的 `OrderAggregate`（购物车）。
    c. 调用 `orderAggregate.addItem(product, quantity)`。**所有核心业务规则（查库存、算总价）在此刻于领域模型内部执行。**
    d. 调用 `OrderRepository.save(orderAggregate)`。
4.  `PrismaOrderRepository` 的 `save` 方法被执行：
    a. 它使用 `OrderPrismaMapper` 将 `OrderAggregate` 及其内部的 `OrderItem` 实体“翻译”成 Prisma 可以理解的数据结构。
    b. 在一个数据库事务中，更新 `Order` 和 `OrderItem` 表。
5.  控制权返回，`OrderController` 将 `OrderService` 返回的 DTO 序列化为 JSON，响应给客户端。

---

## 核心原则与拓展思考

- **聚合根绝不能调用仓储**：领域核心必须保持纯净。让聚合根调用仓储会污染领域、破坏事务边界、并使单元测试变得极其困难。对于跨领域的查写操作，两种做法：1. 在领域服务编排层进行 Repository 抽象类的调用。2. 在 Repository 内使用领域事件。孰优孰劣在[领域服务 (Domain Service) vs. 领域事件 (Domain Event)](#service-vs-event)
- **可靠的领域事件**：一个简单的内存 `EventEmitter` 在生产中是不可靠的。为了保证事件不丢失（例如，一个订阅者失败了），在单体项目中使用，我们需要引入**“事务性发件箱”模式**。而在微服务中，则需要引入消息队列。
- CQRS：查写分离。

## 结语

领域驱动设计是一场将代码从“如何实现”的技术细节中解放出来，回归到“业务是什么”的本质的旅程。它通过精巧的分层和一系列核心构建块，帮助我们构建出一个模型清晰、职责单一、高度内聚、松散耦合的系统。
