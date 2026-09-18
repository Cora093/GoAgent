# Jev Fit Assessment（TypeSafe AI System One）

评估对象：TypeSafe AI **Jev**（`jev-1.13.0` / `jev-latest`）是否适合作为 **GoAgent 仓库** 的辅决策层。

口径：Jev 是结构化、可计价的判断模型（Choice / Score / Noul），**不是**写代码或讲棋的 LLM。棋理事实仍由 KataGo 判定；自然语言讲解仍由用户配置的多模态 LLM 生成。本文件只做适配分析，**未改业务代码、未加依赖**。

结论：**部分适合**。值得在老师智能体的工具风险门与低置信意图分流上做离线/旁路实验；不值得替换 KataGo 数值分类、知识几何匹配，或把 Jev 当讲棋模型。

---

## 1. 仓库画像

| 项 | 现状 |
| --- | --- |
| 产品 | 本地优先、跨平台 Electron 围棋学习工作台（GoAgent） |
| 目标用户 | 围棋学生；老师对话区用自然语言下达复盘/训练任务 |
| 技术栈 | Electron + React + TypeScript；KataGo；OpenAI-compatible 多模态 LLM（`openai` SDK + `zod`）；本地知识库 |
| Agent | 有。`TeacherAgentRuntime`（`src/main/services/teacherAgent.ts`）带 tool calling |
| 工具 | `library.findGames`、`sgf.readGameRecord`、`katago.*`、`knowledge.searchLocal`、`studentProfile.*`、`web.searchGoKnowledge`、`settings.writeAppConfig`、`filesystem.read`、`shell.exec`、`report.saveAnalysis` 等 |
| 分类/路由 | `classifyTeacherIntent`：正则加权，不是 LLM |
| 审核门控 | `qualityGate` / `claimVerifier` / `evaluateTeachingReadiness` 已实现；**质量门未接入老师运行时** |
| 自动化流水线 | GitHub Actions：typecheck + build；大量 `eval:*` 黄金夹具脚本。不是会 apply 补丁的 coding harness |
| 网站 | `website/` 为 Astro 静态站点，无决策点 |

架构优先级（`docs/TEACHER_AGENT.md`）：KataGo 是事实裁判；截图辅助形状理解；本地知识卡提供稳定教学语言；联网搜索只作补充。冲突时 **KataGo 胜出**。

隐私边界：棋谱、画像、报告默认在 `~/.goagent`；当前手讲解会把棋盘图 + KataGo JSON + 知识摘录发到**用户配置的 LLM**。联网搜索按规定只发泛化围棋主题。

---

## 2. 为什么是「部分适合」而不是「适合 / 不适合」

**适合的部分（Jev 的典型用法能对上真实模块）：**

1. **Agent 工具风险门**。`executeAgentToolCall` 目前对 LLM 选出的工具几乎立刻执行。`TeacherToolPolicy` 类型只有 `'auto'`。`shell.exec` 仅靠 `dangerousShellCommand` 正则黑名单；`filesystem.read` 可读任意存在的路径；`web.searchGoKnowledge` **没有**在代码里校验 query 是否泛化、是否含 PII。这正是 Jev 的 Choice/Noul 场景：auto / review / block。
2. **脆弱 if-else 意图分流**。`intentClassifier.ts` 用中英日韩正则给 `current-move | game-review | batch-review | training-plan | open-ended | move-range` 打分。快路径（前端 `mode=`）应保持确定性；**低置信开放 prompt** 才适合 Jev Choice。
3. **讲解质量门 / 声明核验**。`qualityGate.ts` + `claimVerifier.ts` 已有坐标、百分比、定式、绝对化措辞规则，但运行时 `teacherAgent.ts` **没有调用** `runTeacherQualityGate`。Jev 适合做「是否过度绝对 / 是否该修稿」的旁路评分，而不是替代坐标白名单。

**不适合 / 不应让 Jev 插手的部分：**

- **KataGo 数值分类**：`analysis/classifier.ts`、`pvConfidence.ts`、`readinessGate.ts`、`adaptiveProfile.ts` 已是 visits / winrateLoss / scoreLoss 的确定性规则。用云端模型重判「这手是 blunder 吗」会引入延迟、费用和与引擎不一致的风险，且违背「KataGo 是事实裁判」。
- **棋形/定式几何匹配**：`knowledge/matchEngine.ts`、joseki trie、形状识别是本地结构匹配，不是开放判断。
- **讲棋正文**：多模态 LLM 的职责；Jev 不做开放文本生成。
- **静态网站、打包脚本、CI typecheck**：无明显结构化决策点。
- **本仓开发用 coding harness**：CI 不做自动 apply 补丁；用 Jev 给 PR 打爆炸半径分对这个 Electron 桌面仓收益很低。

---

## 3. 具体插入点（仅设计，未实现）

以下问题只应接收**脱敏后的程序状态**（工具名、命令摘要、意图分数、已提取声明类型），**禁止**把 SGF 全文、棋盘 PNG、学生姓名、API Key、本机路径发给 TypeSafe。

### 3.1 工具风险门 — `teacherAgent.ts` `executeAgentToolCall` / `runShell` / `searchWebForGoKnowledge`

| 类型 | 示例 typed question | 期望策略 |
| --- | --- | --- |
| Choice | `Given tool={name}, args_summary={redacted}, which action?` options: `auto` / `ask_user` / `block` | `library.*` / `katago.analyzePosition` / `knowledge.searchLocal` → **auto**；`settings.writeAppConfig` / `studentProfile.write` / `web.searchGoKnowledge` → 高置信 **auto**，低置信 **review**；`shell.exec` / `filesystem.read` 超出围棋环境探测 → **block** 或 **review** |
| Noul | `Is this shell command destructive or out of GoAgent's KataGo/env-setup scope?` | P(yes) ≥ 0.8 → **block**（叠加现有正则）；0.4–0.8 → **review**；否则走正则黑名单 |
| Noul | `Does this web-search query contain student names, SGF moves, coordinates dump, paths, or secrets?` | P(yes) ≥ 0.7 → **block**（落实 SECURITY.md，当前代码未做） |

现有正则黑名单必须保留为 **fail-closed 底线**。Jev 超时或无 Key 时回退到正则，不得「默认放行」。

### 3.2 意图分流 — `teacher/intentClassifier.ts`（仅 `confidence === 'low'`）

| 类型 | 示例 typed question | 期望策略 |
| --- | --- | --- |
| Choice | `Classify the teacher task.` options: `current-move` / `game-review` / `batch-review` / `training-plan` / `move-range` / `open-ended` | 与正则一致且 Jev 高置信 → **auto** 走对应快路径；不一致 → **review**（走 `open-ended` agent，不静默改路径） |
| Score | `How safe is skipping KataGo for this prompt?` 1–5 | ≥4 且无 `gameId` → 允许纯知识/计划路径；否则必须先分析 |

前端已指定 `mode=current-move|move-range` 时 **不要**调用 Jev。

### 3.3 讲解质量旁路 — `teacher/qualityGate.ts`（先 shadow，再决定是否修稿）

| 类型 | 示例 typed question | 期望策略 |
| --- | --- | --- |
| Noul | `Does the teacher text make an absolute joseki/life-death claim unsupported by the provided motif flags?` | 输入只用 `claimVerifier` 已抽出的 flags + 短句，不用棋盘图。P(yes) 高 → **review**（触发已有 vision/claim 修稿环），不直接 **block** 用户可见输出，直到误报率可接受 |
| Score | `Groundedness of this teaching turn vs provided evidence flags` 1–5 | ≤2 → 附加质量门注释（`appendTeacherQualityGateNote`）；不必为了 Jev 再打一轮讲棋 LLM |

确定性违规（非法坐标、>100% 胜率）继续用 `claimVerifier`，不要交给 Jev。

### 3.4 （可选，优先级低于上三项）模型路由 — `llm/openaiCompatibleProvider.ts`

当前只有用户配置的单一 `llmModel`，没有 cheap/reasoning 双模型设置。若未来增加「轻量模型 / 推理模型」：

| 类型 | 示例 | 策略 |
| --- | --- | --- |
| Choice | `Route this teacher turn:` `cheap_multimodal` / `reasoning` | `current-move` 有完整 KataGo+截图 → cheap；`open-ended` 且要写训练计划/改设置 → reasoning。**auto** 切换；不确定 → 保持用户所选模型 |

未配置第二模型前不要接这条。

### 3.5 明确不插入

- `analysis/classifier.ts`、`analysis/readinessGate.ts`、`analysis/pvConfidence.ts`、`analysis/adaptiveProfile.ts`
- `knowledge/matchEngine.ts` 几何分数
- `website/`、electron-builder、签名/公证脚本
- 用 Jev「优化代码」或替代 Claude/Codex 写补丁

---

## 4. 推荐接入方式

**推荐：Electron 主进程内的 HTTP `POST https://api.typesafe.ai/v1/systemone`，或官方 JS SDK。**

| 方式 | 对本仓 |
| --- | --- |
| HTTP / JS SDK | 适合。主进程已有 `fetch`、设置里已有 API Key + `safeStorage` 模式，可复用为可选 `jevApiKey`。无 LangChain 依赖，不必为 Jev 引入。 |
| LangChain TypeSafeClassifier | 不适合作为第一步。仓库没有 LangChain。 |
| MCP `jev-mcp` | 不适合产品运行时。GoAgent 不是 MCP 主机；MCP 更像 Cursor 侧 coding agent，而本仓要优化的是桌面老师智能体。 |
| 无 | 若隐私/Key/延迟实验失败，保持现状：正则工具黑名单 + 正则意图 + 本地质量门。 |

---

## 5. 风险与成本

| 风险 | 说明 |
| --- | --- |
| API Key | 桌面端又多一个云厂商密钥。必须可选、默认关闭、走现有 secret store；renderer 不得拿到明文。 |
| 隐私 | 与「本地优先」冲突。Jev 请求不得含 SGF、PNG、姓名、路径、Key。工具门只发 tool name + 脱敏 args。 |
| 延迟 | 厂商口径约 70–500ms。叠在 KataGo + 多模态 LLM 之后，对「当前手」主路径不友好。应只挡高风险工具和低置信意图。 |
| 误判 | 围棋术语（征子、扑、点杀）可能被通用安全模型误判。策略必须 **fail 到现有规则**，且高风险工具 fail-closed。 |
| 职责边界 | Jev ≠ 讲棋模型 ≠ coding LLM。禁止用它生成复盘正文或改业务代码。 |
| 费用 | 输入约 $0.042/MTok、输出免费（以官方为准）。工具门每回合 token 很少；不要对每手 KataGo 候选点调用。 |
| 可用性 | 无网、无 Key、超时应视为「Jev 未参与」，不是「允许执行」。 |

---

## 6. MVP（1–3 步，不改老师主路径）

1. **Shadow 日志（本地）**：在 `executeAgentToolCall` 与 `classifyTeacherIntent` 旁记录脱敏 JSON（tool、intent、confidence），不调用外网。用真实会话看决策点频率。
2. **离线脚本打 TypeSafe**：用 20–50 条合成样本（危险 shell、含姓名的 web query、模糊「帮我看看」prompt）POST System One，对比正则。**不把脚本打进产品依赖。**
3. **仅当准确率明显优于正则时**：用设置开关接入工具风险门；超时回退 `dangerousShellCommand`；质量门先只写 note，不拦截。

成功标准：高风险 `shell.exec` / 隐私 web query 召回优于纯正则，且当前手讲解延迟无明显回归。失败则停在步骤 1–2，不合并运行时依赖。

---

## 7. 本评估改动范围

未改业务代码，未增加 npm 依赖，未改 CI。仅新增本分析文档。
