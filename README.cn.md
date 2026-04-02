**中文版** | [English](README.md)

# 代码外科分析目录

这个仓库用于发布双语代码外科分析报告，聚焦 agent 系统及其相邻运行时。每份报告都强调具体执行路径，而不是停留在高层印象或功能清单。

## 仓库内容

- 以中英文 README 作为稳定首页入口。
- 所有正式发布的报告都放在 `reports/` 下，并且以英文文件 + 中文文件成对出现。
- 统一的分析框架：范围、正常路径、状态/调度映射、副作用、故障恢复，以及可复制性判断。

## 方法论

每份报告都应回答同一组核心问题：

- 精确分析的是哪个子系统，哪些内容被有意排除？
- 一个真实 turn 或任务在正常路径下是如何流动的？
- 实际的控制中心分别在哪里处理调度、状态、工具、上下文和持久化？
- 哪些机制适合直接复制、深度内化，或者仅作为学习对象？

## 发布约定

- 每份正式分析都必须同时提供 EN + CN。
- 根目录 README 双文件始终作为仓库主页。
- 新分析通过在 `reports/` 下新增一组 slug 文件，并在下面的目录表中追加一行来接入。
- 英文与中文是对等发布物，不是“以后再补”的附属版本。

## 报告目录

| 系统 | 聚焦范围 | 日期 | English | 中文 |
|---|---|---|---|---|
| Claude Code | Agent 运行时与工具循环架构 | 2026-04-01 | [EN](reports/claude-code-agent-runtime-surgical-analysis.md) | [中文](reports/claude-code-agent-runtime-surgical-analysis.cn.md) |
| OpenClaw | 面向 local-first 到 SaaS shell 的核心 agent 运行时 | 2026-04-01 | [EN](reports/openclaw-core-agent-runtime-surgical-analysis.md) | [中文](reports/openclaw-core-agent-runtime-surgical-analysis.cn.md) |

## 说明

- Claude Code 报告保留了仓库原先“单 README 发布”的主体内容，并迁移到 `reports/` 下。
- OpenClaw 报告来自比较工作区中的研究产物，并在此仓库中以稳定 slug 重新发布。
- 这个仓库是纯文档仓库，不声明构建、测试或代码生成工作流。
