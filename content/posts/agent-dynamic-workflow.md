+++
title = 'Dynamic Workflow 介绍'
date = 2026-08-27T16:53:28+08:00
draft = false
+++

Anthropic 在 Claude Code v2.1.154 版本之后发布了 **Dynamic Workflows（动态工作流）** 特性。简单来说，这个特性让"流程"本身变成一段可以被动态编排的脚本：一个 Dynamic Workflow 脚本会编排（orchestrate）很多个 subagent，让它们**并行**处理各自负责的事情；同时，背后有一个运行时（runtime）在后台执行这段脚本，替你把整个任务完成。