---
Title: Soybean Admin NestJS 学习(4)：使用Casbin实现用户角色授权策略
Draft: false
tags:
  - ERP
  - 开源项目
  - TypeScript
  - NestJS
Author: Ruby Ceng
---

# Soybean Admin NestJS 学习(4)：使用 Casbin 实现用户角色授权策略

## 1. 引入 Casbin 实现基于角色的访问控制 (RBAC)

### 1.1 授权流程概述

**完整授权流程**：用户请求 → 登录认证 → 接口鉴权 → 放行

1. **JWT 守卫**：JwtAuthGuard 验证 Bearer Token
2. **权限守卫**：AuthZGuard 使用 Casbin 检查权限
3. **角色获取**：从 Redis 缓存读取用户角色（如果需要）
4. **权限验证**：enforcer.enforce(role, resource, action, domain)

### 1.2 为什么需要 RBAC

到目前为止，我们只解决了"你是谁"的问题（认证）。任何一个成功登录的用户，都可以访问 `GET /user` 接口。但一个真正的系统需要更精细的控制："你能做什么？"（授权）。例如：

- **管理员 (admin)**：应该能访问所有接口
- **普通用户 (user)**：可能只能查看自己的信息，不能查看用户列表
- **游客 (guest)**：在登录前，除了登录接口外，什么都不能访问

### 1.3 Casbin 的核心思想

Casbin 将权限逻辑从代码中抽离出来，用一套简单的访问控制模型来管理。最经典的 PERM 模型是：

```
p, sub, obj, act
```

**参数说明：**

- `p`: policy，表示这是一条策略规则
- `sub`: subject，主体，即"谁"，通常是用户 ID 或角色 ID
- `obj`: object，对象，即"想要访问什么资源"，通常是 API 路径（如 `/user`）
- `act`: action，动作，即"想做什么操作"，通常是 HTTP 方法（如 `GET`, `POST`）

**示例策略：**

```
p, admin, /user, GET
```

它的意思是：角色为 `admin` 的主体，允许对 `/user` 这个资源执行 `GET` 操作。

我们将把这些策略规则存入数据库中。这样，当一个请求进来时，我们只需要查询数据库，用 Casbin 引擎判断一下当前用户是否匹配某条策略即可。

## 2. 数据库配置

### 2.1 更新数据库 Schema

添加 CasbinRule（存储 Casbin 策略）模型：

```prisma
// schema.prisma
model CasbinRule {
  id    Int     @id @default(autoincrement())
  ptype String
  v0    String?
  v1    String?
  v2    String?
  v3    String?
  v4    String?
  v5    String?

  @@map("casbin_rule")
}
```

### 2.2 执行数据库迁移

```bash
pnpm prisma migrate dev --name add_rbac_models
```

## 3. Casbin 访问控制模型配置

### 3.1 定义访问控制模型

使用 Casbin 需要使用到 `enforcer`，它接受两个参数：模型文件路径`model.conf`和策略文件路径（在数据库中则是实现自定义的适配器`adapter`）。

创建 `model.conf`：

```conf
# resources/model.conf
# Casbin RBAC model configuration

[request_definition]
r = sub, obj, act, dom

[policy_definition]
p = sub, obj, act, dom, eft

[role_definition]
g = _, _, _

[policy_effect]
e = some(where (p.eft == allow)) && !some(where (p.eft == deny))

[matchers]
m = g(r.sub, p.sub, r.dom) && r.obj == p.obj && r.act == p.act && r.dom == p.dom
```

### 3.2 创建权限控制守卫

这个守卫的职责是：

1. 获取请求方法上声明的权限点，没有设定默认放行
2. 在每个受保护的请求到达时，从 `req.user` 中获取当前用户的角色
3. 获取当前请求的 API 路径（`req.path`）和 HTTP 方法（`req.method`）
4. 调用 `CasbinService.enforce()` 方法，将用户的角色、请求路径和方法传递过去，判断用户是否有权限
5. 如果有权限，则放行；否则，抛出 `ForbiddenException` (403 Forbidden) 异常

```typescript
// authz.guard.ts
@Injectable()
export class AuthZGuard implements CanActivate {
  constructor(
    private readonly reflector: Reflector,
    @Inject(AUTHZ_ENFORCER) private readonly enforcer: casbin.Enforcer,
    @Inject(AUTHZ_MODULE_OPTIONS) private readonly options: AuthZModuleOptions
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    // 1. 获取方法上的权限声明
    const permissions: Permission[] = this.reflector.get<Permission[]>(
      PERMISSIONS_METADATA,
      context.getHandler()
    );

    if (!permissions) {
      return true; // 没有权限要求，直接通过
    }

    // 2. 从上下文获取当前用户
    const user = this.options.userFromContext(context);
    if (!user) {
      throw new UnauthorizedException();
    }

    // 3. 从Redis获取用户角色
    const userRoles = await RedisUtility.instance.smembers(
      `${CacheConstant.AUTH_TOKEN_PREFIX}${user.uid}`
    );

    if (userRoles && userRoles.length <= 0) {
      return false; // 没有角色，拒绝访问
    }

    // 4. 验证所有权限 (AND 逻辑)
    return await AuthZGuard.asyncEvery<Permission>(
      permissions,
      async (permission) =>
        this.hasPermission(
          new Set(userRoles),
          user.domain,
          permission,
          context,
          this.enforcer
        )
    );
  }

  // 权限验证核心逻辑
  async hasPermission(
    roles: Set<string>,
    domain: string,
    permission: Permission,
    context: ExecutionContext,
    enforcer: casbin.Enforcer
  ): Promise<boolean> {
    const { resource, action } = permission;

    // 5. 对每个角色进行权限检查 (OR 逻辑)
    return AuthZGuard.asyncSome<string>(Array.from(roles), async (role) => {
      return enforcer.enforce(role, resource, action, domain);
    });
  }
}
```

### 3.3 权限装饰器

```typescript
// use-permissions.decorator.ts
import { SetMetadata, ExecutionContext } from "@nestjs/common";
import { PERMISSIONS_METADATA } from "../constants/authz.constants";
import { Permission } from "../interfaces";

/**
 * 定义多个权限，只有当所有权限都满足时，才能访问路由
 */
export const UsePermissions = (...permissions: Permission[]) => {
  return SetMetadata(PERMISSIONS_METADATA, permissions);
};
```

### 3.4 使用示例

在控制器中使用权限控制：

```typescript
// menu.controller.ts
@Controller("menu")
@UseGuards(AuthZGuard)
export class MenuController {
  constructor(private readonly menuService: MenuService) {}

  @Post()
  @UsePermissions({ resource: "menu", action: "create" })
  create(@Body() createMenuDto: CreateMenuDto) {
    return this.menuService.create(createMenuDto);
  }
}
```

## 4. Casbin 模块封装

### 4.1 目录结构

```
libs/infra/casbin/
├── src/
│   ├── constants/
│   │   └── authz.constants.ts
│   ├── interfaces/
│   │   ├── permission.interface.ts
│   │   └── authz-module-options.interface.ts
│   ├── decorators/
│   │   └── use-permissions.decorator.ts
│   ├── guards/
│   │   └── authz.guard.ts
│   ├── services/
│   │   └── authz.service.ts
│   ├── adapter/
│   │   └── casbin-prisma.adapter.ts
│   ├── authz.module.ts
│   └── index.ts
```

### 4.2 常量定义

```typescript
// constants/authz.constants.ts
export const AUTHZ_MODULE_OPTIONS = "AUTHZ_MODULE_OPTIONS";
export const AUTHZ_ENFORCER = "AUTHZ_ENFORCER";
export const PERMISSIONS_METADATA = "**PERMISSIONS**";
```

### 4.3 接口定义

```typescript
// interfaces/permission.interface.ts
export interface Permission {
  resource: string;
  action: string;
}

// interfaces/authz-module-options.interface.ts
import { ExecutionContext } from "@nestjs/common";

export interface AuthZModuleOptions {
  userFromContext: (context: ExecutionContext) => any;
}
```

### 4.4 权限装饰器

```typescript
// decorators/use-permissions.decorator.ts
import { SetMetadata } from "@nestjs/common";
import { PERMISSIONS_METADATA } from "../constants/authz.constants";
import { Permission } from "../interfaces";

export const UsePermissions = (...permissions: Permission[]) => {
  return SetMetadata(PERMISSIONS_METADATA, permissions);
};
```

### 4.5 Prisma 适配器

```typescript
// adapter/casbin-prisma.adapter.ts
import { PrismaClient } from "@prisma/client";
import { Adapter, Model } from "casbin";

export class PrismaAdapter implements Adapter {
  private prisma: PrismaClient;

  constructor(prisma?: PrismaClient) {
    this.prisma = prisma || new PrismaClient();
  }

  static async newAdapter(prisma?: PrismaClient): Promise<PrismaAdapter> {
    const adapter = new PrismaAdapter(prisma);
    return adapter;
  }

  async loadPolicy(model: Model): Promise<void> {
    const rules = await this.prisma.casbinRule.findMany();

    for (const rule of rules) {
      const line = [rule.v0, rule.v1, rule.v2, rule.v3, rule.v4, rule.v5]
        .filter((v) => v !== null)
        .join(", ");

      model.addDef(rule.ptype, rule.ptype, line);
    }
  }

  async savePolicy(model: Model): Promise<boolean> {
    await this.prisma.casbinRule.deleteMany({});

    const rules = [];

    model.model.get("p").forEach((ast, ptype) => {
      ast.policy.forEach((rule) => {
        rules.push({
          ptype,
          v0: rule[0] || null,
          v1: rule[1] || null,
          v2: rule[2] || null,
          v3: rule[3] || null,
          v4: rule[4] || null,
          v5: rule[5] || null,
        });
      });
    });

    model.model.get("g").forEach((ast, ptype) => {
      ast.policy.forEach((rule) => {
        rules.push({
          ptype,
          v0: rule[0] || null,
          v1: rule[1] || null,
          v2: rule[2] || null,
          v3: rule[3] || null,
          v4: rule[4] || null,
          v5: rule[5] || null,
        });
      });
    });

    await this.prisma.casbinRule.createMany({ data: rules });
    return true;
  }

  async addPolicy(sec: string, ptype: string, rule: string[]): Promise<void> {
    await this.prisma.casbinRule.create({
      data: {
        ptype,
        v0: rule[0] || null,
        v1: rule[1] || null,
        v2: rule[2] || null,
        v3: rule[3] || null,
        v4: rule[4] || null,
        v5: rule[5] || null,
      },
    });
  }

  async removePolicy(
    sec: string,
    ptype: string,
    rule: string[]
  ): Promise<void> {
    await this.prisma.casbinRule.deleteMany({
      where: {
        ptype,
        v0: rule[0] || null,
        v1: rule[1] || null,
        v2: rule[2] || null,
        v3: rule[3] || null,
        v4: rule[4] || null,
        v5: rule[5] || null,
      },
    });
  }

  async removeFilteredPolicy(
    sec: string,
    ptype: string,
    fieldIndex: number,
    ...fieldValues: string[]
  ): Promise<void> {
    const where: any = { ptype };

    fieldValues.forEach((value, index) => {
      if (value) {
        where[`v${fieldIndex + index}`] = value;
      }
    });

    await this.prisma.casbinRule.deleteMany({ where });
  }
}
```

### 4.6 授权模块

```typescript
// authz.module.ts
import { DynamicModule, Module, Provider } from "@nestjs/common";
import {
  AUTHZ_ENFORCER,
  AUTHZ_MODULE_OPTIONS,
} from "./constants/authz.constants";
import { AuthZModuleOptions } from "./interfaces";
import { AuthZGuard } from "./guards/authz.guard";

export interface AuthZModuleAsyncOptions {
  imports?: any[];
  enforcerProvider: Provider;
  userFromContext: AuthZModuleOptions["userFromContext"];
}

@Module({})
export class AuthZModule {
  static register(options: AuthZModuleAsyncOptions): DynamicModule {
    const optionsProvider: Provider = {
      provide: AUTHZ_MODULE_OPTIONS,
      useValue: {
        userFromContext: options.userFromContext,
      },
    };

    return {
      module: AuthZModule,
      imports: options.imports || [],
      providers: [optionsProvider, options.enforcerProvider, AuthZGuard],
      exports: [AuthZGuard, AUTHZ_ENFORCER],
      global: true,
    };
  }
}
```

## 5. 应用集成

### 5.1 在应用模块中集成

```typescript
// app.module.ts
import { Module } from "@nestjs/common";
import { ConfigModule, ConfigService } from "@nestjs/config";
import * as casbin from "casbin";
import {
  AuthZModule,
  AUTHZ_ENFORCER,
  PrismaAdapter,
} from "./libs/infra/casbin";

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),

    // 注册Casbin模块
    AuthZModule.register({
      imports: [ConfigModule],
      enforcerProvider: {
        provide: AUTHZ_ENFORCER,
        useFactory: async () => {
          // 创建Prisma适配器
          const adapter = await PrismaAdapter.newAdapter();

          // 创建Casbin执行器
          return casbin.newEnforcer("./model.conf", adapter);
        },
      },
      userFromContext: (ctx) => {
        const request = ctx.switchToHttp().getRequest();
        return request.user; // 从JWT认证中获取用户信息
      },
    }),
  ],
})
export class AppModule {}
```

## 6. 使用示例

### 6.1 模型配置文件

```conf
# model.conf
[request_definition]
r = sub, obj, act, dom

[policy_definition]
p = sub, obj, act, dom, eft

[role_definition]
g = _, _, _

[policy_effect]
e = some(where (p.eft == allow)) && !some(where (p.eft == deny))

[matchers]
m = g(r.sub, p.sub, r.dom) && r.obj == p.obj && r.act == p.act && r.dom == p.dom
```

### 6.2 数据库表结构

```prisma
// Prisma schema.prisma
model CasbinRule {
  id    Int     @id @default(autoincrement())
  ptype String
  v0    String?
  v1    String?
  v2    String?
  v3    String?
  v4    String?
  v5    String?

  @@map("casbin_rule")
}
```

### 6.3 在控制器中使用

```typescript
// user.controller.ts
import { Controller, Get, Post, UseGuards } from "@nestjs/common";
import { AuthZGuard, UsePermissions } from "../libs/infra/casbin";

@UseGuards(AuthZGuard) // 启用授权守卫
@Controller("users")
export class UserController {
  @Get()
  @UsePermissions({ resource: "user", action: "read" })
  async getUsers() {
    // 只有拥有 user.read 权限的用户才能访问
    return "user list";
  }

  @Post()
  @UsePermissions({ resource: "user", action: "create" })
  async createUser() {
    // 只有拥有 user.create 权限的用户才能访问
    return "user created";
  }

  @Post("batch-delete")
  @UsePermissions(
    { resource: "user", action: "delete" },
    { resource: "admin", action: "manage" } // 需要同时满足两个权限
  )
  async batchDelete() {
    // 需要同时拥有 user.delete AND admin.manage 权限
    return "users deleted";
  }
}
```

### 6.4 权限管理服务

```typescript
// permission.service.ts
import { Injectable, Inject } from "@nestjs/common";
import * as casbin from "casbin";
import { AUTHZ_ENFORCER } from "../libs/infra/casbin";

@Injectable()
export class PermissionService {
  constructor(
    @Inject(AUTHZ_ENFORCER)
    private readonly enforcer: casbin.Enforcer
  ) {}

  // 添加角色权限
  async addPermission(
    role: string,
    resource: string,
    action: string,
    domain: string = "default"
  ) {
    await this.enforcer.addPolicy(role, resource, action, domain, "allow");
  }

  // 删除角色权限
  async removePermission(
    role: string,
    resource: string,
    action: string,
    domain: string = "default"
  ) {
    await this.enforcer.removePolicy(role, resource, action, domain, "allow");
  }

  // 为用户分配角色
  async assignRole(userId: string, role: string, domain: string = "default") {
    await this.enforcer.addRoleForUser(userId, role, domain);
  }

  // 检查用户权限
  async checkPermission(
    userId: string,
    resource: string,
    action: string,
    domain: string = "default"
  ) {
    return await this.enforcer.enforce(userId, resource, action, domain);
  }

  // 获取用户所有角色
  async getUserRoles(userId: string, domain: string = "default") {
    return await this.enforcer.getRolesForUser(userId, domain);
  }

  // 获取角色所有权限
  async getRolePermissions(role: string) {
    return await this.enforcer.getPermissionsForUser(role);
  }
}
```

### 6.5 初始化权限数据

```typescript
// permission.seed.ts
async function seedPermissions() {
  const enforcer = await casbin.newEnforcer("./model.conf", adapter);

  // 添加角色权限策略
  await enforcer.addPolicy("admin", "user", "create", "default", "allow");
  await enforcer.addPolicy("admin", "user", "read", "default", "allow");
  await enforcer.addPolicy("admin", "user", "update", "default", "allow");
  await enforcer.addPolicy("admin", "user", "delete", "default", "allow");

  await enforcer.addPolicy("operator", "user", "read", "default", "allow");
  await enforcer.addPolicy("operator", "user", "update", "default", "allow");

  await enforcer.addPolicy("guest", "public", "read", "default", "allow");

  // 添加用户角色关系
  await enforcer.addRoleForUser("user123", "admin", "default");
  await enforcer.addRoleForUser("user456", "operator", "default");
  await enforcer.addRoleForUser("user789", "guest", "default");

  // 保存到数据库
  await enforcer.savePolicy();
}
```

## 7. 总结

通过以上步骤，我们成功实现了基于 Casbin 的用户角色授权策略：

1. **模块化设计**：将 Casbin 功能封装为独立的模块，便于复用和维护
2. **数据库集成**：使用 Prisma 适配器将权限策略持久化到数据库
3. **装饰器支持**：通过 `@UsePermissions` 装饰器声明接口权限要求
4. **灵活的权限控制**：支持多种权限组合和域隔离
5. **易于管理**：提供完整的权限管理 API，便于动态调整权限策略

这套权限系统具有高度的可扩展性和灵活性，能够满足复杂企业应用的权限控制需求。
