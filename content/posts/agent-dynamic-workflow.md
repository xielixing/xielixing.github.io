+++
title = 'Dynamic Workflows'
date = 2026-08-27T16:53:28+08:00
lastmod = 2026-09-01T23:30:00+08:00
draft = false
+++

> Anthropic 在 Claude Code v2.1.154 发布了 Dynamic Workflows [\[2\]](#ref-2)——Claude 可以帮你的任务写一段 JavaScript 编排脚本，由运行时在后台执行，这段脚本可以编排几十到几百个 subagent 来并行干活，非常适合复杂任务的场景。

## 常用模式

<figure>
  <img src="/images/agent-dynamic-workflow-patterns.png" alt="Dynamic Workflows 常见模式总览">
  <figcaption>图片转载自 Thariq (@trq212) 的<a href="https://x.com/trq212/status/2061907337154367865">推文</a>，原文见 <a href="https://claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code">Claude 官方博客</a></figcaption>
</figure>

你可以直接让 Claude 建一个 workflow 就开始用；但建立起对工作原理的心智模型，能帮你判断什么时候该用、以及如何用 prompt 引导。下面是六个常见模式，每个都标注了对应的脚本原语——它们可以组合使用：

### Classify-and-act（分类并行动）

用一个分类（classifier）agent 判断任务的类型，然后根据任务类型路由给不同的 agents，或选择不同的行为分支；也可以把分类器放在最后，用来决定如何输出结果。**原语：普通分支逻辑（if/else）。**

### Fan-out-and-synthesize（扇出并汇总）

把任务拆成许多更小的步骤，每个步骤跑一个 agent，最后把结果汇总（synthesize）起来。当小步骤数量很大、或者每个步骤都受益于自己干净的上下文窗口时尤其有用。汇总这一步是一个屏障（barrier）：等所有扇出的 agents 都完成，再把它们的结构化输出合并成一个结果。**原语：pipeline() / parallel() + 一个汇总 agent。**

### Adversarial verification（对抗性验证）

对每个派生的 agent，再派生一个独立的 agent，去对抗性地核验它的输出是否符合评分标准（rubric）或准则。**原语：两层 agent() 调用。**

### Generate-and-filter（生成并筛选）

围绕一个主题生成一批想法，再按评分标准或核验结果筛选、去重（dedupe），只返回质量最高、经过检验的想法。**原语：生成 + schema 评分 + filter() / 排序。**

### Tournament（锦标赛）

不是分工，而是让 agents 同台竞争：派生 N 个 agents，各自用不同的方法尝试同一个任务，再由评判 agent 进行两两对决（pairwise），直到产生赢家。**原语：parallel() + 评委 agent。**

### Loop until done（循环直到完成）

对于工作量未知的任务，循环派生 agents，直到满足某个停止条件（比如没有新的发现、日志里不再有错误、连续两轮无进展）为止，而不是固定跑固定轮数。**原语：while 循环。**

### 示例 Prompts

以下 Prompt 全部逐字引自一手来源——官方文档 [\[1\]](#ref-1) 与 Thariq 的原帖 [\[3\]](#ref-3)——保留英文原文，可直接粘贴使用；每条标注它演示的是上面哪种模式（可组合）：

| 场景 | 演示的模式 | Prompt 原文 |
| --- | --- | --- |
| 审计大量文件的同一类问题 | Fan-out + Adversarial verification | use a workflow to audit every route handler under src/routes/ for missing authentication checks, and adversarially verify each finding before reporting it |
| 修复直到检查通过 | Loop until done | use a workflow to run npx tsc --noEmit and keep fixing the reported errors until the type check passes or two rounds in a row make no progress |
| 复现竞态条件 | Loop until done + Adversarial verification | This test fails maybe 1 in 50 runs. Set up a workflow to reproduce it, form theories and adversarially test them in worktrees /goal don't stop until one theory works. |
| 大规模并行迁移 | Fan-out-and-synthesize | use a workflow to migrate every component under src/components/ from JavaScript to TypeScript, working on each file in its own isolated copy |
| 逐文件审查并汇总 | Fan-out-and-synthesize | use a workflow to review every file changed in this PR for correctness issues, then merge the per-file findings into one ranked summary |
| 跨来源调研对比 | Fan-out-and-synthesize | use a workflow to research how our three competitors handle rate limiting: read their public docs and recent changelog entries in parallel, then compare the approaches |
| 从会话历史提炼规则 | Fan-out-and-synthesize | Using a workflow, go through my last 50 sessions and mine them for corrections I keep making and turn the recurring ones into CLAUDE.md rules |
| 多视角对抗评审 | Fan-out-and-synthesize | Take my business plan and run a workflow where different agents tear it apart from an investor's, a customer's, and a competitor's perspective. |

## 什么是 Dynamic Workflows

一个 Dynamic Workflow 就是一段 JavaScript 脚本：你描述任务，Claude 写出编排脚本，运行时（runtime）在后台执行它，通过调用大量 subagent 并行处理。skill 当然也可以使用自然语言去派生大量的 subagents，但是编排本身就需要消耗上下文，并且不能保证每个subagents传入的上下文足够准确，后面会举例子来说明这点。

脚本保存在 `.claude/workflows/` 下，长这样（官方文档示例）：

```js
export const meta = {
  name: 'audit-routes',
  description: 'Audit every route handler for missing auth checks',
}

const found = await agent('List every .ts file under src/routes/.', {
  schema: { type: 'object', required: ['files'], properties: { files: { type: 'array', items: { type: 'string' } } } },
})

const audits = await pipeline(found.files, file =>
  agent(`Audit ${file} for missing authentication checks.`, { label: file }),
)

return audits.filter(Boolean)
```

### 脚本原语

- **meta**：工作流的元信息，`name` 与 `description` 供 Claude Code 识别。必须是纯字面量，否则不会出现在 `/` 自动补全里。
- **agent(prompt, opts)**：派生一个 subagent 执行任务；失败或被停止时返回 `null`。
- **schema**：要求 agent 按给定的 JSON Schema 返回结构化结果，而不是自由文本——结果直接存入脚本变量，供后续脚本使用。
- **pipeline(items, fn)**：把一组输入逐个送过同一道工序，每个 item 一个 agent，并行执行、互不阻塞。
- **parallel(thunks)**：同时跑一组任务并等待全部完成（屏障语义）。
- **phase(title) / log(msg)**：在进度视图里给 agent 分组、输出一条进度信息。
- **args**：全局变量，保存的 workflow 在调用时传入的参数（比如一串 issue 编号），省去每次改脚本。
- **label**：给 agent 起个标签，方便在进度视图里区分并行任务。
- **filter(Boolean)**：惯用法——`pipeline` 会把失败的 `null` 保留在结果数组里，最后过滤掉，只留下成功的结果。

两个约束值得一提：脚本里的 `Date.now()`、`Math.random()` 会被运行时禁用（保证暂停后重跑时 agent 调用序列一致）；`import()` 会让脚本在启动前就失败，需要第三方库的工作放进 agent 的任务里执行。

## 为什么有效：把计划从上下文搬进代码

用 subagents、skills 或 agent teams 时，Claude 自己就是编排者：逐 turn 决定下一步派生什么，每个结果都落回它的上下文窗口。任务一大，规划和执行挤在同一个窗口里，上下文快速膨胀、注意力被稀释。

长任务下有几个经典失败模式：

- **Agentic laziness（代理惰性）**：Claude 在处理某个特别复杂的多部分任务时，可能在只完成部分进度后就宣告任务完成，例如一次安全审计有 50 个条目，它只处理了 35 个就停了。
- **Self-preferential bias（自我偏好偏差）**：Claude 倾向于偏爱自己的结果或发现，尤其是当它被要求按评分标准（rubric）核验或评判自己的产出时。
- **Goal drift（目标漂移）**：在跨越很多轮次的运行中，对原始目标的忠实度会逐渐丢失，尤其是在上下文压缩（compaction）之后。每一次摘要（summarization）都是有损的，像边界情况的需求或「不要做 X」这类约束，可能会在这个过程中丢失。

Dynamic Workflow 的做法是把「计划」物化成代码：循环、分支、中间结果全部由脚本持有，agent 之间互不共享上下文，Claude 的主上下文只接收最终答案。两个直接推论：

1. **编排本身零上下文成本**——几百个 agent 的调度信息不占主 agent 一字节；
2. **每个 agent 的输入可以精确裁剪**——干净的上下文窗口意味着更少的交叉污染、更高的注意力密度。

除此之外，把编排放进代码还解锁了一种质量模式：**对抗性互评**。让独立的 agent 按评分标准核验彼此的产出，或多角度起草方案再相互权衡——这比单个 agent 的单次 pass 可信得多（下文的 Adversarial verification 模式就是它的典型形态）。

## 和其他 Agent 能力的区别

|     | Subagents | Skills | Agent teams | Workflows |
| --- | --- | --- | --- | --- |
| 是什么 | Claude 派生（spawn）出来的 worker | Claude 遵循的指令（instructions） | 一个监督各 peer session 的 lead agent | 由运行时（runtime）执行的脚本 |
| 谁决定下一步运行什么 | Claude，每个 turn 逐个决定 | Claude，按照 prompt 决定 | lead agent，每个 turn 逐个决定 | 脚本 |
| 中间结果存在哪里 | Claude 的上下文窗口 | Claude 的上下文窗口 | 共享的任务列表 | 脚本变量 |
| 可重复使用的是什么 | worker 的定义 | 指令的定义 | team 的定义 | 编排（orchestration）本身 |
| 规模  | 每个 turn 转派少量任务 | 与 subagents 相同 | 少数几个长期运行的 peers | 每次运行可编排几十到几百个 agents |
| 中断时 | 该 turn 重新开始 | 该 turn 重新开始 | teammates 继续运行 | 可在同一会话中恢复 |

来源：[Claude Code 官方文档 · Workflows](https://code.claude.com/docs/en/workflows)

Claude Code 维护者 Boris Cherny 的总结很精炼：与 agent teams 相比，workflow 一是可以支撑多 1–2 个数量级的并行 agent，二是以分阶段、半结构化的方式推进工作 [\[4\]](#ref-4)。一句话记忆：**Subagents 是「派工人」，Skills 是「发操作手册」，Agent teams 是「组个班组」，Workflow 是「写一段程序，让工人们按程序干」**。

## 上手：触发、运行、复用

### 触发

| 方式 | 说明 |
| --- | --- |
| prompt 里带 `ultracode` 关键字 | 例如 `ultracode: audit every API endpoint under src/routes/ for missing auth checks`；用自然语言说「use a workflow」效果相同 |
| `/effort ultracode` | 会话级开关：之后每个实质任务 Claude 都会自动规划 workflow（叠加 `xhigh` 推理档位），token 消耗和耗时都会明显上升 |
| 内置 `/deep-research` | 开箱即用的调研工作流：多角度扇出搜索、交叉核验来源、产出带引用的报告 |

注意两点：v2.1.160 之前的触发关键字是 `workflow`；另外在 v2.1.218 实测，`ultracode` 关键字在 `claude -p` 无头模式下同样能触发 workflow（本文实验就是无人值守跑的，见下文），「`-p` 不触发」的说法已过时；但保存的 workflow 用 `/<name>` 调用仍然只在交互式会话里生效。

### 运行

Workflow 在后台运行，会话保持可用。用 `/workflows` 打开进度视图：每个 phase 显示 agent 数、token 消耗和耗时，可以一路下钻到单个 agent 的 prompt、工具调用和结果。视图里还能暂停/恢复（`p`）、停掉单个 agent 或整个运行（`x`）。

恢复的语义是：已完成的 agent 直接返回缓存结果；如果某个 agent 失败、或你改了脚本导致某个 agent 的 prompt 变化，它和它之后的所有 agent 会重跑。

### 复用

跑完一次满意的 workflow，在 `/workflows` 里按 `s` 就能把它存成命令：项目级存到 `.claude/workflows/`（随仓库共享），个人级存到 `~/.claude/workflows/`（所有项目可用）。之后在 `/` 自动补全里以 `/<name>` 调用，还可以通过 `args` 传参，比如 `Run /triage-issues on issues 1024, 1025, and 1030`。

## 成本与限制

### 成本：先泼一盆冷水

官方明确说 workflow 的消耗会「meaningfully more than a typical session」[\[1\]](#ref-1)——一次运行可能相当于普通会话几倍甚至几十倍的 token。

- 进度面板会在 agent 超过 25 个、或预估 token 超过 150 万时给出 `Large workflow` 警告（只是提醒，不会拦截）；
- 可以用 size guideline 控制规模：`small`（<5 个 agent）、`medium`（<15，默认）、`large`（<50）；
- 省 token 的实践：先在小范围试跑（一个目录而不是整仓）、给 fan-out 阶段指定更小的模型、明确告诉 Claude 哪些步骤不需要最强模型。

社区里的真实翻车案例 [\[4\]](#ref-4)：有人在 5x Max 计划上跑出一个 62 个 Opus 1M subagent 的 workflow，约 5 小时的额度 18 分钟烧完；也有人 review 一个不大的包被派出 90 个 agent，第一次打满了 Claude Max 的用量上限。

### 限制

| 限制  | 原因  |
| --- | --- |
| 运行中途不能接受用户输入 | 只有 agent 的权限提示（permission prompts）才能暂停运行。如果需要在阶段之间做人工确认（sign-off），就让每个阶段各自作为一个独立的工作流运行 |
| workflow 自身不能直接访问文件系统或执行 shell 命令 | 文件的读写和命令执行都由 agents 完成，脚本只负责协调（coordinate）agents |
| 不支持加载模块：脚本里包含 `import()` 会在运行开始前就失败 | 脚本主体是纯 JavaScript。需要用到第三方库的工作，请放进 agent 的任务里执行 |
| 最多 16 个并发 agents；当 Claude Code 可用的 CPU 较少时数量会更少（包括在受 CPU 限制的容器内） | 限制本地资源的使用 |
| 单次 `parallel()` / `pipeline()` 最多 4096 项，超长列表直接报错 | 静默截断会在不给提示的情况下丢掉一部分工作负载 |
| 在 fan-out（扇出）场景中，与第一个 agent 共享 prompt-cache 前缀的 agents，默认会在第一个 agent 启动后最多 5 秒才开始启动 | 除了第一个 agent 外，其余 agent 都直接读取第一个 agent 缓存好的前缀，而不是各自对未缓存的内容重新处理一遍 |
| 每次运行最多总共 1,000 个 agents | 防止失控的循环（runaway loops） |

## 我的看法

HN 上对这项特性最大的批评是「token 焚烧炉」：让一群 agent 互相 review、反复循环，更像是在烧钱而不是解决问题 [\[4\]](#ref-4)。这个批评有道理，但展开看会发现，真正决定值不值得的不是 agent 数量，而是**任务有没有清晰的验证标准**。

开头那组官方内部用例之所以成立，不是因为 agent 多，而是因为裁判是现成的：token 用量降了几成、SDK 启动快了多少、CI 有没有变绿、删掉了多少行代码，全都是可以机械度量的指标 [\[4\]](#ref-4)。反过来，如果任务没有机械的验证标准，几百个 agent 只会把一份「不确定」放大成几十份昂贵的不确定。

所以我的判断标准是：**任务可机械验证、规模大且重复、步骤结构清晰，三者占其二就值得上 workflow**。一次性小任务直接对话更快；需要频繁人工介入的探索性工作则要谨慎——运行中途无法注入你的想法是硬伤，这意味着 prompt 的质量必须前置投入，而不是边跑边调。

一句话总结：Dynamic Workflow 买到的不是「更多答案」，而是**确定性和干净的上下文**。验证标准越清晰，这笔交易越划算。

（本节为综合官方文档与社区讨论的观点整理，非第一手实测结论。）

## 实测：脚本编排 vs 自然语言编排

上面的判断标准值得用数据验一验。我搭了一套实验基建（fixture 仓库、transcript 逐消息 token 统计、自动评分），放在 [dynamic-workflows-lab](https://github.com/xielixing/dynamic-workflows-lab)，两个实验各跑 5 轮 × 2 臂，全部用 `claude -p` 无人值守执行（v2.1.218，单一模型、本地代理配置，每轮全新会话）。这是工程级实证：n=5、合成 fixture、单一环境，不是论文。

### 实验 1：大规模重复审计

24 个 TS 文件预埋 6 处「async 调用结果被当布尔值用」（必须读代码才能发现，tsc 报不出来，还撒了诱饵）。两臂同一个任务：「审计每个文件，输出每文件报告」。

| 指标（5 轮均值） | 自然语言编排 | Workflow |
| --- | --- | --- |
| 交付 recall（最终报告含全部预埋 bug） | 3/5 轮交付 | **5/5** |
| 集体 recall（子代理实际发现） | 5/5 | 5/5 |
| 报告轮间稳定性（Jaccard） | 0.4（recall 标准差 0.49） | **1.0（零方差）** |
| 文件覆盖 | 24/24 | 24/24（脚本强制一文件一 agent） |
| 主线程 token | 369,840 | **229,855** |
| 全会话 token | ≥374,922\* | **287,480** |
| 耗时 | 178s | 207s |

\* 该版本 transcript 不记录 Task 子代理用量，NL 臂总 token 被低估。所以保守结论是：**WF 的全会话总消耗（287k）低于 NL 仅主线程的消耗（370k）**——还不算 NL 那些干活的子代理。

最有意思的是交付 recall 和集体 recall 的分离：**两臂的子代理都全找到了 6 处 bug**，但 NL 编排者在 2/5 的轮次里没把结果合并成交付报告就停了——文章开头说的 agentic laziness，抓了个现行。WF 臂把交付质量钉死在 1.0，且五轮报告的发现集合完全一致（Jaccard=1.0）：脚本保证覆盖，屏障保证汇总。

### 实验 2：复杂任务修到全绿

换一个结构更复杂的任务：6 个跨模块 bug（含 1 条类型错误链），12 个测试（6 失败 + 6 守卫），验证标准机械——`npm test` 全绿且 `tsc --noEmit` 干净，且不许改测试。这次 NL 编排可以逐步重规划，看 workflow 还能不能赢。

| 指标（5 轮均值） | 自然语言编排 | Workflow |
| --- | --- | --- |
| 合法成功率（全绿 + 无作弊） | 4/5 | **5/5** |
| 根因命中 | 5.2/6（失败轮只修了 2/6 就放弃） | **6/6** |
| 作弊率（改测试 / @ts-ignore / 吞异常） | 0 | 0 |
| 全会话 token | **468,392** | 499,570 |
| 耗时 | **166s** | 757s |

复杂任务上 workflow 依然全胜成功率，NL 臂的失败模式还是老样子：一轮 77 秒后放弃，只修了 2/6。代价也清楚：**WF 慢 3–4 倍、token 略高**——主线程要跑验证循环、消化测试输出，这部分上下文成本脚本省不掉。两臂作弊率都是 0，「对抗验证降作弊率」的假设没有被区分出来，如实报告。另一个意外发现：所有成功轮（两臂都是）的最终 diff 全部是 +6/-6 的外科手术式修复，没有一例打补丁。

### 实验给判断标准的反愧

两个实验合起来正好落在「我的看法」那套标准上：**机械可验证 + 规模大且重复 + 结构清晰，占得越全，workflow 赢面越大、赢势越稳**。实验 1（三项全占）里 WF 是碾压性的确定性优势；实验 2（占两项，结构不算清晰）里 WF 赢在成功率，但要用耗时和 token 去换。反过来，如果你的一次性小任务一项都不占，直接对话仍然是对的。

## References

<a id="ref-1"></a>[1] Anthropic. Claude Code 官方文档 · Orchestrate subagents at scale with dynamic workflows. [code.claude.com/docs/en/workflows](https://code.claude.com/docs/en/workflows)

<a id="ref-2"></a>[2] Anthropic. Introducing dynamic workflows in Claude Code. 2026-05-28. [claude.com/blog/introducing-dynamic-workflows-in-claude-code](https://claude.com/blog/introducing-dynamic-workflows-in-claude-code)

<a id="ref-3"></a>[3] Anthropic. A harness for every task: dynamic workflows in Claude Code. [claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code](https://claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code)

<a id="ref-4"></a>[4] Hacker News. Dynamic Workflows in Claude Code（200 points · 135 条讨论）. [news.ycombinator.com/item?id=48311705](https://news.ycombinator.com/item?id=48311705)

<a id="ref-5"></a>[5] Thariq (@trq212). Dynamic Workflows 模式总览图. [x.com/trq212/status/2061907337154367865](https://x.com/trq212/status/2061907337154367865)
