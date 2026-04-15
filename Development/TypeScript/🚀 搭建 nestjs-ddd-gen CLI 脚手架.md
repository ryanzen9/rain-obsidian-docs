## 安装相关依赖与配置文件

```bash
pnpm add -D commander zod handlebars change-case globby @mrleebo/prisma-ast @types/node typescript param-case pluralize
pnpm add -D ts-node tsconfig-paths tsx
```

`tsconfig.json` 配置文件如下:

```json
{
  "compilerOptions": {
    "outDir": "dist",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "target": "ES2022",
    "rootDir": "src",
    "declaration": true,
    "composite": false,
    "noEmit": false,
    "skipLibCheck": true,
    "strict": true
  },
  "include": ["src"]
}
```

`package.json` 配置文件如下:

```json
{
  "name": "ddd-cqrs-gen",
  "version": "1.0.0",
  "private": false,
  "description": "ddd-cqrs-gen",
  "type": "module",
  "bin": {
    "ddd-cqrs-gen": "./dist/index.js"
  },
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "files": ["dist", "templates"],
  "scripts": {
    "build": "tsc",
    "dev": "tsx watch src/index.ts",
    "ddd-gen": "node dist/index.js"
  },
  "devDependencies": {
    "@mrleebo/prisma-ast": "^0.13.0",
    "@types/node": "^24.2.1",
    "change-case": "^5.4.4",
    "commander": "^14.0.0",
    "globby": "^14.1.0",
    "handlebars": "^4.7.8",
    "param-case": "^4.0.0",
    "pluralize": "^8.0.0",
    "ts-node": "^10.9.2",
    "tsconfig-paths": "^4.2.0",
    "tsx": "^4.20.3",
    "typescript": "^5.9.2",
    "zod": "^4.0.17"
  },
  "keywords": ["ddd", "cqrs", "ddd-cqrs-gen"],
  "license": "MIT",
  "packageManager": "pnpm@10.13.1"
}
```

声明配置文件`.dddgen.json`

```json
{
  "schemaPath": "prisma/schema.prisma",
  "projectRoot": ".",
  "output": {
    "moduleRoot": "src/modules",
    "moduleName": "iam",
    "resourceName": "user"
  },
  "idStrategy": "domain",
  "immutableFields": ["username", "password"],
  "baseFields": {
    "id": "id",
    "isDeleted": "isDeleted",
    "isActive": "isActive",
    "createdAt": "createdAt",
    "updatedAt": "updatedAt",
    "createdBy": "createdBy",
    "updatedBy": "updatedBy"
  },
  "softDeleteField": "isDeleted",
  "read": {
    "pageSizeDefault": 10
  },
  "modelMap": {
    "SysUsers": "User"
  }
}
```

## 编写 CLI 核心代码

CLI 入口代码

```typescript
// src/index.ts
import { Command } from 'commander';
import { loadConfig } from './lib/config.js';
import { inspectSchema } from './lib/prisma.js';
import { generateAll, listModels } from './lib/generate.js';

const program = new Command();

program.name('rcerp-gen').description('RCERP code generator from prisma schema').version('0.1.0');

program
  .command('init')
  .description('create .rcerpgen.json config at project root')
  .option('-f, --force', 'overwrite if exists', false)
  .action(async (opts) => {
    const { writeDefaultConfig } = await import('./lib/init.js');
    await writeDefaultConfig(!!opts.force);
    console.log('Created .rcerpgen.json');
  });

program
  .command('models')
  .description('list prisma models')
  .option('-c, --config <path>', 'config path', '.rcerpgen.json')
  .action(async (opts) => {
    const cfg = await loadConfig(opts.config);
    const schema = await inspectSchema(cfg.schemaPath);
    console.table(listModels(schema));
  });

program
  .command('generate')
  .description('generate code for a model or all')
  .option('-c, --config <path>', 'config path', '.rcerpgen.json')
  .option('-m, --model <modelName>', 'single prisma model name')
  .option('--dry-run', 'do not write files', false)
  .action(async (opts) => {
    const cfg = await loadConfig(opts.config);
    const schema = await inspectSchema(cfg.schemaPath);
    await generateAll({ cfg, schema, model: opts.model, dryRun: !!opts.dryRun });
    console.log('Generation completed');
  });

program.parseAsync(process.argv);
```

配置校验

```typescript
// src/lib/config.ts
import { promises as fs } from 'fs';
import { z } from 'zod';
import path from 'node:path';

const OutputSchema = z.object({
  moduleRoot: z.string(), // e.g. apps/base-system/src/modules
  moduleName: z.string(), // e.g. iam
  resourceName: z.string(), // e.g. user
});

export const ConfigSchema = z.object({
  schemaPath: z.string(),
  projectRoot: z.string().default('.'),
  output: OutputSchema,
  idStrategy: z.enum(['domain', 'db']).default('domain'),
  immutableFields: z.array(z.string()).default([]),
  baseFields: z.object({
    id: z.string(),
    isDeleted: z.string(),
    isActive: z.string(),
    createdAt: z.string(),
    updatedAt: z.string(),
    createdBy: z.string(),
    updatedBy: z.string(),
  }),
  softDeleteField: z.string().optional(),
  read: z
    .object({
      pageSizeDefault: z.number().default(10),
    })
    .default({ pageSizeDefault: 10 }),
  modelMap: z.record(z.string()).default({}),
});

export type GenConfig = z.infer<typeof ConfigSchema>;

export async function loadConfig(configPath: string): Promise<GenConfig> {
  const abs = path.resolve(process.cwd(), configPath);
  const raw = await fs.readFile(abs, 'utf-8');
  const json = JSON.parse(raw);
  const parsed = ConfigSchema.parse(json);
  // normalize schema path
  parsed.schemaPath = path.resolve(path.dirname(abs), parsed.schemaPath);
  return parsed;
}
```

默认配置文件生成

```typescript
// src/lib/init.ts
import { promises as fs } from 'fs';
import path from 'node:path';

export async function writeDefaultConfig(force = false) {
  const target = path.resolve(process.cwd(), '.rcerpgen.json');
  try {
    if (!force) {
      await fs.access(target);
      throw new Error('.rcerpgen.json already exists, use --force to overwrite');
    }
  } catch {
    /* not exists */
  }
  const tpl = {
    schemaPath: 'prisma/schema.prisma',
    projectRoot: '.',
    output: { moduleRoot: 'apps/base-system/src/modules', moduleName: 'iam', resourceName: 'user' },
    idStrategy: 'domain',
    immutableFields: ['username', 'password'],
    baseFields: {
      id: 'id',
      isDeleted: 'isDeleted',
      isActive: 'isActive',
      createdAt: 'createdAt',
      updatedAt: 'updatedAt',
      createdBy: 'createdBy',
      updatedBy: 'updatedBy',
    },
    softDeleteField: 'isDeleted',
    read: { pageSizeDefault: 10 },
    modelMap: { SysUsers: 'User' },
  };
  await fs.writeFile(target, JSON.stringify(tpl, null, 2));
}
```

prisma 解析

```typescript
// src/lib/prisma.ts
import { promises as fs } from 'fs';
import { getSchema, Model, Schema } from '@mrleebo/prisma-ast';

export async function inspectSchema(schemaPath: string): Promise<Schema> {
  const sdl = await fs.readFile(schemaPath, 'utf-8');
  return getSchema(sdl);
}

export function getModels(schema: Schema): Model[] {
  return (schema.list || []).filter((n: any): n is Model => n.type === 'model') as Model[];
}

export function listModels(schema: Schema) {
  return getModels(schema).map((m) => ({
    model: m.name,
    fields: m.properties.filter((p: any) => p.type === 'field').length,
  }));
}
```

项目模块骨架搭建

```typescript
// src/lib/generate.ts
import path from 'node:path';
import { promises as fs } from 'fs';
import { pascalCase, paramCase, camelCase } from 'change-case';
import Handlebars from 'handlebars';
import { Schema, Model } from '@mrleebo/prisma-ast';
import { GenConfig } from './config.js';
import { getModels } from './prisma.js';

Handlebars.registerHelper('pascal', pascalCase);
Handlebars.registerHelper('param', paramCase);
Handlebars.registerHelper('camel', camelCase);
Handlebars.registerHelper('json', (v) => JSON.stringify(v));

const TEMPLATES = {
  domain: 'domain.aggregate.root.hbs',
  service: 'application.service.hbs',
  portsRead: 'ports.read.repo.hbs',
  portsWrite: 'ports.write.repo.hbs',
  commandsCreate: 'command.create.hbs',
  commandsUpdate: 'command.update.hbs',
  tokens: 'tokens.hbs',
};

type GenCtx = {
  Entity: string; // User
  entity: string; // user
  kebab: string; // user
  model: string; // SysUsers
  fields: { name: string; tsType: string; optional: boolean }[];
  base: GenConfig['baseFields'];
  immutable: string[];
};

function prismaTypeToTs(t: string) {
  switch (t) {
    case 'String':
      return 'string';
    case 'Int':
    case 'Float':
    case 'BigInt':
      return 'number';
    case 'Boolean':
      return 'boolean';
    case 'DateTime':
      return 'Date';
    case 'Decimal':
      return 'string';
    default:
      return 'any';
  }
}

function buildCtx(cfg: GenConfig, m: Model): GenCtx {
  const modelName = m.name;
  const Entity = cfg.modelMap[modelName] ?? modelName;
  const entity = camelCase(Entity);
  const kebab = paramCase(Entity);

  const fields = m.properties
    .filter((p: any) => p.type === 'field')
    .map((f: any) => ({
      name: f.name,
      tsType: prismaTypeToTs(f.fieldType),
      optional: !!f.optional,
    }));

  return { Entity, entity, kebab, model: modelName, fields, base: cfg.baseFields, immutable: cfg.immutableFields };
}

async function renderTemplate(file: string, ctx: any) {
  const tplPath = path.resolve(path.dirname(new URL(import.meta.url).pathname), '../../templates', file);
  const tpl = await fs.readFile(tplPath, 'utf-8');
  const compiled = Handlebars.compile(tpl, { noEscape: true });
  return compiled(ctx);
}

async function writeFileIfChanged(target: string, content: string, dryRun: boolean) {
  if (dryRun) {
    console.log(`[dry] write ${target}`);
    return;
  }
  await fs.mkdir(path.dirname(target), { recursive: true });
  await fs.writeFile(target, content);
}

export async function generateAll(params: { cfg: GenConfig; schema: Schema; model?: string; dryRun: boolean }) {
  const { cfg, schema, model, dryRun } = params;
  const models = getModels(schema).filter((m) => (model ? m.name === model : true));
  for (const m of models) {
    const ctx = buildCtx(cfg, m);
    const modRoot = path.resolve(
      cfg.projectRoot,
      cfg.output.moduleRoot,
      cfg.output.moduleName,
      ctx.entity, // resourceName 一级子目录
    );
    // domain
    const domain = await renderTemplate(TEMPLATES.domain, ctx);
    await writeFileIfChanged(path.join(modRoot, 'domain', `${ctx.kebab}.aggregate.root.ts`), domain, dryRun);
    // application
    const service = await renderTemplate(TEMPLATES.service, ctx);
    await writeFileIfChanged(
      path.join(modRoot, 'application', 'services', `${ctx.kebab}.writer.service.ts`),
      service,
      dryRun,
    );

    const portsR = await renderTemplate(TEMPLATES.portsRead, ctx);
    await writeFileIfChanged(path.join(modRoot, 'application', 'ports', `${ctx.kebab}.read.repo.ts`), portsR, dryRun);

    const portsW = await renderTemplate(TEMPLATES.portsWrite, ctx);
    await writeFileIfChanged(path.join(modRoot, 'application', 'ports', `${ctx.kebab}.write.repo.ts`), portsW, dryRun);

    const cmdC = await renderTemplate(TEMPLATES.commandsCreate, ctx);
    await writeFileIfChanged(
      path.join(modRoot, 'application', 'commands', `${ctx.kebab}-create.command.ts`),
      cmdC,
      dryRun,
    );

    const cmdU = await renderTemplate(TEMPLATES.commandsUpdate, ctx);
    await writeFileIfChanged(
      path.join(modRoot, 'application', 'commands', `${ctx.kebab}-update.command.ts`),
      cmdU,
      dryRun,
    );

    const tokens = await renderTemplate(TEMPLATES.tokens, ctx);
    await writeFileIfChanged(path.join(modRoot, 'application', 'tokens.ts`'.replace('`', '')), tokens, dryRun);
  }
}

export function listModels(schema: Schema) {
  return getModels(schema).map((m) => ({ model: m.name }));
}
```

## 配置代码模板

`templates/domain.aggregate.root.hbs`

```hbs
import { AggregateRootBase, BaseProps, WithoutBaseAnd } from '@rcerp/core/aggregate-root.base';
import { randomUUID } from 'crypto';

type {{Entity}}Props = {
{{#each fields}}
  {{this.name}}: {{this.tsType}};
{{/each}}
} & BaseProps;

type {{Entity}}CreateProps = WithoutBaseAnd<{{Entity}}Props>;
type {{Entity}}UpdateProps = Partial<WithoutBaseAnd<{{Entity}}Props, '{{#each immutable}}{{this}}{{#unless @last}}' | '{{/unless}}{{/each}}'>>;

export class {{Entity}}AggregateRoot extends AggregateRootBase<{{Entity}}Props> {
  static create(props: {{Entity}}CreateProps): {{Entity}}AggregateRoot {
    const now = new Date();
    return new {{Entity}}AggregateRoot({
      ...props,
      id: randomUUID(),
      createdAt: now,
      updatedAt: now,
      isDeleted: false,
      isActive: true,
      createdBy: null,
      updatedBy: null,
    });
  }

  static rehydrate(props: {{Entity}}Props): {{Entity}}AggregateRoot {
    return new {{Entity}}AggregateRoot(props);
  }

  public update(props: {{Entity}}UpdateProps): void {
    this.setProps({
      ...props,
{{#each fields}}
      {{this.name}}: (props as any).{{this.name}} ?? (this as any).props.{{this.name}},
{{/each}}
    });
  }

  public delete(): void {
    this.setProps({ isDeleted: true });
  }

{{#each fields}}
  public get {{this.name}}(): {{this.tsType}} {
    return (this as any).props.{{this.name}};
  }
{{/each}}
}

```

`templates/application.service.hbs`

```hbs
import { Transactional } from '@nestjs-cls/transactional';
import { Inject, Injectable } from '@nestjs/common';
import { BizException, ErrorCode } from '@rcerp/infra/errors/error-code.enum';
import { {{Entity}}AggregateRoot } from '../../domain/{{param Entity}}.aggregate.root';
import { {{Entity}}CreateCommand } from '../commands/{{param Entity}}-create.command';
import { {{Entity}}UpdateCommand } from '../commands/{{param Entity}}-update.command';
import { I{{Entity}}WriteRepo } from '../ports/{{param Entity}}.write.repo';
import { I{{Entity}}WriteRepoToken } from '../tokens';

@Injectable()
export class {{Entity}}WriteService {
  constructor(
    @Inject(I{{Entity}}WriteRepoToken)
    private readonly repo: I{{Entity}}WriteRepo,
  ) {}

  @Transactional()
  async create{{Entity}}(command: {{Entity}}CreateCommand): Promise<void> {
    const agg = {{Entity}}AggregateRoot.create({
{{#each fields}}
      {{this.name}}: (command as any).{{this.name}},
{{/each}}
    });
    await this.repo.save(agg);
  }

  @Transactional()
  async update{{Entity}}(command: {{Entity}}UpdateCommand): Promise<void> {
    const agg = await this.repo.getByIdForWrite(command.id);
    if (!agg) {
      throw new BizException(ErrorCode.BIZ_ERROR, '查无该{{entity}}');
    }
    agg.update({
{{#each fields}}
      {{this.name}}: (command as any).{{this.name}},
{{/each}}
    });
    await this.repo.save(agg);
  }

  @Transactional()
  async delete{{Entity}}(id: string): Promise<void> {
    const agg = await this.repo.getByIdForWrite(id);
    if (!agg) {
      throw new BizException(ErrorCode.BIZ_ERROR, '查无该{{entity}}');
    }
    agg.delete();
    await this.repo.save(agg);
  }
}
```

`templates/ports.read.repo.hbs`

```hbs
import { PaginationResult } from '@rcerp/shared/prisma/pagination';
import { {{Entity}}ReadModel } from '../read-models/{{param Entity}}.read.model';
import { Page{{Entity}}sQuery } from '../queries/page-{{param Entity}}s.query';

export interface I{{Entity}}ReadRepo {
  page{{Entity}}s(query: Page{{Entity}}sQuery): Promise<PaginationResult<{{Entity}}ReadModel>>;
  findById(id: string): Promise<{{Entity}}ReadModel>;
}
```

`templates/ports.write.repo.hbs`

```hbs
import { {{Entity}}AggregateRoot } from '../../domain/{{param Entity}}.aggregate.root';

export interface I{{Entity}}WriteRepo {
  getByIdForWrite(id: string): Promise<{{Entity}}AggregateRoot | null>;
  save(agg: {{Entity}}AggregateRoot): Promise<void>;
}
```

`templates/commands.create.hbs`

```hbs
export class
{{Entity}}CreateCommand { constructor(
{{#each fields}}
  readonly
  {{this.name}}{{#if this.optional}}?{{/if}}:
  {{this.tsType}},
{{/each}}
) {} }
```

`templates/commands.update.hbs`

```hbs
export class
{{Entity}}UpdateCommand { constructor( readonly id: string,
{{#each fields}}
  readonly
  {{this.name}}{{#if this.optional}}?{{/if}}:
  {{this.tsType}},
{{/each}}
) {} }
```

`templates/tokens.hbs`

```hbs
export const I{{Entity}}ReadRepoToken = Symbol('I{{Entity}}PrismaReadRepo'); export const I{{Entity}}WriteRepoToken =
Symbol('I{{Entity}}PrismaWriteRepo');
```
