# PoA 开发文档

PoA（Program of Agent）是 brainary 提供的一条通道：用普通 JavaScript 驱动一批 AI agent，由程序调度 AI，而不是由 AI 调度工具。

通常的 agent 系统里，模型是决策者，它决定下一步调哪个工具、读哪个文件、什么时候收手。

PoA 把这个关系倒过来：调度、分发、汇总、判定全部写在程序代码里，模型退到每个子 agent 内部，只负责那些必须读懂内容才能回答的部分。

于是编排逻辑变成确定的：`for` 就是循环，`Promise.all` 就是并发，`if` 就是分支。这段编排不消耗任何模型调用，每次运行行为一致。

```js
// 列出目录 → 每个派一个 agent → 收齐 → 汇总
const prompt = (folder) => `分析 ./${folder}，只回一行 JSON：{"purpose":"..."}`;

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

> `shellLines` / `runBatch` / `parseJsonReply` 不是内置的，是一层普通 JS 封装。全文在《07-api-reference.md》§2.5，要抄进程序才有；不抄的等价写法见《05-writing.md》§12.2。

---

## 文档目录

| # | 文档 | 文件 | 内容 |
| --- | --- | --- | --- |
| 01 | 概览 | 《01-overview.md》| PoA 是什么、解决什么问题、与自建编排框架的分工、brainary 把它接在 codex 的哪一层、程序的典型形状 |
| 02 | 快速开始 | 《02-quickstart.md》| 取得二进制 → 写 `config.toml` → 跑探针包 → 接上第一个 agent → 打开可选工具组 |
| 03 | 核心概念 | 《03-concepts.md》| 四节：三个角色与 cell、prelude 与内置的分界、沙箱里有什么、怎么确定是哪一代后端 |
| 04 | 工作原理 | 《04-how-it-works.md》| code mode 是什么、`tools.foo()` 发生了什么、为什么并发不等于流式 |
| 05 | 编写指南 | 《05-writing.md》| 主体文档。从空文件到能跑的完整动线，含四处必写的容错与调试习惯、抄与不抄 prelude 的两版完整范例 |
| 06 | 模式库 | 《06-patterns.md》| 四种可复用形状 + 一个负结果 + 十条反模式 |
| 07 | API 参考 | 《07-api-reference.md》| 12 个全局 primitive、16 个 prelude primitive（含可直接抄的全文）、32 个内置工具的完整声明 |
| 08 | 边界与限制 | 《08-limits.md》| 硬边界清单，以及每一条在写代码时的具体样子 |
| 09 | 故障排查 | 《09-troubleshooting.md》| 症状 → 原因 → 处置；三个不报错的失败单独标出 |
| 10 | 打包与分发 | 《10-packaging.md》| 提交链路、`.poa` 包格式、自带 MCP、打包发给别人 |

---

## 术语

以下术语全文反复出现。

| 术语 | 含义 |
| --- | --- |
| brainary | 本项目。它建在 codex 之上：codex 提供 V8 沙箱、工具面与子 agent；brainary 提供 PoA 这条通道、`.poa` 包格式与包自带 MCP |
| codex | PoA 所在的那套 agent 运行时。文中凡是说"codex 内置""codex 的某某"，指的都是这一层提供的东西 |
| PoA | Program of Agent，"驱动 agent 的程序"。既指开发者编写的那段 JavaScript，也指这整条通道 |
| `.poa` 包 | brainary 定义的分发形态：一个 zip，根上放 `manifest.toml`，里面是程序本体和它自带的 MCP server。提交接口只收这一种形状，只持有一段 JS 时同样封装成一个"申报零能力"的包 |
| manifest | 包里的 `manifest.toml`：`[poa]` 元数据 + `[[capabilities.mcp]]` 能力申报。申报了的 server 起不来就拒跑 |
| cell | 一次提交的那整段 JS 的一次运行。一次提交 = 一个 cell = 一次跑完 |
| 宿主 | 跑这段程序的那个 codex 运行时，即 `codex exec --poa` 或 `codex app-server` 那个进程。它持有工具注册表与分发链路、审批与沙箱策略、子 agent 的注册表与名额，并拉起 `codex-code-mode-host` 那个独立进程来承载 V8。文中"宿主侧""宿主配置的"指的都是它，与开发者自己的进程相对 |
| code mode | codex 已有的一种工具投送方式：所有工具挂在 JS 全局对象 `tools` 上，由沙箱里的 JS 调用。PoA 借的就是这条路 |
| 工具面 | 一次运行中 `tools.` 上实际挂着的那批工具。它在 cell 起跑前一次性捕获，运行过程中不变 |
| 曝光度 | 每个工具的一档注册属性，决定它对模型、对 code mode 各自可见与否。`DirectModelOnly` 只给模型直接调用，code mode 拿不到；`Deferred` 在 code mode 里能调，但工具说明文本里没有它的参数声明 |
| 子 agent | 被 PoA 程序派出去的独立 AI 会话，有自己的上下文和工具 |
| handle | 派出一个子 agent 后拿到的句柄，后续等待、追问、关闭都靠它 |
| 并行派发（扇出 / fan-out） | 一次派出 N 个子 agent 并行执行的形状 |
| 收口（fan-in） | 把派出去的结果收齐、解析、合并的那一步 |
| prelude | 一层编排 primitive，16 个，不是内置的。全文在《07-api-reference.md》§2.5，要抄进程序才有 |
| 后端 v1 / v2 | codex 的两代多 agent 工具，接口形状完全不同。prelude 存在的理由就是抹平这个差异 |
