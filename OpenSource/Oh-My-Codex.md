
## Prompt:

```markdown
请参考 `https://github.com/Yeachan-Heo/oh-my-codex`(Oh-My-Codex,以下简称 OMC) 的相关资料，查询社区的相关使用案例，跟我介绍一下它是什么？ 我应该如何使用它。 
```

## What is this ?

运行在 Codex Cli 上的一个编排层，负责组织工作流，任务调度，状态管理的脚手架。

它保留了 Codex 作为执行引擎，并简化了以下操作：

- start a stronger Codex session by default  
    默认启动更强大的 Codex 会话
- run one consistent workflow from clarification to completion  
    从澄清到完成，运行一致的工作流程
- invoke the canonical skills with `$deep-interview`, `$ralplan`, `$team`, and `$ralph`  
    使用 `$deep-interview` 、 `$ralplan` 、 `$team` 和 `$ralph` 来调用规范技能。
- keep project guidance, plans, logs, and state in `.omx/`  
    将项目指南、计划、日志和状态保存在 `.omx/` 目录中

## Install

Install Dependency

```shell
npm install -g @openai/codex oh-my-codex
omx setup
omx --madmax --high
```

使用 ClaudeCode 进行项目重构的头脑风暴
```
Use `Plan Mode`, First please use `Deep Research` analysis this project, and tell me: if i want to use java refactoring this project,What does the project architecture and the related transformation plan look like? Provide a complete transformation checklist, including the target architecture, implementation objectives, and execution steps. The entire response should be in Chinese.
```

Then work normally inside Codex:  

```
$deep-interview "clarify the authentication change"
$ralplan "approve the auth plan and review tradeoffs"
$ralph "carry the approved plan to completion"
$team 3:executor "execute the approved plan in parallel"
```