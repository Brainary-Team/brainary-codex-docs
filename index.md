---
title: PoA 开发文档
description: 用普通 JavaScript 程序去驱动一批 AI agent —— 机制、写法、API 参考与边界
---

# PoA 开发文档

**PoA（Program of Agent）是一条用普通 JavaScript 驱动一批 AI agent 的通道：由程序调度 AI，而不是由 AI 调度工具。**

通常的 agent 系统里，模型是决策者：它决定下一步调哪个工具、读哪个文件、什么时候收手。PoA 把这个关系倒过来——**调度、分发、汇总、判定全部写在程序代码里**，模型退到每个子 agent 内部，只负责那些"必须读懂内容才能回答"的部分。

于是编排逻辑变成确定的：`for` 就是循环，`Promise.all` 就是并发，`if` 就是分支。这段编排**不消耗任何模型调用，每次运行行为一致**。

```js
// 一个完整的 PoA 程序：列出目录 → 每个派一个 agent → 收齐 → 汇总
const folders = await shellLines("ls -d */ | sed 's#/$##'", {
  validate: (line) => SAFE_NAME.test(line),
});

const batch = await runBatch(
  folders.map((folder, i) => ({ message: prompt(folder), name: `scan_${i}`, meta: { folder } })),
  { concurrency: 3 },
);

text(JSON.stringify(batch.map(({ handle, reply }) => ({
  folder: handle.meta.folder,
  summary: parseJsonReply(reply).value,
}))));
```

---

## 从哪里开始

| 场景 | 从这里进 |
| --- | --- |
| 初次接触 PoA，需要了解它解决什么问题 | [概览](./01-overview.md) → [快速开始](./02-quickstart.md) |
| 环境已就绪，需要马上跑通 | [快速开始](./02-quickstart.md) |
| 要动手写一个新程序 | [编写指南](./05-writing.md)，遇到具体形状去 [模式库](./06-patterns.md) |
| 查某个函数收什么参数 | [API 参考](./07-api-reference.md) |
| 程序行为不符合预期 | [故障排查](./09-troubleshooting.md) |
| 判断某件事能不能做 | [边界与限制](./08-limits.md) |

---

## 文档目录

| # | 文档 | 内容 |
| --- | --- | --- |
| 01 | [概览](./01-overview.md) | PoA 是什么、解决什么问题、它怎么接进 codex（code mode）、什么时候该用与不该用 |
| 02 | [快速开始](./02-quickstart.md) | 环境准备 → 探针 → 跑通第一个程序 → 写出自己的第一个 |
| 03 | [核心概念](./03-concepts.md) | 三个角色、cell、沙箱面、prelude 与内置的分界、两代后端 |
| 04 | [工作原理](./04-how-it-works.md) | code mode 是什么、`tools.foo()` 发生了什么、为什么并发不等于流式 |
| 05 | [编写指南](./05-writing.md) | **主体文档。** 从空文件到能跑的完整动线，含四处必写的容错与调试习惯 |
| 06 | [模式库](./06-patterns.md) | 四种可复用形状 + 一个负结果 + 六条反模式 |
| 07 | [API 参考](./07-api-reference.md) | 12 个全局 primitive、12 个 prelude primitive、32 个内置工具的完整声明 |
| 08 | [边界与限制](./08-limits.md) | 硬边界清单，以及每一条在写代码时的具体样子 |
| 09 | [故障排查](./09-troubleshooting.md) | 症状 → 原因 → 处置；两个**不报错**的坑单独标出 |

---

## 术语

这几个词全文反复出现，先对齐。

| 术语 | 含义 |
| --- | --- |
| **PoA** | Program of Agent，"驱动 agent 的程序"。既指开发者编写的那段 JavaScript，也指这整条通道 |
| **cell** | 一次提交的那整段 JS 的一次运行。一次提交 = 一个 cell = 一次跑完 |
| **code mode** | codex 的一种工具投送方式：所有工具挂在 JS 全局对象 `tools` 上，由沙箱里的 JS 调用。PoA 走的就是这条路 |
| **子 agent** | 被 PoA 程序派出去的独立 AI 会话，有自己的上下文和工具 |
| **handle** | 派出一个子 agent 后拿到的句柄，后续等待、追问、关闭都靠它 |
| **并行派发**（扇出 / fan-out） | 一次派出 N 个子 agent 并行干活的形状 |
| **收口**（fan-in） | 把派出去的结果收齐、解析、合并的那一步 |
| **prelude** | `workflow-demos/lib/prelude.js`，一层自备的编排 primitive，运行前被拼在程序代码前面 |
| **后端 v1 / v2** | codex 的两代多 agent 工具，接口形状完全不同。prelude 存在的理由就是抹平这个差异 |
| **profile** | `workflow-demos/config/` 下的一份配置模板，决定用哪代后端 |

---

## 两条最容易踩的线

这两条在正文里各有详述，但值得提前知道，因为它们**都不报错**：

> [!WARNING]
> **① 忘了 `await` 的 promise 会被静默丢弃。** 程序求值一结束沙箱就没了，没等的活直接消失，不报错、不警告，只是那部分工作没有发生。
>
> **② shell 的报错信息和正常输出走同一个通道。** 不过滤的话，`No such file or directory` 会被当成数据喂给子 agent，而 **agent 会认认真真地去分析它**，最终产出一份看起来很正常的错误结论。

---

## 这份文档的范围

**覆盖**：PoA 程序的写法、可用的 API、可观察的运行机制、当前的能力边界。

**不覆盖**：codex 内核如何实现这条通道（合成工具调用、Rust 侧的分发链路）、code mode 沙箱本身的实现、子 agent 内部的模型行为。

文档内容基于 brainary fork 当前的代码与 `workflow-demos/` 的实跑记录。**工具面会随模型、provider 与配置变化**——任何时候以[探针](./02-quickstart.md#1-第一步跑探针)的输出为准，而不是以本文的表格为准。
