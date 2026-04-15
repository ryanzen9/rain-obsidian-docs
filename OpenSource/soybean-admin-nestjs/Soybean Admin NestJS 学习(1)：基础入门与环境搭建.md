---
Title: Soybean Admin NestJS 学习(1)：基础入门与环境搭建
Draft: false
tags:
  - ERP
  - 开源项目
  - TypeScript
  - NestJS
Author: Ruby Ceng
---

# Soybean Admin NestJS 学习(1)：基础入门与环境搭建

## 1. 项目初始化

### 1.1 创建 NestJS 项目

NestJS 官方的命令行工具（CLI）来创建一个新项目。这会自动生成一个标准的项目骨架，包含所有必需的配置文件和基础代码。

```sh
# 使用 npx 来执行 NestJS CLI
npx @nestjs/cli new my-soybean-backend

# 执行
pnpm start:dev
```

项目结构：

- `src/`: 存放我们所有业务逻辑的源代码
- `test/`: 存放测试文件
- `node_modules/`: 存放项目依赖
- `package.json`: 定义项目依赖和脚本
- `tsconfig.json`: TypeScript 配置文件
- `nest-cli.json`: NestJS CLI 的配置文件

### 1.2 集成 Prisma ORM

Prisma 是一个现代化的数据库工具集，它将帮助我们以类型安全的方式操作数据库。

首先，我们需要安装 Prisma 的客户端和命令行工具。

- @prisma/client: 在我们的代码中与数据库交互的客户端。
- prisma: 开发时使用的命令行工具，用于数据库迁移等操作。

执行命令：

```bash
pnpm add @prisma/client && pnpm add -D prisma && npx prisma init
```

配置 env 文件

```env
# .env
DATABASE_URL="postgresql://root:lfFxGjULezLVgO2DTQAm@10.0.5.134:8432/my-soybean-backend?schema=public"
DIRECT_DATABASE_URL="postgresql://root:lfFxGjULezLVgO2DTQAm@10.0.5.134:8432/my-soybean-backend?schema=public"
```

更新 Prisma 配置

```prisma
// schema.prisma

// This is your Prisma schema file,
// learn more about it in the docs: https://pris.ly/d/prisma-schema

// Looking for ways to speed up your queries, or scale easily with your serverless or edge functions?
// Try Prisma Accelerate: https://pris.ly/cli/accelerate-init

generator client {
  provider = "prisma-client-js"
  // 通常将其输出到 node_modules 中一个更标准的位置。这可以简化我们的导入路径。
  // output   = "@prisma/client"
}

datasource db {
  provider  = "postgresql"
  url       = env("DATABASE_URL")
  directUrl = env("DIRECT_DATABASE_URL")
}
```

## 2. 数据库模型设计

### 2.1 定义 User 模型

编辑 `prisma/schema.prisma` 文件，在里面定义一个 User 模型。这个模型会映射到数据库中的 users 表。

为了起步，我们先定义一些最基础的用户字段，参考原项目但进行简化：

- `id`: 主键，自增整数
- `username`: 用户名，唯一且不能为空
- `password`: 密码，不能为空
- `nickname`: 昵称
- `createdAt`: 创建时间，自动生成
- `updatedAt`: 更新时间，自动更新

```prisma
// schema.prisma
// 添加以下内容
//================================================================================
// User Model
//================================================================================
model User {
  id        Int      @id @default(autoincrement())
  username  String   @unique
  password  String
  nickname  String?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@map("users")
}
```

执行迁移命令：

```bash
npx prisma migrate dev --name init_user_model
```

### 2.2 创建 Prisma 服务模块

创建 Prisma 相关模块与服务：

```bash
# 使用 NestJS 命令
npx @nestjs/cli g module prisma
npx @nestjs/cli g service prisma
```

现在，我们需要修改 `src/prisma/prisma.service.ts` 文件的内容，让它真正地去连接数据库。

这个服务需要做几件事：

1. 继承自 `PrismaClient`，这样它就拥有了所有与数据库模型交互的方法（如 `this.user.findMany()`）
2. 实现 `OnModuleInit` 接口，以确保在应用模块初始化时能够成功连接到数据库

```typescript
// prisma.service.ts
import { Injectable, OnModuleInit } from "@nestjs/common";
import { PrismaClient } from "@prisma/client";

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit {
  async onModuleInit() {
    // This method is called once the module has been initialized.
    // We connect to the database here.    await this.$connect();
  }
}
```

```typescript
// prisma.module.ts
import { Module } from "@nestjs/common";
import { PrismaService } from "./prisma.service";

@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class PrismaModule {}
```

当前结构：

```bash
├── README.md
├── eslint.config.mjs
├── nest-cli.json
├── package.json
├── pnpm-lock.yaml
├── prisma
│   ├── migrations
│   │   ├── 20250723025008_init_user_model
│   │   │   └── migration.sql
│   │   └── migration_lock.toml
│   └── schema.prisma
├── src
│   ├── app.controller.spec.ts
│   ├── app.controller.ts
│   ├── app.module.ts
│   ├── app.service.ts
│   ├── main.ts
│   └── prisma
│       ├── prisma.module.ts
│       ├── prisma.service.spec.ts
│       └── prisma.service.ts
├── test
│   ├── app.e2e-spec.ts
│   └── jest-e2e.json
├── tsconfig.build.json
└── tsconfig.json
```

## 3. 用户模块开发

### 3.1 创建 User 模块结构

与创建 PrismaModule 的流程类似，我们将使用 NestJS CLI 来一次性生成 user 模块所需的所有基础文件。这能确保我们的项目结构保持一致和规范。

```bash
npx @nestjs/cli g module user      # 创建 user 模块
npx @nestjs/cli g service user     # 创建 user 服务，用于处理业务逻辑
npx @nestjs/cli g controller user  # 创建 user 控制器，用于处理 HTTP 请求
```

目标是实现一个查询所有用户的功能。这需要在 UserService 中完成。

首先，我们需要在 UserService 中注入之前创建的 PrismaService，这样我们才能访问数据库。然后，我们创建一个 `findAll` 方法，在该方法中使用 `prisma.user.findMany()` 来获取所有用户记录。

```typescript
// user.service.ts
import { Injectable } from "@nestjs/common";
import { PrismaService } from "src/prisma/prisma.service";

@Injectable()
export class UserService {
  constructor(private readonly prisma: PrismaService) {}

  findAll() {
    return this.prisma.user.findMany();
  }
}

// user.contorller.ts
import { Controller, Get } from "@nestjs/common";
import { UserService } from "./user.service";

@Controller("user")
export class UserController {
  constructor(private readonly userService: UserService) {}

  @Get()
  findAll() {
    return this.userService.findAll();
  }
}

// user.module.ts
import { Module } from "@nestjs/common";
import { UserService } from "./user.service";
import { UserController } from "./user.controller";
import { PrismaModule } from "../prisma/prisma.module";

@Module({
  imports: [PrismaModule],
  providers: [UserService],
  controllers: [UserController],
})
export class UserModule {}
```

### 3.2 数据验证与密码加密

为了实现数据验证和密码加密，我们需要安装几个新的库：

- `class-validator`: 提供了一系列基于装饰器的验证规则
- `class-transformer`: 用于将普通的请求体（plain object）转换为我们定义的 DTO 类的实例
- `bcrypt`: 一个非常流行和安全的库，用于哈希和校验密码
- `@types/bcrypt`: bcrypt 库的 TypeScript 类型定义

执行安装命令：

```bash
pnpm add class-validator class-transformer bcrypt && pnpm add -D @types/bcrypt
```

```typescript
// main.ts
// ... 其他代码 ...
// 加入以下内容：
// Enable global validation pipe
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,
    transform: true, // Automatically transform payloads to DTO instances
  })
);
```

### 3.3 创建用户功能实现

模拟创建用户

```typescript
// create-user.dto.ts
export class CreateUserDto {
  @IsString()
  @IsNotEmpty()
  @MinLength(3)
  username: string;

  @IsString()
  @IsNotEmpty()
  @MinLength(6)
  password: string;

  @IsString()
  @IsOptional()
  nickname?: string;
}

// user.service.ts
// 新增如下：
async create(createUserDto: CreateUserDto) {
  const saltRounds = 10;
  const hashedPassword = await bcrypt.hash(createUserDto.password, saltRounds);

  return this.prisma.user.create({
    data: {
      username: createUserDto.username,
      password: hashedPassword,
      nickname: createUserDto.nickname,
    },
  });
}

// user.controller.ts
@Controller("user")
export class UserController {
  constructor(private readonly userService: UserService) {}

  @Post()
  create(@Body() createUserDto: CreateUserDto) {
    return this.userService.create(createUserDto);
  }

  @Get()
  findAll() {
    return this.userService.findAll();
  }
}
```

## 4. 测试与验证

### 4.1 API 测试

使用 curl 命令测试

```bash
# 这个命令会向 http://localhost:3000/user 发送一个 POST 请求，请求体为一个 JSON 对象
curl -X POST http://localhost:3000/user \
-H "Content-Type: application/json" \
-d '{
  "username": "john_doe",
  "password": "password123",
  "nickname": "Johnny"
}'

# 这个命令发送一个不符合我们 DTO 规则的数据（username 太短）
curl -X POST http://localhost:3000/user \
-H "Content-Type: application/json" \
-d '{
  "username": "jo",
  "password": "password123"
}'
```

## 5. 扩展功能

### 5.1 使用 @nestjs/mapped-types

为了高效地创建更新操作的 DTO（UpdateUserDto），我们将安装一个官方的辅助包 `@nestjs/mapped-types`。它可以基于 CreateUserDto 自动生成一个所有字段都为可选的 DTO。

安装命令：

```bash
pnpm add @nestjs/mapped-types
```

```typescript
// update-user.dto.ts
import { PartialType } from "@nestjs/mapped-types";
import { CreateUserDto } from "./create-user.dto";

export class UpdateUserDto extends PartialType(CreateUserDto) {}
```
