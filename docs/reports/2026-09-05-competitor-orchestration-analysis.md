# 竞品编排能力全景分析（16 仓库源码级）

- **日期**: 2026-09-05
- **方法**: 浅克隆 16 个竞品仓库到 `E:\coding\_competitor-analysis\`（shallow clone, depth=1），8 个并行只读 agent 按统一 8 维度 rubric 逐仓剖析，全部结论带 file:line 源码证据；先对 AIOS 自身编排内核做摸底作为对比基线。
- **对比基线**: AIOS 自身摸底见本文第二节；此前四轮竞品分析见 `docs/reports/2026-05-22-competitor-agent-team-analysis.md`、`2026-06-04-competitor-*.md`、`2026-07-09-competitor-refresh-actionable.md`。

## TL;DR

16 个竞品（3 个闭源以 docs 分析、13 个全源码）呈现的高度一致的行业规律是：**编排的"智能判断"交给模型，但编排的"状态推进与验证"全部落在 runtime 硬闸门上**。AIOS 的 Fact→Capability→Evidence 协议在"确定性状态机"方向领先多数竞品，但三个关键环节落后于行业主流：

1. **验证闭环是 prompt 级技能而非 runtime 强制**（几乎全体竞品都用 hook/权限网关/guardrail 在工具执行前后硬拦截）；
2. **checkpoint 是日志型而非状态快照型**（langgraph/codex/crewAI/MetaGPT/autogen 都实现了状态粒度持久化与续跑）；
3. **记忆检索只有 token-overlap**（主流已是 embedding+BM25+实体加成混合检索，且多数可纯本地实现）。

次级差距：并行编排蓝图静态（竞品已普遍动态 spawn + 声明式 recipe）、无沙箱后端、模型路由未接主路径、无预算熔断。本文第五节给出 P0/P1/P2 共 8 项改进建议，每项映射到借鉴来源与 AIOS 具体落地文件。

---

## 一、调研范围

| 类别 | 项目 | 语言 | 形态 |
| --- | --- | --- | --- |
| 编码 agent harness | openai/codex | Rust | Session/Task/Turn 三层运行时 |
| | google-gemini/gemini-cli | TS | AsyncGenerator 事件流 + PolicyEngine |
| | anthropics/claude-code | 闭源 | 主 agent + subagent 分发（docs 分析） |
| | opencode-ai/opencode（Go 归档版） | Go | Broker 事件总线 + SQLite 持久化 |
| | block/goose | Rust | 状态机内核 + recipe 工作流（远端源码分析） |
| | All-Hands-AI/OpenHands | TS/Python | 事件溯源 + agent-sdk（本地为其前端镜像） |
| | SWE-agent/SWE-agent | Python | ACI-first + yaml 模板化 agent |
| | Aider-AI/aider | Python | 编辑格式范式 + git 骨架 |
| 通用编排框架 | langchain-ai/langgraph | Python | Pregel 图 + checkpointer 协议 |
| | microsoft/autogen | Python | actor 模型 + 群聊协议 |
| | crewAIInc/crewAI | Python | 角色型 crew + Flow 事件驱动 |
| | geekan/MetaGPT | Python | SOP 流水线 + 订阅消息池 |
| | FoundationAgents/OpenManus | Python | ReAct + 可插拔 Flow |
| | huggingface/smolagents | Python | code-as-action + AST 白名单执行器 |
| | agiresearch/AIOS（Rutgers，同名） | Python/Rust | 进程式内核隐喻 |
| 记忆层 | letta-ai/letta | TS/Python | memory blocks + self-editing + sleeptime |
| | mem0ai/mem0 | Python | 抽取/合并记忆管道 |

> 注：goose 与 letta 本地克隆不完整，分析 agent 已改用远端源码/文档补齐，证据为文件路径+符号名；opencode 本地检出为 Go 归档版而非现役 TS 版（sst/opencode），其结论仍有效但代表旧架构。

## 二、我们自身的编排基线（摸底结论）

AIOS 是"确定性协议外壳 + 模型显式声明"的文件系统状态机控制面：

- **核心链路**: `scripts/lib/planning/workflow-policy.mjs`（direct/guarded/planned 路由，只认显式声明不从文本猜语义）→ `auto-gate.mjs`（纯评估 + 落盘）→ `scripts/lib/ctx-agent-core/run.mjs`（executePrompt 主管线）→ rex-harness `Fact→Capability→Evidence`（`rex-harness/src/composition-root.mjs` 一次只推进一个能力）。
- **多 agent**: team = 依赖感知 job DAG（`scripts/lib/harness/subagent-runtime/dispatch-executor.mjs`）但节点集合来自**静态 blueprint spec**（4 角色 × 4 蓝图，`blueprints.mjs`）。
- **长任务**: solo harness = tmpdir git worktree + **日志型 checkpoint**（ContextDB `session#C<seq>`，resume 按 lastIteration+1 续跑；worktree 丢失后 resume 得到空 worktree）。
- **记忆**: memo 文件型事件存储 + ContextDB 检索均为 **token-overlap**（`mcp-server/src/contextdb/semantic.ts`），无 embedding。
- **验证**: rex evidence envelope 是代码级硬门（好），但 pre-edit-safety-gate / verification-before-completion 是 **prompt 级技能**（`workflow-policy.mjs:448` 只声明不执行）；回执型 evidence（passing-test-observed 等 6 类）已由 runtime 校验磁盘执行回执与退出码（`rex-harness/src/application/validate-command-evidence.mjs:26-55`），缺口在非回执类 evidence 仅格式校验、且回执文件可被有写权限的 agent 伪造。
- **其他**: 模型路由（`scripts/lib/model-router.mjs`）已被 team/subagent 路径消费（含成本遥测 `phase-job-finalize.mjs:92-94`），但 solo/one-shot 主执行路径未接、无预算控制；压缩是引用外置（ref_id+hash）。

**三大薄弱环节**：①记忆/检索层封顶经验复用上限；②并行编排静态、隔离靠约定无强制；③验证闭环非 runtime 强制、checkpoint 非状态快照、无预算熔断。

## 三、竞品逐仓亮点速览

每个仓库挑与编排最相关的 2-4 个杀手锏（完整 8 维度分析见各 agent 报告，要点已并入第五节）：

- **codex**: rollout JSONL 逐事件持久化 + sqlite 状态库支持断点回放；`spawn_agent` v2 子 agent 独立线程/上下文/角色→模型映射；OS 级沙箱三后端（seatbelt SBPL 基线策略 / landlock / bwrap）；auto-compact 有专属"压缩窗口"状态机且压缩本身可降级到备用模型。
- **gemini-cli**: 请求级**模型策略链路由**（Fallback→Override→Classifier→Default）+ 错误分类驱动 fallback；确定性 PolicyEngine 在工具执行前裁决 ALLOW/DENY/ASK（TOML 热更）；双层压缩（chat 0.5 阈值 + 文件四级 FULL/PARTIAL/SUMMARY/EXCLUDED 并保护近 2 轮）；哈希+LLM 双重死循环检测；每次工具调用前 git 快照 checkpoint。
- **claude-code**（docs）: subagent frontmatter 契约（name/description/tools/model）+ maxTurns 超限返回 partial 结果可用 SendMessage 续跑；hooks 运行时拦截（PreToolUse exit code block）；permission modes + bash 沙箱域名白名单；checkpointing `/rewind` 双轨回滚（会话+文件）；分层 CLAUDE.md 记忆 + skills 渐进披露。
- **opencode（Go）**: 权限是阻塞式 pubsub Service（工具内 Request() 阻塞等 Grant/Deny + 持久授权）；Edit 工具返回前等待 **LSP diagnostics 回灌 tool result**——验证发生在工具层，模型下一轮必然看到；摘要复用原会话 + SummaryMessageID 游标截断重放。
- **goose**: **recipe YAML 声明式工作流**（类型化参数/response json_schema/retry/subrecipe 委托/cron 调度器）；权限三档 `AlwaysAllow/AskBefore/NeverAllow` 严格优先 + MCP read_only_hint 自动分类；压缩**摘要不截断**（摘要 agent-visible、原文留给用户）+ 每回合注入剩余 token/turn budget 驱动收敛；子代理独立 Agent 实例/模型/工具白名单 + 工具调用事件实时流回 lead。
- **OpenHands**: 一切皆 Event（压缩/ hook 执行也是事件，可回放审计）；**condenser 是可组合配置对象**（`{kind,max_size,keep_first,usage_id}`，压缩可独立计费归因）；子会话 launch 协议（自包含 brief + worktree/shared 隔离枚举 + server 侧父子链接可恢复）；`/goal` 循环带 judge（每轮 verdict{score,complete,missing}，missing 直接回灌下一轮任务）；hook 返回 blocked+reason 硬拦。
- **SWE-agent**: ACI-first——坏输出层内 requery（解析失败不入环境，模板化重问至上限）；agent = yaml 模板组装件（模型+工具 bundle+history processor 链+预算硬限一个文件）；`per_instance_cost_limit` 双层记账；history processor 类型化可组合（LastNObservations/CacheControl/RemoveRegex）。
- **aider**: 异常/lint/test 全部归约为统一的 **reflected_message 重试原语**（带 max_reflections 硬上限）；repo-map = tree-sitter + networkx PageRank（按 chat 文件与提及标识符个性化加权）+ token 预算控制；weak_model 分工降级链（commit message/摘要/title 走弱模型，超限自动 fallback）；git 自动提交 + `aider_commit_hashes` 白名单 undo 即天然 checkpoint。
- **langgraph**: **checkpoint 协议**（`(thread,ns,checkpoint_id)` 三元组 + `put_writes` 保存单任务 pending writes 支持部分失败恢复）；`interrupt()` 持久化审批中断 + `Command(resume)` 从节点头幂等重放；**Durability 三档**（sync/async/exit）；`Send` API 动态 fork 任意数量并行分支；`get_state_history` time-travel 任意回溯；节点级 CachePolicy 跨 run 复用。
- **autogen**: MagenticOne 编排器每轮输出结构化 **progress ledger**（is_complete/in_progress/facts/任务分配 + max_stalls 卡死换路）；termination condition 组合器（And/Or + MaxMessage/TokenUsage）；`TokenLimitedChatCompletionContext` 按剩余 token 自动裁剪；intervention handler 消息进 runtime 前拦截审批；模型 replay 录制回放（测试零成本）。
- **crewAI**: Task 级 **guardrail 字段**（valid+feedback，失败带 feedback 重做 + retry_count 递归，超限阻塞）；Crew.restore checkpoint 续跑（首个无输出任务索引落盘，kickoff 幂等）；shallow/deep 双档记忆召回（deep = LLM 蒸馏子查询→多 scope 并行检索→置信度路由）。
- **MetaGPT**: SOP 编码为**订阅链**（角色 watch 产物类型，PRD/设计文档结构化流转）；Message 携带 pydantic `instruct_content`（消息即带 schema 的数据契约，下游按 key 消费）；ActionNode schema 校验非法即重试；`rc.state` 角色内状态机指针支持 recover 续跑；CostManager 实时计费 + `max_budget` 超支抛 NoMoneyException 中断。
- **OpenManus**: plan 即工具（七命令 + active plan 指针）；DockerSandbox mem/cpu 硬限 + 空闲回收；stuck 检测（重复输出→注入换策略提示）。
- **smolagents**: **code-as-action**（一次输出=一段可组合代码，N 次工具调用合一、状态存本地变量不进上下文）；AST 级白名单执行器（危险模块黑名单 + 每表达式求值结果二次校验 + 导入白名单）+ executor_type 可切 docker/e2b；planning_interval 每 N 步周期性重规划。
- **AIOS（Rutgers）**: 进程式隐喻最彻底（agent=进程、调用=syscall、FIFO/RR 时间片调度）；**gen_snapshot/gen_recover 按 pid 挂起/恢复生成中状态**（LLM 流式打断时保存已收 token+游标）；SmartRouting 用相似历史 query 的 perf/cost 双分数选模；本地 sentence-embedding + 余弦检索（无外部 API）+ MemoryWriteBarrier 并发写排空。
- **letta**: **memory blocks**（具名、带 char limit、行号化渲染 + 余量元数据让模型自监控）；self-editing memory tools（memory_replace/insert/rethink，编辑必须精确匹配而非自由声明）；**sleeptime agent** 后台复盘会话沉淀教训并 git commit 记忆；热记忆常驻 system prompt、冷记忆按工具调取。
- **mem0**: 两阶段记忆管道（先取相似既有记忆再单次 LLM 抽取）；**混合检索三信号加法融合**（语义分先卡阈值 + BM25 sigmoid 归一 + 实体 boost）；链接式冲突消解（新事实不覆盖、`linked_memory_ids` 挂链、等价跳过）+ TTL 过期剪枝；抽取 prompt 即写入门槛（只收自包含事实、排除客套话、相对时间锚定为绝对日期）。

## 四、跨竞品七大规律

1. **验证强制化是行业底线**：claude-code hooks（exit code block）、OpenHands hook blocked+reason、gemini PolicyEngine、goose 权限三档、crewAI Task guardrail、SWE-agent blocklist 前置拦截、smolagents AST 白名单——没有任何一个成熟竞品把"该不该做/做没做对"留在 prompt 层。AIOS 的 pre-edit-safety-gate / verification-before-completion 是少数仍停留在 prompt 级的。
2. **checkpoint 的正确形态是"状态快照 + 恢复协议"**：langgraph（三元组+put_writes+interrupt/resume 幂等重放+time-travel）、codex（JSONL+sqlite+回溯扫描）、crewAI（任务索引幂等续跑）、MetaGPT（rc.state 指针）、opencode（SummaryMessageID 游标）、autogen（整团快照）。AIOS 的日志型 checkpoint + lastIteration 续跑 + 空 worktree 兜底是最弱的一种。
3. **记忆层主流 = 抽取门槛 + 混合检索 + 冲突消解 + 后台整理**四件套。AIOS 的"模型自声明 verified=true 落库 + token-overlap 检索 + 只增不减"四件全缺。且竞品证明这四件可以纯本地实现（mem0 ollama/faiss、Rutgers AIOS 本地 all-MiniLM、letta MemFS git 版本化）。
4. **并行编排的胜负手是"动态生成 + 节点契约 + 结构化回传"**：langgraph Send 动态 fork、MagenticOne ledger 驱动调度、codex spawn_agent v2 角色映射、OpenHands child conversation brief、goose subrecipe、claude-code frontmatter。静态 blueprint 只是过渡形态。
5. **上下文压缩的共识做法**：阈值触发、摘要不截断（goose：摘要 agent-visible/原文留给用户）、分级降级（gemini 文件四级）、保护近期轮次、压缩事件化可审计（OpenHands CondensationEvent）、死循环检测三处独立出现（gemini 哈希+LLM、OpenManus 重复检测、autogen max_stalls）。
6. **成本与模型路由已从"锦上添花"变"标配"**：请求级策略链路由（gemini）、弱模型分工（aider/opencode/claude-code subagent model 字段）、预算熔断（MetaGPT/SWE-agent/OpenHands max_budget_per_task）、逐步 token 聚合（smolagents RunResult/autogen RequestUsage）、录制回放省测试成本（autogen）。
7. **声明式资产化**：goose recipe、SWE-agent yaml 模板、claude-code frontmatter、langgraph Durability 档位、OpenHands condenser 配置——"模型/工具/权限/预算/输出 schema"正在被打包成一个可分发、可复用、可校验的声明对象，而不是散落在代码里。

## 五、对 AIOS 的改进建议

### P0（直接补齐行业底线，收益最大）

**P0-1 把验证闭环从 prompt 技能升级为 runtime 强制闸门**
- 问题：`pre-edit-safety-gate` / `verification-before-completion` 只在 `workflow-policy.mjs:448` 声明、无代码执行点；非回执类 evidence ref 只校验 URI 格式与 placeholder 黑名单（`rex-capability-runtime.mjs:47,68`）——回执型 evidence 已有 exitCode 硬校验（`validate-command-evidence.mjs:26-55`），改进重点是**扩大回执覆盖面并防回执伪造**，而非从零建校验。
- 借鉴：claude-code PreToolUse hook exit-code block（`examples/hooks/bash_command_validator_example.py:14`）；OpenHands hook 返回 blocked+reason（`hook-execution-event.ts:6-24`）+ ConfirmRisky 策略对象；goose 权限三档 never>always>ask（`config/permission.rs`）；gemini PolicyEngine TOML 热更（`policy-engine.ts:121`）；crewAI guardrail valid+feedback+retry_count（`utilities/guardrail.py:126`）。
- 落地（AIOS 已有地利）：
  1. 在 `ctx-agent-core/run.mjs` 的 provider 执行管线（现有 `ingestCapabilityProviderOutput` 证据摄取点）挂 **PreToolUse/PostToolUse hook 槽位**：guarded 路由下，Edit/Bash 类操作先过 hook，非零退出即 blocked 并把 reason 回注下一轮输入；
  2. verification 阶段把 `verification-evidence.mjs` 从"收集声明"升级为**执行验证命令并校验退出码**，产物才是合法 evidence（补上"ref 指向的命令真的跑过且成功"）；
  3. 规则用 TOML/JSON 声明、支持热更，安全/归属类失败继续走现有 human-gate。

**P0-2 checkpoint 从日志型升级为"状态快照 + 恢复协议"**
- 问题：solo harness resume 按 lastIteration+1 续跑（`solo-runtime/loop.mjs`），worktree 丢失后重建的是空 worktree（`lifecycle/harness/worktree.mjs:16-24`），改动不随 checkpoint 恢复。
- 借鉴：langgraph checkpointer 协议（`checkpoint/base/__init__.py:177`，`put_writes` 支持部分失败恢复，`interrupt()`/`Command(resume)` 幂等重放）；opencode SummaryMessageID 游标；crewAI 首个无输出任务索引幂等续跑（`crew.py:1554`）；MetaGPT `rc.state` 指针；codex rollout JSONL+sqlite 双层。
- 落地：
  1. 定义 `CheckpointSaver` 接口（内存/SQLite 双实现），记录 **DAG/任务节点粒度状态**（node_id、artifact 输出、evidence 引用）而非只有 iteration 序号；
  2. resume 时跳过已完成节点 + 校验 worktree diff 与 checkpoint 记录的一致性，不一致时用 pre-mutation snapshot 恢复（dispatch-executor 已有 snapshot 机制可复用）；
  3. 把"待审批/待验证"做成 langgraph 式**持久化中断**（落盘 + resume 值重入），替代进程内等待——现有 human-gate 语义可直接映射；
  4. 低成本补充：aider 式 git 自动提交 + 提交哈希白名单，作为 worktree 恢复的物理兜底。

**P0-3 记忆层四件套（纯本地可达成）**
- 问题：检索仅 token-overlap（`contextdb/semantic.ts:14`、`memo/storage/query.mjs:122`）；写入靠模型自声明 `verified=true`（`run.mjs:391-431`）；只增不减、无冲突消解、无遗忘。
- 借鉴：mem0 混合检索三信号（`main.py:1628-1681` + `scoring.py:60-139`：语义分卡阈值 + BM25 sigmoid + 实体 boost）与抽取式写入门槛（`prompts.py:490-535`：只收自包含事实、排除客套、相对时间锚定）与链接式冲突消解+TTL（`prompts.py:511`、`main.py:442`）；letta memory blocks + self-editing 工具（精确替换而非自由声明）；letta sleeptime 后台整理；Rutgers AIOS 本地 all-MiniLM 余弦（`retrievers.py:16-63`）证明 embedding 可无外部 API。
- 落地（按投入从低到高）：
  1. **零 embedding 先行**：memo/ContextDB 加 BM25（复用现有分词）+ 实体加成 + sigmoid 归一融合排序；加 `links`/`expires` 字段做冲突消解与过期剪枝；
  2. 写入门槛：`verified=true` 自声明替换为**抽取 prompt 契约**（单次调用输出结构化条目 `{fact, entities[], date, evidence_ref, confidence}`；实体由模型结构化声明，**禁止正则/关键词抽取**）+ runtime 仅校验结构合法性（schema/ISO 日期/去重哈希/TTL）；`verified` 改为「引用且仅引用一条 runtime 实测退出码=0 的证据」后由协议打标——把语义声明降维成证据链校验；
  3. pinned 区改造成 limit 化 memory block（行号化渲染 + 余量元数据），编辑收敛为精确替换工具；
  4. 会话结束挂 **sleeptime 整理子任务**：candidates 合并入 pinned、按 useful 反馈调权、记忆库 git 版本化（可回滚）；
  5. 可选：接入本地 embedding（ollama/fastembed，数据不出机器）做粗排 + token-overlap 精排的混合检索。

### P1（结构性提升）

**P1-4 并行编排动态化 + 节点契约化（recipe 化）**
- 问题：team 的 phase 图来自静态 spec（4 角色 × 4 蓝图，`blueprints.mjs:74-111`），rex-planning 产出的 parallelGroups/frontier 只是文本 artifact，runtime 不会据此生成并行计划。
- 借鉴：goose recipe YAML（类型化参数 + response json_schema + retry + subrecipe 委托，`recipe/mod.rs`）；langgraph Send 动态 fork（`types.py:704`）；MagenticOne progress ledger 每轮结构化重排（`_magentic_one_orchestrator.py:300-401`）；codex spawn_agent v2（`multi_agents_v2/spawn.rs:36`，role→config/model 映射）；OpenHands child conversation 自包含 brief + 隔离枚举；claude-code subagent frontmatter（tools/model 白名单）+ partial 结果 SendMessage 续跑；MetaGPT 订阅链 + typed instruct_content（产物类型订阅触发下游）。
- 落地：
  1. blueprint 升级为**参数化 recipe 资产**：YAML 声明 roles/tools 白名单/model/max_turns/budget/output schema/retry/子蓝图引用；
  2. dispatch-executor 支持**运行期动态增删节点**：rex-planning 产出的 frontier 映射为 DAG 节点，每完成 K 个节点触发 re-plan（读 Evidence→增删/重排），阈值与 smolagents planning_interval 一致；
  3. 节点间通信改"产物类型订阅"：上游产出带 schema 校验的 artifact 即触发下游，替代 prompt 级传递；
  4. 失败/超限节点返回 partial 结果 + 续跑句柄，不整 DAG 重启。

**P1-5 沙箱分级与权限网关**
- 问题：隔离仅 tmpdir worktree + 约定式 ownedPathPrefixes，无强制沙箱。
- 借鉴：smolagents AST 白名单执行器（危险模块黑名单 + 逐表达式结果校验 + 导入白名单，`local_python_executor.py:105-237`）；codex OS 级后端抽象（seatbelt SBPL/landlock/bwrap 统一 manager）；OpenManus DockerSandbox mem/cpu 硬限（`sandbox/core/sandbox.py:62-64`）；goose 权限三档 + MCP read_only_hint 自动分类；claude-code bash 沙箱域名白名单。
- 落地：team/harness 节点声明 `sandbox:{level,network,fs}`；executor 层按平台选后端（Windows 上可用 Job Object/受限 token 起步），无可用后端时**拒绝并行执行写操作**（降级串行 + 加强审批）——把"能不能并行"与"隔离是否强制"绑定。

**P1-6 模型路由接入主路径 + 预算熔断**
- 问题：`model-router.mjs` 有 registry/COST_ORDER/fallback，team/subagent 路径已消费（仅遥测）；solo/one-shot 主执行路径未接，全项目无预算熔断。
- 借鉴：gemini 请求级策略链（Fallback→Override→Classifier→Default，`modelRouterService.ts:40-56`）+ 错误分类 fallback（`fallback/handler.ts:26-64`）；aider weak_model 分工与超限降级链（`models.py:603-623`）；MetaGPT CostManager 实时计费 + max_budget 抛中断（`cost_manager.py:35-58`）；SWE-agent `per_instance_cost_limit`；OpenHands `switch_llm` 运行中换模 + `max_budget_per_task`；Rutgers AIOS SmartRouting 用相似历史 query 的 perf/cost 双分数选模（`routing.py:182`）。
- 落地：direct/guarded/planned 映射不同模型档位；旁路任务（摘要/标题/记忆抽取/评审）固定弱模型；harness 层按任务记账（复用 lex turn 事件），DAG 各分支独立配额，超支降级小模型或 fail-fast。
- ⚠️ 反模式约束（2026-09-06 审核补充）：gemini 的 Classifier 自动分类档**不要照搬**——按 prompt 内容分类选模属于语义猜测，违反北极星。改为模型在 planning/请求阶段显式声明 `task-type` 字段，runtime 查路由表；无声明回退 Default（与 workflow-policy 显式 intent 原则同构）。

### P2（体验与效率增强）

**P2-7 上下文压缩管线**
- 借鉴：gemini 双层压缩（chat 0.5 阈值 + 文件四级降级 + 保护近 2 轮，`chatCompressionService.ts:41`、`contextCompressionService.ts:116`）；goose 摘要不截断（摘要 agent-visible、原文留给用户）+ 后台批量摘要旧工具对 + 每回合注入剩余 token/turn budget；codex auto-compact 窗口状态机 + 压缩模型可降级；OpenHands condenser 可组合配置 + 压缩事件化审计。
- 落地：AIOS 现有"引用外置压缩"（ref_id+hash）之上加**摘要式压缩层**：阈值触发、被压缩事件入库 ContextDB（为语义检索提供召回素材）、压缩动作本身写 evidence 可审计、主循环注入剩余预算驱动收敛。

**P2-8 散点机制**
- 死循环检测：gemini 哈希+LLM 双重（`loopDetectionService.ts:208`）/ OpenManus 重复输出检测 / autogen max_stalls → 在 ctx-agent 主循环加重复动作检测与换路提示。⚠️ 只做哈希比对档（客观事实检测）；「runtime 调 LLM 判死循环」不要做——改为 ReAct 式让模型在 observations 里自报告 `progress_made:false + blocked_reason`，runtime 只计连续声明次数触发换路。
- ACI requery：SWE-agent 解析失败不入环境、模板化重问至上限（`agents.py:1107-1136`）→ AIOS evidence envelope 解析失败时给模型结构化 requery 而非直接 blocked。
- 结构化输出契约：MetaGPT ActionNode schema 校验非法重试（`action_node.py:428-452`）→ rex handoff JSON 已有严格校验，可加"校验失败→模板化 requery"替代一次失败即终止。
- 录制回放：autogen replay（测试零成本）→ AIOS harness 增加 turn 级录制回放，用于回归测试 provider 管线。
- code-as-action：smolagents 证明单次代码输出可合并 N 次工具调用省 token → 可作为 AIOS 压缩技能的可选输出形态实验。

## 六、建议落地路线图

| 阶段 | 内容 | 涉及模块（现有文件） | 规模估计 |
| --- | --- | --- | --- |
| 第 1 步 | P0-1 验证 runtime 化（hook 槽位 + evidence 实测） | `ctx-agent-core/run.mjs`、`rex-capability-runtime.mjs`、`verification-evidence.mjs` | 中 |
| 第 2 步 | P0-3 记忆四件套之 1-2（BM25 融合 + 抽取式写入门槛 + links/TTL） | `mcp-server/src/contextdb/`、`scripts/lib/memo/` | 中 |
| 第 3 步 | P0-2 CheckpointSaver 接口 + 节点粒度状态 + worktree 一致性校验 | `scripts/lib/harness/solo-runtime/`、`lifecycle/harness/` | 中大 |
| 第 4 步 | P1-6 模型路由接主路径 + 预算熔断 | `model-router.mjs`、`one-shot.mjs`、solo-runtime | 小 |
| 第 5 步 | P1-4 recipe 化 blueprint + 动态 DAG + 产物订阅 | `harness/orchestrator/blueprints.mjs`、`subagent-runtime/dispatch-executor.mjs` | 大 |
| 第 6 步 | P1-5 沙箱分级 + P0-3 之 3-5（blocks/sleeptime/embedding）+ P2 | 跨模块 | 持续迭代 |

排序依据：P0-1/P0-3(1-2) 改动面小、复用现有证据管线与分词器，先拿行业底线；P0-2 是 solo harness 可信度的根，紧随其后；模型路由是纯接线工作、性价比最高；recipe 化与沙箱是结构性改造，放在基础设施稳固之后。

## 附录：分析产物位置

- 竞品克隆：`E:\coding\_competitor-analysis\`（16 仓库，浅克隆，可整体删除）
- 本报告引用的全部 file:line 证据来自 8 个并行调研 agent 的逐仓报告（codex+gemini-cli、opencode+aider、letta+mem0、MetaGPT+smolagents、OpenHands+SWE-agent、langgraph+autogen、crewAI+goose、claude-code+OpenManus+AIOS）。
- 局限：goose/letta 本地克隆不完整，证据为远端源码的文件路径+符号名（无精确行号）；opencode 为 Go 归档版；claude-code 闭源，从官方仓库 docs/CHANGELOG 提炼。

## 附录 B：审核勘误（2026-09-06，v5.10.0 发布前复核）

经子代理逐条复核（详见 [2026-09-06-v510-worklist-audit.md](./2026-09-06-v510-worklist-audit.md)），本文涉及 AIOS 自身代码的 11 条断言中 8 条 CONFIRMED、3 条已按实测修正并回写正文：

1. evidence 管线并非"只做格式校验"——回执型 evidence 已有 exitCode 硬校验（`validate-command-evidence.mjs:26-55`），P0-1 已改为"扩大回执覆盖面 + 防伪造"；
2. model-router 消费面更广（team/subagent 含遥测），"未接主路径"仅对 solo/one-shot 成立；
3. rex-harness-adapter 的"唯一边界"是声明性的：`rex-capability-runtime.mjs`、`rex-activation-store.mjs`、`rex-long-running-delivery-store.mjs` 三个同级模块直接 import `rex-harness/src`（局限在 `scripts/lib/workflows/` 内）。

另：merge-gate 实为代码级 block（`merge-gate.mjs:47-56` + `handoffs.mjs:29-82`），非文本约定——真实弱点是校验对象为子代理自报的 filesTouched 而非磁盘 diff；此修正影响 P1-4 的改进方向（改为校验磁盘 diff 而非新建冲突规则）。竞品侧证据本次未复核。
