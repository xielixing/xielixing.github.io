+++
title = 'Dynamic Workflow 介绍'
date = 2026-08-27T16:53:28+08:00
draft = false
+++

## 什么是 Dynamic Workflows

Anthropic 在 Claude Code v2.1.154 版本之后发布了Dynamic Workflows的特性，一个Dynamic Workflow脚本会编排很多subagents去并行的处理事情，有一个运行时会去后台执行这段脚本来完成人类的任务。

## 什么时候需要 Dynamic Workflows

使用Dynamic Workflow的场景通常是你需要的任务过于复杂，需要很多个subagents协同工作才可以完成，比如你需要对一个超大的代码仓进行迁移，开放性研究问题需要交叉验证等

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