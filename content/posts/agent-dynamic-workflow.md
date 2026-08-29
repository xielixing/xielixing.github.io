+++
title = 'Dynamic Workflow 介绍'
date = 2026-08-27T16:53:28+08:00
draft = false
+++

## 什么是 Dynamic Workflows

Anthropic 在 Claude Code v2.1.154 版本之后发布了Dynamic Workflows的特性，一个Dynamic Workflow脚本会编排很多subagents去并行的处理事情，这段脚本会由一个运行时去执行

用户可以用两种方式来使用dynamic workflows：一种是在prompt里加入workflows关键字来触发，另一种是使用 ultracode 这个effort。

### 示例 Prompts

我们可以通过一些例子来看dynamic workflows可以做什么，以下是Claude Code团队推荐的一些常用的prompt

> "这个测试大概每 50 次运行就会失败 1 次。设置一个 workflow 来复现它。针对这个竞态（race）问题提出几种相互竞争的假设，并且不要停下来，直到有一个假设经受住证据的检验。"

> "用一个 workflow 翻看我最近 50 个会话，挖掘出我反复犯的错误，并把反复出现的那些整理成 CLAUDE.md 里的规则。"

> "用一个 workflow 翻阅 Slack 里过去六个月的 #incidents 频道，找出反复出现但还没人提交工单的根因。"

> "拿我的商业计划书跑一个 workflow，让不同的 agents 分别从投资人、客户、竞争对手的视角把它批得体无完肤。"

> "这里有一个装着 80 份简历的文件夹，用一个 workflow 为后端岗位给它们排序，并复核前十名。然后使用 AskUserQuestion 工具对我进行面试，以确定评分标准（rubric）。"

> "我需要给这个 CLI 工具起个名字。用一个 workflow 头脑风暴出一堆候选名字，然后跑一个锦标赛式的淘汰赛，选出前 3 名。"

> "用一个 workflow 把我们的 User 模型在所有地方统一重命名为 Account。"

> "用一个 workflow 检查我的博客草稿，把每一个技术论断都和代码库对照核实，我可不想发布任何有错的内容。"

来源：[Claude 官方博客 · A Harness for Every Task: Dynamic Workflows in Claude Code](https://claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code)

## 什么时候需要 Dynamic Workflows

可以看出来以上的任务都是一些比较复杂的大型任务，大量的任务意味着大量的上下文，全部放到一个上下文窗口里面会导致注意力稀释，用脚本编排不同的subagent更加具有确定性，换句话说：

1. 编排subagent需要的上下文是零成本的，不占主agent的上下文空间
2. 每个subagent可以传入的上下文也可以做到非常精确的把控，dynamic workflows非常适合大量subagent并行工作的场景。

### Dynamic Workflows从根本来说解决了什么问题

Claude Code在执行任务的时候，如果在同一个上下文窗口进行规划(决定工作流如何编排)和执行(执行对应的工作流)会导致上下文占用过多，虽然Claude Code已经可以在绝大部分场景表现得足够好了，但是在之后提到的一些典型场景：大规模并行、高度结构化或对抗性验证的任务上，表现的可能不太好，根本原因还是完成任务需要的上下文太长。

比如我们常见的agent会犯的一些问题：

- **Agentic laziness（代理惰性）**：Claude 在处理某个特别复杂的多部分任务时，可能在只完成部分进度后就宣告任务完成，例如一次安全审计有 50 个条目，它只处理了 35 个就停了。
- **Self-preferential bias（自我偏好偏差）**：Claude 倾向于偏爱自己的结果或发现，尤其是当它被要求按评分标准（rubric）核验或评判自己的产出时。
- **Goal drift（目标漂移）**：在跨越很多轮次的运行中，对原始目标的忠实度会逐渐丢失，尤其是在上下文压缩（compaction）之后。每一次摘要（summarization）都是有损的，像边界情况的需求或「不要做 X」这类约束，可能会在这个过程中丢失。

创建 workflow 正是对抗这些失败模式的方式：编排多个拥有各自上下文窗口、目标聚焦且相互隔离的 Claude subagents。

## 和其他 Agent 能力的区别

| | Subagents | Skills | Agent teams | Workflows |
| --- | --- | --- | --- | --- |
| 是什么 | Claude 派生（spawn）出来的 worker | Claude 遵循的指令（instructions） | 一个监督各 peer session 的 lead agent | 由运行时（runtime）执行的脚本 |
| 谁决定下一步运行什么 | Claude，每个 turn 逐个决定 | Claude，按照 prompt 决定 | lead agent，每个 turn 逐个决定 | 脚本 |
| 中间结果存在哪里 | Claude 的上下文窗口 | Claude 的上下文窗口 | 共享的任务列表 | 脚本变量 |
| 可重复使用的是什么 | worker 的定义 | 指令的定义 | team 的定义 | 编排（orchestration）本身 |
| 规模 | 每个 turn 转派少量任务 | 与 subagents 相同 | 少数几个长期运行的 peers | 每次运行可编排几十到几百个 agents |
| 中断时 | 该 turn 重新开始 | 该 turn 重新开始 | teammates 继续运行 | 可在同一会话中恢复 |

来源：[Claude Code 官方文档 · Workflows](https://code.claude.com/docs/en/workflows)

## 动态编排的优势

workflows可以把plan的过程放到代码里面，可以精心的把上面提到的subagents、skills、agent teams编排到工作流里面，可以精确的指定不同subagent需要用到的上下文，workflow的脚本负责编排，和保存中间过程的值，因此可以使得最终产出的上下文只有最终的答案。把这个编排的过程放到脚本里面最直观的意义就是这个过程可重复(虽然skills也是让流程可重复，但是他的约束性不如dynamic workflow，因为编排的部分都用脚本安排了)，更重要的是让不同的agents给出答案之前可以对抗性的评估彼此的运行结果，当然这里有很多常见的模式，在后面会详细描述，总的来说dynamic workflow运行出来的结果不管是从流程的编排来说还是给答案的准确率来说都会有提升。

一个workflow最适合这样的任务：任务大到单个agent的上下文装不下，或者同一个步骤需要在许多条目上重复执行。下面的prompt展示了常见的几种形态。每一个prompt都是让Claude为该任务编写并运行一个workflow——脚本不是你自己写的，而是Claude写的。

特别值得了解的一点是：dynamic workflows 可以决定每个 agent 使用哪个模型，以及 subagents 是否在各自独立的工作树（worktree）中运行——这让 Claude 能按需选择所需的智能水平和隔离程度。

## 使用 Dynamic Workflows 的常用模式

你可以直接让 Claude 创建一个 dynamic workflow 就开始使用；也可以使用触发词「ultracode」，确保 Claude Code 会创建一个 workflow。

但建立起对 dynamic workflows 工作原理的心智模型（mental model），能帮你理解什么时候该用它们，以及如何通过 prompt 引导 Claude。

下面是 Claude 构建 workflow 时可能会用到、也会组合起来使用的一些常见模式：

<figure>
  <img src="/images/agent-dynamic-workflow-patterns.png" alt="Dynamic Workflows 常见模式总览">
  <figcaption>图片转载自 Thariq (@trq212) 的<a href="https://x.com/trq212/status/2061907337154367865">推文</a>，原文见 <a href="https://claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code">Claude 官方博客</a></figcaption>
</figure>

### Classify-and-act（分类并行动）

用一个分类（classifier）agent 判断任务的类型，然后根据任务类型把任务路由（route）给不同的 agents，或选择不同的行为分支；也可以把分类器放在最后，用来决定如何输出结果。

### Fan-out-and-synthesize（扇出并汇总）

把任务拆成许多更小的步骤，每个步骤跑一个 agent，最后把这些结果汇总（synthesize）起来。当小步骤数量很大、或者每个步骤都受益于自己干净的上下文窗口（互不干扰、不交叉污染）时，这个模式尤其有用。汇总这一步是一个屏障（barrier）——它会等所有扇出的 agents 都完成，再把它们的结构化输出合并成一个结果。

### Adversarial verification（对抗性验证）

对每个派生的 agent，再派生一个独立的 agent，去对抗性地核验它的输出是否符合评分标准（rubric）或准则。

### Generate-and-filter（生成并筛选）

围绕一个主题生成一批想法，再按评分标准或核验结果进行筛选，去重（dedupe）后只返回质量最高、经过检验的想法。

### Tournament（锦标赛）

不是分工，而是让 agents 同台竞争：派生 N 个 agents，各自用不同的方法尝试同一个任务，再由 prompt 或模型用评判 agent 进行两两对决（pairwise）式的评判，直到产生赢家。

### Loop until done（循环直到完成）

对于工作量未知的任务，循环派生 agents，直到满足某个停止条件（比如没有新的发现、或日志里不再有错误）为止，而不是固定跑固定轮数。

## 如何触发 Dynamic Workflows

在Claude code中使用dynamic workflow给了两种方式：1：使用prompt触发，比如输入：ultracode: audit every API endpoint under src/routes/ for missing auth checks 其中有ultracode或者workflows的这两个关键字的时候会自动触发，2：使用/effort ultracode

## 工作流脚本解析

workflows会保存在：.claude/workflows/下面，大概长这样：

```ts
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

### 字段含义

简单解释一下这段脚本里各字段的含义：

- **meta**：工作流的元信息。`name` 是工作流的名字，`description` 是用途描述——Claude Code 会根据它们识别这份工作流（比如通过名字或描述里的关键词触发，对应前面说的 `ultracode` / `workflows` 关键字）。
- **agent(...)**：派生一个 subagent 执行任务，第一个参数是给它下达的任务描述。
- **schema**：要求 subagent 按给定的 JSON Schema 返回结构化结果，而不是自由文本——这里要求返回 `{ files: string[] }`，结果直接存入 `found` 变量，供后面的脚本使用。
- **found.files**：第一个 subagent 的结构化输出。注意它只是"脚本变量"里的中间结果，不会塞回上下文窗口（对应前面表格里"中间结果存在哪里"一列）。
- **pipeline(...)**：把一组输入逐个送过同一道工序——这里把 `found.files` 里的每个文件交给一个新的 subagent 去审计，它们并行执行互不阻塞。
- **label**：给 subagent 起个标签，方便在运行界面里区分各个并行任务。
- **filter(Boolean)**：过滤掉执行失败的 subagent 返回的 `null`（agent 失败时是 `null`），只保留成功的结果——这就是最终产出。

## Dynamic Workflows 的限制

这是claude code的dynamic workflows的限制：

| 限制 | 原因 |
| --- | --- |
| 运行中途不能接受用户输入 | 只有 agent 的权限提示（permission prompts）才能暂停运行。如果需要在阶段之间做人工确认（sign-off），就让每个阶段各自作为一个独立的工作流运行 |
| workflow 自身不能直接访问文件系统或执行 shell 命令 | 文件的读写和命令执行都由 agents 完成，脚本只负责协调（coordinate）agents |
| 不支持加载模块：脚本里包含 `import()` 会在运行开始前就失败 | 脚本主体是纯 JavaScript。需要用到第三方库的工作，请放进 agent 的任务里执行 |
| 最多 16 个并发 agents；当 Claude Code 可用的 CPU 较少时数量会更少（包括在受 CPU 限制的容器内） | 限制本地资源的使用 |
| 在 fan-out（扇出）场景中，与第一个 agent 共享 prompt-cache 前缀的 agents，默认会在第一个 agent 启动后最多 5 秒才开始启动 | 除了第一个 agent 外，其余 agent 都直接读取第一个 agent 缓存好的前缀，而不是各自对未缓存的内容重新处理一遍 |
| 每次运行最多总共 1,000 个 agents | 防止失控的循环（runaway loops） |

## References

[1] Anthropic. Claude Code 官方文档 · Workflows. [code.claude.com/docs/en/workflows](https://code.claude.com/docs/en/workflows)

[2] Anthropic. A Harness for Every Task: Dynamic Workflows in Claude Code. [claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code](https://claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code)

[3] Thariq (@trq212). A Harness for Every Task: Dynamic Workflows in Claude Code. [x.com/trq212/status/2061907337154367865](https://x.com/trq212/status/2061907337154367865)