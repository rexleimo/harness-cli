# Prompt Authoring Norms（提示词撰写规范）

适用范围：`skill-sources/**/SKILL.md`、`rex-harness/skill-sources/**/SKILL.md`、所有 agent 角色卡（`agent-sources/roles/*`）以及任何随 AIOS 分发、用于引导模型的提示词/脚本。

## 一句话判据

**代码只校验客观事实（退出码、sha256、schema、哈希、diff 一致性）；一切对模型或用户"意图"的猜测（关键词表、正则匹配、语义分类）都是反模式。** 引导模型的方式是显式声明协议与结构化输出契约（Plan-and-solve / ReAct 式），这与项目北极星（`scripts/lib/planning/workflow-policy.mjs` 头注释、`rex-harness/src/application/derive-facts.mjs`：只从结构化 Observation 与显式 intent 建事实）完全一致。

## 三条规则（DO / DON'T）

### 1. 输出契约优先

- **DO**：每个 SKILL 开头声明输入前提（当前 Command / activationId）与输出契约（结构化 evidence kinds + 恰好一个信封）。`skill-sources/rex-planning/SKILL.md` 是范本。
- **DON'T**：写"若用户提到 X/Y 词则进入 Z 流程"的关键词分支。改为要求模型输出显式 `intent` / `task-type` 字段，供 policy 查表路由（无声明回退默认值）。

### 2. 语义判断改为模型自报告字段（ReAct 式观察-声明循环）

- **DO**：把"是否只读？是否多步？是否完成？是否卡住？"等语义判断写成"请在结构化输出中自报告字段 X"，并给出字段定义与 few-shot 示例。例如卡死检测：模型每轮在 observations 里自报告 `progress_made: false + blocked_reason`，runtime 只计连续声明次数。
- **DON'T**：把中英文关键词清单、正则表达式写进 SKILL 或配套脚本充当"理解器"（意图分类器、情绪判断、完成判定）。也不要在 runtime 里调 LLM 替代自报告——那是把语义判断藏进更贵的黑盒。

### 3. 硬门槛写成验证协议

- **DO**：需要强制保证的事（测试通过、证据存在、预算未超）写成验证协议——"命令必须真实执行并在 evidence 中引用退出码/哈希"，SKILL 只描述义务，runtime 执行校验（现有实现：`rex-harness/src/application/validate-command-evidence.mjs`）。
- **DON'T**：用"请确保测试通过"之类的提示词替代 runtime 门禁；也不要在 runtime 里为提示词失灵写文本启发式兜底——那会同时违反北极星并掩盖协议缺口。

## 反模式速查

| 反模式 | 为什么错 | 正确做法 |
| --- | --- | --- |
| 关键词/正则判断用户意图 | 文本猜测语义，北极星明令禁止 | 模型显式声明 `intent`，policy 查表 |
| 正则抽取实体/关键词打分做检索加成 | 硬编码理解器，换语言即碎 | 记忆写入时模型输出结构化 `entities:[]` 字段，检索端只做确定性加权 |
| 调 LLM 判"是否死循环/是否完成" | 语义判断外包给黑盒，不可复现 | ReAct 自报告字段 + runtime 计数（参考 MagenticOne progress ledger） |
| 提示词要求"确保测试通过" | 无强制力 | evidence envelope 引用真实执行回执（退出码=0） |
| 策略规则匹配消息语义 | 规则引擎变成隐式 NLU | 规则只匹配工具名/命令文本/路径等客观字段，并写明该边界 |

## 已有机制对照（写提示词前先看）

- 显式意图路由：`workflow-policy.mjs` 的显式 intent → route 映射；
- 事实推导：`rex-harness/src/application/derive-facts.mjs` 的 `OBSERVATION_TO_FACT`；
- 证据协议：`scripts/lib/workflows/rex-capability-runtime.mjs` 的单行 evidence envelope；
- 能力选择：`rex-harness/src/composition-root.mjs`（按 facts 与 priority 取首个触发，不猜语义）。
