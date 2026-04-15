## 静态注册 (Static Registration)

在 NestJS 中，每个应用至少有一个根模块（通常是 AppModule）。模块本质上是一个带有 @Module() 装饰器的类。这个装饰器提供了 元数据，告诉 Nest 如何组织应用程序结构。

一个标准的 @Module() 装饰器接受一个对象，包含以下四个核心属性：

- providers: 将被 Nest 注入器实例化的服务，它们在模块内部是共享的。
- controllers: 该模块定义的必须被实例化的控制器集合。
- imports: 导入其他模块，以获得它们导出的 提供者（Providers）。
- exports: 导出本模块中的 providers，供其他导入了本模块的模块使用。

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

// 在 AppModule 中引入
@Module({
  imports: [UsersModule], // 静态引入
})
export class AppModule {}
```

## 全局注册 (Global Registration)

使用 @Global() 装饰器可以将一个模块声明为全局模块。一旦在根模块注册，整个应用程序中的任何位置都可以使用该模块导出的服务，而无需在每个模块中重复 导入。

适用场景：通用的工具库、数据库连接辅助类、全局守卫等。

注意：不要过度使用全局模块，这会降低代码的 可维护性 和解耦程度。

```ts
import { Module, Global } from '@nestjs/common';
import { HelperService } from './helper.service';

@Global() // 标记为全局
@Module({
  providers: [HelperService],
  exports: [HelperService],
})
export class SharedModule {}
```

## 动态注册 (Dynamic Registration)

动态模块 允许模块在注册时接受参数，并根据这些参数动态生成模块的元数据（如 providers、imports 等）。这通常用于通过 .register(), .forRoot(), 或 .forFeature() 方法来配置模块。

> 提示：从 @nestjs/common 导入 DynamicModule。

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
// imports: [ConfigModule.register({ folder: './config' })];
```

在 NestJS 中，register、forRoot 和 forFeature 并非 TypeScript 的语法关键字，而是 NestJS 社区和官方约定俗成的 命名规范（Naming Conventions）。

它们本质上都是返回 DynamicModule 的静态方法，但在设计意图、作用域和生命周期上有着明确的分工。

以下为官方社区准则：

使用以下命令创建模块时：

**register**： 你希望配置一个具有特定配置的动态模块，仅供调用模块使用。例如，使用 Nest 的 @nestjs/axios：HttpModule.register({ baseUrl: 'someUrl' })。如果在另一个模块中使用 HttpModule.register({ baseUrl: 'somewhere else' })，它将具有不同的配置。你可以根据需要对任意数量的模块执行此操作。

**forRoot**，你期望配置一个动态模块一次并在多个地方重用该配置（尽管可能在不知不觉中因为它被抽象掉了）。这就是为什么你有一个 GraphQLModule.forRoot()、一个 TypeOrmModule.forRoot() 等。

**forFeature**：你希望使用动态模块 forRoot 的配置，但需要修改一些特定于调用模块需求的配置（即该模块应该访问哪个存储库，或者日志器应该使用的上下文。）

通常，所有这些都有对应的 async、registerAsync、forRootAsync 和 forFeatureAsync，意思相同，但也使用 Nest 的依赖注入进行配置。

> 注意： 强依赖于 forRoot 已经初始化的上下文，在当前模块的上下文中注入 forRoot 已经初始化模块中特定的资源。

## 实现

forRoot 和 forFeature 通常用于需要全局配置的模块，例如数据库连接、日志记录等。register 则更适合于需要多次实例化且配置不同的模块。

forRoot 实现：

通常会把配置 Provider 放到 exports 中，供全应用使用。

```typescript
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
```

forFeature 实现：

它通常不创建新连接，而是利用已有的机制注入特定对象。

```typescript
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
```

## 另见

[NestJs 官方文档](https://docs.nestjs.com/)
