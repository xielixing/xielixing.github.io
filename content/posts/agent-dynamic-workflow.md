+++
title = 'Dynamic Workflow 介绍'
date = 2026-08-27T16:53:28+08:00
draft = false
+++

本文介绍 Claude Code 的 **Dynamic Workflows（动态工作流）** 特性。它是 Anthropic 在 Claude Code v2.1.154 版本之后发布的能力：Agent 不再只能按预先写死的固定流程执行，而是可以在运行时根据当前情况，动态地决定和调整接下来的处理步骤。需要说明的是，要使用这一特性，Claude Code 的版本需要不低于 v2.1.154（官方原文：*Dynamic workflows require Claude Code v2.1.154*）。