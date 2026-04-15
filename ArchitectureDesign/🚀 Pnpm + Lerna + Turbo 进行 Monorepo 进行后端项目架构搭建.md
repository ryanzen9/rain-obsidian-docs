---
Title: 🚀 Pnpm + Lerna + Turbo 进行 Monorepo 进行后端项目架构搭建
Draft: false
tags:
  - Monorepo
  - Pnpm
  - Lerna
  - Turbo
Author: Ruby Ceng
---

## 引言

起因是公司要更换新的技术栈，从传统 **.NET** 迁移到 **Node.js** 这一套上面来。老项目是一个大单体项目，各种服务全耦合在一起了。因此打算安排先重做一个商城，一个一个的往分布式转移。

最开始我打算直接采用**微服务**的设计模式。但是奈何 **Node.js** 的生态，找了一圈实现没有什么适合的中间件与工作流。与 **AI** 聊了下发现可以先用 **Monorepo** 进行实现，到了项目后期也可以往微服务进行无缝迁移，遂有此文。

## 什么是 Monorepo？

**Monorepo**（单一代码仓库）是一种软件开发策略，即将多个逻辑上独立的项目（**projects**）、应用（**apps**）或库（**libs**）存放在同一个版本控制仓库（比如一个 **Git** 仓库）中。

这与更传统的 **Multirepo**（多代码仓库）策略形成对比，后者是为每个项目/应用/库创建独立的仓库。

## 为什么采用 Monorepo 架构？

### 避免过早抽象

**HTTP** 接口需要定义协议（**gRPC**/**REST**）、错误码、重试机制... 这些在项目初期是**纯成本**。**Monorepo** 让您先专注业务逻辑，等系统稳定后再抽取服务（如将 **auth** 模块拆为独立服务）。

**HTTP** 调用在初期是反模式：

您描述的"基础设施"本质是**领域模型**（如用户、角色），而非独立业务域。用 **HTTP** 暴露领域模型会导致：

- 产生大量琐碎 **API**（`GET /users/{id}`, `POST /roles`...）
- 业务系统需处理网络错误（超时/重试），而基础设施本应是高可用的
- **数据一致性风险**：例如商城系统创建订单时，需先调用基础设施获取用户权限，再操作订单库——这需要分布式事务，复杂度陡增。

> **经验法则**：当且仅当某个模块**可独立交付价值**（如支付服务、短信服务）时，才拆分为独立服务。用户/角色管理是**支撑性能力**，不适合早期拆分。

### 共享代码零成本

**场景**：你的 **NestJS API** (app) 和另一个 **Node.js** 脚本（比如一个定时任务 app）可能需要共用同一套 **Prisma Schema**、数据库客户端、**DTOs**（数据传输对象）或者业务逻辑函数。

**在 Monorepo 中**：你可以创建一个共享的库（**lib**），比如叫 `data-access` 或 `shared-utils`。所有应用都可以直接导入这个库，代码完全复用。修改一次，所有地方都生效。

### 统一的工具链和开发标准

所有项目共享一套配置，比如 **ESLint**（代码规范）、**Prettier**（代码格式化）、**Jest**（测试框架）、**TypeScript**（类型检查）。这确保了整个代码库的风格一致，新成员加入项目时也能更快地适应。

## 项目架构搭建

### 引入 Lerna 和 Turbo

#### 什么是 Lerna？

- [Lerna 官网](https://lerna.js.org/)
- [Lerna 中文文档](https://www.lernajs.cn/docs/getting-started)

**Lerna** 是一个用于管理 **JavaScript** 多包存储库（**monorepo**）的工具。它的核心功能在于**版本控制和包发布**。它允许你在一个仓库中管理多个包，并提供了一系列命令来简化包的开发、发布和维护。

**Lerna 使用场景**：

- **版本控制**：Lerna 可以管理多个包的版本，并提供统一的版本控制命令，也可以根据更改的子项目，自动更新版本号。
- **简化发布流程**：可以管理多个包的发布，并提供统一的发布命令，它能自动检测自上次发布以来有变更的包，自动更新版本号，生成 `CHANGELOG.md`，创建 **git tag**，并最终将更新后的包发布到 **npm**。
- **批量执行命令**：您可以在仓库根目录通过 `lerna run <script>` 命令，在所有或指定的子项目中并行或串行地执行 **npm** 脚本（如 `test`, `build` 等）。

> **注意**：lerna 在 8.0 后不负责在源码仓库中安装和链依赖项，官方更推荐用软件包管理器的 **workspaces** 功能。故使用 [pnpm workspace](https://pnpm.io/workspaces) 进行依赖管理。

#### 什么是 Turbo？

- [Turbo 官网](https://turbo.build/repo/docs)
- [Turbo 中文文档](https://turborepo-zh.vercel.app/)

**Turborepo** 是一个针对 **JavaScript** 和 **TypeScript** 代码库进行优化的**智能构建系统**。Turborepo 使用缓存来加速本地设置并加快 **CI** 速度。Turborepo 被设计为可逐步采用，因此您可以在几分钟内将其添加到大多数代码库中。

**Turbo** 天然支持 **monorepo** 架构，并且可以与 **Lerna** 无缝集成。

**Turbo 使用优势**：

- **极致的速度 (Extreme Speed)**

  - **本地缓存**：Turbo 会为你执行过的任务（如 `build`, `lint`）创建缓存。当你再次运行同一个任务，如果相关的代码没有改变，它会直接从缓存中读取结果，耗时可能是毫秒级的。
  - **远程缓存 (Remote Caching)**：这是团队协作的"杀手锏"。你可以将缓存上传到云端（如 **Vercel** 或你自己的服务器）。当你的同事或 **CI/CD** 流水线需要构建项目时，他们可以直接下载你已经生成好的缓存，无需在自己的机器上重复计算。这意味着，一个项目在一天内只需要被完整构建一次！

- **智能的任务调度 (Intelligent Task Scheduling)**

  - 它能理解你项目内部的依赖关系。比如，你的 **app** 依赖于共享组件库 **ui**。当你运行 `turbo run build` 时，它知道必须先构建 **ui**，然后再构建 **app**。
  - 它会最大限度地利用你 **CPU** 的所有核心，并行执行那些没有相互依赖的任务，进一步压缩时间。
  - **简化的命令行 (Simplified CLI)**：告别繁琐的 `cd ../../packages/ui && pnpm run build`。你只需要在项目根目录运行 `turbo run build`，它会自动处理所有事情。你还可以使用强大的筛选功能，例如 `turbo run build --filter=my-app` 来只构建 **my-app** 及其依赖。

- **极少的配置 (Minimal Configuration)**
  - **Turbo** 遵循"约定优于配置"的原则。对于大多数项目，你只需要一个简单的 `turbo.json` 文件来定义任务之间的"管道关系"即可，学习成本很低。

### 正式操作

创建相关脚手架：

```bash
# 使用 create-turbo 创建项目
npx create-turbo@latest

# 删除无用的文件
rm -rf apps/web apps/docs

# 安装相关依赖
pnpm install

# 创建 api 项目
mkdir -p apps/api
cd apps/api
pnpm init -y
```

#### 创建示例项目

修改 `api/package.json`：

```json
{
  "name": "@my-repo/api",
  "version": "1.0.0",
  "private": true,
  "main": "src/index.ts",
  "scripts": {
    "start": "node index.js"
  }
}
```

创建 `api/index.js`：

```javascript
// api/index.js
console.log("Hello World");
```

配置 `turbo.json`：

```json
{
  "$schema": "https://turborepo.com/schema.json",
  "ui": "tui",
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "inputs": ["$TURBO_DEFAULT$", ".env*"],
      "outputs": [".next/**", "!.next/cache/**"]
    },
    "lint": {
      "dependsOn": ["^lint"]
    },
    "check-types": {
      "dependsOn": ["^check-types"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    },
    "start": {
      "cache": false
    }
  }
}
```

测试运行：

```bash
# 运行项目
pnpm start

# 构建项目
pnpm build

# 运行项目
pnpm start
```

#### 使用 nest-cli 创建 NestJS 项目

```bash
# 安装 nest-cli
pnpm install -g @nestjs/cli

# 创建 nest 项目
cd ../ && nest new nest-api
```

测试运行：

```bash
# 运行项目
pnpm start

# 构建项目
pnpm build
```

#### 整合 Lerna

在项目根目录下执行：

```bash
pnpm add lerna -wD
```

根目录创建 `lerna.json`（如果要将包独立维护版本，将 `version` 设置为 `independent`）：

```json
{
  "$schema": "node_modules/lerna/schemas/lerna-schema.json",
  "version": "0.0.0",
  "packages": ["packages/*", "apps/*"],
  "npmClient": "pnpm",
  "useWorkspaces": true
}
```

编辑相关文件 `package.json` 和 `turbo.json`：

根目录 `package.json` 脚本配置：

```json
{
  "private": true,
  "scripts": {
    "build": "turbo run build",
    "dev": "turbo run dev --parallel",
    "test": "turbo run test",
    "lint": "turbo run lint",
    "version": "lerna version",
    "release": "lerna publish from-package"
  }
}
```

`turbo.json` 补充配置：

```json
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "version": {
      "cache": false
    },
    "release": {
      "cache": false
    }
  }
}
```

#### 工作流

```bash
# 安装依赖
pnpm install

# 运行项目
pnpm dev

# 构建项目
pnpm build

# 发布项目
# 1. 更新版本号
pnpm version

# 2. 发布包
pnpm publish
```

#### 最终的项目结构

```bash
> tree -I "node_modules|dist|docs|sql" -L 3 -C

.
├── apps
│   ├── api
│   │   ├── index.js
│   │   └── package.json
│   └── demo
│       ├── eslint.config.mjs
│       ├── nest-cli.json
│       ├── package.json
│       ├── README.md
│       ├── src
│       ├── test
│       ├── tsconfig.build.json
│       └── tsconfig.json
├── lerna.json
├── package.json
├── packages
│   ├── eslint-config
│   │   ├── base.js
│   │   ├── next.js
│   │   ├── package.json
│   │   ├── react-internal.js
│   │   └── README.md
│   ├── typescript-config
│   │   ├── base.json
│   │   ├── nextjs.json
│   │   ├── package.json
│   │   └── react-library.json
│   └── ui
│       ├── eslint.config.mjs
│       ├── package.json
│       ├── src
│       └── tsconfig.json
├── pnpm-lock.yaml
├── pnpm-workspace.yaml
├── README.md
└── turbo.json
```
