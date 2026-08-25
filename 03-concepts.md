---
title: 核心概念
description: 三个角色、cell、沙箱面、prelude 与内置的分界、两代后端
---

# 核心概念

[← 快速开始](./02-quickstart.md) · [返回目录](./index.md) · 下一篇：[工作原理](./04-how-it-works.md)

---

## 1. 三个角色

把这三个角色理清楚，后面就都好说了。

```mermaid
graph TD
    subgraph SANDBOX["V8 沙箱：没有 Node、没有文件系统、没有网络、没有 console"]
      PROG["<b>PoA 程序</b><br/>一段 JavaScript<br/>一次提交，一次跑完"]
    end

    PROG -->|"直接调用，当场拿到结果"| TOOLS["<b>工具</b><br/>全局对象 tools 上的一批函数<br/>exec_command —— 跑 shell（是个真 shell）<br/>apply_patch —— 改文件<br/>view_image —— 看图"]
    PROG -->|"派出去，先拿到一个 handle"| AGENT["<b>子 agent</b><br/>一个独立的 AI 会话，有自己的上下文<br/>接收一段任务文字<br/>自己决定读什么、怎么读<br/>干完交回一段回答"]
    AGENT -->|"它也能用同一批工具"| TOOLS
    AGENT -.->|"何时去等由程序决定"| PROG

    style PROG fill:#dae8fc,stroke:#6c8ebf,stroke-width:2px
    style TOOLS fill:#f5f5f5,stroke:#666666
    style AGENT fill:#fff2cc,stroke:#d6b656
```

**PoA 程序**：一段 JS，一次提交、一次跑完。

**工具**：沙箱里有个全局对象 `tools`，上面挂着可调用的函数。`tools.exec_command` 提供一个真正的 shell。

**子 agent**：一个独立的 AI 会话，有自己的上下文和工具。它接收一段任务文字，自己决定读哪些文件、怎么读，最后交回一段回答。**它是异步的**——派出去先拿到一个 handle，何时去等由程序决定。

> 一句话概括：**程序是主角，工具是它的手，子 agent 是它雇的临时工。**

---

## 2. cell：一次提交，一次跑完

**cell** 是一次提交的那整段 JS 的一次运行。

这个概念之所以要单列，是因为它决定了几件反直觉的事：

| 事实 | 后果 |
| --- | --- |
| 一次提交 = 一个 cell = 一次跑完 | 没有"分阶段提交"，整个流程必须写在同一段代码里 |
| cell 结束时 V8 isolate 立即销毁 | **未 `await` 的 promise 被静默丢弃**，不报错 |
| cell 有整体超时（首行 pragma 的 `yield_time_ms`） | 整个并行派发必须在这个时间内跑完 |
| cell 中途没有办法交出部分结果再继续 | 长任务只能靠把超时调大硬扛，不能靠分段 |
| cell 跑起来之后无法从外部中止 | 跑飞了只能杀进程 |

后三条的完整说明见[边界与限制](./08-limits.md)。

---

## 3. 沙箱面：有什么、没什么

这一节最容易想错——沙箱比"一个 JS 环境"要**窄得多，同时又宽得多**。

### 被删掉的

`console`、`Atomics`、`SharedArrayBuffer`、`WebAssembly`。

### 完全没有的

没有 Node，没有文件系统 API，没有网络，没有 `require` / `import`，没有 `fetch`。

### 装进来的

| 全局 | 作用 |
| --- | --- |
| `tools` | 所有工具，`await tools.exec_command(...)` |
| `ALL_TOOLS` | `{name, description}` 数组——可以在 JS 里按名字筛工具 |
| `text(v)` / `image(v)` / `audio(v)` / `generatedImage(v)` | 往本次运行的返回值里追加一条内容项 |
| `store(k, v)` / `load(k)` | 会话级 KV，程序私有，模型看不见 |
| `notify(v)` | 不等程序结束，立刻额外送出一条内容。**PoA 下客户端收不到** |
| `yield_control()` | 先把已攒的输出交出去，程序继续跑。**PoA 下约等于提前结束** |
| `exit()` | 顶层提前 return |
| `setTimeout` / `clearTimeout` | **沙箱里全部的定时器能力就这两个** |

`Date.now()` 是有的——所以要时间戳不必额外开工具。同时也意味着**同一段程序重跑两次结果可能不同**。

各 primitive 的完整语义见 [API 参考 §1](./07-api-reference.md#1-全局-primitive12-个)。

> [!IMPORTANT]
> **JS 本身没有网络和文件系统，但 `tools.exec_command` 提供了一个真 shell。**
> 沙箱管的是 JS 引擎，**不是能力边界**——能力边界仍然是 codex 原本那套审批与沙箱策略。
> 别把"V8 沙箱"读成"这段代码干不了坏事"。

---

## 4. 一个必须先分清的界：prelude vs 内置

> [!IMPORTANT]
> **写 demo 时，写下的那个文件不是被原样提交的。**

跑 `demos/*.js` 时，runner 会把 `workflow-demos/lib/prelude.js`（243 行）**整个拼在程序代码前面**，然后把拼接结果当作 `entry` 提交。

所以下面这两类名字，来源完全不同：

| 类别 | 有哪些 | 来源 | 改得动吗 |
| --- | --- | --- | --- |
| **prelude 提供的** | `AGENT_BACKEND`、`requireAgents`、`mapLimit`、`spawnAgent`、`spawnMany`、`collectAll`、`runBatch`、`sendAndWait`、`closeAll`、`parseJsonReply`、`shellLines`、`SAFE_NAME`、`hasTool`、`callTool`、`shapeOf`、`timed` | 仓库自己写的一层薄封装 | ✅ 就在 `workflow-demos/lib/prelude.js`，243 行，不是黑盒 |
| **codex 内置的** | `tools`、`ALL_TOOLS`、`text`、`notify`、`exit`、`store`、`load`、`setTimeout` 等 12 个全局 primitive | codex 本体 | ❌ 只能查，改不了 |

**为什么这个区分重要**：

- 换一个客户端、不拼 prelude 的话，**前一类全都不存在**
- prelude 那些函数的行为可以直接读源码确认；内置那些只能查文档
- 出问题时，知道该去哪一侧排查

> [!CAUTION]
> **写成 `.poa` 包时，prelude 不会被拼进去。** 包的 `entry` 是原样提交的——这是格式的定义，不是配置。
> 于是上表第一行那 16 个名字**全部不存在**，`runBatch is not defined` 是包作者最常见的第一个报错。
> 要用就把 `prelude.js` 抄进包里、在 `entry` 顶部内联，或者干脆只用内置那一档。详见 [§9](#9-poa-包自带能力的那条路)。

### prelude 为什么存在

一句话：**codex 的多 agent 工具有两代，形状完全不同，而且各有毛病。**

| 问题 | v1 后端 | v2 后端 |
| --- | --- | --- |
| `wait_agent` 返回什么 | 直接返回最终答复 | **只是一个"有动静了"的信号**，不带内容 |
| 收 N 个结果怎么写 | 一次等多个只会反复返回最先完成的那个，所以得**一个一个等** | 状态和答复要另外去 `list_agents` 里捞，所以是个**轮询循环** |
| agent 用什么标识 | spawn 返回的 `agent_id` | 调用方给的 `task_name`，**且返回的是全限定路径** |
| 有没有"关闭"操作 | 有，且**已完成的 agent 不关掉仍占并发名额** | 没有 |

prelude 把这些差异全部盖住了，所以**同一段程序在两代后端上写法完全一样**。代价是这一层里带后端分支的代码占了将近一半。

---

## 5. handle 与 `meta`

派出一个子 agent，拿到的是一个 handle：

```js
{ key, label, meta }
```

| 字段 | 是什么 |
| --- | --- |
| `key` | 后端认的标识，后续等待 / 追问 / 关闭都靠它 |
| `label` | 面向人的名字（v1 下是自动起的昵称） |
| `meta` | **留给调用方塞上下文的口袋。** prelude 不解释它，只负责原样带回来 |

> [!WARNING]
> **`meta` 不是可选的锦上添花。**
> agent 的回答文本里**不包含"我是谁"**，程序必须自己记。收口时靠 `handle.meta.folder` 这类字段
> 把结果对回输入，不然收到的只是一堆匿名回复。

---

## 6. 两代后端与 profile

### 后端不由本地配置决定

用 v1 还是 v2 后端，**由模型元数据决定**，写在 codex 的模型清单里。没有任何 feature 开关能强制回到 v1。

唯一的杠杆是**替换整个模型清单**——`v1-forced` 这个 profile 干的就是这件事：它现场派生一份把所有模型钉到 v1 的清单。

**推论**：同一个模型换个 profile，code mode 里能看到的 agent 工具数会从 0 变到 5 或 6。所以：

> **第一条命令永远是探针。** 换模型、换 provider、换 profile 之后都要重跑。

### 四个 profile

| profile | 后端 | agent 工具 | 工具总数 | 说明 |
| --- | --- | --- | --- | --- |
| `v1` | 取决于模型 | 0 | 8 | **陷阱。** 多数模型自报 v2，而 v2 工具默认进不了 code mode |
| `v1-forced` | v1 | 5 | 13 | **推荐默认。** 把所有模型钉到 v1 |
| `v2` | v2 | 6 | 14 | 需要更新的模型；额外显式打开了 v2 工具进 code mode 的开关 |
| `full` | v1（钉法同 `v1-forced`） | 5 | 20 | **差在工具面，不在后端。** 把可选工具组全打开 |

`./run.sh` 不给 profile 时的默认值恰好是那个坏的 `v1`。**永远显式写。**

> [!IMPORTANT]
> **第一行取决于模型，不取决于 profile。** 自带清单里只有 `gpt-5.6-sol` 和 `gpt-5.6-terra` 标了 `multi_agent_version: v2`，`gpt-5.6-luna` 本来就是 v1——所以用 luna 时 `v1` 这个 profile 会正常解析到 v1 并给出 5 个 agent 工具，`v1-forced` 对它是空操作。
>
> 上面那张表是"某个模型 + 某份清单"的一张快照。**你那套是什么样，只有探针答得了。**

`full` 打开的是这几组（各自的门不止一道）：

| 工具 | 需要什么 |
| --- | --- |
| `clock__curr_time` | `[features.current_time_reminder] enabled` |
| `get_context_remaining` | `[features] token_budget` |
| `request_permissions` | `[features] request_permissions_tool`，且得存在一个 environment |
| `memories__*`（4 个） | `[features] memories` **且** `[memories] dedicated_tools`——**两个都要，少一个是零个工具而不是一半** |

写操作还额外要一个可写沙箱，那个走环境变量不在 profile 里：

```bash
CODEX_DEMO_SANDBOX=workspace-write ./run.sh demos/08_file_and_image.js full
```

### 两代的差异对写程序的影响

大部分差异被 prelude 盖住了，但有三件事漏出来，写程序时必须知道：

| 差异 | v1 | v2 |
| --- | --- | --- |
| **并发名额** | 默认 **6**，不减 root | 上限**减 1**（root 线程占一个）。默认上限 4 → **实际只剩 3** |
| **递归深度刹车** | 有，且拦两道——子 agent 天然派不出孙 agent | **没有**，无条件放行。刹车只能在 JS 里自己踩 |
| **`closeAll` 有没有用** | 有效，且**必须调**（已完成的 agent 仍占名额） | 空操作，因为 v2 那组工具里根本没有"关闭" |

第二条是实打实的风险，详见[编写指南 §6.4](./05-writing.md#64-递归深度v2-下没有刹车)。

---

## 7. 提交链路：代码是怎么进去的

知道这条链有助于判断报错落在哪一侧。**现在有两条**，`run.sh` 按目标形态自动分流：给它一个目录或 `.poa` 文件走下面那条，给它一个 `.js` 走上面那条。

```mermaid
graph LR
    YOU["开发者"] --> RUNSH["run.sh<br/>读 .env，渲染 profile"]

    RUNSH -->|"*.js"| RUNPY["run_workflow.py<br/>拼 prelude + 程序代码<br/>把 pragma 提回第 1 行<br/>包成零能力的 .poa"]
    RUNSH -->|"目录 / *.poa"| EXEC["codex exec --poa<br/>打包 + 本地校验 manifest"]

    RUNPY --> RPC["JSON-RPC over stdio<br/>threadId + package"]
    EXEC --> RPC
    RPC --> CODEX["codex app-server<br/>解包 → 起包内 MCP → 等就位"]
    CODEX --> CELL["V8 里的一个 cell"]
    CELL --> SUB["子 agent × N"]

    style CELL fill:#dae8fc,stroke:#6c8ebf,stroke-width:2px
    style SUB fill:#fff2cc,stroke:#d6b656
    style EXEC fill:#d5e8d4,stroke:#82b366
```

四件值得记住的事：

**① 改 demo 或 prelude 不需要重建二进制**——它们是运行时被读成字符串送进去的。包也一样：`codex exec --poa <目录>` 每次现打包，没有"先 build 再跑"这一步。

**② 线上只有一种形状。** 不管哪条路，最后发出去的都是 `{ threadId, package }`——手里只有一段 JS 的客户端（比如 `run_workflow.py`）就现造一个"申报零能力"的包。**没有第二个只收源码的入口**。

**③ pragma 会被自动提到第 1 行——只在上面那条路上。** prelude 有 243 行，直接拼在前面会把 `// @exec:` 挤到第 244 行，那就**不生效了，而且不报错**。runner 专门做了这一步提取。走包这条路时 `entry` 原样提交，pragma 天然在第 1 行；但换成自研客户端拼源码时，**没有任何环节会代劳**。

**④ 报错落在哪一侧决定去哪查**：宿主机侧的失败在 Python traceback 或 shell 里（走包这条路时连 Python 都没有，是 `codex` 自己报的 usage error）；V8 侧的失败在 RPC 返回的内容里。

---

## 8. 全程无人值守

这不是一个选项，是这条通道的性质：

- 客户端对服务端的**反向请求一律自动批准**，配合 profile 里的"从不询问"审批策略，整条链路上**不存在任何人工审批点**
- 程序**没有跟人对话的手段**——那个能问用户问题的工具只给模型，程序调不到

**推论**：任何一次阻塞询问都会把整条链路挂死，所以 PoA 程序必须能在没有人的情况下从头跑到尾。需要人判断的地方，只能把判断也写成代码（比如数票），或者把问题留到最后输出里。

---

## 9. PoA 包：自带能力的那条路

> **2026-08 新增（PR #29）。** 在这之前，程序的能力只能来自宿主的 `config.toml`——[边界与限制 §6](./08-limits.md#6-自带能力只到-stdio-mcp) 原来就是这么写的。现在程序可以像一个安装包那样把 MCP server 带在身上。

### 一个包长什么样

```
poas/00_echo/
├── manifest.toml       [poa] 元数据 + [[capabilities.mcp]] 能力申报
├── main.js             程序本体，原样提交（不拼 prelude）
└── mcp/echo.mjs        随包分发的 stdio MCP server，零依赖
```

`.poa` 就是这个目录打成的 zip，**`manifest.toml` 必须在根上**。

```toml
[poa]
name = "echo_expert"
version = "0.1.0"
runtime = "codex-v8"      # 只认这一个值，别的直接反序列化失败
poa_api_version = 1       # 只认 1
entry = "main.js"         # 必须 .js/.mjs，必须在包内
description = "..."       # 可选

[[capabilities.mcp]]
name = "echo"                     # 只能用字母数字和 _ -，且不能重名
type = "stdio"                    # 只有这一种；写 network 会在解析阶段报错
shell = "node mcp/echo.mjs"       # 按 shell 语法切成 argv，直接派生，不经 sh -c
description = "..."               # 可选
```

### 跑起来的时候发生了什么

```
codex exec --poa <目录|.poa>
  ├─ 目录就现打包，.poa 就直接读
  ├─ 本地先校验一遍 manifest        ← 坏包在这里就是 usage error，不会起半个会话
  └─ base64 → { threadId, package }
                 ↓
       codex app-server
         ├─ 解包到一个临时目录
         ├─ 每条 [[capabilities.mcp]] 注册成 thread 级 MCP server（cwd = 包根）
         ├─ 等这些 server 全部就位，再捕获工具面   ← 顺序是关键，见下
         ├─ 任何一个没在工具面上留下工具 → 拒跑
         └─ 把 entry 的内容当源码送进 V8
```

四件必须知道的事：

**① 组装在服务端，不在客户端。** CLI 只是"把字节递过去"的一个薄客户端。**TUI、SDK、自研客户端跑一个包都是发同一份字节**，不需要各自实现一遍解包和起 server。

**② server 随 thread 生死，不写全局配置。** 包起的 server 只对这一个 thread 可见，`config.toml` 一个字都不改，thread 关掉进程就回收。

**③ 顺序不能反。** 包必须在"捕获工具面"之前发布，因为捕获工具面这一步才是建工具计划的时候。晚一步注册的 server，对这个 cell 来说等于不存在。

**④ 起不来就拒跑。** 判据是"这个 server 有没有在工具面上留下工具"，所以起来了但一个工具都不报的 server**同样算缺席**。这是刻意与采样循环相反：模型少个工具可以换条路走，程序不行。启动超时 30 秒。

### 两个一定会踩的坑

> [!WARNING]
> **① 工具名不能硬编码。** `mcp__echo__echo` 是当前配置下的产物——前缀取决于 `prefix_mcp_tool_names()`，命名空间会被清洗、重名时还会加哈希后缀。**正确写法是从 `ALL_TOOLS` 里按后缀找，并断言只找到一个**：
>
> ```js
> const matches = ALL_TOOLS.map((t) => t.name).filter((n) => /(^|__)echo__echo$/.test(n));
> if (matches.length !== 1) throw new Error(`expected exactly one echo tool, found ${matches.length}`);
> const result = await tools[matches[0]]({ message: "ping" });
> ```
>
> 断言那一句不是防御性冗余：命名规则一变，**没有断言就是静默调错工具**。

> [!WARNING]
> **② 没有标注的工具会触发审批弹窗，在无人值守下就是挂死。** 默认 `auto` 审批模式下，codex 对一个没有标注的 MCP 工具的假设是"它可能写、可能联网"，于是发起一次 elicitation——而这条链路上[没有人](#8-全程无人值守)。
>
> **给工具标上 `readOnlyHint`**。顺带的好处是：只读标注同时也是[并行安全的判据](./04-how-it-works.md#哪些工具是并行安全的)，所以包自带的只读 server **是可以真并行的**（`ext/poa` 建的 config 里 `supports_parallel_tool_calls` 硬编码为 `false`，服务器级的整体豁免用不了，**只能逐个工具标只读**）。

### 边界

| 能 | 不能 |
| --- | --- |
| `stdio` MCP server | `network` 传输（解析阶段拒，不是静默忽略） |
| 任意随包文件 | `[build]` 段、resources |
| 根线程的工具面 | **子 agent 的工具面**——它跑在自己的 thread 上，拿不到包里的 server |

还有几个数字：**包上限 64 MiB**（因为要 base64 塞进一条 JSON-RPC 消息），解压后 256 MiB / 100 倍膨胀比 / 4096 个条目三道闸，**解出来的文件权限固定 `0644`**（所以 server 要写成 `node mcp/xxx.mjs` 这种带解释器的命令行，别指望可执行位）。

### 分发：`build.sh`

```bash
./build.sh poas/00_echo               # → ./00_echo.poa
./run.sh   poas/00_echo v1-forced     # 目录直接跑，不用先 build
codex exec --poa poas/00_echo         # 连 run.sh 都不要
codex exec --poa ./00_echo.poa
```

**`build.sh` 只为"给别人一个文件"而存在**——`codex exec --poa <目录>` 本来就现打包。它写文件前先校验，时间戳钉在 zip 纪元所以**打两次字节一样**。

> [!NOTE]
> **包不需要 provider 也能跑。** 包本体不采样，所以 `run.sh` 对包目标会兜底一套占位的 key / model / base_url，而且 base_url 故意用一个**关闭的端口**（`127.0.0.1:9`）——万一程序真的去采样了，应当当场炸掉，而不是连上碰巧在监听的某台主机。这套兜底排在 `.env` 和 `CODEX_DEMO_*` **下面**，配了真 provider 照常生效。

代码位置：格式与校验在 `codex-rs/poa/`，起 server 在 `codex-rs/ext/poa/`。

---

[← 快速开始](./02-quickstart.md) · [返回目录](./index.md) · 下一篇：[工作原理](./04-how-it-works.md)
