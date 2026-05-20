
## 项目初始化

```
make example-projects && cd example-projects

pnpm init -y # 在 package.json 中设置 "private": true
```

配置 workspace

```yaml
# pnpm-workspace.yaml
packages:
  - packages/*
```

安装相关 TS 依赖

```bash
pnpm add -D -w typescript @types/node
```

创建根 `tsconfig.json`

```json
{
  "compilerOptions": {
    "module": "commonjs",
    "declaration": true,
    "target": "es2017",
    "strict": true,
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,
    "skipLibCheck": true
  }
}
```

#### 创建第一个包

```bash
mkdir -p packages/my-module/src
```

`packages/my-module/package.json`:

```json
{
  "name": "@my-scope/nestjs-my-module",
  "version": "0.0.0",
  "main": "lib/index.js",
  "files": ["lib"],
  "scripts": {
    "build": "tsc --build tsconfig.build.json",
    "test": "vitest run",
    "typecheck": "tsc --noEmit"
  },
  "peerDependencies": {
    "@nestjs/common": "^11.0.0",
    "@nestjs/core": "^11.0.0"
  },
  "publishConfig": { "access": "public" }
}
```

`packages/my-module/tsconfig.json`:

```json
{
  "extends": "../../tsconfig.json",
  "compilerOptions": { "outDir": "./lib", "rootDir": "./src" },
  "include": ["./src"]
}
```

`packages/my-module/tsconfig.build.json`:

```json
{
  "extends": "./tsconfig.json",
  "exclude": ["./src/**/*.spec.ts"]
}
```

创建你的 NestJS 模块

```
packages/my-module/src/
├── index.ts
├── my-module.module.ts
├── my-module.decorators.ts
├── my-module.interfaces.ts
├── my-module.constants.ts
└── my-module-module-definition.ts
```


### 配置 Lint 和 Format 工具

为项目引入 `oxlint` 和 `oxfmt`（Oxc 生态的格式化工具）可以极大地提升代码检查和格式化的性能。它们均由 Rust 编写：**oxlint** 是一个比 ESLint 快 50-100 倍的 Lint 工具，而 **oxfmt** 则是旨在替代 Prettier 的高性能格式化工具。

> **what is oxc**:
> 
> The Oxidation Compiler is a collection of high-performance tools for JavaScript and TypeScript written in Rust.
> Oxc is part of [VoidZero](https://voidzero.dev/)'s vision for a unified, high-performance toolchain for JavaScript. It powers [Rolldown](https://rolldown.rs/) ([Vite](https://vitejs.dev/)'s future bundler) and enables the next generation of ultra-fast development tools that work seamlessly together


安装 VSCode Plugin `oxc`, 更改项目内 IDE 设置 `.vscode/setting.json`

```diff
+ "oxc.enable": true,
+ "oxc.enable.oxlint": true,
+ "oxc.enable.oxfmt": true,
+ "editor.formatOnSave": true,
+ "editor.defaultFormatter": "oxc.oxc-vscode",
+ "editor.formatOnSaveMode": "file",
```

安装相关依赖

```bash
pnpm add -D -w oxlint oxfmt lint-staged
```
#### 配置 Oxlint

在 package.json 下配置

```diff
{
  "scripts": {
+   "lint": "oxlint",
+   "lint:fix": "oxlint --fix"
  }
}
```

执行 `pnpm oxlint --init`, 获取最小化配置文件 `.oxlintrc.json`

```json
{
  "$schema": "./node_modules/oxlint/configuration_schema.json",
  "categories": {
    "correctness": "warn"
  },
  "rules": {
    "eslint/no-unused-vars": "error"
  }
}
```

相关配置： [oxlint 配置]('https://oxc.rs/docs/guide/usage/linter/config.html')

#### 配置  Oxfmt

添加相关脚本，在 `package.json` 中添加如下内容：

```diff
{
  "scripts": {
+   "fmt": "oxfmt",
+   "fmt:check": "oxfmt --check"
  }
}
```

### 配置 GIT 提交相关 Tool

安装相关依赖

```bash
pnpm add -wD @commitlint/cli husky @commitlint/config-conventional commitizen cz-conventional-changelog
```

安装相关包工具, 在 `package.json` 中添加以下内容：

```diff
{
  "name": "nestjs",
  "version": "1.0.0",
  "private": "true",
  "description": "",
  "keywords": [],
  "license": "ISC",
  "author": "Ryan Zeng",
  "main": "index.js",
  "scripts": {
+   "commit": "cz",
+   "prepare": "husky"  
  },
  "devDependencies": {
+   "@commitlint/cli": "^21.0.1",
+   "@commitlint/config-conventional": "^21.0.1",
+   "commitizen": "^4.3.1",
+   "cz-conventional-changelog": "^3.3.0",
+   "husky": "^9.1.7",
    "typescript": "^6.0.3"
  },
+ "config": {
+   "commitizen": {
+     "path": "cz-conventional-changelog"
+   }
+ },
  "packageManager": "pnpm@10.13.1"
}
```

**使用 husky +  commitlint 拦截格式非法的提交**

执行 `pnpm husky init` ， 编辑 `.husky/commit-msg`

```bash
pnpm exec commitlint --edit $1
```

**使用 husky 进行提交前测试**

编辑 `.husky/pre-commit`

```bash
pnpm test
```

使用 `git add . && pnpm commit` 尝试进行第一次提交。
### 发布相关 Tool

采用 `Changesets` 结合 `PNPM` 取代 `lerna` + `standard-verison` , 作为发布管理方案.
#### npm 包的发布与管理

> pnpm 从 v8+ 就已经原生支持 monorepo publish: `pnpm publish -r`

执行 `npm whoami` 查看登录情况，如果未登录执行 `npm login`
```bash
❯ npm login             
npm notice Log in on https://registry.npmjs.org/
Login at:
https://www.npmjs.com/login?next=/login/cli/********
Press ENTER to open in the browser...

Logged in on https://registry.npmjs.org/.

❯ npm whoami
ryanzeng
```

#### 对 ChangeLog 进行自动化生成与维护

[**Changesets**](https://github.com/changesets/changesets) : 管理版本号和 CHANGELOG 的自动化工具，它的核心思想是：**开发者只需要描述自己做了什么改动，工具自动计算下一个版本号、生成变更日志**。

```bash
pnpm add -w -D @changesets/cli
```

工作流程概览

1. **添加 changeset**: feat/fix 后执行 `npx changeset`, 交互式描述自己做了什么改动。
2. **累积 changeset**：  随着其他贡献者不断添加 changeset，`.changeset` 里会积累很多小的 markdown 文件。这些文件只描述改动，**此时并不修改任何 package.json 或 CHANGELOG**。
3. **消费 changeset，升级版本** ： 发布时运行 `npx changeset version`，它会自动计算版本号以及更新日志
4. **发布**： 执行 `npx changeset publish`, 他会自动执行 `pnpm publish`,发布发生变更的包。

执行 `npx changeset init` 进行初始化:

`.changeset/config.json`

```json
{
  "$schema": "https://unpkg.com/@changesets/config@3.1.4/schema.json",
  "changelog": "@changesets/cli/changelog",
  "commit": false,
  "fixed": [],
  "linked": [],
  "access": "restricted",
  "baseBranch": "main",
  "updateInternalDependencies": "patch",
  "ignore": []
}
```

`package.json`

```diff
{
  "scripts": {
+   "prep-release": "changeset",
+   "prep-version": "changeset version",
+   "release": "changeset publish"
  },
  "devDependencies": {
+   "@changesets/cli": "^2.31.0",
  }
}

```

通过将“发版意图”与“代码提交”解耦，让版本管理和更新日志（Changelog）的生成变得非常清晰。

#### 发布模拟

使用 `npm whoami` 与 `npm login` 查询登录信息

执行 `pnpm prep-release` 选择需要发布的包并且填写相关信息。

执行 `git add .` 和 `pnpm commit` 进行消息提交

使用 `pnpm prep-version` 进行版本号自动添加以及标签添加。 使用 pnpm commit 提交 `chore(release): vx.x.x`

使用 `pnpm release` 进行发布