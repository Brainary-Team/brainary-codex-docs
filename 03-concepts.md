# 核心概念

本篇覆盖编写 PoA 程序所需的全部背景，共四节。把程序打成包分发属于另一个阶段，见《10-packaging.md》文档。

---

## 1. 认清三个角色和一次运行

```mermaid
graph TD
    subgraph SANDBOX["V8 沙箱：没有 Node、没有文件系统、没有网络、没有 console"]
      PROG["<b>PoA 程序</b><br/>一段 JavaScript<br/>一次提交，一次跑完"]
    end

    PROG -->|"直接调用，当场拿到结果"| TOOLS["<b>工具</b><br/>全局对象 tools 上的一批函数<br/>exec_command —— 跑 shell（是个真 shell）<br/>apply_patch —— 改文件<br/>view_image —— 看图"]
    PROG -->|"派出去，先拿到一个 handle"| AGENT["<b>子 agent</b><br/>一个独立的 AI 会话，有自己的上下文<br/>接收一段任务文字<br/>自己决定读什么、怎么读<br/>干完交回一段回答"]
    AGENT -->|"它也能用同一批工具"| TOOLS
    PROG -.->|"何时去等由程序决定"| AGENT

    style PROG fill:#dae8fc,stroke:#6c8ebf,stroke-width:2px
    style TOOLS fill:#f5f5f5,stroke:#666666
    style AGENT fill:#fff2cc,stroke:#d6b656
```

- **PoA 程序**：一段 JS，一次提交、一次跑完。
- **工具**：沙箱里有个全局对象 `tools`，上面挂着可调用的函数。`tools.exec_command` 提供一个真正的 shell。
- **子 agent**：一个独立的 AI 会话，有自己的上下文和工具。它接收一段任务文字，自己决定读哪些文件、怎么读，最后交回一段回答。它是异步的，派出去先拿到一个 handle，何时去等由程序决定。
- **cell**：一次提交的那整段 JS 的一次运行。一次提交 = 一个 cell = 一次跑完。

cell 单列成一个词，是因为它决定了以下几件事：

| 事实 | 后果 |
| --- | --- |
| 一次提交 = 一个 cell = 一次跑完 | 没有"分阶段提交"，整个流程必须写在同一段代码里 |
| cell 结束时 V8 isolate 立即销毁 | 未 `await` 的 promise 被静默丢弃，不报错 |
| cell 有整体超时（首行 pragma 的 `yield_time_ms`） | 整个并行派发必须在这个时间内跑完 |
| cell 中途没有办法交出部分结果再继续 | 长任务只能靠把超时调大，不能靠分段 |
| cell 跑起来之后无法从外部中止 | 程序失控时只能杀进程 |

后三条的完整说明见《08-limits.md》文档。

---

## 2. 分清 prelude 和内置

程序里能用的名字分两类，来源完全不同：

| 类别 | 有哪些 | 来源 |
| --- | --- | --- |
| codex 内置的 | 全局对象 `tools`，外加 `ALL_TOOLS`、`text`、`notify`、`exit`、`store`、`load`、`setTimeout` 等 12 个全局 primitive | codex 本体，沙箱起来时就在 |
| prelude 提供的 | `AGENT_BACKEND`、`requireAgents`、`mapLimit`、`spawnAgent`、`spawnMany`、`collectAll`、`runBatch`、`sendAndWait`、`closeAll`、`parseJsonReply`、`shellLines`、`SAFE_NAME`、`hasTool`、`callTool`、`shapeOf`、`timed` | 一层普通 JS 封装，全文在《07-api-reference.md》§2.5 |

包的 `entry` 是**原样提交**的，前面不会被拼入任何内容。要用第二类那 16 个名字，就把《07-api-reference.md》§2.5 整段抄进 `main.js`，抄在首行 pragma 之后——`// @exec:` 只在第 1 行生效，且不在第 1 行时不报错。另一种做法是只用内置那一档，《05-writing.md》§12.2 的范例即如此。

后续各篇的示例默认已经抄了这一层。看到 `runBatch` / `shellLines` / `parseJsonReply` 就是它，看到 `tools.` 开头的就是内置。

### prelude 为什么存在

codex 的多 agent 工具有两代，形状完全不同，且各有局限。

| 问题 | v1 后端 | v2 后端 |
| --- | --- | --- |
| `wait_agent` 返回什么 | 直接返回最终答复 | 只是一个"有动静了"的信号，不带内容 |
| 收 N 个结果怎么写 | 一次等多个只会反复返回最先完成的那个，所以得一个一个等 | 状态和答复要另外去 `list_agents` 里捞，所以是个轮询循环 |
| agent 用什么标识 | spawn 返回的 `agent_id` | 调用方给的 `task_name`，且返回的是全限定路径 |
| 有没有"关闭"操作 | 有，且已完成的 agent 不关掉仍占并发名额 | 没有 |

这一层把差异全部盖住，所以同一段程序在两代后端上写法完全一样。代价是它里面带后端分支的代码占了将近一半。

---

## 3. 判断一个 JS 特性在不在沙箱里

沙箱在标准 JS 环境的基础上删掉了一批全局对象，同时注入了一批 codex 专有的全局函数。

被删掉的：`console`、`Atomics`、`SharedArrayBuffer`、`WebAssembly`。

本来就没有的：Node、文件系统 API、网络、`require` / `import`、`fetch`。

注入的：

| 全局 | 作用 |
| --- | --- |
| `tools` | 所有工具，`await tools.exec_command(...)` |
| `ALL_TOOLS` | `{name, description}` 数组——可以在 JS 里按名字筛工具 |
| `text(v)` / `image(v)` / `audio(v)` / `generatedImage(v)` | 往本次运行的返回值里追加一条内容项 |
| `store(k, v)` / `load(k)` | 会话级 KV，程序私有，模型看不见 |
| `notify(v)` | 不等程序结束，立刻额外送出一条内容。PoA 下客户端收不到 |
| `yield_control()` | 先把已攒的输出交出去，程序继续跑。PoA 下约等于提前结束 |
| `exit()` | 顶层提前 return |
| `setTimeout` / `clearTimeout` | 沙箱里全部的定时器能力就这两个 |

`Date.now()` 是有的，所以要时间戳不必额外开工具。同时也意味着同一段程序重跑两次结果可能不同。

各 primitive 的完整语义见《07-api-reference.md》§1。

JS 本身没有网络和文件系统，但 `tools.exec_command` 提供了一个真 shell。沙箱管的是 JS 引擎，不是能力边界；能力边界仍然是 codex 原本那套审批与沙箱策略。"运行在 V8 沙箱里"不等于"这段代码不具备破坏性操作的能力"。

---

## 4. 确定当前是哪一代后端

判定顺序只有两步：

1. `[features.multi_agent_v2] enabled = true` → **强制 v2**，压过模型元数据
2. 否则看模型元数据。自带清单里只有 `gpt-5.6-sol` 和 `gpt-5.6-terra` 标了 v2；`gpt-5.6-luna` 明确标了 v1，其余几个没标，落到默认值也是 v1

没有任何 feature 开关能反过来强制到 v1。所以一个标着 v2 的模型只有两条路：打开 v2 那组开关，或者换模型。

还有一层：v2 那 6 个工具默认是 `DirectModelOnly`，只给模型直接调，PoA 完全拿不到。

```toml
[features.multi_agent_v2]
enabled = true
non_code_mode_only = false               # 少这行就是 0 个工具
max_concurrent_threads_per_session = 8   # 默认 4，root 占一个，所以默认只剩 3
```

> 第一条命令永远是探针。换模型、换 provider、改配置之后都要重跑。写法见《02-quickstart.md》§2。

可选工具组（memories、clock、token 预算、权限）默认都不在工具面上，各自的开关见《02-quickstart.md》§4。它们影响的是工具面，不是后端。

### 两代的差异对写程序的影响

大部分差异被 prelude 那一层盖住了，但有三件事漏出来，写程序时必须知道：

| 差异 | v1 | v2 |
| --- | --- | --- |
| 名额计的是什么 | 一次运行里累计派出的 agent 数，默认 6，root 不占 | 同时驻留的 agent 数，配置值减 1（root 占一个） |
| 超额时 | 派发抛异常 | 派发抛异常；驻留位满时驱逐已完成的 agent，其结果静默变 `null` |
| 递归深度限制 | 有，且拦两道，子 agent 天然派不出孙 agent | 没有，无条件放行。限制只能在 JS 里自行实现 |
| `closeAll` | 有效，且必须调（已完成的 agent 不关掉仍占名额） | 空操作，v2 那组工具里没有"关闭" |

前两条决定程序要不要写成分批，见《05-writing.md》§6.3；第三条是递归风险，见《05-writing.md》§6.4。
