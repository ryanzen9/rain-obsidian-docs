---
Title: 🚀 一、 Prisma 实战指南：从零到一，优雅地构建与演进 NestJS 应用
Draft: false
tags:
  - ORM
  - Prisma
Author: Ruby Ceng
---

## 📦 源码获取

本文涉及的完整项目模版已开源，您可以通过以下方式获取：

### GitHub 仓库

🔗 **项目地址**：[prisma_todo_example](https://github.com/rubyceng/ruby-project-example/tree/main/prisma_todo_example)

## 引言

在现代后端开发中，与数据库的交互是不可或缺的核心环节。一个强大而易用的 ORM (对象关系映射) 工具能极大地提升开发效率和代码质量。Prisma 正是这个领域的佼佼者，它通过 **类型安全**、**自动生成** 和 **直观的迁移系统**，正在彻底改变我们与数据库打交道的方式。

本文将通过一个完整的实战项目——一个从简单到复杂演进的 TODO 应用——来带您深入探索 Prisma 的核心概念与高效工作流。

## 💡 1. Prisma 的核心概念

在深入代码之前，我们先理解 Prisma 的三大核心组件：

- **Prisma Client**: 一个自动生成的、完全类型安全的数据库客户端。你应用中的所有数据库查询都通过它进行，它能确保你的查询在编译时就是类型正确的，告别因字段名写错导致的运行时错误。
- **Prisma Migrate**: 一个声明式的数据库迁移工具。你只需要 `schema.prisma` 文件中定义你期望的数据库 _最终状态_，Prisma Migrate 就会自动生成 SQL 迁移脚本来达到这个状态。
- **Prisma Studio**: 一个内置的可视化数据库 GUI，可以让你方便地浏览和编辑数据库中的数据，非常适合在开发和调试时使用。

---

## 🛠️ 2. 工作流第一阶段：项目初始化与基础构建

我们的第一个需求是创建一个简单的 TODO 应用，可以增删改查 `Todo` 项。

### **步骤 1: 定义你的“唯一事实来源” - `schema.prisma`**

对项目进行初始化

```
# 安装相关依赖
npm i @nestjs/cli

# 安装prisma client -- 与数据库交互的核心
npm i @prisma/client

# 安装prisma js包生成器
npm install prisma --save-dev
```

所有 Prisma 项目的核心都是 `schema.prisma` 文件。它是你数据库结构的 **唯一事实来源 (Single Source of Truth)**。所有数据库的结构变更都应该从这里开始。

在我们的项目中，初始的 `schema.prisma` 文件如下：

```prisma
// prisma/schema.prisma

// ---------------------------------
// 1. 定义数据源 (Datasource)
//    告诉 Prisma 我们要连接哪个数据库
// ---------------------------------
datasource db {
  provider = "sqlite" // 使用 SQLite
  url      = env("DATABASE_URL") // 连接字符串从 .env 文件读取
}

// ---------------------------------
// 2. 定义生成器 (Generator)
//    告诉 Prisma 我们要生成什么类型的客户端
// ---------------------------------
generator client {
  provider = "prisma-client-js"
}

// ---------------------------------
// 3. 定义数据模型 (Model)
//    这会映射为数据库中的一张 `Todo` 表
// ---------------------------------
model Todo {
  id          Int      @id @default(autoincrement())
  title       String
  isCompleted Boolean  @default(false)
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
}
```

> - **`model Todo`**: 定义了一个名为 `Todo` 的模型。
> - **`@id @default(autoincrement())`**: 声明 `id` 字段是主键，并且是自增的。
> - **`@default(false)`** 和 **`@default(now())`**: 为字段提供默认值。
> - **`@updatedAt`**: 特殊的指令，确保每次记录更新时，该字段的时间戳都会自动更新。

### **步骤 2: 从 Schema 到数据库 - 迁移**

定义好模型后，我们需要用它来创建真实的数据库表。这通过 `migrate` 命令完成：

```bash
npx prisma migrate dev --name init
```

这个命令做了四件重要的事：

1.  **比较差异**：将你当前的 `schema.prisma` 与上一次的迁移状态进行比较。
2.  **生成 SQL**：基于差异，生成一个带有时间戳和名称的 SQL 迁移文件，并存放在 `prisma/migrations` 目录下。这使得你的数据库结构变更有了版本控制。
3.  **应用迁移**：执行生成的 SQL，更新你的数据库。
4.  **生成 Client**：自动调用 `prisma generate`，根据最新的 schema 更新 `node_modules/@prisma/client` 中的客户端代码。

### **步骤 3: 在 NestJS 中优雅集成 Prisma**

为了在 NestJS 中使用 Prisma，我们创建了一个可注入的 `PrismaService`。

```typescript
// src/prisma.service.ts
import { Injectable, OnModuleInit } from "@nestjs/common";
import { PrismaClient } from "@prisma/client";

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit {
  // 确保在应用启动时连接数据库
  async onModuleInit() {
    await this.$connect();
  }
}
```

然后，在 `AppModule` 中将其注册为 `provider`，这样就可以在其他服务中通过依赖注入来使用它了。

```typescript
// src/app.module.ts
import { Module } from "@nestjs/common";
import { PrismaService } from "./prisma.service";
// ... 其他导入

@Module({
  imports: [],
  controllers: [AppController],
  providers: [AppService, PrismaService], // 在这里注册
})
export class AppModule {}
```

---

## 🔄 3. 工作流第二阶段：项目演进与结构变更

随着业务发展，需求变更了：我们不再只有一个全局的 TODO 列表，而是需要引入“**计划 (Plan)**”，每个计划拥有自己独立的 TODO 列表。

这需要修改数据库结构，引入 `Plan` 表，并建立与 `Todo` 表的 **一对多关系**。

### **步骤 1: 演进 Schema - 添加关系**

我们再次回到“唯一事实来源”——`schema.prisma` 文件，来描述新的数据结构。

```prisma
// prisma/schema.prisma

model Todo {
  id          Int      @id @default(autoincrement())
  title       String
  isCompleted Boolean  @default(false)
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  // --- ✨ 新增内容 ✨ ---
  // 1. 添加外键字段
  planId Int
  // 2. 定��与 Plan 的关系
  //    - fields: [planId] 指定本模型的外键
  //    - references: [id] 指定关联到 Plan 模型的哪个字段
  //    - onDelete: Cascade 当 Plan 被删除时，其下的所有 Todo 也一并删除
  plan   Plan   @relation(fields: [planId], references: [id], onDelete: Cascade)
}

// --- ✨ 新增模型 ✨ ---
model Plan {
  id        Int      @id @default(autoincrement())
  name      String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  // 定义关系的反向链接，表示一个 Plan 包含一个 Todo 列表
  todos     Todo[]
}
```

### **步骤 2: 再次迁移**

修改完 Schema 后，我们再次运行迁移命令。

```bash
npx prisma migrate dev --name add-plan-model
```

Prisma 会智能地检测到这是一个破坏性更改（因为向现有 `Todo` 表添加了一个没有默认值的非空字段 `planId`），并给出提示。在清空开发数据库后，迁移成功执行，`Plan` 表被创建，`Todo` 表被更新，Prisma Client 也随之更新。

### **步骤 3: 在代码中使用新的关系**

Prisma Client 更新后，我们的代码中就可以直接使用这些关系了。

#### **创建关联数据**

在 `AppService` 中，创建 `Todo` 时必须提供 `planId`。

```typescript
// src/app.service.ts
async createTodo(title: string, planId: number): Promise<Todo> {
  return this.prisma.todo.create({
    data: {
      title,
      planId, // 直接提供 planId 来建立关系
    },
  });
}
```

#### **查询关联数据**

在 `PlanService` 中，我们可以使用 `include` 选项来方便地查询一个计划及其下的所有 TODO。

```typescript
// src/plan/plan.service.ts
async findPlanById(id: number) {
  return this.prisma.plan.findUnique({
    where: { id },
    // `include` 选项让 Prisma 在查询 Plan 的同时，
    // 自动执行一个额外的查询来获取所有关联的 todos
    include: {
      todos: true,
    },
  });
}
```

> `include` 的神奇之处在于，它返回的结果是 **完全类型安全** 的。TypeScript 知道返回的对象会有一个 `todos` 属性，它是一个 `Todo` 数组。

---

## 📜 4. Prisma 工作流总结

通过以上实战，我们可以总结出 Prisma 的核心开发循环：

1.  **✍️ 修改 Schema**: 一切从编辑 `prisma/schema.prisma` 文件开始。无论是添加新模型、修改字段，还是建立关系。
2.  **🚀 运行迁移**: 执行 `npx prisma migrate dev --name <migration-name>`。这会使你的数据库与 Schema 同步，并自动更新你的 Prisma Client。
3.  **💻 编写代码**: 在你的应用服务中，使用刚刚更新的、类型安全的 Prisma Client 来实现你的业务逻辑。
4.  **🔁 重复**: 当新的需求到来时，回到第一步，开始新的循环。

这个清晰、可预测且安全的工作流，正是 Prisma 强大的原因。它将数据库管理的复杂性降到了最低，让开发者可以更专注于业务逻辑的实现。
