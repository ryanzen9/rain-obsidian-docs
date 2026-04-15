---
Title: 🚀 四、Prisma 实战指南：构建基于NestJS+Prisma的DDD项目模版
Draft: false
tags:
  - 架构设计
  - DDD领域驱动设计
  - Prisma
  - ORM
Author: Ruby Ceng
---

## 📦 源码获取

本文涉及的完整项目模版已开源，您可以通过以下方式获取：

### GitHub 仓库

🔗 **项目地址**：[prisma_nestjs_example](https://github.com/rubyceng/ruby-project-example/tree/main/prisma_nestjs_example)

## 🎯 引言

在软件开发的征途中，我们总在寻求那个"更好"的起点——一个结构清晰、高度可扩展、能够驾驭业务复杂性的项目模版。一个简单的 `nest new` 固然能让我们快速启动，但它离一个能支撑企业级应用的专业架构还有很长的路。

本文是我们一系列深度探讨的最终结晶。我们将一步步、从无到有地构建一个基于 NestJS 和 Prisma 的领域驱动设计（DDD）项目模版。这个模版不仅仅是文件和目录的堆砌，它是一套经过深思熟虑的架构决策、编码范式和最佳实践的集合，旨在为你未来的任何复杂项目提供一个坚如磐石的起点。

## 🏗️ 蓝图：专业级的 DDD 分层架构

在敲下第一行代码前，让我们先展示最终的目标。这是一个清晰、可扩展、职责分离的 DDD 分层目录结构，也是我们构建过程的导航地图。

```
src
├── modules/
│   └── (业务模块, e.g., product, order...)
│
└── shared/ # 跨模块共享的通用代码
    ├─ domain/
    │  ├─ aggregate-root.base.ts      # 聚合根基类 (含事件处理)
    │  ├─ domain-event.base.ts        # 领域事件基类
    │  └─ ...
    │
    └─ infrastructure/
       ├─ prisma/
       │  ├─ prisma.module.ts
       │  └─ prisma.service.ts
       └─ messaging/ # 用于未来可靠事件分发 (e.g., Outbox Pattern)
```

这个结构的核心思想是**关注点分离 (Separation of Concerns)** 和 **依赖倒置 (Dependency Inversion)**。依赖关系永远指向中心的 `domain` 层，外部的技术实现细节（如数据库）不能污染纯粹的业务核心。

## 🔧 构建流程：从地基到封顶

我们将分三个阶段来完成这个模版的构建：

### 📦 阶段一：基础设置 (Project & Prisma)

#### 步骤 1.1：初始化项目

**目标**：使用 NestJS CLI 创建一个标准项目。

```bash
npm i -g @nestjs/cli
nest new ddd-template
cd ddd-template
```

#### 步骤 1.2：集成 Prisma

**目标**：安装 Prisma 依赖，并用其 CLI 初始化项目。

```bash
npm install @prisma/client
npm install prisma --save-dev
npx prisma init --datasource-provider postgresql
```

**为什么选择 Prisma？**

我们选择 Prisma 作为 ORM，因为它通过 `schema.prisma` 这个"唯一真理之源"来管理数据模型，并提供类型安全的客户端，与 DDD 的思想天然契合。将数据库连接信息放在 `.env` 中是保证配置与代码分离的安全实践。

#### 步骤 1.3：第一次数据库迁移

在 `schema.prisma` 中定义一个简单的模型，并运行迁移命令。

```prisma
// prisma/schema.prisma
model Product {
  id    String @id @default(cuid())
  name  String
  // ...其他字段
}
```

```bash
npx prisma migrate dev --name init
```

**声明式迁移的优势**：

1. Prisma 在 `prisma/migrations` 目录下生成了一个 SQL 迁移文件
2. 这个 SQL 被应用到你的数据库，创建了 `Product` 表
3. 一个名为 `_prisma_migrations` 的历史表被创建，用于记录已应用的迁移，实现"留痕"

我们将数据库的演进也作为代码的一部分进行版本控制。这种**"声明式迁移"**范式保证了任何环境（开发、测试、生产）的数据库结构都可以通过代码库被精确地、自动化地复现，解决了传统手动变更数据库带来的混乱和风险。

> **⚠️ 重要提醒**：`prisma/migrations` 目录必须与源码一同提交到版本控制中。

### 🏛️ 阶段二：构筑共享核心 (The DDD Toolkit)

这是模板的精髓所在。我们创建一系列可被所有业务模块复用的基础构件。

#### 步骤 2.1：打造全局 PrismaService

**目标**：创建一个可被全局注入的 `PrismaService` 和 `PrismaModule`。

**关键源码** (`src/shared/infrastructure/prisma/prisma.service.ts`)：

```typescript
import { INestApplication, Injectable, OnModuleInit } from "@nestjs/common";
import { PrismaClient } from "@prisma/client";

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit {
  async onModuleInit() {
    await this.$connect();
  }

  async enableShutdownHooks(app: INestApplication) {
    process.on("beforeExit", async () => {
      await app.close();
    });
  }
}
```

**关键源码** (`src/shared/infrastructure/prisma/prisma.module.ts`)：

```typescript
import { Global, Module } from "@nestjs/common";
import { PrismaService } from "./prisma.service";

@Global() // 设为全局模块
@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class PrismaModule {}
```

**优势**：

- 我们可以在任何模块中直接注入 `PrismaService` 而无需在每个模块中单独导入 `PrismaModule`
- 在应用启动时自动连接数据库、在应用关闭时优雅断开连接的 `PrismaService`
- 通过 `@Global()` 装饰器，提供了统一的数据库访问入口，并妥善管理了数据库连接的生命周期

#### 步骤 2.2：定义强大的领域基类

**关键源码** (`src/shared/domain/aggregate-root.base.ts`)：

```typescript
export interface AggregateRootProps {
  id?: string;
  createdAt?: Date;
  updatedAt?: Date;
}

export abstract class AggregateRoot {
  private _domainEvents: IDomainEvent[] = [];

  public get domainEvents(): IDomainEvent[] {
    return this._domainEvents;
  }

  protected addDomainEvent(domainEvent: IDomainEvent): void {
    this._domainEvents.push(domainEvent);
  }

  public clearDomainEvents(): void {
    this._domainEvents = [];
  }

  // 创建聚合根 - 子类必须重写
  public static create(props: any): AggregateRoot {
    throw new Error(`${this.name}.create() must be implemented`);
  }

  // POJO再水合 - 子类必须重写
  public static reconstitute(props: any): AggregateRoot {
    throw new Error(`${this.name}.reconstitute() must be implemented`);
  }
}
```

**设计理念**：

一个高度规范的聚合根基类。它强制所有具体的聚合根都必须明确区分**"创建(Create)"**和**"再水合(Reconstitute)"**这两种不同的对象生命周期。

1. `create()` 方法是业务规则的"堡垒"，用于从无到有创建一个全新的、符合业务不变量的领域对象
2. `fromPersistence()` 方法则是一个纯粹的"数据恢复"通道，用于从数据库中恢复已有对象的状态，它信任持久化层的数据，不执行创建时的业务校验
3. `protected constructor` 阻止了从外部随意 `new` 一个聚合根，强制开发者必须通过意图明确的工厂方法来实例化对象，极大地提升了架构的健壮性

### 🎨 阶段三：构建示例模块 (Putting It All Together)

一个没有实际范例的模板是空洞的。我们将构建一个完整的 `product` 模块，它将作为未来所有业务模块的"活文档"和"最佳实践指南"。

#### 步骤 3.1：创建模块骨架

**目标**：创建 DDD 分层架构所需的所有目录。

```bash
mkdir -p src/modules/product/domain/aggregates
mkdir -p src/modules/product/domain/repositories
mkdir -p src/modules/product/application/services
mkdir -p src/modules/product/application/dtos
mkdir -p src/modules/product/application/mappers
mkdir -p src/modules/product/infrastructure/repositories
mkdir -p src/modules/product/infrastructure/mappers
mkdir -p src/modules/product/presentation
```

#### 步骤 3.2：领域核心 (The Domain Layer)

这是最重要的一步，我们在这里用代码定义"什么是商品"。

##### 1. 定义 ProductAggregate

**关键源码** (`src/modules/product/domain/aggregates/product.aggregate-root.ts`)：

```typescript
import { randomUUID } from "crypto";
import {
  AggregateRoot,
  AggregateRootProps,
} from "../../../../shared/domain/aggregate-root.base";

// 1. 定义Product自身的Props接口
export interface ProductProps extends AggregateRootProps {
  name: string;
  description?: string | null;
  price: number;
  sku: string;
  stock: number;
}

export class ProductAggregate extends AggregateRoot<ProductProps> {
  // 2. 构造函数设为受保护，强制使用工厂方法
  protected constructor(props: ProductProps) {
    super(props);
  }

  // 3. 实现create工厂方法，包含业务校验规则
  public static create(
    props: Omit<ProductProps, "id" | "createdAt" | "updatedAt">
  ): ProductAggregate {
    if (props.price < 0) {
      throw new Error("Product price cannot be negative.");
    }
    if (props.stock < 0) {
      throw new Error("Product stock cannot be negative.");
    }

    const defaultProps: ProductProps = {
      ...props,
      id: randomUUID(),
      createdAt: new Date(),
      updatedAt: new Date(),
    };
    return new ProductAggregate(defaultProps);
  }

  // 4. 实现fromPersistence工厂方法，用于从数据库恢复对象
  public static fromPersistence(props: ProductProps): ProductAggregate {
    return new ProductAggregate(props);
  }

  // 5. 封装业务行为，而不是暴露setter
  public decreaseStock(quantity: number): void {
    if (this.props.stock < quantity) {
      throw new Error("Insufficient stock.");
    }
    this.props.stock -= quantity;
    this.props.updatedAt = new Date();
  }

  // 6. 提供清晰的Getters
  public get name(): string {
    return this.props.name;
  }

  public get price(): number {
    return this.props.price;
  }
  // ... 其他Getters
}
```

**设计亮点**：

- **充血模型**：`ProductAggregate` 不仅仅是数据的容器，它还包含了 `decreaseStock` 这样的业务行为。所有与商品相关的业务规则（如价格不能为负）都被封装在内部，防止了逻辑泄露
- **不变量保护**：`create` 工厂方法是业务规则的"守门人"，确保了任何新创建的商品实例都是有效的
- **封装性**：`protected constructor` 和只读的 `props`，以及通过 `getter` 暴露数据，共同保护了聚合根的内部状态不被外部随意篡改

##### 2. 定义 ProductRepository 接口

**关键源码** (`src/modules/product/domain/repositories/product.repository.ts`)：

```typescript
import { ProductAggregate } from "../aggregates/product.aggregate-root";

export abstract class ProductRepository {
  abstract save(product: ProductAggregate): Promise<void>;
  abstract findById(id: string): Promise<ProductAggregate | null>;
  abstract findBySku(sku: string): Promise<ProductAggregate | null>;
  abstract findAll(): Promise<ProductAggregate[]>;
}
```

**设计理念**：

这是**依赖倒置原则**的体现。领域层不关心数据是存在 PostgreSQL 还是 MongoDB，它只定义了"我需要保存和查找商品"这个需求。这使得我们的核心业务逻辑可以独立于任何数据库技术进行测试和演进。

#### 步骤 3.3：构筑基础设施 (The Infrastructure Layer)

现在，我们来实现上一层定义的"契约"。

##### 1. 实现 PrismaProductRepository

**关键源码** (`src/modules/product/infrastructure/repositories/prisma-product.repository.ts`)：

```typescript
import { Injectable } from "@nestjs/common";
import { PrismaService } from "../../../../shared/infrastructure/prisma/prisma.service";
import { ProductRepository } from "../../domain/repositories/product.repository";
import { ProductAggregate } from "../../domain/aggregates/product.aggregate-root";
import { ProductPrismaMapper } from "../mappers/product.prisma-mapper";

@Injectable()
export class PrismaProductRepository implements ProductRepository {
  constructor(private readonly prisma: PrismaService) {}

  async save(product: ProductAggregate): Promise<void> {
    const persistenceData = ProductPrismaMapper.toPersistence(product);
    await this.prisma.product.upsert({
      where: { id: product.id },
      update: persistenceData,
      create: persistenceData,
    });
  }

  async findById(id: string): Promise<ProductAggregate | null> {
    const prismaProduct = await this.prisma.product.findUnique({
      where: { id },
    });
    // 如果找到了，就用Mapper将其"再水合"成领域对象
    return prismaProduct ? ProductPrismaMapper.toDomain(prismaProduct) : null;
  }
  // ... 其他方法的实现 ...
}
```

**设计理念**：

这是技术细节的"实现区"。所有与 Prisma 相关的代码都被限制在这一层，如果未来需要更换 ORM，我们只需要修改 `infrastructure` 层，而 `domain` 和 `application` 层无需任何改动。

##### 2. 创建 ProductPrismaMapper

**关键源码** (`src/modules/product/infrastructure/mappers/product.prisma-mapper.ts`)：

```typescript
import { Product as PrismaProduct } from "@prisma/client";
import {
  ProductAggregate,
  ProductProps,
} from "../../domain/aggregates/product.aggregate-root";

export class ProductPrismaMapper {
  // 将Prisma模型转换为领域聚合根（再水合）
  static toDomain(prismaProduct: PrismaProduct): ProductAggregate {
    const props: ProductProps = {
      /* ... 属性映射 ... */
    };
    return ProductAggregate.fromPersistence(props);
  }

  // 将领域聚合根转换为Prisma可理解的输入数据
  static toPersistence(
    product: ProductAggregate
  ): Prisma.ProductUncheckedCreateInput {
    // 访问聚合根的内部props，进行转换
    const rawProps = (product as any).props;
    return {
      /* ... 属性映射 ... */
    };
  }
}
```

**设计理念**：

Mapper 是"防腐层"（Anti-Corruption Layer）的一种体现。它隔离了两种不同的模型：一个是为业务逻辑服务的、拥有丰富行为的领域模型；另一个是为数据库存储服务的、扁平的数据模型。这层翻译官的存在，保证了两种模型可以独立演化而不互相污染。

#### 步骤 3.4：编排用例 (The Application Layer)

创建 `product.service.ts` 来编排一个完整的业务用例，如"创建一个新商品"。同时创建 DTOs 用于数据传输。

**关键源码** (`src/modules/product/application/services/product.service.ts`)：

```typescript
import { Inject, Injectable, ConflictException } from "@nestjs/common";
import { ProductRepository } from "../../domain/repositories/product.repository";
import { CreateProductDto } from "../dtos/create-product.dto";
import { ProductAggregate } from "../../domain/aggregates/product.aggregate-root";
import { ProductMapper } from "../mappers/product.mapper";

@Injectable()
export class ProductService {
  constructor(
    @Inject(ProductRepository) // 依赖于抽象，而不是具体实现
    private readonly productRepository: ProductRepository
  ) {}

  async create(createProductDto: CreateProductDto): Promise<ProductDto> {
    // 1. 应用层逻辑：检查SKU是否唯一
    const existing = await this.productRepository.findBySku(
      createProductDto.sku
    );
    if (existing) {
      throw new ConflictException(
        `Product with SKU ${createProductDto.sku} already exists.`
      );
    }

    // 2. 调用领域层的工厂方法创建聚合根
    const product = ProductAggregate.create(createProductDto);

    // 3. 通过仓储持久化
    await this.productRepository.save(product);

    // 4. 使用DTO Mapper转换为安全的数据传输对象返回
    return ProductMapper.toDto(product);
  }
}
```

**设计理念**：

`ProductService` 本身不包含核心业务规则，它像一个"项目经理"，负责协调各个部分（仓储、领域对象）来完成一个完整的用户故事。这种"薄应用层"的设计使得业务流程清晰可见。

#### 步骤 3.5：暴露 API 并最终组装 (Presentation & Module)

1. **创建 ProductController**

   - 这一层只关心 Web 技术（HTTP 方法、路由、状态码等），它将应用的 Web 接口与内部业务逻辑完全解耦

2. **组装 ProductModule**
   - 创建 `product.module.ts`，利用 NestJS 的依赖注入容器，将前面创建的所有 `providers`、`controllers` 等组装起来，尤其是将 `ProductRepository` 这个"抽象"与其 `PrismaProductRepository` 这个"具体"绑定
   - 这是整个模块能够协同工作的"启动器"。它声明了模块内部的依赖关系以及向外部暴露的接口，是高内聚、低耦合模块化设计的核心

## 🚀 模版进阶：面向生产的可靠性设计

一个好的模版不仅要结构清晰，还要为生产环境的复杂性做好准备。

### 1. 可靠的领域事件：事务性发件箱模式

在我们的探讨中，我们认识到一个简单的内存 `EventEmitter` 无法保证事件在生产环境中被可靠地处理。对于 ERP 这类系统，事件丢失是不可接受的。

**解决方案：事务性发件箱 (Transactional Outbox) 模式**

**实现步骤**：

1. 在 `schema.prisma` 中新增一个 `OutboxEvent` 表，用于存储待发布的事件
2. 修改仓储的 `save` 方法，使其在一个数据库事务中，同时**保存业务数据**和**将领域事件存入`OutboxEvent`表**
3. 创建一个独立的后台进程（如 Cron Job），轮询 `OutboxEvent` 表，将"待处理"的事件捞出并进行分发
4. 为分发失败的事件添加重试和告警机制

**核心优势**：

- **保证不丢失**：因为事件的记录和业务数据的保存是原子性的，所以事件绝对不会丢失
- **高性能**：核心业务事务非常快，因为它只写数据库，不等待任何异步处理
- **可追溯**：任何处理失败的事件都有据可查，便于排错和人工干预
- **最终一致性**：这是构建可扩展、高弹性系统的基石

### 2. 领域服务的预留之地

我们的模版虽然只有一个简单的 `product` 模块，没有用到领域服务，但一个完整的 DDD 模板必须为它预留位置（`domain/services` 目录）。

**重要区别**：

- **领域服务**：用于处理需要**同步返回结果**的**跨聚合业务计算**（如复杂的折扣计算）。它与主流程在同一事务中
- **领域事件**：用于通知系统其他部分一个**已发生的事实**，是**异步**的、解耦的，每个处理都在独立事务中

在模版的文档中清晰地说明这一点，能引导未来的开发者在正确的场景下使用正确的工具。

## 📚 如何使用此模版

1. **克隆或复制**整个 `ddd-template` 项目
2. **配置你的数据库**，修改 `.env` 文件
3. **删除或重命名** `src/modules/product` 示例模块
4. **创建你自己的业务模块**，例如 `user` 或 `inventory`，完全遵循 `product` 模块的分层结构和编码范式
