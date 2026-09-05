# v5.10.0 工作清单审核报告（真实性 / 必要性 / 反模式）

- **日期**: 2026-09-06
- **背景**: 准备发布 v5.10.0。发布前对两份报告（[2026-09-05 竞品编排分析](./2026-09-05-competitor-orchestration-analysis.md)、[2026-09-06 优缺点审计](./2026-09-06-strengths-weaknesses-audit.md)）提炼的工作清单做交叉审核。用户约束：①清单内容必须真实，不得有错误与虚假；②甄别真正必要的条目；③不重蹈"用正则/硬编码判断逻辑代替提示词引导"的覆辙——凡引导模型行为处应采用 Plan-and-solve / ReAct / 显式声明式设计。
- **方法**: 3 个并行只读子代理（竞品报告证据核实、审计报告事实核实、实现风格审查）+ 主线 git/plan 分诊（第 4 个子代理两次中断后由主线完成）。

## 一、真实性审核结论

**总体判定：清单无虚构断言，全部可溯源；但有 3 处低估了 AIOS 自身能力、4 处行号/计数偏差，已就地回写修正。** 修正后的净效果是：部分工作量比原清单预估更小。

### 竞品报告·自身代码断言（11 条：8 CONFIRMED / 3 PARTIAL）

| # | 断言 | 裁定 | 修正 |
| --- | --- | --- | --- |
| 1 | workflow-policy requiresPreEditSafety 无运行时执行点（:448）、disposition 五值（:14-20）、isReadOnlyMessage 恒 false（:277-280） | CONFIRMED | 全仓 grep 仅 tests/benchmarks/docs 消费；仓库自认"currently only a decision field"（`docs/plans/2026-07-28-context-lifecycle-w0-w1-decisions.md:79`） |
| 2 | evidence ref 只做格式校验 | **PARTIAL** | 回执型 evidence（passing-test-observed 等 6 类）已有 exitCode 硬校验（`rex-harness/src/application/validate-command-evidence.mjs:26-55`）；真缺口 = 非回执类仅格式校验 + 回执文件可被有写权限的 agent 伪造（存 sha256 但不复验） |
| 3 | run.mjs 证据摄取失败改写 exitCode（:381-384）、verified=true 自声明落库（:391-431） | CONFIRMED | — |
| 4 | checkpoint 日志型、lastIteration+1 续跑 | CONFIRMED | `loop.mjs:98`、`checkpoint.mjs:79-82` |
| 5 | worktree 丢失重建空树 | CONFIRMED | `worktree.mjs:16-23`、`solo-worktree.mjs:54` |
| 6 | blueprint 静态 4×4 | CONFIRMED | 但"merge gate 是文本约定"**低估**：实为代码级 block（`merge-gate.mjs:47-56` + `handoffs.mjs:29-82` + `file-policy.mjs`）；真弱点 = 校验对象是子代理**自报** filesTouched 而非磁盘 diff |
| 7 | ContextDB 仅 token-overlap、无 embedding | CONFIRMED | 补充：仅在 `CONTEXTDB_SEMANTIC=1` 时启用（`semantic.ts:93`） |
| 8 | model-router 未接主路径 | **PARTIAL** | team/subagent 路径已消费（含成本遥测 `phase-job-finalize.mjs:92-94`）；"未接"仅对 solo/one-shot 成立 |
| 9 | dispatch-executor 依赖感知 DAG 全套机制 | CONFIRMED | `:29-171` 逐项复现 |
| 10 | solo/one-shot 无模型路由与预算 | CONFIRMED | 三处 grep 为空；team 有遥测但无预算控制 |
| 11 | adapter 唯一适配边界、composition-root 一次推进一个能力 | PARTIAL | "唯一"是声明性的：`rex-capability-runtime.mjs:5-13`、`rex-activation-store.mjs`、`rex-long-running-delivery-store.mjs` 直接 import `rex-harness/src`（局限在 `scripts/lib/workflows/` 内） |

### 优缺点审计报告·事实断言（12 组：10 CONFIRMED / 1 PARTIAL / 行号微偏）

**已修正的事实错误**：①`clients/core/definitions.mjs` 实为 **7** 客户端（codex/claude/gemini/opencode/hermes/grok/workbuddy），原写 6 漏计 workbuddy——README"6 客户端"宣传反而**少报**了；②根目录 4 个调试日志已被 `.gitignore:17 *.log` 覆盖、**未入库**（属工作区卫生而非仓库增重）；③行号微偏（release.yml 证据门 :84 非 :87）；④pptx 365 个 tracked 文件（原写 364，差 1 不影响结论）。

**全部精确复现的硬事实**：351 个 node_modules 误入库、.gitignore:125、665 个证据文件、sha256 drift guard（:272）、AGENTS.md:31 browser 叙事、mkdocs.yml:3 旧 description、216 测试文件/1937 test/8716 断言、7 个 workflow、printf shim（:54）、npm install（windows-shell-smoke:27）、零 secret 扫描/零 SHA pin、tag 类型混杂、credentials.py 存在、state-root 双路径、5a6bd078 改 11 文件、docs/plans 618、examples 4 个 md、opencode.json 跟踪与 ignore 自相矛盾、VERSION=5.9.0。

## 二、反模式审查（清单本身的实现风格）

**判据一句话：校验客观事实 = 合理硬编码 [D]；猜测模型/用户意图 = 反模式 [X]；用结构化输出契约引导模型自报告 = 正确路线 [M]。** 这与项目北极星（确定性协议外壳 + 模型显式声明，`workflow-policy.mjs:66-70`、`derive-facts.mjs:14-18`）完全同构。

### 全清单归类

- **[D] 合理硬编码（保持）**：hook 槽位 + 退出码校验、CheckpointSaver 状态机、BM25+sigmoid 融合排序、recipe/动态 DAG/产物订阅、AST 白名单（语法层非语义层）、权限三档、预算熔断、哈希死循环检测、CRLF 归一化、供应链补齐、全部 B/C 工程项。退出码、sha256、diff 一致性、schema 合法性、哈希重复、token 超限都是可机器复核的客观事实——代码强制不是反模式，恰是"状态推进交给可校验协议"的本体。
- **[M] 模型引导（保持并推广）**：记忆抽取 prompt 契约、MagenticOne progress ledger（模型每轮输出结构化 ledger、runtime 只读字段调度）、sleeptime 整理、requery 模板、压缩摘要（弱模型任务）。
- **[X] 反模式风险（4 处，已给出替代设计）**：

| 反模式点 | 问题 | 符合北极星的替代设计 |
| --- | --- | --- |
| gemini 式 Classifier 自动选模档 | 按 prompt 内容分类意图 = 语义猜测 | 模型在 planning/请求阶段显式声明 `task-type` 字段 → runtime 查路由表；无声明回退 Default（与显式 intent 原则同构） |
| 实体加成检索的实体抽取 | 若用正则/分词抽实体即硬编码理解器 | 写入时抽取 prompt 契约要求模型输出结构化 `entities:[]` 字段；检索端只做确定性 boost |
| 死循环检测的 LLM 档 | runtime 调 LLM 判"是否死循环" = 语义判断 | ReAct 循环内要求模型每轮 observations 自报告 `progress_made:false + blocked_reason`；runtime 只数连续声明次数触发换路 |
| TOML 策略规则 | 一旦规则匹配"消息语义"即越界 | 文档化限定：规则只匹配工具名/命令文本/路径等**客观字段**，写入 PolicyEngine 边界说明 |

### verified=true 矛盾的调和（记忆写入门槛的最终设计）

矛盾不在"自声明"而在声明对象：`verified` 是对客观世界的断言、runtime 无法校验，等价无门槛。改为两层：

1. **语义层 [M]**：抽取 prompt 契约（Plan-and-solve 式单次调用）产出结构化条目 `{fact, entities[], date, evidence_ref, confidence}`——模型显式声明，禁止关键词黑名单式"客套话过滤"；
2. **结构层 [D]**：runtime 只校验 schema 合法、ISO 日期、去重哈希、TTL、evidence_ref 指向"实测退出码=0"的真实证据；`verified` 不再由模型自由声明，而是**引用且仅引用一条 runtime 实测证据**后由协议打标。

### SKILL/提示词写法规范（建议写入贡献文档与 skill-sources 模板）

1. **DO** 每个 SKILL 开头声明输入前提（当前 Command/activationId）与输出契约（结构化 evidence kinds + 恰好一个信封），rex-planning 已是范本；**DON'T** 写"若用户提到 X/Y 词则进入 Z 流程"的关键词分支——改为要求模型输出显式 intent 字段供 policy 查表。
2. **DO** 一切语义判断（只读？多步？完成？卡住？）写成"请在结构化输出中自报告字段 X"并给出字段定义与 few-shot（ReAct 式观察-声明循环）；**DON'T** 把中英文关键词表、正则清单写进 SKILL 或配套脚本充当"理解器"。
3. **DO** 硬门槛写成验证协议——"命令必须真实执行并引用退出码/哈希"，SKILL 只描述义务、runtime 执行校验；**DON'T** 用"请确保测试通过"的提示词替代 runtime 门禁，也不要在 runtime 里为提示词失灵写文本启发式兜底（同时违反北极星并掩盖协议缺口）。

## 三、5.10.0 必要性分诊

### 变更集底座（实测）

- `git log v5.9.0..HEAD` = **仅 1 个提交**（`fbab2193` fix(skills): rexai-image-generation i2i 修复）。CHANGELOG 无 Unreleased/5.10 段。→ **5.10.0 的内容完全由本清单定义**，没有"已完成待发布"的存量，范围可控。
- 既有 plan 映射：**无现行 plan** 覆盖 hook runtime 化 / CheckpointSaver / 记忆四件套 / recipe 化 / 沙箱（全部是报告新提，落地需先立 plan 走 provider 管线）；有旧参考（`2026-05-08/05-14 model-router` 修复、`2026-05-11 harness-stage-checkpoint-strengthening`、`2026-07-22 proportional-capability-evidence-gates` 与 `rex-evidence-rejection-reasons`），说明方向有历史脉络但均非现行契约。
- 发布需同步文件（照 5a6bd078 惯例）：`VERSION`、`CHANGELOG.md`、`docs-site/{index,changelog}.md` + `docs-site/zh/*`、`docs/zh-CN/CHANGELOG.md`、多语言 release notes；且 **tag 指向的 commit 必须包含 step 13 门禁证据**（v5.9.0 教训）。

### 分诊表

| 条目 | 档位 | 理由 |
| --- | --- | --- |
| a. CRLF 红测根治（.gitattributes eol=lf + drift guard 归一化） | **必须** | 消灭"红=正常"信号腐蚀；小改动；发版前测试基线必须干净 |
| b. 证据门时序/存储修复（gitignore 白名单免 add -f、preflight 校验 tag 含证据、证据门前移） | **必须** | 直接防 v5.9.0 式白跑重打 tag；发布链自身缺陷必须先修 |
| c. 叙事统一（AGENTS.md browser 表述、mkdocs description、README 调用口径） | **必须** | 纯文档、零风险；5.10 发布会带流量，叙事错误直接暴露 |
| d. 仓库减负（pptx 351 个 node_modules 出库、删 4 个根日志、.cache 110 字体、opencode.json 矛盾规则） | **必须** | `git rm --cached` 级操作；减小发布包与 clone 成本 |
| f. 供应链补齐（Actions SHA pin、gitleaks、windows smoke npm ci、npm audit） | **必须** | 公开仓供应链风险；一次性配置，半天级 |
| n. 提示词写法规范落地（第二节 3 条 DO/DON'T → 贡献文档 + skill-sources SKILL 模板） | **必须** | 直接回应用户踩坑约束；纯文档；为后续所有编排迭代立规矩 |
| e. generatedTargets 补 gemini/hermes/workbuddy | **可进** | 复用 sync-skills 现成机制；与 5.9"memory across clients"主题延续；但属行为变化需各客户端投影验证 |
| j. 模型路由接 solo/one-shot 主路径（接线部分） | **可进** | team 路径已有先例、router 现成；注意用 task-type 声明式而非 Classifier；预算熔断部分延后 |
| g. 验证 runtime 化（hook 槽位） | **延后 5.11+** | 行为大改、无现行 plan、需 provider 契约评审与 skill 认证；其中"扩大回执型 evidence 覆盖面 + 防伪造"若时间允许可作 5.10 可选增量 |
| h. CheckpointSaver + worktree 一致性 | **延后 5.11+** | 同上；有 2026-05-11 旧 plan 可参考，但需新 plan |
| i. 记忆四件套 | **延后 5.11+** | 中大改动；embedding 选型（本地 ollama vs 远端）需用户决策 |
| k. recipe 化 + 动态 DAG | **延后 5.11+** | 最大结构改造；依赖 g/h 前置 |
| l. 沙箱分级 | **延后 5.11+** | 平台相关工作量大；Windows 后端选型需调研 |
| m. 压缩管线/死循环检测/CLI 收敛/examples/plans 归档/CHANGELOG 生成化 | **延后 5.11+** | 无一阻塞发布（CHANGELOG 生成化很小，若顺手可随 b 一起做） |

### 建议

- **5.10.0 主题句**：「可信与减负」——让既有承诺变真（修红测、修门禁时序、统一叙事、清理仓库、补齐客户端投影、立提示词规范），为 5.11 编排升级（g/h/i）铺路。
- **编排大项不要混进 5.10**：g/h/i/k/l 都是行为大改，按仓库自己的门禁要求需要独立 plan + test scope + skill 认证，混装会把 5.10 变成高风险版本。建议 5.10 发布窗口内并行启动 g/h/i 的 plan（走 rex-planning），代码进 5.11。
- **排序**：a/c/n（当天可完成）→ b/f（半天）→ d（半天）→ e/j（各一天，可选）→ 打 tag 发布。

## 四、结论

1. **真实性**：清单全部断言可溯源、无虚构；3 处低估自身能力（evidence 回执校验已存在、merge-gate 是代码级、model-router 已部分接线）已修正——修正后相关工作量比原清单更小；4 处行号/计数偏差已回写。
2. **反模式**：清单 30+ 条目中仅 4 处借鉴条目有"硬编码理解器"风险，均已给出 Plan-and-solve / ReAct / 显式声明式替代设计；[D]/[M] 两类占绝对多数，说明清单整体与北极星一致。
3. **必要性**：5.10 建议 = 6 必须 + 2 可选（a/b/c/d/f/n + e/j），全部是小到中规模、无行为大改；编排 P0 三项启动规划、代码进 5.11。
