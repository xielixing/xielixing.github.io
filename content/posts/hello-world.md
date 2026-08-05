+++
title = '在 BitFun 里嵌入 LoopX：让 Agent 持续修完一个仓库的 Issue'
date = 2026-08-05T14:30:00+08:00
draft = false
summary = '用 BitFun × LoopX 的真实集成过程讲清 LoopX 的设计哲学：State Kernel 与 goal 的关系、heartbeat / monitor / gate / quota 这些概念如何协同、GitHub 上的人工操作如何回流成状态收敛，以及 heartbeat 对模型 API 限流的缓解作用。附一夜产出 16 个 PR 的实战数据与四个踩坑记录。'
tags = ['loopx', 'agent', 'bitfun', 'state-kernel', 'automation']
categories = ['Engineering']
+++

上周我给 [BitFun](https://github.com/GCWing/BitFun)（一个 Tauri 桌面端的 AI 编码工具）接入了 [LoopX](https://github.com/huangruiteng/loopx)，目标是一个听起来很朴素的功能：**在面板里勾选一批 issue，点一下启动，然后就不用管了**。

实际效果：勾选 28 个 issue 后启动，一夜产出 **16 个 PR**，第二天累计 **26 个**；每个 issue 独立 worktree、独立分支、带验证；判定"不需要写代码"的 issue 走了评论+关闭路线；需要人类权限的动作（发评论、关 issue、merge）全部停下来等我批准。整个过程消耗了配额预算的 1.5%（22/1440 个 slot）。期间应用崩过、重启过、状态清过——循环每次都能从断点继续。

这篇文章用这个集成过程，把 LoopX 的设计哲学讲清楚。

## 一、LoopX 不是 Agent，goal 也不是 Prompt

先厘清最容易混淆的两个概念。

**朴素做法**是把目标塞进一个长对话："帮我修完这个仓库的所有 issue"，然后让 agent 一直跑。问题所有人都遇到过：上下文越滚越长、结论开始漂移、进程一断全部归零、模型把三轮前自己的猜测当成事实。**对话是最糟糕的持久化状态**。

LoopX 的回答是把这两样东西拆开：

- **LoopX 是一个 State Kernel（状态内核）**——它不写代码、不调模型，甚至"没有编码能力"。它管理的是控制面事实：现在有哪些目标、哪些工作项、谁认领了、边界在哪、配额还剩多少、哪一步需要人类批准。
- **goal 是内核里的一个对象**——注册在项目的 `.loopx/registry.json` 里，持久状态写在 `ACTIVE_GOAL_STATE.md`。我的 goal 陈述是一句英文：

> Continuously repair repository issues explicitly selected in BitFun, keep each issue isolated, validate every change, and surface authority gates for human decisions.

所以"LoopX 和 goal 的区别"类似操作系统和进程的区别：goal 是数据，LoopX 是机制。LoopX 作者有一个关键论断：**原生能力是单 issue 的全生命周期（选题→复现→修复→发 PR→跟 CI/review→收口）；把 goal 设成"持续修 issue"，多 issue 就自然涌现了**——每个 issue 是一个独立的一次性工作项，串成链；不需要为"批量"写任何专门代码。实测确实如此：28 个 issue 就是 28 个 todo，Kernel 一次调度一个，做完解锁下一个。

BitFun 原本有自己的 Thread Goal 机制（把目标绑在会话上），第一版集成试图把 LoopX 的决策映射到 Thread Goal 上——后来整个删掉了。原因就是上面这条哲学：**不要建第二套状态机**。状态只能有一个主人。

## 二、四层分工

LoopX 的设计文档里有一句总纲：

> Kernel 管控制权，Domain State 管领域连续性，Capability 管翻译，Provider 管可替换的外部 I/O；任何 projection 都只负责展示。

| 层 | 回答的问题 | 例子 | 不拥有什么 |
| --- | --- | --- | --- |
| State Kernel | 谁现在能在什么边界内做哪一步？ | goal、todo、认领/租约、配额、user gate、运行历史 | 不懂 `CHANGES_REQUESTED` 是什么意思 |
| Domain State | 这个 issue/PR 在领域上被观察成了什么？ | `feasibility.jsonl`、`pr-lifecycle.jsonl`（紧凑观察+指纹） | 不能自己获得写仓库、发消息的权限 |
| Capability | 领域事实如何翻译成通用工作？ | 把"CI 挂了"翻译成一个修复 todo | 不做调度决定 |
| Provider | 可选的外部 I/O 由谁实现？ | 消息通知 sink、记忆服务 | 缺席时 capability 仍然成立 |

这个分层最反直觉但最重要的推论是：**GitHub 才是外部权威事实源，LoopX 保存的只是"有出处的紧凑观察"**。PR 是否 merge 了，以 GitHub 为准；LoopX 只记"我上次看到它是什么状态、指纹是多少"。看板、面板全是只读投影，不允许从 UI 反向修改事实。

## 三、BitFun 怎么当 Host

LoopX 自己不会动：它需要一个 host 提供三样东西——**编码能力**（真正会写代码的 agent）、**心跳**（周期性唤醒）、**人机界面**（把 gate 和进度投影给人）。BitFun 的集成就是补齐这三样：

```text
┌─ BitFun (host) ─────────────────────────────────┐
│  Cron 服务: 每 10 分钟唤醒一次 (heartbeat)        │
│    └→ 向一个 Agent 会话注入: 英文 host 前导语     │
│        + `loopx heartbeat-prompt --compact` 契约  │
│  Issue-Fix 面板: 只读投影 + 类型化命令            │
│    (勾选 issue / 回答 gate / 停止)                │
│  Agent 会话: 真正读代码、写补丁、跑验证、发 PR    │
└──────────────────┬──────────────────────────────┘
                   │  只通过 CLI 交互 (JSON)
┌─ LoopX (kernel) ─▼──────────────────────────────┐
│  goal / todo / quota / gate / monitor / 领域账本 │
└─────────────────────────────────────────────────┘
```

解耦是刻意保持的，几条硬边界：

- BitFun 对 LoopX 内部**唯一**的直接读取是 `registry.json` 里的身份（goal id、agent id）；其余一切读写都走 `loopx --format json <命令>`；
- 没有改过 LoopX 一行源码。连 Windows 中文环境的编码坑（LoopX 的 subprocess 按 GBK 解码 UTF-8 输出）都是用 `PYTHONUTF8=1` 环境变量在 host 侧解决的；
- 用户勾选 issue 后，BitFun 做的仅仅是把每个 issue 物化成一个 LoopX todo（`action_kind=issue_fix_intake`），从此队列归 Kernel 所有——面板刷新时从 Kernel 重建视图，BitFun 自己不存任何 issue 队列。

## 四、一次心跳的解剖

核心概念都藏在一拍心跳的流程里，顺着走一遍就全认识了。

**heartbeat（心跳）**：host 每 10 分钟把同一段 prompt 注入 agent 会话。prompt 分两截——BitFun 写的英文前导语（你是谁、host 环境有什么、规矩是什么），加上 `loopx heartbeat-prompt --compact` 生成的通用生命周期契约。心跳哲学的核心一句话：**每拍从 Kernel 的 source state 重放，而不是从上一段对话继续猜**。前导语里有一条对应的硬规则："LoopX CLI 是唯一事实源，无视本对话早前与当前状态冲突的结论"。

**quota should-run（守卫）**：agent 醒来第一件事是问 Kernel"这一拍我该不该干活、干什么"。返回的判定可能是：

- `should_run=true` + 选中一个 todo → 做一个**有界片段**（比如把 #1318 从复现推进到发出 PR），验证、写回、记账一个 slot，然后**主动停下等下一拍**；
- `state=operator_gate` → 有 gate 等人回答，本拍不做实现工作；
- 到期的 monitor → 只做一次只读观察轮询；
- 什么都没有 → 安静退出，不消耗任何配额。

**todo（工作项）**：只有五类——`advancement_task`（推进）、`continuous_monitor`（监控）、`user_gate`（需人类决策）、`user_action`（需人类操作）、`blocker`。issue 修复、CI 修复、review 整改，全部是普通 advancement todo，Kernel 不需要懂 GitHub。串行由 `resume_when: todo_done:<前一个>` 实现。

**gate（权限门）**：agent 判断"这一步需要人类权威"（发公开评论、关别人的 issue、merge）时，不是在聊天里问一句，而是创建一个**类型化的 user_gate todo**，并声明它阻塞哪个工作项。BitFun 面板把 gate 投影成卡片（批准/拒绝/取消 + 理由），回答通过 `todo complete --decision-outcome approve` 写回。关键设计：**聊天回复永远不构成授权**，授权必须是控制面里的类型化事实。批准过的权限沉淀为持续 authority，同类操作不再重复问。

**monitor（监控）**：PR 发出去之后进入监控。这里有个精妙设计——**不是一个 PR 一个监控**，而是按"仓库 × 生命周期桶"聚合（checks_pending 一个桶、changes_requested 一个桶……），Kernel 只为非空的桶调度监控；观察结果带指纹，无变化的轮询不写状态、不算交付。26 个 PR 只需要几个监控 todo，避免了监控风暴。

## 五、GitHub 上的操作如何回流

这是我觉得整套设计里最优雅的部分。我在 GitHub 上 merge 一个 PR 之后，**不需要告诉 LoopX 任何事**。同一个"PR 已合并"会以三种身份存在于三层：

```text
GitHub:  PR state = MERGED          ← 外部权威事实
   ↓ (下一拍 monitor 回读)
Domain State: 观察 + 指纹 + 显式 issue 关联   ← 紧凑领域连续性
   ↓ (capability 翻译)
Kernel: monitor 完成 + terminal transition + 后继恢复  ← 控制面收敛
```

各种人工操作的回流路径：

| 你在 GitHub 上做了什么 | 下一拍 LoopX 发生什么 |
| --- | --- |
| merge 了一个 PR | monitor 观察到 MERGED → 终局收尾：监控 todo 完成、关联 issue 关闭、"待你处理"里对应项消失 |
| PR 的 CI 挂了 | `checks_failed` 桶非空 → 生成一个修 CI 的 advancement todo，agent 下拍去修 |
| 留了 change request | 生成 review 整改 todo；agent 修完推送后回到"等待 re-review"的监控 |
| 关闭了 PR（不合并） | 终局收尾，记录结构化的 no-follow-up |
| 什么都没做 | 指纹无变化，安静轮询，零状态写入 |

缺一层就 fail closed：GitHub 已 merge 但没有显式 issue 关联时**不猜**；面板显示什么状态都不构成事实。这就是为什么进程崩溃、应用重启、甚至我把缓存状态全清了重来，循环都能继续——**一切都能从三层状态重新推导**。

## 六、这一路真实跑过的 LoopX 命令

| 命令 | 谁调用 | 作用 | 一个值得记的细节 |
| --- | --- | --- | --- |
| `loopx bootstrap` | 一次性 | 创建 goal、registry、初始 ACTIVE_GOAL_STATE.md | |
| `loopx todo add --role agent --task-class advancement_task --action-kind issue_fix_intake ...` | BitFun（点启动时） | 把勾选的 issue 物化为 Kernel todo | 按文本去重，重复启动幂等 |
| `loopx todo list` | BitFun（面板 30s 轮询） | 读全部 todo，重建面板投影 | **零写入**，所以才敢拿来轮询 |
| `loopx quota should-run --available-capability ...` | agent 每拍 + BitFun 手动刷新 | 该不该跑、跑哪个、gate 状态 | **每次调用都追加一条 rollout event**——UI 高频轮询绝不能用它，这是我们踩过的坑 |
| `loopx heartbeat-prompt --compact` | BitFun（启动/答复 gate 时） | 生成心跳契约文本 | 纯生成、无副作用；`--thin` 档依赖 host 没有的技能包，所以选 `--compact` |
| `loopx todo claim` | agent | 认领工作项（软路由，不是锁） | |
| `loopx todo complete --role user --decision-outcome approve\|reject\|cancel` | BitFun（gate 卡片提交） | 把人类决策写成类型化事实 | 只有 approve 消耗权威 |
| `loopx quota spend-slot --execute` | agent（交付验证写回后） | 记账一个配额槽 | 安静拍、失败拍不记账 |
| `loopx refresh-state` | agent | 把进度/验证/下一步写回 active state | |
| `loopx evidence-log --thin` | agent | 回看最近的证据流水 | |
| `loopx check` | agent | 检查 adapter/连通性 | |

## 七、heartbeat 能缓解模型 API 限流吗？

可以，而且缓解的方式比"限速"更本质。持续 agent 循环对 API 的需求模式是**突发且不可中断的**：一旦开跑就要连续高频调用，被限流打断就等于任务失败。heartbeat 把它改造成了**离散、有界、可中断**的模式：

1. **削峰**：每 10 分钟一拍、单飞不重叠（上一拍没结束下一拍自动合并跳过）、每拍只做一个有界片段——API 消耗从突发洪峰摊平成低稳态速率。28 个 issue 一夜修完，靠的不是并发轰炸而是细水长流。
2. **预算化**：quota slot（这次是 1440 个的预算池）让消耗可审计、可截断；安静拍（`should_run=false`）几乎零消耗——夜里大量心跳其实只是问一句"有事吗"就睡了。
3. **限流从灾难降级为延迟**：这是最关键的一条。如果某一拍中途被 429 打死，损失只有这一拍——下一拍从 Kernel 状态重放，已完成的验证和写回都在。**任务对 API 的依赖从"必须连续可用"弱化成"间歇可用即可"**，这恰好是限流环境下唯一靠得住的假设。

要诚实的是它不是硬限流器：单拍内部 agent 依然可能几十轮调用，瞬时速率仍需 provider 层兜底。heartbeat 解决的是**时间维度的形状问题**——把长任务从"对高速 API 的持续依赖"改写成"对低速 API 的间歇依赖"。

## 八、踩过的四个坑

真实运行两天，每个坑都换来一条设计教训：

1. **Agent 把宿主杀了**。它在 worktree 里编译时遇到文件锁，跑了 `Stop-Process -Force` 清理"陈旧进程"——被杀的 PID 里有承载它自己的 BitFun。教训：host 前导语必须声明宿主边界（"不许杀死非本回合启动的进程"），而且要用通用表述，不点名任何具体工具。
2. **Gate 静默不可见**。gate 投影最初依赖 quota 摘要里被截断到 2 条的预览数组，且要求 gate 显式关联到受管 todo——agent 建了个没关联的 gate，面板上永远看不到，循环空转。教训：**开着的 gate 永远不能不可见**，投影必须从完整的 todo list 枚举并做兜底。
3. **PR 被特性分支污染**。agent 从 host checkout 的当前分支（而不是 origin/main）开修复分支，三个 PR 各自带上了 8000 行无关改动。教训：契约里必须钉死基分支——"从远端默认分支开分支，永远不基 host 的 HEAD"。
4. **读操作有写副作用**。`quota should-run` 每次调用都追加事件日志，拿它做 UI 轮询会无界污染运行历史。教训：轮询只能用真正只读的命令（`todo list`），每个"读"命令都要先验证它的写语义。

第 1、3 条的修复还引出一条元原则：这套东西是**面向任意仓库的通用 host**，所有仓库特定的知识（工具链、验证命令）都不进 prompt，而是写进每个 goal 自己的 active state——LoopX 契约原文就是这么要求的："Keep project-specific branching out of the automation prompt."

## 九、结语

回头看，这次集成里 BitFun 写的代码其实不多：一个 CLI 桥、一个每 10 分钟的定时器、一个只读面板。真正的能力来自分层本身——**控制权、领域观察、翻译、外部事实各归其位之后，"持续修完一个仓库的 issue"就只是一个 goal 陈述，而不是一个需要专门开发的功能**。

而人的位置也很清楚：agent 修不动的它自己会记 blocker；能修的它推进到 PR；**merge 与否、公开写与否，权威始终在人手里**——你在 GitHub 上点下的每个按钮，都会在下一拍变成状态机里的一次干净收敛。

（本文描述的集成代码见 BitFun 的 [PR #2006](https://github.com/GCWing/BitFun/pull/2006)；LoopX 的设计哲学原文见其仓库的 [state-kernel-domain-state-case-study](https://github.com/huangruiteng/loopx/blob/main/docs/capabilities/issue-fix/state-kernel-domain-state-case-study.zh-CN.md)。）
