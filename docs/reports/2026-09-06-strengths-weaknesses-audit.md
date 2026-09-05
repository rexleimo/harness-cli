# AIOS 优缺点全景提取——扬长避短行动清单

- **日期**: 2026-09-06
- **目的**: 从近期变更（v5.9.0 发布周期）与整个项目材料中系统提取优点与缺点，形成"扬长避短"的可执行依据。
- **材料来源（5 路）**:
  1. 自身编排内核摸底（`workflow-policy` / `auto-gate` / `ctx-agent-core` / rex-harness / harness / memo，全部 file:line 证据）；
  2. 16 竞品源码级对比（见 [2026-09-05-competitor-orchestration-analysis.md](./2026-09-05-competitor-orchestration-analysis.md)，本文与其交叉引用，不重复展开）；
  3. 工程/发布质量审计（测试、CI/CD、跨平台、代码组织、安全）；
  4. 产品面/仓库卫生审计（承诺一致性、命令面、客户端适配、仓库杂物、文档站、上手体验）；
  5. v5.9.0 发布周期实战记录（`.workbuddy/memory/2026-09-04.md` + git log `6a50dbcd..9284171a`）。

## TL;DR

**一句话画像**：AIOS 的护城河是"确定性证据链 + 证据驱动的工程文化"——这在全部 16 个竞品中独一无二（没有任何竞品有发布级 sha256 证据门与 1937 用例真行为回归网）；拖累它的是"静态与弱强制"（编排三缺口）与"叙事漂移 + 仓库增重"（工程三短板）。

**扬长 TOP3**：①把"可验证交付"（Evidence envelope → 用户可见完成报告）升级为核心卖点；②用单一真源投影体系兑现并超越 6 客户端承诺；③以现有测试+发布门禁为安全网，低风险推进 P0 编排改造。
**避短 TOP3**：①编排三缺口按竞品报告 P0 顺序补齐（验证 runtime 化 → checkpoint 状态化 → 记忆四件套）；②消灭"红=正常"（Windows CRLF 红测）与证据门时序陷阱（tag 必须含证据）；③仓库减负与叙事统一（清 364 个误入库文件、修 AGENTS.md/README/mkdocs 三处口径矛盾）。

---

## 一、优点清单（扬长：每条含"如何放大"）

### A. 编排内核

| # | 优点 | 证据 | 放大动作 |
| --- | --- | --- | --- |
| A1 | **确定性协议外壳**：状态推进全锁进可校验 JSON 信封，"智能判断"刻意推给模型显式声明，不从自由文本猜语义 | `workflow-policy.mjs:14-20` disposition 五值、`isReadOnlyMessage` 恒 false（:277-280）；rex evidence envelope 单行协议 `rex-capability-runtime.mjs:23,182-234` | 把这一点写进 docs-site 首页与 README 的差异化段落——竞品普遍靠 prompt 约束，AIOS 靠协议。这是对比 langgraph/autogen/crewAI 时的独有卖点 |
| A2 | **一次只推进一个能力**的激活状态机，避免整链注入失控 | `composition-root.mjs:83-93` 按 priority 取首个触发；activation 持久化原子写+锁 `rex-activation-store.mjs:162-175` | 保持；后续 P0 改造（hook/检查点）一律落在 rex-harness 适配层（`scripts/lib/workflows/rex-harness-adapter.mjs` 唯一边界），不动内核 |
| A3 | **依赖感知的并行 DAG 调度**已具备：dependsOn 就绪、并发上限、merge-gate、pre-mutation snapshot、依赖环统一 blocked | `dispatch-executor.mjs:29-171` | 这是动态化的现成地基——竞品报告 P1-4 的"运行期动态增删节点"只需把静态 blueprint 换成 plan 产出的节点集，调度器本身不用重写 |

### B. 验证与发布工程

| # | 优点 | 证据 | 放大动作 |
| --- | --- | --- | --- |
| B1 | **真行为回归网**：216 文件 / 1937 test / 8716 断言，spawn 真实 CLI + 临时目录，非快照式 | `scripts/tests/aios-orchestrator-agents.test.mjs:21-31`；分层 suite 由 `test-suite-runner.mjs:7-24` 驱动 | 作为 P0/P1 改造的"不退化"安全网对外宣传（开发者信任）；补覆盖率工具（c8）让数字可见 |
| B2 | **纵深发布门禁**（竞品没有的形态）：preflight 三方互检（tag↔VERSION↔CHANGELOG↔sync）+ 全量回归 + mcp typecheck/test/build + skill 证据 sha256 硬门 | `release-preflight.sh:16-23`；`release.yml:61,65-69,88-91`；`training-gate.mjs:65-71` | 把"发布门禁拦截过多少次坏发布"做成 blog 素材；同时按避短 C2 修时序弱点 |
| B3 | **证据驱动文化**：665 个认证证据文件、63 份 docs/reports、618 份 docs/plans，决策可追溯 | `docs/evidence/skill-training/`、`docs/reports/`、`docs/plans/` | 沉淀惯例保持（本报告即产物）；给 plans 加归档轮替机制防膨胀（见避短 C6） |

### C. 客户端生态

| # | 优点 | 证据 | 放大动作 |
| --- | --- | --- | --- |
| C1 | **单一真源投影体系**：agent-sources 19 张角色卡 + skill-sources 25 个 canonical skill → generatedTargets 统一投影，投影目录声明"可弃置重建" | `agent-sources/manifest.json`；.gitignore "managed by sync-skills.mjs"；`clients/core/definitions.mjs` 7 客户端注册表（含 workbuddy） | 兑现全部客户端能力只差把 gemini/hermes/workbuddy 加进 generatedTargets（现仅 claude/codex/opencode/grok，见避短 C4）——这是低成本高回报的一步 |
| C2 | **无 hook 客户端有明确降级路径**（hermes 走 config.yaml、workbuddy 走 skills+mcp.json） | `clients/core/definitions.mjs` | 把降级矩阵写进 docs-site 客户端支持页，把"功能深度不一"从免责声明变成能力矩阵表 |

### D. 产品面

| # | 优点 | 证据 | 放大动作 |
| --- | --- | --- | --- |
| D1 | **营销承诺可当场复现**：README quick tour 8 条命令与真实命令面一一对应 | README Quick tour ↔ `commander/specs/` 11 模块 | 保持"每个 README 命令必须真实可跑"的纪律；顺手统一 `aios` vs `node scripts/aios.mjs` 调用口径（避短 C5） |
| D2 | **双语重文档站已上线**：docs-site 132 篇双语 + blog 209 篇 + 架构图嵌双语 README | docs-site/、blog-site/、cli.rexai.top | 内容资产是复用竞品分析（P0-P2 报告）改写教程的现成管道 |

---

## 二、缺点清单（避短：每条含风险 + 缓解动作）

### A. 编排能力缺口（对竞品落后，详见竞品报告 P0/P1/P2）

| # | 缺点 | 证据/对照 | 缓解动作（引用竞品报告） |
| --- | --- | --- | --- |
| A1 | 验证闭环 prompt 级、evidence ref 不验"测试真跑过" | `workflow-policy.mjs:448` 只声明；`rex-capability-runtime.mjs:47,68` 仅格式校验 | 竞品报告 P0-1：hook 槽位 + 实测退出码入 evidence |
| A2 | checkpoint 日志型、worktree 丢失 resume 得空树 | `solo-runtime/loop.mjs`、`lifecycle/harness/worktree.mjs:16-24` | 竞品报告 P0-2：CheckpointSaver + 节点粒度 + 一致性校验 |
| A3 | 记忆仅 token-overlap、写入靠自声明、只增不减 | `contextdb/semantic.ts:14`、`run.mjs:391-431` | 竞品报告 P0-3：BM25 融合 + 抽取门槛 + links/TTL（纯本地可达成） |
| A4 | 并行蓝图静态、无沙箱、模型路由未接主路径、无预算熔断 | `blueprints.mjs:4-5,74-111`；`model-router.mjs` 仅 groupchat 消费 | 竞品报告 P1-4/5/6 |

### B. 工程短板（本次审计新发现）

| # | 缺点 | 证据 | 风险 | 缓解动作 |
| --- | --- | --- | --- | --- |
| B1 | **Windows 2 个固有红测**（字节级 sha256 drift guard 撞 CRLF） | `aios-orchestrator-agents.test.mjs:272`；memory 记录"红=正常"误判风险 | 信号腐蚀：真回归被淹没 | 根治：`.gitattributes` 对 canonical markdown/生成物加 `eol=lf`（比测试内归一化更根本）；短期给这 2 个测试的哈希前归一化或 skip 标注 |
| B2 | **证据门时序陷阱**：证据被 gitignore（:125）却要求提交，门排在全量测试后（step 13 > step 9），漏提交 = 整轮白跑 + 重打 tag | `.gitignore:125`；665 个 `git add -f` 跟踪文件；v5.9.0 实战（9284171a 差点没进 tag） | 发布周期拉长、tag 重打惯例化 | ①`.gitignore` 改白名单子集免 add -f；②preflight 增加"tag commit 已含全部门禁证据"本地校验；③证据门前移到 step 9 之前 |
| B3 | **供应链盲区**：无 secret 扫描、无 npm audit 门、Actions 按 tag 引用非 SHA pin、windows smoke 用 `npm install` 非 `npm ci` | workflows 全目录无 gitleaks/audit 步骤；`windows-shell-smoke.yml:27`；`release.yml:14` | 公开仓供应链风险 | 一次性补齐：SHA pin + gitleaks + `npm ci` + audit 门（半天工作量） |
| B4 | tag 惯例不一致（v5.8.0/v5.9.0 annotated，v5.8.1/v5.8.2 lightweight），证据门 base 取 `git describe` 依赖 tag 拓扑 | `git for-each-ref` 实测；`release.yml:84` | base 错位 → 门禁误判 | 发版 checklist 固定 annotated tag；preflight 校验 tag 类型 |
| B5 | CI 用假 CLI shim 掩盖 provider 集成断裂 | `release.yml:48-54` printf shim | 集成回归晚发现 | smoke 层加一个真实 provider 冒烟（可手动触发档） |
| B6 | `scripts/` 8.8M/806 文件 + 根 60+ npm scripts 混装 RL 实验/基准；state-root 双路径并存；`credentials.py` 混在 JS 库；37 个散装 `scripts/*.mjs` 入口 | `du`/`ls` 实测；`state-root.mjs:10-12,43-45` preferLegacyExisting | 维护面持续膨胀 | 定边界：实验移 `experiments/`；legacy 路径定 sunset 版本；CLI 双轨收敛到 specs（见 C5） |
| B7 | CHANGELOG 三处手工同步（CHANGELOG.md / docs/zh-CN / docs-site/zh） | 发版提交 5a6bd078 改 11 文件 | 漏同步=门禁失败 | 由 preflight 从 CHANGELOG.md 生成另两份（生成物化） |

### C. 产品叙事与仓库卫生（本次审计新发现）

| # | 缺点 | 证据 | 风险 | 缓解动作 |
| --- | --- | --- | --- | --- |
| C1 | **叙事三处矛盾**：AGENTS.md 称仓库 "centered on browser automation via MCP"（mcp-server 91M），README 主文零浏览器；mkdocs.yml site_description 还是旧叙事 "local-first Graph Engine" | `AGENTS.md:31`、`README.md`、`mkdocs.yml` | 新用户/贡献者第一印象错位 | 一次性统一三处口径（AGENTS.md 改为"编码客户端编排控制面 + legacy browser MCP"；mkdocs description 更新） |
| C2 | **误入库杂物**：pptx-ai-coding-share/ 364 个 tracked 文件（351 个是 node_modules，8M）；.cache/ 110 个 mkdocs 字体 tracked 且与 ignore 规则冲突；opencode.json tracked 与 ignore 规则自相矛盾 | `git ls-files pptx-ai-coding-share` 实测 | 仓库增重、clone 变慢、发布包体积风险 | `git rm -r --cached` + ignore 规则去重；blog-site（226 文件）迁独立内容仓库或 LFS |
| C3 | 根目录散落调试日志：asset-test.log(995 行)、asset-test2.log(481)、verification-site-redesign.log(1971)、docs-serve.log | `wc -l` 实测（均已被 `.gitignore:17 *.log` 覆盖、未入库） | 工作区卫生差 | 直接删除即可，无需 git 操作 |
| C4 | generatedTargets 仅 4 客户端（claude/codex/opencode/grok），README 宣传 6 个，client registry 实际已注册 7 个（含 workbuddy） | `agent-sources/manifest.json`；`clients/core/definitions.mjs` | 承诺与交付落差（registry 能力反而超过宣传） | 把 gemini/hermes/workbuddy 加入投影目标（复用现有 sync-skills 机制） |
| C5 | CLI 三套解析层并存，`plan` 仍走旧 `parse-args` 双轨维护；README 混用两种调用口径 | `cli/commander` vs `cli/parse-args`；`root-program.mjs` 40 行转发 | 双轨漂移 | 收敛计划：新命令一律 specs，旧 parse-args 逐个迁移并记录在 docs/plans |
| C6 | docs/plans 618 份无归档机制；examples/ 仅 4 个 md 无可运行端到端模板；新用户隐含前置条件（先装客户端、先 init 投影）未写明 | `ls docs/plans | wc -l`；examples/ 14K | 上手转化率受损 | plans 按季度归档；补 1 个可运行模板 + "无客户端也能跑"路径说明 |

### D. 近期变更（v5.9.0 周期）的制度化教训

来自 `.workbuddy/memory/2026-09-04.md` 与 git log，按"教训 → 制度化"提炼：

1. **tag 必须含门禁证据**（9284171a 只在本地 main、tag 指向旧 commit → 白跑）→ 制度化：preflight 增加 `git merge-base --is-ancestor` 校验证据提交 ∈ tag 指向 commit。
2. **Windows CRLF 陷阱反复出现**（grep $'\r' 不可靠、git archive 导出 CRLF、生成物 CRLF 伪影）→ 制度化：canonical markdown 全部 `.gitattributes eol=lf`；检测 CRLF 一律用 node `includes('\r\n')` 写进贡献指南。
3. **判断领先/落后前必须先 fetch**（曾误判 ahead 149）→ 已是个人教训；可写进 release checklist。
4. **发版涉及 11+ 文件多语言手工同步** → 制度化：发版生成脚本（B7）。
5. **安装布局 shim/runtime-root 复杂**（本机 5.8.2 vs 源码 5.9.0 混淆、v5.9.0 曾从未发布）→ 制度化：`aios doctor` 显式打印 runtime root 路径与版本，消除"装没装上/装了哪个版本"的支持成本。

---

## 三、行动优先级总表（合并两轮分析）

| 优先级 | 行动 | 来源 | 估摸规模 |
| --- | --- | --- | --- |
| P0 | 编排：验证 runtime 化（hook 槽位 + 实测 evidence） | 竞品报告 P0-1 | 中 |
| P0 | 记忆：BM25 融合 + 抽取式写入门槛 + links/TTL | 竞品报告 P0-3(1-2) | 中 |
| P0 | 工程：消灭红=正常（CRLF 根治）+ 证据门时序/存储修复（B1/B2） | 本文避短 | 小 |
| P0 | 叙事：AGENTS.md/README/mkdocs 三处口径统一（C1） | 本文避短 | 极小 |
| P1 | checkpoint 状态化 CheckpointSaver | 竞品报告 P0-2 | 中大 |
| P1 | 模型路由接主路径 + 预算熔断 | 竞品报告 P1-6 | 小 |
| P1 | 供应链补齐：SHA pin/gitleaks/npm ci/audit（B3）+ tag 惯例（B4） | 本文避短 | 小 |
| P1 | 仓库减负：pptx node_modules 出库、blog-site 迁移、删根日志、generatedTargets 补 6 客户端（C2/C3/C4） | 本文避短 | 小 |
| P2 | blueprint recipe 化 + 动态 DAG + 沙箱分级 | 竞品报告 P1-4/5 | 大 |
| P2 | 压缩管线、死循环检测、录制回放、CLI 双轨收敛、examples 模板、plans 归档、CHANGELOG 生成化（B5/B6/B7/C5/C6） | 本文 + 竞品报告 P2 | 持续 |

**排序逻辑**：P0 先做"小而关键"的工程/叙事修复（B1/B2/C1 成本极低、消除的是系统性风险）与竞品报告的两项中规模编排补强并行；P1 里模型路由与供应链是纯接线/一次性工作，性价比最高；P2 大改造放在安全网与基础设施稳固之后。

## 四、局限说明

- 工程与产品审计为只读静态审计（file:line 与 git 实测），未运行完整测试/发布流程复核；
- "承诺 vs 支撑"判断基于 README/docs 与代码的对照，未含用户访谈维度；
- 编排缺口的竞品对照细节以 [2026-09-05-competitor-orchestration-analysis.md](./2026-09-05-competitor-orchestration-analysis.md) 为准，本文未重复其证据。

## 附录：审核勘误（2026-09-06，v5.10.0 发布前复核）

经子代理逐条实测复核（12 组事实断言：10 组精确复现、1 组部分属实、3 处行号/计数微偏），正文已就地修正：

1. `clients/core/definitions.mjs` 实为 **7** 客户端注册表（原写 6，漏计 workbuddy）——README 的"6 客户端"宣传反而少报了；
2. `release.yml` 证据门实际行号 **:84**（原引 :87）；
3. 根目录 4 个调试日志已被 `.gitignore:17 *.log` 覆盖、**未入库**——属工作区卫生而非仓库增重，直接删除即可。

其余断言（pptx 351 个 node_modules、.gitignore:125、665 个证据文件、sha256 drift guard :272、AGENTS.md browser 叙事、mkdocs 旧 description、7 个 workflow、无 secret 扫描/SHA pin、tag 类型混杂、state-root 双路径、5a6bd078 改 11 文件、docs/plans 618、examples 4 个 md、opencode.json 矛盾、VERSION=5.9.0）全部复现。完整审核与 v5.10.0 工作清单分诊见 [2026-09-06-v510-worklist-audit.md](./2026-09-06-v510-worklist-audit.md)。
