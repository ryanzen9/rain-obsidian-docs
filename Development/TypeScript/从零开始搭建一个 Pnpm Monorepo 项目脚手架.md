
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