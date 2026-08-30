# PoA 开发文档

PoA（Program of Agent）是 brainary 提供的一条通道：用普通 JavaScript 驱动一批 AI agent，由程序调度 AI，而不是由 AI 调度工具。

通常的 agent 系统里，模型是决策者，它决定下一步调哪个工具、读哪个文件、什么时候收手。PoA 把这个关系倒过来：调度、分发、汇总、判定全部写在程序代码里，模型退到每个子 agent 内部，只负责那些必须读懂内容才能回答的部分。

于是编排逻辑变成确定的：`for` 就是循环，`Promise.all` 就是并发，`if` 就是分支。这段编排不消耗任何模型调用，每次运行行为一致。

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
| 初次接触 PoA，需要了解它解决什么问题 | 概览 → 快速开始 |
| 环境已就绪，需要马上跑通 | 快速开始 |
| 要动手写一个新程序 | 编写指南，遇到具体形状去 模式库 |
| 要把程序打成包交给别人跑（含自带 MCP） | 核心概念 §9 → 编写指南 §13 |
| 查某个函数收什么参数 | API 参考 |
| 程序行为不符合预期 | 故障排查 |
| 判断某件事能不能做 | 边界与限制 |

---

## 文档目录

| # | 文档 | 内容 |
| --- | --- | --- |
| 01 | 概览 | PoA 是什么、解决什么问题、brainary 把它接在 codex 的哪一层、什么时候该用与不该用 |
| 02 | 快速开始 | 环境准备 → 探针 → 跑通第一个程序 → 写出自己的第一个 |
| 03 | 核心概念 | 三个角色、cell、沙箱面、prelude 与内置的分界、两代后端、`.poa` 包与自带 MCP |
| 04 | 工作原理 | code mode 是什么、`tools.foo()` 发生了什么、为什么并发不等于流式 |
| 05 | 编写指南 | 主体文档。从空文件到能跑的完整动线，含四处必写的容错与调试习惯、打成 `.poa` 包时要改的五件事 |
| 06 | 模式库 | 四种可复用形状 + 一个负结果 + 八条反模式 + 五个能力样本 demo |
| 07 | API 参考 | 12 个全局 primitive、16 个 prelude primitive、32 个内置工具的完整声明 |
| 08 | 边界与限制 | 硬边界清单，以及每一条在写代码时的具体样子 |
| 09 | 故障排查 | 症状 → 原因 → 处置；两个不报错的坑单独标出 |

---

## 术语

这几个词全文反复出现，先对齐。

| 术语 | 含义 |
| --- | --- |
| **brainary** | 本项目。它建在 codex 之上：codex 提供 V8 沙箱、工具面与子 agent，brainary 提供 PoA 这条通道，让模型以外的东西也能提交那段 JS，以及 `.poa` 包格式与包自带 MCP |
| **codex** | brainary 所基于的那套 agent 运行时。文中凡是说"codex 内置""codex 的某某"，指的都是这一层已有的东西，不是 brainary 加的 |
| **PoA** | Program of Agent，"驱动 agent 的程序"。既指开发者编写的那段 JavaScript，也指这整条通道 |
| **`.poa` 包** | brainary 定义的分发形态：一个 zip，根上放 `manifest.toml`，里面是程序本体和它自带的 MCP server。提交接口只收这一种形状，只持有一段 JS 时同样封装成一个"申报零能力"的包 |
| **manifest** | 包里的 `manifest.toml`：`[poa]` 元数据 + `[[capabilities.mcp]]` 能力申报。申报了的 server 起不来就拒跑 |
| **cell** | 一次提交的那整段 JS 的一次运行。一次提交 = 一个 cell = 一次跑完 |
| **code mode** | codex 已有的一种工具投送方式：所有工具挂在 JS 全局对象 `tools` 上，由沙箱里的 JS 调用。PoA 借的就是这条路 |
| **子 agent** | 被 PoA 程序派出去的独立 AI 会话，有自己的上下文和工具 |
| **handle** | 派出一个子 agent 后拿到的句柄，后续等待、追问、关闭都靠它 |
| **并行派发**（扇出 / fan-out） | 一次派出 N 个子 agent 并行干活的形状 |
| **收口**（fan-in） | 把派出去的结果收齐、解析、合并的那一步 |
| **prelude** | `workflow-demos/lib/prelude.js`，一层自备的编排 primitive，运行前被拼在程序代码前面 |
| **后端 v1 / v2** | codex 的两代多 agent 工具，接口形状完全不同。prelude 存在的理由就是抹平这个差异 |
| **profile** | `workflow-demos/config/` 下的一份配置模板，决定用哪代后端 |

---

## 两条最容易踩的线

这两条正文里各有详述，但值得提前知道，因为它们都不报错：

> [!WARNING]
> **① 忘了 `await` 的 promise 会被静默丢弃。** 程序求值一结束沙箱就没了，没等的活直接消失，不报错、不警告，只是那部分工作没有发生。
>
> **② shell 的报错信息和正常输出走同一个通道。** 不过滤的话，`No such file or directory` 会被当成数据喂给子 agent，而子 agent 会照着它认真分析一通，最终产出一份看起来很正常的错误结论。

---

## 这份文档的范围

**覆盖**：PoA 程序的写法、可用的 API、可观察的运行机制、当前的能力边界。

**不覆盖**：codex 内核如何实现这条通道（合成工具调用、Rust 侧的分发链路）、code mode 沙箱本身的实现、子 agent 内部的模型行为。

全文描述的是 brainary `main` 分支 `56a8b1307`（2026-08-20）时的实现。代码与文档之间只有这一个锚点，读到与实际行为不符的地方，以代码为准。

---

## 怎么读这份文档

全文混排着两类内容，稳定性完全不同，读之前先分清：

| | 是什么 | 变了意味着什么 | 怎么标记 |
| --- | --- | --- | --- |
| **契约** | 接口与格式：`manifest.toml` 的字段、`{ threadId, package }`、尺寸上限、`0644`、拒跑判据、审批与并行的判定顺序 | 那是一次 breaking change | 不标记，默认即是 |
| **快照** | 某一次实测的结果：工具数量、profile 表、曝光度档位、并发名额、prelude 的函数名单 | 什么都没变，只是换了一套模型 / provider / 配置 | 就近标出 **⚠ 快照** |

> 别照着"快照"写死代码。那类表只说明大概会看到什么形状，
> 现在实际有什么只有探针答得了，换模型、换 provider、换 profile 之后都要重跑。
