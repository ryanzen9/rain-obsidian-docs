## VsCode 开发相关配置 

安装依赖： 
- GitHub Copilot
- GitHub Copilot Chat

Copilot 会读取 `.github/copilot-instructions.md`

### 提交消息自动生成

项目内创建：

```
.github/commit-message-instructions.md
```

内容：

```markdown
# Commit Message Rules

Generate commit messages using Conventional Commits.

Rules:
- Use type(scope): subject
- subject must be lowercase
- keep under 72 chars
- explain WHY not only WHAT

Allowed types:
- feat
- fix
- refactor
- docs
- test
- chore
- perf

Examples:
- feat(auth): add jwt refresh flow
- fix(api): handle null response in user endpoint
- refactor(cache): simplify redis key generation

Avoid:
- update code
- fix bug
- minor changes
```

### 提交前自动 Review

项目内创建：

```
.github/commit-message-instructions.md
```

内容：

```

创建 `code-review-instructions.md` 文件

```markdown
# Code Review Rules

When reviewing code:

Focus on:
- correctness
- edge cases
- performance
- security
- readability
- architecture consistency

Avoid:
- praising
- generic comments
- obvious explanations

Prefer:
- actionable review comments
- concrete fixes
- detecting hidden bugs
- identifying race conditions
- checking async correctness

Review style:
- concise
- technical
- direct
```