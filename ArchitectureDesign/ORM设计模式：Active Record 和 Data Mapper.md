---
Title: ORM设计模式：Active Record 和 Data Mapper
Draft: false
tags:
  - ORM
  - 架构设计
Author: Ruby Ceng
---

## 相关文章

- [https://www.mehdi-khalili.com/orm-anti-patterns-part-1-active-record](https://www.mehdi-khalili.com/orm-anti-patterns-part-1-active-record)
- [https://kore-nordmann.de/blog/why_active_record_sucks.html](https://kore-nordmann.de/blog/why_active_record_sucks.html)

## 引言

在与数据库打交道的应用开发中，[对象关系映射（ORM）](https://zh.wikipedia.org/wiki/对象关系映射)是不可或缺的一环。它允许我们使用面向对象的方式来操作数据库，而不是手写繁琐的 SQL 语句。在众多 ORM 实现中，其核心思想主要源于两种经典的设计模式：**Active Record** 和 **Data Mapper**。理解这两种模式的差异，能帮助我们根据项目需求选择最合适的工具和架构。

这篇笔记将探讨这两种模式的核心理念，并分别使用 `TypeORM` 进行代码示例。

> [TypeORM](https://typeorm.io/) 是一个流行的 TypeScript ORM，它同时支持 **Active Record** 和 **Data Mapper** 两种模式。

## 一、Active Record（活动记录）模式 —— 数据与行为合一

> WIKI 给出的解释如下：
>
> > 在[软件工程](https://en.wikipedia.org/wiki/Software_engineering)中，**活动记录模式**是一种[架构模式](<https://en.wikipedia.org/wiki/Architectural_pattern_(computer_science)>)。它存在于将内存对象数据存储在[关系数据库](https://en.wikipedia.org/wiki/Relational_database)中的软件中。
>
> > [数据库表](https://en.wikipedia.org/wiki/Database_table "Database table")或[视图](<https://www.google.com/search?q=https://en.wikipedia.org/wiki/View_(database)> "View (database)")被包装成一个[类](<https://en.wikipedia.org/wiki/Class_(computer_science)> "Class (computer science)")。因此，一个[对象](<https://en.wikipedia.org/wiki/Object_(computer_science)> "Object (computer science)")实例与表中的一行绑定。对象创建后，保存时会向表中添加一行新数据。任何加载的对象都会从数据库中获取其信息。当对象更新时，表中的相应行也会更新。

**Active Record**（以下简称 AR）模式将数据模型（如一个 `User` 类）与数据库的持久化逻辑紧密地绑定在一起。这意味着，一个模型实例不仅代表了数据表中的一行数据，它自身还包含了 `.save()`、`.update()`、`.remove()` 等直接与数据库交互的方法。使用 AR 的过程中，实际上对对象的操作也就是对数据库表行的操作。

对活动记录模式的主要批评是，由于数据库交互和应用程序逻辑的强耦合，活动记录对象不遵循[单一职责原则](https://zh.wikipedia.org/wiki/单一功能原则)**和**[关注点分离](https://zh.wikipedia.org/wiki/关注点分离) 。

- **优点**：简单直观，上手快，代码量少，非常适合快速原型开发和简单的 CRUD（增删改查）应用。
- **缺点**：违反了**单一职责原则**（数据模型要同时封装并且处理：对数据库表的映射，对数据库表的存取，业务流程的封装），导致业务逻辑与持久化逻辑高度耦合，不利于单元测试和长期维护，本质上将 `POJO` 对象和具体的操作业务封装在了一起。类即表示数据映射对象，又表示业务逻辑类。

`TypeORM` 允许你通过继承 `BaseEntity` 类来实现 **Active Record** 模式。

试想一个例子：

- 假设有一个 `Order` (订单) 对象。在不同的业务场景下，对订单的操作是完全不同的：
  - **用户下单时**：需要检查库存、计算总价、应用优惠券。
  - **仓库发货时**：需要检查地址、调用物流接口、更新订单状态为“已发货”。
  - **财务对账时**：需要计算税费、生成发票。
  - **客服处理退款时**：需要验证退款条件、执行退款操作、更新状态为“已退款”。
- 如果使用 **Active Record** 模式，所有这些不同业务领域的方法（`placeOrder()`、`ship()`、`generateInvoice()`、`processRefund()`）都可能被塞进同一个 `Order` 类里。这个类了解了太多不属于其核心职责的业务，变得异常庞大和复杂，任何一个场景的修改都可能影响到其他场景，最终变得无法维护。

### 场景 A：Active Record 导致的“[上帝对象](https://zh.wikipedia.org/wiki/上帝对象)”

`Order` 实体继承自 `BaseEntity`，这使得它自身就具备了 `.save()`、`.remove()` 等数据库操作能力。然后，我们将所有业务逻辑都堆砌在这个类中。

```typescript
// File: src/entity/OrderGodObject.ts
import { BaseEntity, Entity, PrimaryGeneratedColumn, Column } from "typeorm";

// 模拟外部服务接口
const logisticsAPI = { requestShipment: async () => "TRACK12345" };
const paymentGateway = { charge: async () => true };
const inventorySystem = { checkStock: async () => true };

@Entity()
export class OrderGodObject extends BaseEntity {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  status: string;

  @Column("decimal")
  totalPrice: number;

  @Column()
  shippingAddress: string;

  // 问题所在：所有不同业务领域的逻辑都集中在这里

  // 场景1: 用户下单逻辑
  async placeOrder(userId: number, items: any[]) {
    console.log("[User Context] Placing order...");
    const hasStock = await inventorySystem.checkStock();
    if (!hasStock) throw new Error("Inventory not sufficient");

    this.totalPrice = items.reduce((sum, item) => sum + item.price, 0);
    this.shippingAddress = `Address for user ${userId}`; // 伪代码
    this.status = "awaiting_payment";
    await this.save(); // 直接调用数据库操作

    await paymentGateway.charge();
    this.status = "paid";
    await this.save(); // 再次调用
    console.log("[User Context] Order placed and paid.");
  }

  // 场景2: 仓库发货逻辑
  async ship() {
    console.log("\n[Warehouse Context] Shipping order...");
    if (this.status !== "paid") {
      throw new Error("Cannot ship an unpaid order.");
    }
    const trackingNumber = await logisticsAPI.requestShipment();
    console.log(`[Warehouse Context] Got tracking number: ${trackingNumber}`);
    this.status = "shipped";
    await this.save(); // 直接调用数据库操作
    console.log("[Warehouse Context] Order shipped.");
  }

  // 场景3: 市场营销逻辑
  async applyDiscount(couponCode: string) {
    console.log("\n[Marketing Context] Applying discount...");
    // 伪代码：验证优惠券
    if (couponCode === "SUMMER10") {
      this.totalPrice *= 0.9;
      await this.save(); // 直接调用数据库操作
      console.log("[Marketing Context] Discount applied.");
    }
  }
}
```

### 场景 B：提取逻辑到 Service 层导致的“贫血领域模型”

为了解决“**上帝对象**”问题，我们把逻辑抽离出来，但这让 `Order` 对象本身失去了行为能力。将业务逻辑从实体中提取到 Service 中，最终会得到一个架构，其中的领域逻辑分布在多个服务中，这会导致第二个反模式——**[贫血领域模型](https://martinfowler.com/bliki/AnemicDomainModel.html)**（Anemic Domain Model）。

- 为了不让 `Order` 类那么臃肿，开发者可能会创建 `OrderService`、`ShippingService`、`FinanceService` 等服务类来承载业务逻辑。
- 这看起来更好，但问题在于 `Order` 对象本身现在变得“贫血”了——它几乎只剩下一堆 getter 和 setter 方法，没有任何自己的业务行为，成了一个纯粹的数据容器。
- 此时，要理解一个完整的业务流程（比如“下单”），你可能需要跟踪 `OrderController` -\> `OrderService` -\> `InventoryService` -\> `PaymentGateway` 等一系列调用。核心的业务逻辑被分散到各个角落，系统的整体复杂性并没有降低，只是从一个大的类转移到了众多类之间的交互中，同样难以理解和维护。

```typescript
// File: src/entity/AnemicOrder.ts
import { BaseEntity, Entity, PrimaryGeneratedColumn, Column } from "typeorm";

@Entity()
export class AnemicOrder extends BaseEntity {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  status: string;

  @Column("decimal")
  totalPrice: number;

  @Column()
  shippingAddress: string;

  // 这个类现在很“干净”，但它只是一个数据容器，没有任何业务行为。
}

// File: src/service/OrderServices.ts
import { AnemicOrder } from "../entity/AnemicOrder";

// 逻辑现在分散在不同的服务中
export class OrderPlacementService {
  async placeOrder(userId: number, items: any[]): Promise<AnemicOrder> {
    console.log("[Service] Placing order...");
    const order = new AnemicOrder();
    order.totalPrice = 100; // 伪代码
    order.shippingAddress = `Address for user ${userId}`;
    order.status = "paid";
    await order.save(); // 服务直接操作实体并保存
    return order;
  }
}

export class ShippingService {
  async ship(order: AnemicOrder) {
    console.log("\n[Service] Shipping order...");
    if (order.status !== "paid") {
      throw new Error("Cannot ship an unpaid order.");
    }
    order.status = "shipped";
    await order.save(); // 服务直接修改实体状态并保存
    console.log("[Service] Order shipped.");
  }
}
```

## 二、Data Mapper（数据映射）模式 —— 职责分离，保持模型的纯粹性

> WIKI 给出的解释如下：
>
> > 在[软件工程](https://en.wikipedia.org/wiki/Software_engineering)中，**数据映射器模式**是一种[架构模式](<https://en.wikipedia.org/wiki/Architectural_pattern_(computer_science)>)。符合此模式的对象接口将包含创建、读取、更新和删除等功能，这些功能对表示数据存储中域实体类型的对象进行操作。
>
> > **数据映射器**是一个[数据访问层](https://en.wikipedia.org/wiki/Data_access_layer)，它在持久数据存储（通常是[关系数据库](https://en.wikipedia.org/wiki/Relational_database)）和内存数据表示（领域层）之间执行双向数据传输。该模式的目标是使内存数据和持久数据存储彼此独立，并独立于数据映射器本身。

**Data Mapper** 将[富领域模型](https://martinfowler.com/bliki/AnemicDomainModel.html)（Rich Domain Model）与数据库持久化逻辑（映射器/仓库 `Mapper/Repository`）彻底分离。也就是抽象出数据持久层，负责将内存中的业务模型（Domain Model）的更改持久化到数据库中。它不保存对象状态，只提供对数据库操作的封装。而领域模型内部则采用**充血模型**，封装自己领域生命周期内的业务方法。

现在，我们将使用 **Data Mapper** 模式来重构上面例子。核心思想是**将领域模型与持久化模型彻底分离**。

我们创建三个部分：

1.  **富领域模型 (Rich Domain Model)**: `Order` 类，一个纯粹的、包含业务逻辑的 TypeScript 类。
2.  **持久化实体 (Persistence Entity)**: `OrderEntity` 类，一个用于 TypeORM 映射数据库表的“贫血”类。
3.  **数据映射器/仓库 (Data Mapper/Repository)**: `OrderRepository`，负责在 `Order` 和 `OrderEntity` 之间进行转换和持久化。

### 富领域模型 (`Order` Class)

这个类是业务的核心。它不依赖任何框架，不包含任何数据库相关的代码。

```typescript
// File: src/domain/Order.ts

// 定义订单可能的状态，提供类型安全
export type OrderStatus = "pending" | "paid" | "shipped" | "cancelled";

export class Order {
  // 属性是私有的，通过方法暴露，保护了对象的不变性
  private readonly _id: number;
  private _status: OrderStatus;
  private _totalPrice: number;
  private _shippingAddress: string;

  // 工厂方法或构造函数用于创建实例
  constructor(
    id: number,
    status: OrderStatus,
    totalPrice: number,
    shippingAddress: string
  ) {
    this._id = id;
    this._status = status;
    this._totalPrice = totalPrice;
    this._shippingAddress = shippingAddress;
  }

  // 提供对外的只读访问器
  get id(): number {
    return this._id;
  }
  get status(): OrderStatus {
    return this._status;
  }
  get totalPrice(): number {
    return this._totalPrice;
  }
  get shippingAddress(): string {
    return this._shippingAddress;
  }

  // --- 业务逻辑被封装在领域模型内部 ---

  // 支付逻辑
  public pay() {
    if (this._status !== "pending") {
      throw new Error("Only pending orders can be paid.");
    }
    this._status = "paid";
    console.log(`[Domain] Order ${this._id} status changed to 'paid'`);
  }

  // 发货逻辑
  public ship() {
    if (this._status !== "paid") {
      throw new Error("Only paid orders can be shipped.");
    }
    this._status = "shipped";
    console.log(`[Domain] Order ${this._id} status changed to 'shipped'`);
  }

  // 折扣逻辑
  public applyDiscount(percentage: number) {
    if (this._status !== "pending") {
      throw new Error("Discount can only be applied to pending orders.");
    }
    if (percentage <= 0 || percentage >= 1) {
      throw new Error("Invalid discount percentage.");
    }
    this._totalPrice *= 1 - percentage;
    console.log(
      `[Domain] Discount applied to order ${this._id}. New price: ${this._totalPrice}`
    );
  }

  // 取消订单
  public cancel() {
    if (this._status === "shipped") {
      throw new Error("Cannot cancel a shipped order.");
    }
    this._status = "cancelled";
    console.log(`[Domain] Order ${this._id} has been cancelled.`);
  }

  // 工厂方法，用于从无到有地创建订单
  public static create(userId: number, items: any[]): Order {
    console.log("[Domain] Creating a new order...");
    const totalPrice = items.reduce((sum, item) => sum + item.price, 0);
    const shippingAddress = `Address for user ${userId}`;
    // 新创建的订单没有ID，状态是 pending
    // 在真实场景中，ID 会在保存后由数据库生成
    return new Order(null, "pending", totalPrice, shippingAddress);
  }
}
```

### 持久化实体 (`OrderEntity` Class)

这个类非常“贫血”，它的唯一职责就是告诉 `TypeORM` 如何将数据存入数据库。

```typescript
// File: src/infrastructure/OrderEntity.ts
import { Entity, PrimaryGeneratedColumn, Column } from "typeorm";
import { OrderStatus } from "../domain/Order";

@Entity({ name: "orders" }) // 显式指定表名
export class OrderEntity {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  status: OrderStatus; // 使用我们定义的类型

  @Column("decimal")
  totalPrice: number;

  @Column()
  shippingAddress: string;
}
```

### 数据映射器/仓库 (`OrderRepository`)

这是连接领域和持久化的桥梁。

```typescript
// File: src/infrastructure/OrderRepository.ts
import { getManager, Repository } from "typeorm";
import { Order } from "../domain/Order";
import { OrderEntity } from "./OrderEntity";

// Mapper/Repository 类
export class OrderRepository {
  private ormRepository: Repository<OrderEntity>;

  constructor() {
    // 获取 TypeORM 的标准 repository
    this.ormRepository = getManager().getRepository(OrderEntity);
  }

  // 映射器：从持久化实体转换为领域对象
  private toDomain(entity: OrderEntity): Order {
    if (!entity) return null;
    return new Order(
      entity.id,
      entity.status,
      entity.totalPrice,
      entity.shippingAddress
    );
  }

  // 映射器：从领域对象转换为持久化实体
  private toPersistence(domain: Order): Partial<OrderEntity> {
    return {
      id: domain.id,
      status: domain.status,
      totalPrice: domain.totalPrice,
      shippingAddress: domain.shippingAddress,
    };
  }

  // 公共方法：通过 ID 查找
  public async findById(id: number): Promise<Order | null> {
    const entity = await this.ormRepository.findOne({ where: { id } });
    return this.toDomain(entity);
  }

  // 公共方法：保存（创建或更新）
  public async save(order: Order): Promise<Order> {
    console.log(`[Repository] Saving order ${order.id}...`);
    const persistenceData = this.toPersistence(order);
    const savedEntity = await this.ormRepository.save(persistenceData);
    console.log(`[Repository] Order saved with ID: ${savedEntity.id}`);
    // 将保存后（可能包含新ID）的实体转换回领域对象返回
    return this.toDomain(savedEntity);
  }
}

// --- 在应用层/服务层中使用 ---

async function main() {
  // 假设 TypeORM 连接已建立
  const orderRepository = new OrderRepository();

  // 1. 创建订单
  const newOrder = Order.create(123, [{ price: 50 }, { price: 75 }]);
  newOrder.applyDiscount(0.1);

  // 2. 通过 Repository 保存
  const savedOrder = await orderRepository.save(newOrder);

  // 3. 从数据库中取出并执行业务操作
  const retrievedOrder = await orderRepository.findById(savedOrder.id);
  if (retrievedOrder) {
    retrievedOrder.pay();
    retrievedOrder.ship();

    // 4. 再次保存状态变更
    await orderRepository.save(retrievedOrder);
  }
}
```

## 三、总结

**AR** 被称为 ORM anti-patterns（ORM [反模式](https://zh.wikipedia.org/wiki/反面模式)），通常被认为不够现代和正确。**AR** 对于不太复杂的领域逻辑来说是一个不错的选择，比如创建、读取、更新和删除。基于单个记录的派生和验证在此结构中效果很好。

但是，实现捷径和反模式总是始于“在这种情况下会更简单”。`AR` 是数据库记录的 1:1 表示，而 ORM 的重点在于解决数据库结构和领域对象之间的\*\*[对象关系阻抗不匹配](https://www.google.com/search?q=https://en.wikipedia.org/wiki/Object%E2%80%93relational_impedance_mismatch)\*\*问题。使用 `AR` 时，你无法解决这种阻抗不匹配问题，因为你的 `AR` 对象代表的是数据库行，你实际上是将数据库布局与对象绑定在一起。操作对象的同时也就意味着是在操作表行。

实际上这样反而限制了领域对象的封装。不同的业务场景下，插入更新的顺序可能是不同的，我们将被迫往对象上塞入不同领域的封装的不同方法。且如果不同的领域间需要自己特定的属性，对象上会封装很多领域属性，会变得相当臃肿难以维护。

将逻辑从实体中提取业务方法到一些 Service 中，这会将您的领域变成一个不理想的**贫血领域模型**；但这比很多**上帝对象**（God Objects）都要好。在花费了大量精力来清理实体之后，您最终会得到一个架构，其中的领域逻辑分布在多个服务中，领域的生命周期依旧散落在各个 Service 并迷失在这些 Service 之间的交互中。一切都混合在一起。

同时由于对象与数据库的一比一映射，也就意味着如果你必须模拟，或者连接测试库进行测试。

通过对比，我们可以清晰地看到 **Data Mapper** 模式带来的巨大优势：

1.  **关注点分离 (Separation of Concerns)**：

    - **`Order` (领域)**: 只关心业务规则、状态流转和逻辑。它完全不知道数据库的存在。
    - **`OrderEntity` (持久化)**: 只关心如何映射到数据库表结构。
    - **`OrderRepository` (映射)**: 只关心如何在两者之间转换和存取。

2.  **避免反模式**：

    - 它**不是上帝对象**，因为 `Order` 类只包含与订单自身相关的核心业务逻辑，不包含持久化代码。
    - 它**不是贫血模型**，因为 `Order` 类是“富”的，它封装了数据和操作这些数据的行为，能自我验证和保护状态。

3.  **可测试性**：

    - 你可以独立地测试 `Order` 领域类的所有业务逻辑，无需模拟数据库或任何外部框架，这使得单元测试变得极其简单和快速。

4.  **灵活性和可维护性**：

    - 如果需要更换 ORM 框架，甚至从 SQL 换成 NoSQL 数据库，你只需要重写 `OrderRepository` 和 `OrderEntity`，核心的 `Order` 领域模型完全不受影响。
    - 业务逻辑集中在领域模型中，使得新开发者能更快地理解系统的核心业务。

## 四、结论

虽然 **Data Mapper** 模式需要编写更多的代码（一个领域类、一个实体类、一个仓库类），但对于具有一定复杂性、需要长期维护和演进的系统来说，这种前期投入所换来的架构清晰度、可维护性和健壮性是完全值得的。
