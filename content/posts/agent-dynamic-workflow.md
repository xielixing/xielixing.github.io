+++
title = 'Dynamic Workflow 介绍'
date = 2026-08-27T16:53:28+08:00
lastmod = 2026-08-31T19:28:00+08:00
draft = false
+++

> **一句话总结**：Anthropic 在 Claude Code v2.1.154 版本之后发布了 Dynamic Workflows 的特性——Claude 会为你的任务现场写一段 JavaScript 编排脚本，由运行时在后台执行，脚本派生几十到几百个 subagent 并行干活。它适合「任务大到单个上下文装不下」或「同一步骤要在大量条目上重复」的场景，代价是 token 消耗显著高于普通会话。本文介绍它的原理、用法、常用模式，以及什么时候**不该**用它。

## 从一个真实故事说起

2026 年 5 月底，Anthropic 发布 Dynamic Workflows 时，Bun 作者 Jarred Sumner 展示了一个硬核用例：用 Dynamic Workflows 把 Bun 从 Zig 重写为 Rust——约 75 万行 Rust 代码，从第一个 commit 到 merge 只用了 11 天，合入时 99.8% 的既有测试套件通过 [2]。

整个工程的编排大致是这样的：

1. 第一个 workflow 扫描 Zig 代码库，为每个 struct 字段映射出正确的 Rust 生命周期；
2. 第二个 workflow 派生数百个 agent 并行工作，每个 `.rs` 文件都是对应 `.zig` 文件的行为等价移植，且每个文件配两名 reviewer；
3. 一个 fix loop 反复驱动构建和测试套件，直到两者全绿；
4. 移植合入后，一个通宵运行的 workflow 清理多余的数据拷贝，逐条开出 PR 供人终审。

关键不是「模型变聪明了」，而是**一段脚本编排了几百个各带独立上下文窗口的 agent**。这就是本文的主角。

## 什么是 Dynamic Workflows

一个 Dynamic Workflow 就是一段 JavaScript 脚本：你描述任务，Claude 现场写出编排脚本，运行时（runtime）在后台执行它，脚本再派生大量 subagent 并行处理。与在会话里手动派生 subagent 不同，**编排逻辑本身不占 Claude 的上下文——它就在代码里**。该特性对所有付费计划开放（Pro 需在 `/config` 里手动开启）[1]。

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

Claude Code 维护者 Boris Cherny 的总结很精炼：与 agent teams 相比，workflow 一是可以支撑多 1–2 个数量级的并行 agent，二是以分阶段、半结构化的方式推进工作 [4]。一句话记忆：**Subagents 是「派工人」，Skills 是「发操作手册」，Agent teams 是「组个班组」，Workflow 是「写一段程序，让工人们按程序干」**。

## 上手：触发、运行、复用

### 触发

| 方式 | 说明 |
| --- | --- |
| prompt 里带 `ultracode` 关键字 | 例如 `ultracode: audit every API endpoint under src/routes/ for missing auth checks`；用自然语言说「use a workflow」效果相同 |
| `/effort ultracode` | 会话级开关：之后每个实质任务 Claude 都会自动规划 workflow（叠加 `xhigh` 推理档位），token 消耗和耗时都会明显上升 |
| 内置 `/deep-research` | 开箱即用的调研工作流：多角度扇出搜索、交叉核验来源、产出带引用的报告 |

注意两点：v2.1.160 之前的触发关键字是 `workflow`；关键字只在你手动输入的 prompt 里生效，`-p`、定时任务、webhook 进来的 prompt 不会触发。

### 运行

Workflow 在后台运行，会话保持可用。用 `/workflows` 打开进度视图：每个 phase 显示 agent 数、token 消耗和耗时，可以一路下钻到单个 agent 的 prompt、工具调用和结果。视图里还能暂停/恢复（`p`）、停掉单个 agent 或整个运行（`x`）。

恢复的语义是：已完成的 agent 直接返回缓存结果；如果某个 agent 失败、或你改了脚本导致某个 agent 的 prompt 变化，它和它之后的所有 agent 会重跑。

### 复用

跑完一次满意的 workflow，在 `/workflows` 里按 `s` 就能把它存成命令：项目级存到 `.claude/workflows/`（随仓库共享），个人级存到 `~/.claude/workflows/`（所有项目可用）。之后在 `/` 自动补全里以 `/<name>` 调用，还可以通过 `args` 传参，比如 `Run /triage-issues on issues 1024, 1025, and 1030`。

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

以下是 Claude Code 团队在博客里给出的一些典型任务描述 [3]：

| 场景 | Prompt |
| --- | --- |
| 复现竞态 | 「这个测试大概每 50 次运行就会失败 1 次。设置一个 workflow 来复现它。针对这个竞态（race）问题提出几种相互竞争的假设，并且不要停下来，直到有一个假设经受住证据的检验。」 |
| 挖掘个人习惯 | 「用一个 workflow 翻看我最近 50 个会话，挖掘出我反复犯的错误，并把反复出现的那些整理成 CLAUDE.md 里的规则。」 |
| 工单挖掘 | 「用一个 workflow 翻阅 Slack 里过去六个月的 #incidents 频道，找出反复出现但还没人提交工单的根因。」 |
| 多视角评审 | 「拿我的商业计划书跑一个 workflow，让不同的 agents 分别从投资人、客户、竞争对手的视角把它批得体无完肤。」 |
| 简历筛选 | 「这里有一个装着 80 份简历的文件夹，用一个 workflow 为后端岗位给它们排序，并复核前十名。然后使用 AskUserQuestion 工具对我进行面试，以确定评分标准（rubric）。」 |
| 命名锦标赛 | 「我需要给这个 CLI 工具起个名字。用一个 workflow 头脑风暴出一堆候选名字，然后跑一个锦标赛式的淘汰赛，选出前 3 名。」 |
| 大规模重命名 | 「用一个 workflow 把我们的 User 模型在所有地方统一重命名为 Account。」 |
| 博客事实核查 | 「用一个 workflow 检查我的博客草稿，把每一个技术论断都和代码库对照核实，我可不想发布任何有错的内容。」 |

## 成本与限制

### 成本：先泼一盆冷水

官方明确说 workflow 的消耗会「meaningfully more than a typical session」[1]——一次运行可能相当于普通会话几倍甚至几十倍的 token。

- 进度面板会在 agent 超过 25 个、或预估 token 超过 150 万时给出 `Large workflow` 警告（只是提醒，不会拦截）；
- 可以用 size guideline 控制规模：`small`（<5 个 agent）、`medium`（<15，默认）、`large`（<50）；
- 省 token 的实践：先在小范围试跑（一个目录而不是整仓）、给 fan-out 阶段指定更小的模型、明确告诉 Claude 哪些步骤不需要最强模型。

社区里的真实翻车案例 [4]：有人在 5x Max 计划上跑出一个 62 个 Opus 1M subagent 的 workflow，约 5 小时的额度 18 分钟烧完；也有人 review 一个不大的包被派出 90 个 agent，第一次打满了 Claude Max 的用量上限。

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

HN 上对这项特性最大的批评是「token 焚烧炉」：让一群 agent 互相 review、反复循环，更像是在烧钱而不是解决问题 [4]。这个批评有道理，但展开看会发现，真正决定值不值的不是 agent 数量，而是**任务有没有清晰的验证标准**。

Bun 重写之所以成立，不是因为 agent 多，而是因为裁判是现成的：75 万行代码的行为等价性，由 99.8% 通过的既有测试套件来裁决。官方团队的内部用法也是同一个结构——CI 提速、flaky test 修复、SDK 启动时间优化 61%、69 个删掉上万行代码的简化 PR——全都是「有编译器、测试或性能指标兜底」的任务 [4]。反过来，如果任务没有机械的验证标准，几百个 agent 只会把一份「不确定」放大成几十份昂贵的不确定。

所以我的判断标准是：**任务可机械验证、规模大且重复、步骤结构清晰，三者占其二就值得上 workflow**。一次性小任务直接对话更快；需要频繁人工介入的探索性工作则要谨慎——运行中途无法注入你的想法是硬伤，这意味着 prompt 的质量必须前置投入，而不是边跑边调。

一句话总结：Dynamic Workflow 买到的不是「更多答案」，而是**确定性和干净的上下文**。验证标准越清晰，这笔交易越划算。

（本节为综合官方文档与社区讨论的观点整理，非第一手实测结论。）

## References

[1] Anthropic. Claude Code 官方文档 · Orchestrate subagents at scale with dynamic workflows. [code.claude.com/docs/en/workflows](https://code.claude.com/docs/en/workflows)

[2] Anthropic. Introducing dynamic workflows in Claude Code. 2026-05-28. [claude.com/blog/introducing-dynamic-workflows-in-claude-code](https://claude.com/blog/introducing-dynamic-workflows-in-claude-code)

[3] Anthropic. A harness for every task: dynamic workflows in Claude Code. [claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code](https://claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code)

[4] Hacker News. Dynamic Workflows in Claude Code（200 points · 135 条讨论）. [news.ycombinator.com/item?id=48311705](https://news.ycombinator.com/item?id=48311705)

[5] Thariq (@trq212). Dynamic Workflows 模式总览图. [x.com/trq212/status/2061907337154367865](https://x.com/trq212/status/2061907337154367865)
