---
title: "NestJS 模块声明"
date: 2026-05-14
author: Ryan Zeng
tags:
  - NestJS
  - TypeScript
  - 模块化
categories:
  - Development
draft: false
---

## 模块是什么

在 NestJS 中，模块是组织应用程序结构的基本单元。每个应用至少有一个根模块（通常是 `AppModule`），它作为 Nest 组装应用图的起点。模块本质上是一个带有 `@Module()` 装饰器的类，这个装饰器提供了元数据，告诉 Nest 如何组织应用程序结构——哪些控制器负责处理请求，哪些服务提供业务逻辑，哪些功能需要从其他模块引入。

一个标准的 `@Module()` 装饰器接受一个对象，包含以下四个核心属性：

- `providers`：将被 Nest 注入器实例化的服务，它们在模块内部是共享的。
- `controllers`：该模块定义的控制器集合，负责处理传入的请求并返回响应。
- `imports`：导入其他模块，以获得它们导出的提供者（Providers）。
- `exports`：导出本模块中的 providers，供其他导入了本模块的模块使用。

NestJS 提供了三种模块注册方式：静态注册、全局注册和动态注册。它们分别对应不同的使用场景，理解它们的区别是构建可维护应用的关键。

## 静态注册

静态注册是最基本的模块使用方式。你在 `imports` 数组中直接引用模块类，Nest 在编译时就能确定模块的依赖关系。

```ts
import { Module } from '@nestjs/common';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';

@Module({
  controllers: [UsersController],
  providers: [UsersService],
  exports: [UsersService], // 导出 Service 供外部使用
})
export class UsersModule {}
```

`exports` 数组决定了模块对外暴露哪些服务。只有被导出的 Provider 才能被其他模块消费——没有导出的 Provider 对外部模块不可见，这保证了模块的封装性。

在根模块中引入：

```ts
@Module({
  imports: [UsersModule],
})
export class AppModule {}
```

静态注册的特点是配置在编译时确定，模块一旦声明就无法在运行时更改。对于大多数业务模块（如 `UsersModule`、`OrdersModule`），静态注册已经足够。

> 参考：[NestJS Modules 官方文档](https://docs.nestjs.com/modules)

## 全局注册

使用 `@Global()` 装饰器可以将一个模块声明为全局模块。一旦在根模块注册，整个应用程序中的任何位置都可以使用该模块导出的服务，而无需在每个模块中重复导入。

```ts
import { Module, Global } from '@nestjs/common';
import { HelperService } from './helper.service';

@Global() // 标记为全局模块
@Module({
  providers: [HelperService],
  exports: [HelperService],
})
export class SharedModule {}
```

全局模块只需要在根模块注册一次：

```ts
// AppModule 中注册
@Module({
  imports: [SharedModule],
})
export class AppModule {}

// 其他模块无需再 imports SharedModule，即可直接注入 HelperService
@Module({})
export class FeatureModule {}
```

适用场景：通用的工具库、数据库连接辅助类、全局守卫等。这些服务通常在应用中随处可见，逐个导入只会增加样板代码。

需要注意：不要过度使用全局模块。全局模块降低了代码的可维护性和解耦程度——当你在某个模块中注入一个来自全局模块的服务时，无法从模块声明中看出这个依赖从何而来，这会让追踪依赖关系变得困难。只在真正需要全局访问的场合使用它。

> 参考：[NestJS Global Modules](https://docs.nestjs.com/modules#global-modules)

## 动态注册

静态注册和全局注册的模块配置在编译时就已经确定。但很多场景下，模块需要根据运行时参数动态生成元数据——数据库连接需要配置字符串，HTTP 客户端需要指定基础 URL，缓存模块需要设置 TTL。动态模块（Dynamic Module）就是为了解决这个问题而设计的。

动态模块允许模块在注册时接受参数，并根据这些参数动态生成模块的元数据（如 providers、imports 等）。这通常通过 `.register()`、`.forRoot()` 或 `.forFeature()` 静态方法来实现。

```ts
import { DynamicModule, Module } from '@nestjs/common';
import { ConfigService } from './config.service';

@Module({})
export class ConfigModule {
  static register(options: Record<string, any>): DynamicModule {
    return {
      module: ConfigModule,
      providers: [
        {
          provide: 'CONFIG_OPTIONS',
          useValue: options,
        },
        ConfigService,
      ],
      exports: [ConfigService],
    };
  }
}

// 使用方式
@Module({
  imports: [ConfigModule.register({ folder: './config' })],
})
export class AppModule {}
```

`register`、`forRoot` 和 `forFeature` 并非 TypeScript 的语法关键字，而是 NestJS 社区和官方约定俗成的命名规范。它们本质上都是返回 `DynamicModule` 的静态方法，但在设计意图、作用域和生命周期上有着明确的分工：

**register**：你希望配置一个具有特定配置的动态模块，仅供调用模块使用。每个调用可以有不同的配置，互不影响。例如 `HttpModule.register({ baseUrl: 'someUrl' })`，在另一个模块中使用 `HttpModule.register({ baseUrl: 'somewhere else' })` 会创建一个完全独立的实例。

**forRoot**：你期望配置一个动态模块一次，并在全应用范围内重用该配置。这是"全局唯一配置"模式——数据库连接、缓存配置这类只需要初始化一次的基础设施适合放在这里。例如 `TypeOrmModule.forRoot()`、`GraphQLModule.forRoot()`。

**forFeature**：你希望使用 `forRoot` 的全局配置，但需要注册一些仅限当前模块使用的资源。例如 `TypeOrmModule.forFeature([UserEntity])` 注册了 `UserEntity` 对应的 Repository，这个 Repository 只有当前模块的 Service 才会注入。它强依赖于 `forRoot` 已经初始化的上下文。

通常，这三种方法都有对应的异步变体：`registerAsync`、`forRootAsync` 和 `forFeatureAsync`，它们使用 Nest 的依赖注入来异步获取配置。

> 参考：[NestJS Dynamic Modules](https://docs.nestjs.com/fundamentals/dynamic-modules)

## 实现模式

理解了三种动态注册方法的语义之后，接下来看它们的具体实现。

### forRoot

`forRoot` 负责创建全局唯一的配置和连接。通常配合 `@Global()` 装饰器，避免子模块重复导入。配置 Provider 放在 `exports` 中，供全应用使用。

```ts
import { Module, DynamicModule, Global } from '@nestjs/common';

@Global() // 1. 标记为全局，避免子模块重复导入
@Module({})
export class DatabaseModule {
  static forRoot(options: DatabaseConfig): DynamicModule {
    const connectionProvider = {
      provide: 'DATABASE_CONNECTION',
      useFactory: async () => await createConnection(options), // 2. 创建单例连接
    };

    return {
      module: DatabaseModule,
      providers: [connectionProvider],
      exports: [connectionProvider], // 3. 导出给全应用
    };
  }
}

// 使用
@Module({
  imports: [DatabaseModule.forRoot({ host: 'localhost', port: 5432 })],
})
export class AppModule {}
```

`forRoot` 返回的 `DynamicModule` 中 `module` 字段指向自身（`DatabaseModule`），这确保了无论 `forRoot` 被调用多少次，Nest 都能正确识别模块归属。

### forFeature

`forFeature` 不创建新连接，而是利用 `forRoot` 已经建立的全局连接，注册当前模块特有的资源。它注入 `forRoot` 创建的全局 Provider，基于此生成模块级别的 Provider。

```ts
@Module({})
export class DatabaseModule {
  static forFeature(entities: Function[]): DynamicModule {
    const repositories = entities.map((entity) => ({
      provide: `${entity.name}Repository`,
      useFactory: (connection) => connection.getRepository(entity),
      inject: ['DATABASE_CONNECTION'], // 1. 注入 forRoot 创建的全局连接
    }));

    return {
      module: DatabaseModule,
      providers: repositories,
      exports: repositories, // 2. 导出 Repository 给当前 Feature Module 的 Service 使用
    };
  }
}

// 使用：注册 User 实体对应的 Repository
@Module({
  imports: [
    DatabaseModule.forRoot({ host: 'localhost', port: 5432 }),
    DatabaseModule.forFeature([UserEntity]),
  ],
})
export class UsersModule {}
```

`forFeature` 的关键在于 `inject: ['DATABASE_CONNECTION']`——它声明了对 `forRoot` 创建的全局 Provider 的依赖。Nest 的依赖注入系统会确保 `forRoot` 先于 `forFeature` 完成。

### register

`register` 适用于需要多次实例化且配置不同的场景。每次调用都会产生一个独立的模块实例，拥有自己的 Provider。

```ts
@Module({})
export class HttpModule {
  static register(options: HttpModuleOptions): DynamicModule {
    return {
      module: HttpModule,
      providers: [
        {
          provide: 'HTTP_OPTIONS',
          useValue: options,
        },
        HttpService,
      ],
      exports: [HttpService],
    };
  }
}

// 不同模块可以注册不同配置的 HttpService
@Module({
  imports: [HttpModule.register({ baseUrl: 'https://api.service-a.com' })],
})
export class ServiceAModule {}

@Module({
  imports: [HttpModule.register({ baseUrl: 'https://api.service-b.com' })],
})
export class ServiceBModule {}
```

`register` 和 `forRoot` 的核心区别在于：`forRoot` 全局共享一份配置，`register` 每次调用创建一份独立配置。当你需要多个不同配置的实例时，用 `register`；当你只需要一份全局配置时，用 `forRoot`。

### 异步配置

实际项目中，配置往往不是硬编码的，而是从环境变量、配置服务或远程 API 异步获取。NestJS 提供了 `useFactory`、`useExisting` 和 `useClass` 三种异步配置方式，对应 `registerAsync`、`forRootAsync` 和 `forFeatureAsync`。

```ts
import { Module, DynamicModule, Global } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';

@Global()
@Module({})
export class DatabaseModule {
  static forRootAsync(): DynamicModule {
    return {
      module: DatabaseModule,
      providers: [
        {
          provide: 'DATABASE_CONNECTION',
          useFactory: async (configService: ConfigService) => {
            // 从 ConfigService 异步获取配置
            return createConnection({
              host: configService.get('DB_HOST'),
              port: configService.get('DB_PORT'),
              username: configService.get('DB_USERNAME'),
              password: configService.get('DB_PASSWORD'),
              database: configService.get('DB_NAME'),
            });
          },
          inject: [ConfigService], // 注入 ConfigService
        },
      ],
      exports: ['DATABASE_CONNECTION'],
    };
  }
}
```

异步配置的原理很简单：`useFactory` 返回一个 `Promise`，Nest 在应用启动时会等待 Promise resolve 后再完成 Provider 的注册。这确保了模块初始化时配置已经就绪。

## 模块重导出

模块不仅可以导入其他模块，还可以将导入的模块重新导出，简化下游消费者的依赖声明。

```ts
@Module({
  imports: [CommonModule], // 导入 CommonModule
  exports: [CommonModule], // 重导出 CommonModule
})
export class SharedModule {}
```

这样，导入了 `SharedModule` 的模块自动获得了 `CommonModule` 的所有导出 Provider，无需显式声明对 `CommonModule` 的依赖。这在封装内部实现细节时很有用——消费者只需要知道 `SharedModule` 的存在，不需要关心它内部聚合了哪些模块。

## 另见

- [NestJS Modules 官方文档](https://docs.nestjs.com/modules)
- [NestJS Dynamic Modules](https://docs.nestjs.com/fundamentals/dynamic-modules)
- [NestJS Circular Dependency](https://docs.nestjs.com/fundamentals/circular-dependency)