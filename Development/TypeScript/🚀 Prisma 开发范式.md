---
Title: 🚀 Prisma 开发范式
Draft: false
tags:
  - ORM
  - Prisma
Author: Ruby Ceng
---

# 🚀 Prisma 开发范式

## Prisma 文件结构

一个典型的 `Schema.prisma` 文件主要由三个部分组成：

- **数据源 (Datasource)**: 配置数据库连接。
- **生成器 (Generator)**: 指定根据此 Schema 要生成什么（通常是 Prisma Client）。
- **数据模型 (Data Model)**: 定义你的应用程序实体和它们之间的关系。

### 数据源 (Datasource)

```prisma
// 数据库链接信息
datasource db {
  provider  = "postgresql" // 数据库类型
  url       = env("DATABASE_URL") // 数据库连接字符串
  directUrl = env("DIRECT_DATABASE_URL") // 直接连接字符串
  schemas   = ["public"] // 数据库模式, 默认是 public, 可以指定多个 schema
}
```

**关键字详解：**

- `datasource db`: `db` 是这个数据源的名称，你可以自定义，但 `db` 是社区通用惯例。
- `provider`: 指定你使用的数据库类型。常见的值有：
  - `"postgresql"`: PostgreSQL
  - `"mysql"`: MySQL
  - `"sqlite"`: SQLite
  - `"sqlserver"`: Microsoft SQL Server
  - `"mongodb"`: MongoDB
  - `"cockroachdb"`: CockroachDB
- `url`: 数据库的连接字符串。强烈建议使用 `env("DATABASE_URL")` 的方式从环境变量中读取，而不是直接硬编码在文件中。这可以保护你的敏感信息，也便于在不同环境（开发、测试、生产）中切换。
- `directUrl` (可选): 如果你需要直接连接到数据库，而不是通过 Prisma 的连接池，你可以使用 `directUrl`。
- `schemas` (可选): 如果你需要连接到多个数据库模式，你可以指定多个模式。

### 生成器 (Generator)

`generator` 块告诉 Prisma CLI 在你运行 `prisma generate` 命令时要生成哪些资产。最常见的生成器就是 `prisma-client-js`，它会根据你的数据模型生成一个类型安全的 Node.js 客户端。

```prisma
generator client {
  provider = "prisma-client-js"
}
```

**关键字详解：**

- `generator client`: `client` 是生成器的名称，可以自定义，但 `client` 是生成 Prisma Client 时的标准命名。
- `provider`: 指定要使用的生成器包。`"prisma-client-js"` 是默认且最常用的。
- `output` (可选): 指定生成的 Prisma Client 的输出目录。默认是 `node_modules/.prisma/client`。
- `binaryTargets` (可选): 如果你需要在与当前开发环境不同的操作系统上运行（例如，在 macOS 上开发，部署到 Linux 服务器），你需要指定二进制目标。

### 数据模型 (Data Model)

```prisma
/**
 * 模型名称
 */
model ModelName {
    fieldName FieldType [属性/修饰符] @map("field_name") // 字段名
    id        String   @id @default(uuid())    // 主键
    createdAt DateTime @default(now())         // 创建时间
    updatedAt DateTime @updatedAt             // 更新时间
    createdBy String                          // 创建人
    updatedBy String                          // 更新人
    isDeleted Boolean  @default(false)      // 是否删除
    isActive  Boolean  @default(true)       // 是否激活

    @@map("数据库表名")
}
```

#### 模型声明规范

##### 模型和字段命名规范

- **模型名 (ModelName)**: 使用帕斯卡命名法 (PascalCase)，例如 `User`, `BlogPost`。这会映射为 Prisma Client 中的 `prisma.user` 和 `prisma.blogPost`。
- **字段名 (fieldName)**: 使用驼峰命名法 (camelCase)，例如 `firstName`, `createdAt`。
- **列名 (column_name)**: 使用蛇形命名法 (snake_case)，例如 `first_name`, `created_at`。

##### 字段类型

Prisma 支持一系列标量类型，它们会映射到具体的数据库类型。

| Prisma 类型 | 描述                       | 示例                   |
| :---------- | :------------------------- | :--------------------- |
| `String`    | 字符串                     | `"Hello World"`        |
| `Int`       | 整数                       | `123`                  |
| `Float`     | 浮点数                     | `12.34`                |
| `Boolean`   | 布尔值 (`true`/`false`)    | `true`                 |
| `DateTime`  | 日期和时间 (ISO 8601)      | `2025-08-15T10:00:00Z` |
| `Json`      | JSON 对象                  | `{ "foo": "bar" }`     |
| `Bytes`     | 字节数组                   |                        |
| `BigInt`    | 64 位整数                  | `9223372036854775807`  |
| `Decimal`   | 用于精确计算的定点十进制数 | `12.3456`              |

##### 类型修饰符

- `?` (可选): 表示该字段可以为 `NULL`。
- `[]` (列表/数组): 表示该字段是一个数组。这只在支持数组类型的数据库（如 PostgreSQL）中原生支持。

##### 属性/修饰符

**字段属性 (Field Attributes - 以 `@` 开头):**

- `@id`: 将字段标记为主键。
  - `@id @default(autoincrement())`: 自增整数主键 (常见于 SQL 数据库)。
  - `@id @default(cuid())`: 生成一个抗碰撞的唯一 ID (CUID)。
  - `@id @default(uuid())`: 生成一个通用唯一标识符 (UUID)。
- `@unique`: 为该字段添加唯一约束，确保所有记录在该列上的值都是唯一的。
- `@default(...)`: 设置字段的默认值。
  - `@default(now())`: 默认值为当前时间戳。
  - `@default(true)`: 默认值为布尔值 `true`。
  - `@default("some_value")`: 默认值为一个静态字符串。
- `@updatedAt`: 每次记录更新时，自动将该字段的值更新为当前时间戳。
- `@relation(...)`: 定义两个模型之间的关系。我们将在下一部分详细讲解。
- `@map(...)`: 将 Prisma Schema 中的字段名映射到数据库中不同的列名。这非常有用，因为 Prisma 推荐使用 `camelCase` 命名，而许多数据库规范推荐使用 `snake_case`。
  - `@map("column_name_in_db")`

**块级属性 (Block Attributes - 以 `@@` 开头):**

- `@@id([...])`: 定义一个复合主键（由多个字段组成）。
- `@@unique([...])`: 定义一个复合唯一约束。
- `@@index([...])`: 在一个或多个字段上创建数据库索引，以优化查询性能。
- `@@map(...)`: 将 Prisma Schema 中的模型名映射到数据库中不同的表名。
  - `@@map("table_name_in_db")`

## Prisma 中的模型关系

### 字段 (Field)

字段 (Field) 分为两种：

- 关系字段 (Relation Fields)
- 关系标量字段 (Relation Scalar Fields)

- **关系字段**: 这是一个在你的 `model` 中定义的字段，其类型是另一个 `model`。它不存在于数据库的表中，仅用于 Prisma Client，让你能够方便地访问关联数据。例如，`author User`。
- **关系标量字段**: 这就是数据库中实际存储的外键列。它通常是一个 `Int` 或 `String` 类型，并使用 `@relation` 属性与关系字段关联。例如，`authorId Int`。

### 一对多关系

**场景**: 一个 `User` 可以发布多篇 `Post`。一篇 `Post` 只属于一个 `User`。

**实现**:

- 在“一”的那一方 (`User`)，关系字段是一个模型数组，例如 `posts Post[]`。
- 在“多”的那一方 (`Post`)，同时包含关系字段 (`author User`) 和关系标量字段/外键 (`authorId Int`)。

```prisma
model User {
    id    String @id @default(uuid())

    // 关系字段：表示一个用户可以有多篇文章
    // 这个字段不存在于数据库 "users" 表中
    posts Post[]
}

model Post {
    id       String @id @default(uuid())

    // 关系标量字段 (外键)
    // 这个字段实际存在于数据库 "posts" 表中
    authorId Int

    // 关系字段：表示一篇文章属于一个用户
    // @relation 属性是关系的核心定义
    author   User   @relation(fields: [authorId], references: [id])
}
```

### 一对一关系

**场景**: 一个 `User` 只能有一个 `Profile`。一个 `Profile` 只能属于一个 `User`。

```prisma
model User {
    id      Int      @id @default(autoincrement())
    email   String   @unique

    // 关系字段：可选的 Profile。表示一个用户可能有一个 Profile
    // 注意这里的类型是 Profile? 而不是 Profile[]
    profile Profile?
}

model Profile {
    id     Int    @id @default(autoincrement())
    bio    String

    // 关系字段
    user   User   @relation(fields: [userId], references: [id])

    // 关系标量字段 (外键)
    // 必须添加 @unique 约束来保证一对一关系
    userId Int    @unique
}
```

### 多对多关系

**场景**: 一篇博文可以有多个分类。一个分类可以有多个博文。

Prisma 支持两种方式来定义多对多关系：

**a. 隐式多对多关系 (Implicit Many-to-Many) - 推荐**

```prisma
model Post {
    id         Int        @id @default(autoincrement())
    title      String

    // 关系字段：一篇 Post 可以有多个 Category
    categories Category[]
}

model Category {
    id    Int    @id @default(autoincrement())
    name  String @unique

    // 关系字段：一个 Category 可以有多篇 Post
    posts Post[]
}
```

**b. 显式多对多关系 (Explicit Many-to-Many)**

**场景**: 我们不仅想知道哪些 `Post` 属于哪些 `Category`，还想知道是谁以及在什么时间将它们关联起来的。

**实现**: 你需要手动创建一个代表“连接”的模型 (Join Table)。

```prisma
model Post {
    id         Int                 @id @default(autoincrement())
    title      String

    // 关系字段：指向中间表
    categories CategoriesOnPosts[]
}

model Category {
    id    Int                 @id @default(autoincrement())
    name  String              @unique

    // 关系字段：指向中间表
    posts CategoriesOnPosts[]
}

// 显式的中间表模型
model CategoriesOnPosts {
    post       Post     @relation(fields: [postId], references: [id])
    postId     Int // 外键，指向 Post
    category   Category @relation(fields: [categoryId], references: [id])
    categoryId Int // 外键，指向 Category

    // 额外的元数据
    assignedAt DateTime @default(now())
    assignedBy String

    // 定义复合主键，确保 post 和 category 的组合是唯一的
    @@id([postId, categoryId])
}
```

### @relation 属性

`@relation` 属性是控制关系行为的关键，它包含多个可选参数。

- `name: String`: 用于区分同一对模型之间的多个关系。例如，一个 `User` 模型可能有一个 `sentMessages` 字段和 `receivedMessages` 字段，它们都与 `Message` 模型相关。此时就需要 `name` 来区分。
- `fields: Field[]`: 指定当前模型中用作外键的字段。
- `references: Field[]`: 指定关联模型中被引用的字段（通常是主键）。
- `onDelete: Action`: 定义当被引用的记录被删除时，当前记录应执行的操作。
  - `Cascade`: 级联删除。删除 `User` 时，其所有 `Post` 也会被删除。
  - `Restrict`: (默认行为) 禁止删除。如果一个 `User` 还有关联的 `Post`，则无法删除该 `User`。
  - `SetNull`: 将外键字段设置为 `NULL`。这要求外键字段必须是可选的（例如 `authorId Int?`）。
  - `NoAction`: 类似于 `Restrict`，但行为可能因数据库而异。
- `onUpdate: Action`: 定义当被引用的记录的主键更新时，外键应执行的操作。最常见的是 `Cascade`（级联更新）。

### 补充：一言以蔽之，建模主要双方表都要对关系字段进行声明，再通过标量字段或者连接表来表示双方间的关系。

## Prisma Schema 的编写规范

编写 Schema 不仅仅是为了让 Prisma 能看懂，更是为了让你和你的团队成员能轻松地阅读和维护。一个好的 Schema 就像一份清晰的技术文档。

### 命名规范 (Naming Conventions)

- **模型 (Models)**: 使用帕斯卡命名法 (PascalCase)，单数形式。
  - 如: `model User`, `model BlogPost`, `model ProductCategory`
- **字段 (Fields)**: 使用驼峰命名法 (camelCase)。
  - 如: `firstName`, `publishedAt`, `authorId`

### 枚举 (Enums)

- **枚举类型名**: 使用帕斯卡命名法 (PascalCase)。
- **枚举值**: 使用全大写蛇形命名法 (SCREAMING_SNAKE_CASE)。
  - 如: `enum Role { ADMIN, USER }`

### 使用 @map 和 @@map 隔离命名风格

**问题**: Prisma 推荐 `camelCase`，但很多 SQL 数据库（如 PostgreSQL）的传统规范是使用 `snake_case` 来命名表和列（例如 `blog_posts` 表和 `created_at` 列）。

**解决方案**: 在 Prisma Schema 中使用 `camelCase`，然后使用 `map` 属性将其映射到数据库的 `snake_case`。

- `@@map` 用于模型（映射表名）。
- `@map` 用于字段（映射列名）。

**示例**:

```prisma
model BlogPost {
    id          Int      @id @default(autoincrement())
    title       String
    publishedAt DateTime @map("published_at") // 字段映射
    authorId    Int      @map("author_id") // 字段映射

    // ... 关系字段

    @@map("blog_posts") // 模型映射
}
```

这样做的好处:

- **应用层代码清晰**: 在你的 TypeScript/JavaScript 代码中，你可以写 `blogPost.publishedAt`，这非常符合 JS 生态的习惯。
- **数据库层规范**: 在数据库中，表和列是 `blog_posts` 和 `published_at`，这让数据库管理员或使用原生 SQL 查询的人感到舒适。

### 字段顺序的逻辑分组

在一个模型内部，保持字段的逻辑顺序能极大地提升可读性。虽然 Prisma 对顺序没有强制要求，但社区形成了一套事实标准。

**推荐顺序**:

1.  **ID 字段**: 主键永远放在最前面。
2.  **唯一标识符和核心属性**: 例如 `email`, `username`, `title`。
3.  **普通标量字段**: 例如 `firstName`, `bio`, `content`。
4.  **时间戳字段**: `createdAt` 和 `updatedAt`。
5.  **关系标量字段 (外键)**: 例如 `authorId`, `profileId`。
6.  **关系字段**: 放在对应的关系标量字段之后。例如 `author User`。
7.  **块级属性**: `@@id`, `@@unique`, `@@map` 等放在模型定义的最后。

**示例**:

```prisma
model Post {
    // 1. ID 字段
    id        Int      @id @default(autoincrement())

    // 2. 核心属性
    title     String
    slug      String   @unique

    // 3. 普通标量字段
    content   String?
    published Boolean  @default(false)

    // 4. 时间戳字段
    createdAt DateTime @default(now()) @map("created_at")
    updatedAt DateTime @updatedAt @map("updated_at")

    // 5. 关系标量字段 (外键)
    authorId  Int      @map("author_id")

    // 6. 关系字段
    author    User     @relation(fields: [authorId], references: [id], onDelete: Cascade)

    // 7. 块级属性
    @@map("posts")
}
```

### 优先使用必填字段

除非一个字段在业务逻辑上确实是可选的，否则都应该将其设置为必填。

这有助于保证数据的完整性。在 Schema 层面强制要求，比在应用代码中到处做非空检查要好得多。

### 使用枚举 (Enums) 替代魔术字符串

如果一个字段只能取有限的几个值（例如：角色、状态、类型），一定要使用 `enum`。

这能提供编译时的类型安全，防止拼写错误，并让代码意图更清晰。

### 添加注释

`schema.prisma` 支持注释。对于复杂的业务逻辑或不那么直观的字段，请务必添加注释。

- `//` 用于普通注释，解释你的意图。
- `///` 用于文档注释，这些注释会被包含到生成的 Prisma Client 的 JSDoc/TSDoc 中，当你在 VS Code 中悬停在字段上时会显示出来。

**示例**:

```prisma
model Order {
    id Int @id @default(autoincrement())

    /// 订单状态：0=待支付, 1=已支付, 2=已发货, 3=已完成, -1=已取消
    /// 避免使用 Enum，因为状态未来可能会通过后台配置动态增加
    status Int @default(0)
}
```
