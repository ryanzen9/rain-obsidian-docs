
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

安装相关通用依赖

```bash
pnpm add -D -w typescript lerna @commitlint/cli husky lint-staged oxlint oxfmt vitest @commitlint/config-conventional commitizen cz-conventional-changelog
pnpm add -w reflect-metadata rxjs
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

配置 `lerna.json`

```json
{
  "version": "independent",
  "npmClient": "pnpm",
  "command": {
    "publish": { "conventionalCommits": true }
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

### 配置 GIT 提交相关 Tool

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
    "test": "vitest",
    "test:ci": "vitest --run",
    "lint": "oxlint .",
    "lint:fix": "oxlint --fix .",
    "format": "oxfmt .",
    "format:check": "oxfmt --check .",
+   "prepare": "husky"  
  },
  "devDependencies": {
+   "@commitlint/cli": "^21.0.1",
+   "@commitlint/config-conventional": "^21.0.1",
+   "commitizen": "^4.3.1",
+   "cz-conventional-changelog": "^3.3.0",
+   "husky": "^9.1.7",
    "lerna": "^9.0.7",
    "lint-staged": "^17.0.4",
    "oxfmt": "^0.49.0",
    "oxlint": "^1.64.0",
    "typescript": "^6.0.3",
    "vitest": "^4.1.6"
  },
+ "config": {
+   "commitizen": {
+     "path": "cz-conventional-changelog"
+   }
+ },
  "packageManager": "pnpm@10.13.1"
}
```

执行 `pnpm husky init` ， 编辑 `.husky/commit-msg`
```bash


```