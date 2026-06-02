# Document

个人技术文档库，主要面向 Obsidian、GitHub Markdown 和 AI 辅助写作工作流。仓库同时维护项目级 Agent Skills，用来让 Codex、Claude Code 等编码代理在处理文档时复用固定流程。

## 目录结构

| 路径 | 用途 |
| --- | --- |
| `AI/` | AI、Agent、提示词与工具链相关笔记 |
| `ArchitectureDesign/` | 架构设计、DDD、系统设计文章 |
| `CodeBeyond/` | 开发环境、自动化、工具配置 |
| `ComputerBasics/` | 计算机基础知识 |
| `Database/` | 数据库、事务、调优与分布式事务 |
| `Development/` | 编程语言、框架、API、测试与工程实践 |
| `Insight/` | 经验总结与深度思考 |
| `OpenSource/` | 开源项目学习与实践记录 |
| `.agents/skills/` | Codex / Agent Skills 项目级技能 |
| `.claude/skills/` | Claude Code 项目级技能 |

## 已维护的 Skills

| Skill | 位置 | 作用 |
| --- | --- | --- |
| `format-docs` | `.agents/skills/format-docs/`、`.claude/skills/format-docs/` | 检查和统一 Markdown 博客格式，不改写语义 |
| `write-docs` | `.agents/skills/write-docs/`、`.claude/skills/write-docs/` | 辅助技术文档写作、润色和内容补全 |

新增 Skill 时，推荐先维护 `.agents/skills/<skill-name>/SKILL.md`，确认可用后再同步到 `.claude/skills/<skill-name>/SKILL.md`。

## Skill 基本格式

每个 Skill 是一个目录，至少包含一个 `SKILL.md`：

```text
.agents/skills/
  skill-name/
    SKILL.md
    references/
    scripts/
    assets/
```

`SKILL.md` 必须包含 YAML frontmatter 和 Markdown 正文：

```markdown
---
name: skill-name
description: Use when the agent should apply this workflow for specific tasks or symptoms.
---

# Skill Name

## Overview

一句话说明核心原则。

## When to Use

- 明确触发场景
- 明确不该使用的场景

## Workflow

1. 读取上下文
2. 执行检查或生成
3. 验证结果
4. 输出变更摘要
```

命名建议：

- `name` 使用小写字母、数字和连字符，目录名与 `name` 保持一致。
- `description` 只写触发条件，避免把完整流程塞进描述字段。
- 正文写可执行步骤、检查清单、常见错误和输出格式。
- 大段参考资料放进 `references/`，脚本放进 `scripts/`，素材放进 `assets/`。

## 在 Codex 中使用

Codex 当前可以读取本项目的 `.agents/skills/`。项目级 Skill 适合随仓库共享，团队成员克隆仓库后即可使用同一套文档流程。

添加一个新 Skill：

```bash
mkdir -p .agents/skills/drawing-docs
$EDITOR .agents/skills/drawing-docs/SKILL.md
```

在 Codex 对话中触发：

```text
请使用 drawing-docs 帮我为这篇文章补充架构图。
```

验证方式：

```text
请列出你会使用哪个 Skill，并说明为什么。
```

如果 Codex 没有自动触发，通常是 `description` 太泛、关键词不足，或者 Skill 文件不在 `.agents/skills/<name>/SKILL.md` 结构下。

## 在 Claude Code 中使用

Claude Code 项目级 Skill 放在 `.claude/skills/`：

```bash
mkdir -p .claude/skills/drawing-docs
cp .agents/skills/drawing-docs/SKILL.md .claude/skills/drawing-docs/SKILL.md
```

个人全局 Skill 可放在 `~/.claude/skills/`，适合跨项目复用；项目级 `.claude/skills/` 更适合与仓库一起提交。

使用方式：

```text
请使用 drawing-docs 优化这篇 Markdown 的图表说明。
```

修改或新增 Skill 后，建议开启新会话或重启 Claude Code，再用一次明确触发词验证是否被识别。

## 使用 `npx skills` 安装或分享

`skills` CLI 适合从 GitHub、GitLab、git URL 或本地目录安装 Skill。它更像 Agent Skills 生态里的通用安装器，适合把本仓库发布为可被别人安装的 Skill 源。

常用命令：

```bash
# 从 GitHub 仓库安装
npx skills add <owner>/<repo>

# 只安装某个 skill
npx skills add <owner>/<repo> --skill format-docs

# 从本地目录安装，适合发布前测试
npx skills add /Users/ruby/Work/Document/.agents/skills

# 安装为全局 skill
npx skills add -g <owner>/<repo>

# 查看、搜索、更新
npx skills list
npx skills find docs
npx skills update
```

推荐流程：

1. 在本仓库维护 `.agents/skills/<skill-name>/SKILL.md`。
2. 本地用 `npx skills add /Users/ruby/Work/Document/.agents/skills --skill <skill-name>` 测试安装。
3. 推送到 GitHub。
4. 在其他项目运行 `npx skills add <owner>/<repo> --skill <skill-name>`。

## 使用 `npm-skills` 包化分发

`npm-skills` 适合把 Skill 跟 npm 包、私有 registry、版本号和 `package.json` 工作流绑定。它会从依赖包中抽取 `skills/` 目录并复制到工作区，默认目标通常是 `.agents/skills`。

如果只是当前仓库自用，不需要 npm 包化；如果你希望团队通过 npm 私有包同步 Skills，可以使用这个流程。

发布方结构：

```text
my-doc-skills-package/
  package.json
  skills/
    format-docs/
      SKILL.md
    write-docs/
      SKILL.md
```

发布方命令：

```bash
npm install -D npm-skills
npx npm-skills new drawing-docs --folder skills
npm publish
```

使用方命令：

```bash
npm install -D <your-skill-package> npm-skills
npx npm-skills extract <your-skill-package> --output .agents/skills --override
```

也可以在使用方项目的 `package.json` 中添加脚本：

```json
{
  "scripts": {
    "skills:extract": "npm-skills extract --output .agents/skills"
  }
}
```

之后运行：

```bash
npm run skills:extract
```

## 新增 Skill 检查清单

1. 先写一个失败场景：没有这个 Skill 时，代理会漏掉什么、误改什么、或做出什么不稳定判断。
2. 写最小可用的 `SKILL.md`，只覆盖这个失败场景。
3. 用同一个任务重新测试，确认代理会读取并执行 Skill。
4. 补充常见错误、验收标准和输出格式。
5. 同步 `.agents/skills/` 与 `.claude/skills/`。
6. 用真实 Markdown 文章跑一次，确认没有破坏代码块、链接、frontmatter 或正文语义。

## 维护建议

- 项目内优先使用 `.agents/skills/` 和 `.claude/skills/`，让 Skill 跟文档一起版本化。
- 不要安装未审查的第三方 Skill；安装前阅读 `SKILL.md`、`scripts/` 和 `references/`。
- 文档类 Skill 应避免过度泛化，触发词要覆盖“写作、润色、格式检查、图表、Markdown、Obsidian、GitHub”等实际请求。
- 如果 Skill 需要联网、执行脚本或修改文件，必须在正文中写清楚边界和验证方式。

## 参考资料

- [Agent Skills Specification](https://agentskills.io/specification)
- [Claude Code Agent Skills](https://docs.claude.com/en/docs/claude-code/skills)
- [OpenAI Help: Skills in ChatGPT](https://help.openai.com/en/articles/20001066-skills-in-chatgpt)
- [Vercel Skills CLI Documentation](https://skills.sh/docs/cli)
- [npm-skills README](https://npm-skills.com/)
