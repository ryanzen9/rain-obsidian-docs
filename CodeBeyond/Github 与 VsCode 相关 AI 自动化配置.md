## VsCode 开发相关配置 

安装依赖： 
- GitHub Copilot
- GitHub Copilot Chat

Copilot 现在有一整套 instruction 系统, 可以通过相关配置，传入提示词。

### 使用方式

#### 内联使用

在 `.vscode/setting.json` 中

>**github.copilot.chat.commitMessageGeneration.instructions**:  给 VS Code 的 GitHub Copilot “生成提交信息”功能追加自定义提示词。

```json
{
  "github.copilot.chat.commitMessageGeneration.instructions": [
    {
      "text": "Use Conventional Commits format"
    },
    {
      "text": "Use Chinese"
    },
    {
      "text": "Subject line max 72 chars"
    }
  ]
}
```

#### 文件使用

```json
{  
	"file": "./.github/instructions/commit.md"  
}
```
### 提交消息自动生成

项目内创建：

```
.github/instructions/commit-message-instructions.md
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

更改配置文件

```setting.json
```
### 提交前自动 Review

项目内创建：

```
.github/instructions/code-review-instructions.md
```

内容：

```markdown
Review code like a strict senior engineer.

Focus on:

- correctness
- hidden bugs
- race conditions
- async issues
- security risks
- performance regressions
- maintainability
- API contract violations

Avoid:

- compliments
- generic comments
- explaining obvious syntax

Output:

1. Critical Issues
2. Improvements
3. Nitpicks

Keep comments concise and actionable.
```