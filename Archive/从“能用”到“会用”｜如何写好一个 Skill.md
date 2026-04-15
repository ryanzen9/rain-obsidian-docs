URL: https://zhuanlan.zhihu.com/p/2000521052305003174
上次编辑时间: 2026年3月25日 17:54
创建时间: 2026年3月24日 18:00
标签: AI, 好文记录
状态: 完成
目标追踪: AI 学习 (https://www.notion.so/AI-328a89f7e47d805fbbdee2ff9dcf08b5?pvs=21)

> 本文作者：Jiaqi，TRAE 技术文档工程师

之前带大家了解了 [Skill](https://zhida.zhihu.com/search?content_id=269761343&content_type=Article&match_order=1&q=Skill&zhida_source=entity) 的原理和应用，本文继续带领大家从“能用”到“会用”，如何写好一个 Skill 至关重要。

本文为 Skill 设计与开发的一站式最佳实践指南，可助力你：

- 系统厘清 Skill 的定义，规避常见认知误区；
- 熟练掌握高命中率、高稳定性 Skill 的设计标准与核心原则；
- 掌握 “[评测驱动](https://zhida.zhihu.com/search?content_id=269761343&content_type=Article&match_order=1&q=%E8%AF%84%E6%B5%8B%E9%A9%B1%E5%8A%A8&zhida_source=entity)、[失败优先](https://zhida.zhihu.com/search?content_id=269761343&content_type=Article&match_order=1&q=%E5%A4%B1%E8%B4%A5%E4%BC%98%E5%85%88&zhida_source=entity)” 的工程化 Skill 构建与迭代全流程；
- 掌握可维护、可扩展的 Skill 设计技巧；
- 依托 AI 高效完成 Skill 的创建、迭代与优化；
- 凭借[反模式检查清单](https://zhida.zhihu.com/search?content_id=269761343&content_type=Article&match_order=1&q=%E5%8F%8D%E6%A8%A1%E5%BC%8F%E6%A3%80%E6%9F%A5%E6%B8%85%E5%8D%95&zhida_source=entity)，精准规避 Skill 开发中的典型问题。

## **1、什么是 Skill ？**

### **定义**

一个 Skill 是一份清晰、严谨、可执行的指令文档，用于明确告诉模型——在什么条件下（When），按照哪些步骤（How），产出什么结果（What）。

### **常见认知误区**

在编写 Skill 之前，建议先了解以下几个常见误区。这些问题往往会直接导致 Skill 难以被正确触发，或在执行过程中表现不稳定。

**误区一：Skill 等同于一段 [Prompt](https://zhida.zhihu.com/search?content_id=269761343&content_type=Article&match_order=1&q=Prompt&zhida_source=entity)**

Skill 并不是一次性的对话提示。它是一个可长期复用、输入输出明确的能力模块，强调的是稳定、确定且易于工程化维护。而 Prompt 更偏向临时性、探索性和即兴交互，两者在设计目标和工程要求上完全不同。

**误区二：Skill 是写给人看的文档**

Skill 的目标不是“解释原理”，而是“下达指令”。***SKILL.md*** 文件的内容应使用模型可解析的结构化语言，明确约束其行为边界，并精确描述何时使用（When）、如何执行（How）、输出结果（What）。

**误区三：Skill 越复杂越强大**

复杂度并不与 Skill 的能力强度挂钩。模型的推理与决策成本是显著的。职责单一、边界清晰的 Skill，更容易在正确的时机被选中并稳定执行。过于复杂的 Skill 反而会降低命中率。

**提示：**模型的[上下文窗口](https://zhida.zhihu.com/search?content_id=269761343&content_type=Article&match_order=1&q=%E4%B8%8A%E4%B8%8B%E6%96%87%E7%AA%97%E5%8F%A3&zhida_source=entity)是有限且宝贵的公共资源。每一个被加载的 Skill 都在竞争有限的上下文资源。因此，Skill 文档应以最小必要信息为目标，避免冗余解释与不必要的背景铺垫。

## **2、Skill 的设计标准与原则**

以下标准是构建高命中率、高稳定性 Skill 的基础。可将其作为设计与评审 Skill 时的检查清单。

### **精准的[元数据](https://zhida.zhihu.com/search?content_id=269761343&content_type=Article&match_order=1&q=%E5%85%83%E6%95%B0%E6%8D%AE&zhida_source=entity)内容**

Skill 的元数据（name 和 description）是模型发现和识别 Skill 的入口，其设计直接影响触发准确率。

![](https://pic3.zhimg.com/v2-f6bf06bbb360b3b47a9728e7f6aa05b0_1440w.jpg)

### **指导方式的自由度分级**

根据任务复杂度与容错要求，合理控制对模型的约束强度。

![](https://pic4.zhimg.com/v2-adccf7f74bc1465df53ff38ebe9c9879_1440w.jpg)

### **五个核心标准**

![](https://pic3.zhimg.com/v2-fa560edf40f488cf2c22d6595365b094_1440w.jpg)

## **3、构建与迭代 Skill 的最佳流程**

Skill 的开发是一个以失败为起点、评测为牵引，持续迭代优化的工程化过程。

评测并非事后的验证环节，而是 Skill 设计的前提；Skill 也非基于假设的规则集合，而是针对已暴露问题的最小化解决方案。

遵循 “评测驱动、失败优先” 的原则，能确保 Skill 在实际使用场景中拥有清晰的能力边界、稳定的运行表现，以及可回归的质量保障体系。

### **第一步：建立无 Skill 基线，识别真实问题**

在编写任何 Skill 之前，应先不使用 Skill，直接让模型执行目标任务，作为基线对照。

重点观察并记录以下问题：

- 模型在哪些情况下表现不稳定或结果不可复现；
- 哪些输入会引发歧义、误解或走偏；
- 模型是否在错误的时机尝试“主动帮忙”。

这些失败点和不确定行为，本质上就是 Skill 需要解决的真实能力缺口，也是后续评测用例的来源。

### **第二步：以“失败优先”为原则，定义评测用例**

明确核心问题后，需优先编写评测用例，而非直接开发 Skill。评测是约束，Skill 是落地实现。脱离评测约束的 Skill，本质上是在放大模型的行为不确定性。

评测的核心作用是约束 Skill 的行为边界，明确以下信息：

- 什么场景下属于 Skill 的正确使用范围？
- 什么场景下 Skill 必须拒绝执行或判定为失败？
- 什么样的输出结果才算稳定可用？

推荐做法：

- 针对已识别的问题，设计 3–5 个具体、可复现的评测用例；
- 每个用例均需明确 “通过 / 失败” 的判定标准；
- 优先覆盖模型最易误用 Skill 的场景。

### **第三步：编写最小化 Skill，明确最短成功路径**

在评测已经存在的前提下，开始编写 Skill。

此阶段不追求覆盖所有情况，而是只编写刚好能够通过当前评测的最小规则集合，重点关注三个要素：

- **明确失败条件（When NOT to use）**
将评测中识别的失败场景，显式写入 Skill 中，作为第一层防护，避免 Skill 被误触发或滥用。
- **定义最短成功路径**
清晰描述 Skill 最简单、最核心的执行流程，确保最精简的有效输入能得到可预测的稳定输出。
- **保持职责单一**
单个 Skill 仅解决一个明确问题，对应一个核心动作，避免在早期引入额外复杂度。

这一阶段的 Skill，是评测结果的直接产物，而非凭经验预判的方案。

### **第四步：补充边界条件与结构化示例**

当最短成功路径能稳定通过评测后，再逐步扩展 Skill 的适用范围。

完成以下三项核心工作：

- 补充更多边界场景及对应的行为约束；
- 明确 Skill 输入（Input）、输出（Output）的结构化定义；
- 补充关键的输入、输出示例，帮助模型对齐行为预期。

**核心原则：**所有新增规则，均需对应新增或已有评测用例，避免在无评测支撑的情况下将 Skill 复杂化。

### **第五步：评测回归与持续迭代**

Skill 的迭代，需始终与评测结果强绑定，遵循以下规则：

- 新增评测用例 → 推动 Skill 的增量修改；
- 任意修改 Skill → 必须通过已有评测的回归验证；
- 若评测未通过，优先简化 Skill，而非盲目叠加新规则。

通过持续对比模型在 “无 Skill 基线” 与 “当前 Skill + 评测” 中的表现，可验证该 Skill 是否真正提升了模型执行目标任务的成功率与稳定性。

### **第六步：结合真实使用路径进行校准**

评测仅能覆盖已知问题，在真实场景中使用 Skill 时，可能会暴露更多新的问题。

在 Skill 实际使用过程中，需持续观察以下情况：

- 模型是否在非预期场景下误触发 Skill？
- 模型执行 Skill 时，是否遗漏关键参考文件或上下文？
- 模型是否会反复读取某一段内容，形成隐性依赖？

上述这些信号，均需作为新的评测输入，重新进入 Step 2，形成 Skill 迭代的闭环。

## **4、让 Skill 可维护与可拓展**

### **渐进式披露**

***SKILL.md*** 应当作为 Skill 的入口和导航，而不是一个包罗万象的大文件。详细的参考资料、示例、脚本或文档应拆分成独立文件，从而减轻模型初次加载的负担，让信息按需流动。

**信息架构原则：从简单到复杂**

一个 Skill 的目录可以随着功能扩展逐步演化：从单一文件 → 多个参考文件和脚本组成的结构。通过渐进式披露，模型能快速抓住核心信息，再深入了解细节。

**最佳实践**

- **保持 SKILL.md 简洁：**主体内容尽量控制在 500 行以内，只包含必要信息。
- **避免深度嵌套：**所有引用文件最好直接由 ***SKILL.md*** 链接，保持一层引用深度，避免链式引用（A → B → C），防止模型只读取部分内容。
- **为长文件添加目录：**对于超过 100 行的参考文件，在文件顶部添加一个目录（Table of Contents），帮助模型快速了解文件结构。

**示例**

```
# SKILL.md
## 基础用法
描述如何触发 CI/CD 流水线：
- 检查 PR 状态
- 执行单元测试
- 更新 PR 测试状态
## 高级功能
详细说明请参见 `ci-advanced-features.md`：
- 并行执行多分支测试
- 条件触发不同类型的测试
- 自定义失败处理策略
## API 参考
所有方法与参数说明请参见 `ci-api-reference.md`：
- startPipeline(prId: string, branch: string)
- getPipelineStatus(pipelineId: string)
- cancelPipeline(pipelineId: string)
```

### [**工作流](https://zhida.zhihu.com/search?content_id=269761343&content_type=Article&match_order=1&q=%E5%B7%A5%E4%BD%9C%E6%B5%81&zhida_source=entity)与反馈闭环**

对于包含多个步骤、且中间结果会影响最终质量的复杂任务，仅提供最终目标是不够的。必须显式定义工作流（Workflow）和 检查清单（Checklist），引导模型按步骤执行，并在关键节点建立 “验证 → 修正 → 再验证” 的反馈闭环。

工作流负责约束任务执行顺序，检查清单负责追踪任务的执行状态和质量。两者结合可以显著降低遗漏和跑偏的风险。

**分析类任务的工作流**

即使不涉及代码，分析类任务同样适合使用工作流。 检查清单可以帮助模型明确“当前做到哪一步”、“是否可以进入下一步”。例如：

```
## 技术方案评估工作流
在开始执行前复制以下清单，并在每一步完成后显式标记状态。
- Step 1：明确业务目标与技术约束（性能、成本、时限）
- Step 2：列出所有可行的技术方案
- Step 3：从复杂度、可维护性、风险角度逐一评估
- Step 4：对关键差异点进行对比分析；(反馈闭环) 若发现关键信息不足，应返回 Step 2 或 Step 3 补充分析
- Step 5：给出结论性建议，并说明取舍理由；(反馈闭环) 若结论无法支撑目标约束，应重新审视 Step 1 的前提条件
```

**代码类任务的工作流**

代码类任务往往伴随不可逆或影响范围较大的操作，例如重构、依赖升级或配置变更。通过 “Plan → Validate → Execute” 模式，可以有效降低误操作风险。例如：

```
## 依赖版本升级工作流
- Step 1（Plan）：
  - 识别需要升级的依赖及当前版本
  - 阅读目标版本的 Release Notes 与 Breaking Changes
- Step 2（Plan）：
  - 更新依赖配置文件（如 package.json / go.mod）
  - 标注可能受影响的模块
- Step 3（Validate）：
  - 执行依赖冲突检查与静态构建（运行 dependency_check.sh）
  - 确认无版本冲突或构建失败
  - (反馈闭环) 若校验失败，必须回退到 Step 2 调整依赖配置
- Step 4（Execute）：
  - 安装新版本依赖
  - 运行完整测试集
- Step 5（Validate）：
  - 检查核心功能是否受影响
  - 对比升级前后的构建与运行结果
  - (反馈闭环) 若出现回归问题，应回滚升级并记录风险点
```

### **可执行脚本的加固原则**

当 Skill 依赖可执行脚本时，脚本的健壮性应始终优先于代码的巧妙性。

Skill 本身不会理解或阅读你的代码逻辑，它只感知输入与输出。一旦脚本行为不可预测，模型就只能猜测，最终导致不稳定或错误的调用结果。因此，脚本必须做到：失败可预期、输出可理解、参数可解释。

**显式处理错误，而不是让模型猜**

不要将异常直接抛给模型处理。脚本应覆盖常见错误场景，并将技术异常转化为可理解、可决策的输出。

**实践要点：**

- 捕获常见异常（如文件缺失、权限不足、配置错误）
- 为每类错误返回清晰的错误原因和下一步建议

**示例：**配置文件校验脚本

```
ERROR: Config file not found: ./deploy.yaml
HINT: Please check whether the file path is correct or run init-config.sh to generate a default config.
```

**输出自解释的日志与验证结果**

脚本的输出本身就是模型的“上下文”。一个好的脚本不仅说明发生了什么，还说明为什么会这样，以及接下来可以怎么做。

**实践要点：**

- 成功路径和失败路径都要有明确输出
- 验证类脚本应明确列出通过项与失败项

**示例：**构建环境检查脚本

```
CHECK FAILED: Node.js version mismatch
- Required: >= 18.0.0
- Detected: 16.14.0
VALID OPTIONS:
1. Upgrade Node.js to a supported version
2. Switch to a compatible build image
```

**避免魔法数字，让参数“有来由”**

脚本中的常量（如 ***TIMEOUT = 30***）如果缺乏解释，模型和人都无法判断它是否合理。任何影响行为的数值，都应该是可解释、可调整的。

**实践要点：**

- 为常量添加语义化名称
- 说明数值来源或设计依据
- 必要时允许通过参数覆盖默认值

**示例：**部署等待脚本

```
TIMEOUT_SECONDS = 30  # Wait up to 30s because service startup usually completes within 10–20s
```

或在输出中体现：

```
INFO: Waiting for service to become healthy (timeout: 30s)
```

## **5、巧妙使用 AI 来创建和迭代 Skill**

在创建和迭代 Skill 的过程中，可以多用 AI。你负责定义问题和验收结果，AI 负责反复试错、总结规律并封装成可复用的 Skill。

### **阶段一：初次创建（从具体任务中抽象）**

让 AI 从真实任务中抽象出创建 Skill 所需的信息，然后创建初版 Skill。

**1. 让 AI 直接执行真实任务**

提供完整目标与上下文，让 AI 自行尝试完成任务。执行过程中的追问、走偏和修正，本质上就是一次“隐式评测”。

**2. 引导 AI 进行结构化复盘**

任务完成后，要求 AI 从以下维度复盘：

- 成功执行任务的完整步骤；
- 任务执行过程中的不确定性与失败点；
- 可抽象的固定流程与判断逻辑；
- 该流程和判断逻辑的适用场景与不适用场景。

**3. 基于复盘生成 Skill 初稿**

要求 AI 按 Skill 规范生成 SKILL.md，明确：触发条件（When）、如何执行（How）、输出结果（What）、预设失败策略。

**4. 人工快速评审并入库**

你只需关注边界是否合理、步骤是否可执行，其余交由 AI 完成。确认后，让 AI 调用 skills-creator 正式创建该 Skill 并添加至你的项目。

### **阶段二：持续迭代（从使用反馈中优化）**

当 Skill 在使用中暴露新问题时，对其进行优化。

**1. 对齐偏差来源**

引导 AI 分析问题源自 When / What / How 中的哪一部分。

**2. 直接修改 Skill 并验证回归**

更新 ***SKILL.md***，并同时验证：

- 确保本次迭代未破坏原有的“黄金路径”。
- 确保新发现的错误场景已被覆盖。

### **在 TRAE 中创建 Skill 的其他方式**

除了通过 AI 对话创建 Skill 外，你还以手动创建或导入外部 Skill。详细说明点击[技能 - 文档 - TRAE CN](https://link.zhihu.com/?target=https%3A//docs.trae.cn/ide/skills%238be9a856)参考官网文档。

## **6、附录：反模式检查清单**

本表列出了在 Skill 开发中常见的反模式（Anti-Patterns），以及相应的原因、正确示例和错误示例。通过遵循这些规范，可以提升 Skill 的稳定性、可维护性和跨平台兼容性，避免模型误用或运行失败。

该 Checklist 可作为开发、评审和迭代 Skill 时的快速参考指南。

![](https://pic4.zhimg.com/v2-75adc2240bf4e91b004ab349f8dcb94f_1440w.jpg)

​

**参考资料：**[https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/best-practices](https://link.zhihu.com/?target=https%3A//platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/best-practices)