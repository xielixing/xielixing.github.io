+++
title = 'Dynamic Workflow 介绍'
date = 2026-08-27T16:53:28+08:00
draft = false
+++

Anthropic 在 Claude Code v2.1.154 版本之后发布了Dynamic Workflows的特性，一个Dynamic Workflow脚本会编排很多subagents去并行的处理事情，有一个运行时会去后台执行这段脚本来完成人类的任务。

使用Dynamic Workflow的场景通常是你需要的任务过于复杂，需要很多个subagents协同工作才可以完成，比如你需要对一个超大的代码仓进行迁移，开放性研究问题需要交叉验证等