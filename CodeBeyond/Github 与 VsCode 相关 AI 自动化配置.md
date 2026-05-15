## VsCode 开发相关配置 

安装依赖： 
- GitHub Copilot
- GitHub Copilot Chat

Copilot 会读取 `.github/copilot-instructions.md`

项目内创建：

```
.github/copilot-instructions.md
```

内容：

```
# Commit Message RulesGenerate commit messages using Conventional Commits.Rules:- Use type(scope): subject- subject must be lowercase- keep under 72 chars- explain WHY not only WHATAllowed types:- feat- fix- refactor- docs- test- chore- perfExamples:- feat(auth): add jwt refresh flow- fix(api): handle null response in user endpoint- refactor(cache): simplify redis key generationAvoid:- update code- fix bug- minor changes
```