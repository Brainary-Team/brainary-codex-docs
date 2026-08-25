---
title: API 参考
description: 12 个全局 primitive、16 个 prelude primitive、32 个内置工具的完整声明
---

# API 参考

[← 模式库](./06-patterns.md) · [返回目录](./index.md) · 下一篇：[边界与限制](./08-limits.md)

**这一篇是查询用的参考表，不是叙述性文档，不必顺序读完。**

三类东西的来源不同，别混：

| 类别 | 来源 | 数量 |
| --- | --- | --- |
| [全局 primitive](#1-全局-primitive12-个) | codex 内置，直接叫 | 12 |
| [prelude primitive](#2-prelude-primitive12-个) | 仓库自备，运行前拼在程序代码前面 | 12 |
| [内置工具](#3-内置工具) | codex 内置，挂在 `tools` 上 | 32 + provider 相关 |

> [!IMPORTANT]
> **下面的表是"可能有什么"，探针输出的 `all_tools` 才是"现在实际有什么"。**
> 换模型、换 provider、换 profile 之后都要重跑探针。
>
> 写程序时不要硬编码工具名：开头读一次 `ALL_TOOLS` 判断在不在，对缺席路径写降级。

---

## 1. 全局 primitive（12 个）

不在 `tools` 上，直接调用。

| primitive | 干什么 | PoA 下 |
| --- | --- | --- |
| `text(v)` | 往本次运行的返回值里追加一条文本 | ⭐ **唯一可靠的输出手段，必用** |
| `image(v)` / `audio(v)` / `generatedImage(v)` | 追加图片 / 音频 / 生成图，与 `text()` 同族 | 少见 |
| `store(k, v)` / `load(k)` | 会话级 KV。**程序私有，模型看不见** | 偶尔（一次运行只提交一次，通常用 `const` 就够） |
| `ALL_TOOLS` | 当前可用工具的 `{ name, description }` 数组 | 探测环境用。**没有参数 schema** |
| `exit()` | 顶层提前 return | ✅ 正常可用 |
| `setTimeout` / `clearTimeout` | **沙箱里全部的定时器能力就这两个** | 偶尔 |
| `notify(v)` | 不等程序结束，立刻额外送出一条内容 | ❌ **PoA 客户端收不到**，别用它打进度 |
| `yield_control()` | 先把已攒的输出交出去，程序继续跑 | ❌ 没有续跑手段，约等于提前结束 |

**沙箱里没有的**：Node、文件系统 API、网络、`console`、`require` / `import`、`fetch`。
**被删掉的标准全局**：`console`、`Atomics`、`SharedArrayBuffer`、`WebAssembly`。
`Date.now()` 是有的。

> [!WARNING]
> **未 `await` 的 promise 会被静默丢弃。** 程序求值一结束沙箱就没了，没等的活直接消失，**不报错**。

---

## 2. prelude primitive（16 个）

来自 `workflow-demos/lib/prelude.js`（243 行）。**这些不是 codex 内置的**——换一个不拼 prelude 的客户端，它们全都不存在。

> [!CAUTION]
> **写 `.poa` 包时这一整节都不适用。** 包的 `entry` 原样提交，不拼 prelude，这 16 个名字一个都没有。见[编写指南 §13](./05-writing.md#13-从-js-到-poa要改的四件事)。

### 2.1 派 agent / 收结果

| 名字 | 签名（含默认值） | 干什么 | 要注意 |
| --- | --- | --- | --- |
| `runBatch` | `runBatch(specs, { concurrency = 3, timeoutMs = 300000 })` | **最常用的一个。** 派一批 + 等全部完成。`specs` 是 `[{ message, name?, meta? }]`，返回 `[{ handle, reply }]`，**顺序与输入一致** | `reply` 是字符串或 `null`（超时 / 出错）。`timeoutMs` 必须小于首行 pragma 的 `yield_time_ms` |
| `spawnAgent` | `spawnAgent(message, { name?, meta? })` | 派**一个**，立刻返回 handle `{ key, label, meta }` | `meta` 是留给调用方塞上下文的口袋——**agent 的回答里不含"我是谁"，不塞就对不上号**。**`name` 只在 v2 生效**：v1 的派发接口没有这一项，传进去直接被丢掉，`label` 取的是模型给的昵称。所以**别拿 `label` 去认人，认 `meta`** |
| `spawnMany` | `spawnMany(specs, concurrency = 3)` | 派一批，**只派不等**，返回 handle 数组 | 想边派边干别的时用它，否则直接用 `runBatch` |
| `collectAll` | `collectAll(handles, timeoutMs = 300000, pollMs = 15000)` | 等一批 handle 全部到终态，返回 `Map`：`handle.key → 最终答复 \| null` | 没在超时内完成的会被填成 `null`，**不会抛错**，要自己检查。**`pollMs` 只在 v2 生效**——v1 走的是串行单目标等待，压根不轮询 |
| `sendAndWait` | `sendAndWait(handle, message, { timeoutMs = 180000, pollMs = 3000 })` | 对一个**还活着**的 agent 追问，等它给出**新**答复 | 内部会先记下上一轮答复再轮询比对，避免把旧答案当新的。超时返回 `null` |
| `closeAll` | `closeAll(handles)` | 关掉这批 agent，腾出并发名额。**吞掉所有错误，绝不抛** | **只对 v1 后端有效**（v2 那组工具里没有"关闭"）。v1 下不能省——已完成的 agent 不关仍占名额 |

### 2.2 工具函数

| 名字 | 签名 | 干什么 | 要注意 |
| --- | --- | --- | --- |
| `shellLines` | `shellLines(cmd, { validate = null })` | 跑一条 shell 命令，把输出**按行**拿回来：逐行 trim、丢掉空行 | **`validate` 几乎必传。** shell 出错时报错信息和正常输出走同一个通道，不过滤会把 `No such file or directory` 当成数据 |
| `parseJsonReply` | `parseJsonReply(raw)` | 从自由文本里抠 JSON，返回 `{ value, error }` | 匹配是**贪婪**的：取第一个 `{` 到**最后一个** `}` 的整段。回复里出现两段 `{}` 时**整体解析失败**，不会退回第一段——详见下方 WARNING。三种失败各有不同 `error` 文案：不是字符串 / 没找到 `{}` / 解析异常。**永远处理 error 分支** |
| `mapLimit` | `mapLimit(items, limit, fn)` | 限流并发 map，最多 `limit` 个同时在飞，**结果保持输入顺序** | `runBatch` / `spawnMany` 内部就是用它限流的 |
| `SAFE_NAME` | 常量 `/^[A-Za-z0-9._-]+$/` | 现成的白名单正则，配合 `validate` 用 | 只放行字母数字和 `. _ -`，**带空格、斜杠、中文的路径会被一并滤掉** |

> [!WARNING]
> **`parseJsonReply` 的贪婪匹配有一个高频触发场景。**
> 它用的是 `/\{[\s\S]*\}/`——从第一个 `{` 一路吃到最后一个 `}`。
>
> 而[编写指南 §5](./05-writing.md#5-写子-agent-的任务文字) 推荐的 prompt 模板里**本身就带一段 `{...}` 格式示例**。
> agent 只要在正式答案前把那段模板复述一遍，两段 `{}` 就被连成一整块 → **`JSON.parse` 抛错，
> `value` 为 `null`**，而不是退回前面那一段。
>
> 这是 `succeeded / total` 掉下来的常见原因之一。在 prompt 里补一句 `不要复述格式说明` 能挡掉大部分；
> 剩下的靠 `error` 分支兜住并把原文留下来。

### 2.3 环境探测

| 名字 | 是什么 | 干什么 |
| --- | --- | --- |
| `AGENT_BACKEND` | 常量，`"v1"` / `"v2"` / `null` | 启动时扫一遍 `ALL_TOOLS` 自动判定后端。**`null` 就是这套配置下一个 agent 工具都没有** |
| `requireAgents()` | 函数，返回后端名 | 断言有可用后端，没有就抛错——**且错误信息里带上当前全部可用工具名**，这是最快的排错入口 |

### 2.4 能力探测（2026-08 新增）

**为什么需要这一组**：可选工具组（memories、clock、MCP、provider 决定的那几个）**"这套配置下没有"是正常结果，不是 bug**。没有这几个函数，碰一下可选工具要么在 `tools.x is not a function` 上炸掉，要么每个调用点都手写一遍 try/catch。

| 名字 | 签名 | 干什么 | 要注意 |
| --- | --- | --- | --- |
| `hasTool` | `hasTool(name)` | 这个名字在不在当前工具面上 | 就是 `ALL_TOOLS` 的一个 Set 查询，**启动时快照一次**——一次运行里工具面不会变 |
| `callTool` | `callTool(name, args = {})` | 调一个工具，**永不抛**。返回 `{ name, status, value, error, ms }`，`status` 是 `"ok"` / `"absent"` / `"error"` | **`absent` 和 `error` 是两回事**，别合并处理：前者是"这套配置没装"，后者是"装了但调失败"。`args` 原样透传，所以 freeform 工具直接给字符串：`callTool("apply_patch", "*** Begin Patch\n...")` |
| `shapeOf` | `shapeOf(value, depth = 1)` | 把一个运行时值描述成结构串，如 `object{output: string, exit_code: number}` | 给[那 12 个只声明 `Promise<unknown>` 的工具](#34-返回类型12-个没有形状承诺用之前先探一次)用的——**它们对返回形状没有任何承诺，唯一的办法就是看实际回来的是什么** |
| `timed` | `timed(fn)` | 跑 `fn` 并带上墙钟耗时，**永不抛**。返回 `{ ok, value, error, ms }` | 用来把"真并行"和"串行"分开——两个都是 `Promise.all`，只有耗时能区分 |

```js
const r = await callTool("memories__list", {});
if (r.status === "absent") log.push("memories 未配置，跳过");
else if (r.status === "error") log.push(`memories 调用失败：${r.error}`);
else log.push(`memories 返回 ${shapeOf(r.value)}，耗时 ${r.ms}ms`);
```

> [!TIP]
> **包自带的工具不用走这一组。** 申报了的 MCP server 起不来会直接[拒跑](./03-concepts.md#9-poa-包自带能力的那条路)，所以它出现在 `ALL_TOOLS` 里就一定能调。需要探测的是宿主侧那些。

---

## 3. 内置工具

挂在全局对象 `tools` 上，**能不能用取决于配置与 provider**。

### 3.1 速查表

「默认可得」一列：

| 记号 | 含义 |
| :---: | --- |
| ✅ | 默认配置下就在 |
| ⚙ | 需打开对应 feature 或配置 |
| ⚠ | 默认可调用，但**声明不出现**（`Deferred` 档） |
| 探针 | 外挂 MCP server 提供，不是 codex 内建 |

#### 执行与文件（5）

| 工具 | 作用 | 默认可得 | 并行安全 |
| --- | --- | --- | :---: |
| [`exec_command`](#exec_command) | 在 PTY 中跑一条命令；进程未结束时返回可续接的 `session_id` | ✅ ConPTY 可用时 | ✅ |
| [`write_stdin`](#write_stdin) | 向已有 exec 会话写 stdin 并取回最近输出，也可空写用作轮询 | ✅ 同上 | ✅ |
| [`shell_command`](#shell_command) | 跑一条 shell 脚本，无会话概念的一次性执行 | ⚙ 仅回落时，且 code mode 看不见 | ✅ |
| [`apply_patch`](#apply_patch) | 编辑文件。**入参是裸字符串不是对象** | ✅ 需有 environment | ❌ |
| [`view_image`](#view_image) | 把磁盘上已存在的本地图片读进上下文 | ✅ 需有 environment | ✅ |

> **shell 工具二选一，但依据不是操作系统。** 判定在 `codex-rs/tools/src/tool_config.rs:81` 的 `shell_type_for_model_and_features()`：unified exec feature 开启且 `conpty_supported()` 为真 → `exec_command` + `write_stdin`；否则回落到 `shell_command`。而 `conpty_supported()`（`codex-rs/utils/pty/src/pty.rs:47-51`）在**非 Windows 上恒为真**，在 Windows 上只要 Build ≥ `MIN_CONPTY_BUILD` 也为真——**所以现代 Windows 同样拿到 `exec_command`，只有 ConPTY 不可用的老 Windows 才回落**。`shell_command` 另有一个前提：`environments.single_local_environment().is_some()`（`spec_plan.rs:826`），非单一本地环境时它根本不注册。两者**参数名还不一样**（`cmd` vs `command`）。

> [!NOTE]
> **走 `exec_command` 这条路时，`shell_command` 其实也被注册了，但 PoA 调不到。** `spec_plan.rs:846-853` 会把它以 `ToolExposure::Hidden` 注册，注释写明是为兼容 legacy；而 `ToolExposure::is_available_in_code_mode()`（`codex-rs/tools/src/tool_executor.rs:93-98`）把 `Hidden` 判为 `false`。所以它不进 `ALL_TOOLS`、`tools.shell_command(...)` 也调不到——在 `ALL_TOOLS` 里找不到它是**正常现象**，不是配置错了。

#### 计划与目标（4）

| 工具 | 作用 | 默认可得 |
| --- | --- | --- |
| [`update_plan`](#update_plan) | 覆写任务计划列表，同一时刻至多一步 `in_progress` | ✅ |
| [`create_goal`](#create_goal) | 开启一个活跃目标，可带 token 预算 | ✅ |
| [`get_goal`](#get_goal) | 读当前目标的状态、预算、用量 | ✅ |
| [`update_goal`](#update_goal) | 把目标标记为 `complete` 或 `blocked` | ✅ |

> **这一组 PoA 里基本用不上**——它们是给 AI 向人展示进度用的，程序侧用普通变量记状态即可。

#### 上下文与权限（2）

| 工具 | 作用 | 默认可得 |
| --- | --- | --- |
| [`get_context_remaining`](#get_context_remaining) | 查当前上下文窗口还剩多少 token | ⚙ `[features] token_budget` |
| [`request_permissions`](#request_permissions) | 申请额外的文件系统读写或网络权限 | ⚙ `[features] request_permissions_tool`，且需有 environment |

#### 时间（1）

| 工具 | 作用 | 默认可得 |
| --- | --- | --- |
| [`clock__curr_time`](#clock__curr_time) | 返回当前 UTC 时间 | ⚙ `[features] current_time_reminder` |

> JS 里有 `Date.now()`，要时间戳不必开这个。

#### 记忆（4）

| 工具 | 作用 | 默认可得 |
| --- | --- | --- |
| [`memories__list`](#memories__list) | 列出某路径下的直接子文件与子目录 | ⚙ 见下 |
| [`memories__read`](#memories__read) | 按相对路径读一个 memory 文件 | ⚙ 见下 |
| [`memories__search`](#memories__search) | 子串搜索，支持同行 / 同窗口的多子串匹配 | ⚙ 见下 |
| [`memories__add_ad_hoc_note`](#memories__add_ad_hoc_note) | 追加一条临时 memory 笔记 | ⚙ 见下 |

> 这 4 条要 `[features] memories = true` **且** `[memories] dedicated_tools = true`，两个默认都是假，**只开一个得到 0 个记忆工具**。

#### 子代理 v1（5）

| 工具 | 作用 | 默认可得 |
| --- | --- | --- |
| [`multi_agent_v1__spawn_agent`](#multi_agent_v1__spawn_agent) | 派生子代理，返回 `agent_id` | ⚠ 见下 |
| [`multi_agent_v1__wait_agent`](#multi_agent_v1__wait_agent) | 阻塞等待代理进入终态；**多 id 时等最先结束的那个** | ⚠ 见下 |
| [`multi_agent_v1__send_input`](#multi_agent_v1__send_input) | 给已有代理发消息，`interrupt=true` 可立刻改变方向 | ⚠ 见下 |
| [`multi_agent_v1__resume_agent`](#multi_agent_v1__resume_agent) | 恢复已关闭的代理 | ⚠ 见下 |
| [`multi_agent_v1__close_agent`](#multi_agent_v1__close_agent) | 关闭代理及其后代，**释放并发额度** | ⚠ 见下 |

> 这 5 条在默认配置下是 `Deferred`：**调得通，但声明不出现**，`ALL_TOOLS` 里也只有名字和一句描述。下面的声明因此格外有价值——那是运行时根本查不到的东西。

#### 子代理 v2（6）

| 工具 | 作用 | 默认可得 |
| --- | --- | --- |
| [`collaboration__spawn_agent`](#collaboration__spawn_agent) | 按 `task_name` 派生代理，`fork_turns` 控制带多少上下文 | ⚙ 见下 |
| [`collaboration__wait_agent`](#collaboration__wait_agent) | 等任意存活代理的信箱更新，**只返回摘要而非内容** | ⚙ 见下 |
| [`collaboration__send_message`](#collaboration__send_message) | 投递一条消息，**不触发新一轮** | ⚙ 见下 |
| [`collaboration__followup_task`](#collaboration__followup_task) | 下发后续任务，空闲时**触发一轮** | ⚙ 见下 |
| [`collaboration__interrupt_agent`](#collaboration__interrupt_agent) | 打断当前轮次，代理本身还活着 | ⚙ 见下 |
| [`collaboration__list_agents`](#collaboration__list_agents) | 列出存活代理及其状态。**v2 下拿最终答复要靠它** | ⚙ 见下 |

> 这 6 条要 `[features.multi_agent_v2] enabled = true`（默认关）**且** `non_code_mode_only = false`。后者默认为真，此时整组只给模型直接调，**PoA 完全拿不到**。`v2` profile 显式关掉了它。

#### MCP（5）

| 工具 | 作用 | 默认可得 | 并行安全 |
| --- | --- | --- | :---: |
| [`list_mcp_resources`](#list_mcp_resources) | 列出各 MCP server 提供的资源 | ⚙ 配了任意 MCP server | ✅ |
| [`list_mcp_resource_templates`](#list_mcp_resource_templates) | 列出带参数的资源模板 | ⚙ 同上 | ✅ |
| [`read_mcp_resource`](#read_mcp_resource) | 按 server 名 + URI 读取资源 | ⚙ 同上 | ✅ |
| [`mcp__probe__echo`](#mcp__probe__echo) | 示例：**有** `outputSchema` 时的渲染结果 | 探针 | 有条件 |
| [`mcp__probe__no_output_schema`](#mcp__probe__no_output_schema) | 示例：**没有** `outputSchema` 时的渲染结果 | 探针 | 有条件 |

> 除这 3 个 resource 工具外，每个 MCP server 的每个工具还会各渲染出一个 `mcp__<server>__<tool>`，数量取决于挂载了哪些 server——**包括[包自带的那些](./03-concepts.md#9-poa-包自带能力的那条路)**。

> [!CAUTION]
> **`mcp__<server>__<tool>` 这个形状不是契约，别硬编码。** 前缀取决于 `prefix_mcp_tool_names()`，命名空间会被清洗，**重名时还会加哈希后缀**。正确写法是从 `ALL_TOOLS` 里按后缀匹配，并断言只命中一个：
>
> ```js
> const matches = ALL_TOOLS.map((t) => t.name).filter((n) => /(^|__)echo__echo$/.test(n));
> if (matches.length !== 1) throw new Error(`expected exactly one echo tool, found ${matches.length}`);
> ```
>
> 那句断言不是冗余：**命名规则一变，没有断言就是静默调错工具。**

> [!TIP]
> **一般 MCP 工具的并行安全是"有条件"的，而这是唯一不动上游就能用的杠杆。**
> 判据是「工具自己声明并行安全**或**带只读标注」——**一个把自己标成只读的 MCP 工具就是并行安全的**。
> 想让派发出去的重活真的并行，把它放进标了只读的 MCP server 即可。
>
> **包自带的 server 也走这条判据**，但只走得通后半条：`ext/poa` 建的 config 里 `supports_parallel_tool_calls` 硬编码为 `false`，服务器级豁免用不了，**只能逐个工具标 `readOnlyHint`**。

### 3.2 由 provider 决定的三类

**这三类跟 feature 开关无关，是 provider 的能力决定的**，所以在具体环境里可能有也可能没有。**跑探针看 `all_tools` 是唯一可靠的判断方式。**

| 工具 | 作用 | 门槛 |
| --- | --- | --- |
| `web__run` | 联网搜索。**`supports_parallel_tool_calls` 为真**，是少数几个能让并行派发真正并行的工具 | provider 是 OpenAI 系或显式声明支持独立 web search |
| `skills__list` / `skills__read` | 列举与读取 skill | orchestrator skill provider 可用 |
| `image_gen__*` | 生成图片 | provider 能力位与 feature 同时为真 |

> [!NOTE]
> **在 code mode 会话里，"联网搜索是服务端 hosted 工具、PoA 碰不到"这个说法是反的。**
> code mode 那几个模型根本不会收到 hosted 版的搜索规格；同一条件下能出现的反而是扩展工具 `web__run`，
> 而它是**并行安全的**。

### 3.3 PoA 拿不到的

| 工具 | 挡它的机制 | 能否翻盘 |
| --- | --- | --- |
| `request_user_input` | 曝光度 `DirectModelOnly`（`spec_plan.rs:909-915`）。**只给模型，不给程序**——这就是"PoA 全程无人值守"的技术根源 | ❌ 无开关。子 agent 也问不了：`request_user_input.rs` 拒绝非 root |
| `new_context` | 曝光度 `DirectModelOnly`（`spec_plan.rs:923`） | ❌ |
| `clock` 的 `sleep` | 曝光度 `DirectModelOnly`（`handlers/sleep.rs:73`） | ❌ 要等就用 `setTimeout`，或 `exec_command` 跑 `sleep` |
| `collaboration__*`（默认配置下） | 曝光度 `DirectModelOnly`，但**由配置驱动**：`non_code_mode_only` 为真才降级（`spec_plan.rs:986-989`） | ✅ 设为 `false` 即可，见 `workflow-demos/config/v2.toml` |
| `tool_search` | **按 spec 种类整体丢弃，与曝光度无关**（`tools/src/code_mode.rs:176`） | ❌ 改任何配置都没用 |
| hosted 版联网搜索 | 同上，同一行代码。code mode 会话里根本不会下发，取而代之的是 `web__run` | ❌ 同上 |

> [!NOTE]
> **两类机制的区别有实际后果，别混。** 曝光度类的能靠配置或 feature 翻盘；spec 种类类的挡在 `code_mode_tool_definitions_for_spec()` 的这一行：
>
> ```rust
> ToolSpec::ToolSearch { .. } | ToolSpec::WebSearch { .. } => Vec::new(),
> ```
>
> 它在生成 code mode 嵌套工具定义时直接返回空，**曝光度调成什么都进不来**。

### 3.4 返回类型：12 个没有形状承诺，用之前先探一次

下表的计数是一次快照，与 §4 的声明同源；上游增删工具后**不会自动更新**，以 §4 的声明和探针输出为准。

| 返回类型 | 个数 | 是哪些 |
| --- | ---: | --- |
| 结构化（`Promise<{...}>`，字段带注释） | 18 | `exec_command`、`write_stdin`、`view_image`、`clock__curr_time`、`get_context_remaining`、memories 全部 4 个、multi_agent_v1 全部 5 个、collaboration 的 `spawn_agent` / `wait_agent` / `interrupt_agent` / `list_agents` |
| `Promise<unknown>` | 12 | `apply_patch`、`shell_command`、`update_plan`、`create_goal`、`get_goal`、`update_goal`、`request_permissions`、3 个 mcp resource 工具、collaboration 的 `send_message` / `followup_task` |

> [!IMPORTANT]
> **这些声明不产生任何运行时约束——code mode 跑的是纯 JavaScript，不是 TypeScript。** 它们只是给模型和人看的文档。
> 所以"结构化"那 18 个同样不保证字段一定在（agent 挂了、超时了，字段照样缺）；区别只在于**前者有文档承诺，后者连文档承诺都没有**。两类都要防御，后者防御成本更高。

`Promise<unknown>` 意味着**程序侧拿到的是一个没有任何形状承诺的值**：字段名只能自己试出来，上游改一次就崩。

**可操作的做法：对这 12 个，先 `text(JSON.stringify(result))` 打一次形状，再照着写解析。** 别凭猜测取字段——索引签名会让取错的字段安静地返回 `undefined`，而不是报错。

MCP 工具的返回被包成 `CallToolResult`：

```ts
// <TStructured = ...> 是带默认值的泛型参数：不写实参时就退化成"任意键值对"，
// 也就是没有形状承诺。server 声明了 outputSchema 才会填进具体字段
type CallToolResult<TStructured = { [key: string]: unknown }> = {
  _meta?: MetaObject;             // ? = 可选字段，协议层元数据，业务逻辑一般不看
  content: ContentBlock[];        // 唯一必填项：给人/模型看的内容块数组
  isError?: boolean;              // 可选！缺席不等于成功，判错要写 === true
  // 机器要用的那份；没有 outputSchema 时它可能压根不存在
  structuredContent?: TStructured;
  // 索引签名：允许出现任意其他键。代价是写错字段名不会报错，只会安静地拿到 undefined
  [key: string]: unknown;
};
```

---

## 4. 工具声明全文

> 下列声明取自 codex 实际发出的请求体，**声明骨架、工具名、字段名、类型字面量一律保持原样**，只有说明与 `//` 注释是中文。
>
> codex 为每个工具单独渲染一次 `declare const tools: { ... }`，下面保留这个原样，不是笔误。

### 执行与文件

#### `exec_command`

> 在 PTY 中运行一条命令，返回输出，或返回一个可用于后续交互的会话 ID。

```ts
declare const tools: { exec_command(args: {
  // 要执行的 shell 命令。
  cmd: string;
  // 用于 `require_escalated` 的、面向用户的审批问题；其余情况省略。
  justification?: string;
  // true 让 shell 以 -l/-i 语义运行，false 则关闭。默认 true。
  login?: boolean;
  // 输出 token 预算。默认 10000 tokens；更大的请求可能被策略封顶。
  max_output_tokens?: number;
  // 针对 `cmd` 的可复用审批前缀，仅在 `sandbox_permissions: "require_escalated"` 时有效；例如 ["git", "pull"]。
  prefix_rule?: Array<string>;
  // 单条命令级别的沙箱覆盖。默认 `use_default`；需要脱离沙箱执行时用 `require_escalated`。
  sandbox_permissions?: "use_default" | "require_escalated";
  // 要启动的 shell 可执行文件。默认为用户的默认 shell。
  shell?: string;
  // true 为该命令分配一个 PTY；false 或省略则使用普通管道。
  tty?: boolean;
  // 命令的工作目录。默认为本轮的 cwd。
  workdir?: string;
  // 返回输出前的等待时长。默认 10000 ms；有效范围为 250-30000 ms。
  yield_time_ms?: number;
}): Promise<{
  // 响应中报告了分块标识时随之返回的 chunk id。
  chunk_id?: string;
  // 命令在本次调用内结束时的进程退出码。
  exit_code?: number;
  // 输出被截断前的近似 token 数。
  original_token_count?: number;
  // 命令的输出文本，可能已被截断。
  output: string;
  // 进程仍在运行时，用于传给 write_stdin 的会话标识。
  session_id?: number;
  // 等待输出所耗的墙钟时间，单位为秒。
  wall_time_seconds: number;
}>; };
```

> [!TIP]
> **返回的是对象，不是字符串。** 要拿命令的 stdout 得取 `.output`。
> 另外注意它自己也有一个 `yield_time_ms`（默认 10 秒），**跑得久的命令要显式调大**，否则拿到的是被截断的中途输出加一个 `session_id`。

#### `write_stdin`

> 向一个已存在的 unified exec 会话写入字符，并返回最近的输出。

```ts
declare const tools: { write_stdin(args: {
  // 要写入 stdin 的字节。默认为空，此时只轮询、不写入。
  chars?: string;
  // 输出 token 预算。默认 10000 tokens；更大的请求可能被策略封顶。
  max_output_tokens?: number;
  // 正在运行的 unified exec 会话的标识。
  session_id: number;
  // 返回输出前的等待时长。非空写入默认 250 ms、上限 30000 ms；空轮询默认等待 5000-300000 ms。
  yield_time_ms?: number;
}): Promise<{
  chunk_id?: string;
  exit_code?: number;
  original_token_count?: number;
  output: string;
  session_id?: number;
  wall_time_seconds: number;
}>; };
```

#### `shell_command`

> 运行一条 shell 命令并返回其输出。
> - 使用 shell_command 函数时始终设置 `workdir` 参数。非绝对必要不要用 `cd`。

```ts
declare const tools: { shell_command(args: {
  // 要在用户默认 shell 中运行的 shell 脚本。
  command: string;
  // 用于 `require_escalated` 的、面向用户的审批问题；其余情况省略。
  justification?: string;
  // true 以 login shell 语义运行，false 则关闭。默认 true。
  login?: boolean;
  // 针对 `cmd` 的可复用审批前缀，仅在 `sandbox_permissions: "require_escalated"` 时有效。
  prefix_rule?: Array<string>;
  // 单条命令级别的沙箱覆盖。默认 `use_default`。
  sandbox_permissions?: "use_default" | "require_escalated";
  // 命令的最长运行时间。默认 10000 ms。
  timeout_ms?: number;
  // 命令的工作目录。默认为本轮的 cwd。
  workdir?: string;
}): Promise<unknown>; };
```

> [!TIP]
> **`timeout_ms` 与 `exec_command` 的 `yield_time_ms` 长得像，语义相反。** 两者都默认 10000 ms，但 `yield_time_ms` 到点只是**让出**已有输出、进程继续跑（所以才回一个 `session_id`）；`timeout_ms` 到点是**杀进程**，退出码 124（`codex-rs/core/src/exec.rs:65`）。同一条 `sleep 30`，前者给你一个可续接的句柄，后者给你一个被杀的失败。

#### `apply_patch`

> `apply_patch` 工具可用于编辑文件。这是一个 FREEFORM 工具，因此不要把补丁包进 JSON 里。

```ts
declare const tools: { apply_patch(input: string): Promise<unknown>; };
```

> [!WARNING]
> **入参是裸字符串，不是对象**——跟这里其余所有工具的调用形式都不一样。

#### `view_image`

> 需要目视检查时，查看文件系统上的一个本地图片文件。用于磁盘上已经存在的图片。

```ts
declare const tools: { view_image(args: {
  // 图片细节级别。默认 `high`；需要保留精确分辨率时用 `original`。
  detail?: "high" | "original";
  // 图片文件的本地文件系统路径。
  path: string;
}): Promise<{
  // view_image 返回的图片细节提示。
  detail: "high" | "original";
  // 已加载图片的 data URL。
  image_url: string;
}>; };
```

### 计划与目标

#### `update_plan`

> 更新任务计划。提供一段可选的说明，以及一个计划条目列表，每条含一个步骤与一个状态。
> 同一时刻最多只能有一个步骤处于 in_progress。

```ts
declare const tools: { update_plan(args: {
  // 本次计划更新的可选说明。
  explanation?: string;
  // 步骤列表
  plan: Array<{
  // 步骤状态。
  status: "pending" | "in_progress" | "completed";
  // 任务步骤文本。
  step: string;
}>;
}): Promise<unknown>; };
```

#### `create_goal`

> 只有在用户或 system/developer 指令明确要求时才创建目标。若存在未完成的目标则调用失败。

```ts
declare const tools: { create_goal(args: {
  // 必填。要开始推进的具体目标。
  objective: string;
  // 新目标的正数 token 预算。除非明确要求，否则省略。
  token_budget?: number;
}): Promise<unknown>; };
```

#### `get_goal`

> 获取本 thread 的当前目标，包括状态、各项预算、token 与耗时用量，以及剩余 token 预算。

```ts
declare const tools: { get_goal(args: {}): Promise<unknown>; };
```

#### `update_goal`

> 更新已存在的目标。只用本工具把目标标记为已达成，或标记为确实被阻塞。
> 不能用它暂停、恢复目标，也不能设置预算上限。

```ts
declare const tools: { update_goal(args: {
  // 必填。只有当目标已达成且没有剩余必需工作时才设为 `complete`。
  status: "complete" | "blocked";
}): Promise<unknown>; };
```

### 上下文与权限

#### `get_context_remaining`

> 获取当前上下文窗口的剩余 token 数。

```ts
declare const tools: { get_context_remaining(args: {}): Promise<{
  // 当前上下文窗口的剩余 token 数；不可用时为 null。
  tokens_left: number | null;
}>; };
```

#### `request_permissions`

> 向用户申请额外的文件系统或网络权限，并等待客户端授予所申请权限集合的一个子集。

```ts
declare const tools: { request_permissions(args: {
  // 来自 <environment_context> 的环境 id。省略则使用主环境。
  environment_id?: string;
  // 文件系统或网络访问申请。
  permissions: {
  // 文件系统访问申请。
  file_system?: {
  // 要授予读权限的绝对路径；不需要时省略。
  read?: Array<string>;
  // 要授予写权限的绝对路径；不需要时省略。
  write?: Array<string>;
};
  // 网络访问申请。
  network?: {
  // true 表示申请网络访问；false 或省略表示不申请。
  enabled?: boolean;
};
};
  // 可选的简短说明，解释为何需要额外权限。
  reason?: string;
}): Promise<unknown>; };
```

### 时间

#### `clock__curr_time`

> 返回当前的 UTC 时间。

```ts
declare const tools: { clock__curr_time(args: {}): Promise<{
  // 当前 UTC 时间，格式为 YYYY-MM-DD HH:MM:SS UTC。
  current_time: string;
}>; };
```

### 记忆

#### `memories__list`

> 列出 Codex memories 存储中某个路径下的直接子文件与子目录。

```ts
declare const tools: { memories__list(args: { cursor?: string; max_results?: number; path?: string; }): Promise<{ entries: Array<{ entry_type: "file" | "directory"; path: string; }>; next_cursor?: string | null; path?: string | null; truncated: boolean; }>; };
```

#### `memories__read`

> 按相对路径读取一个 Codex memory 文件，可选地从一个 1 起算的行偏移开始读，并限制返回的行数。

```ts
declare const tools: { memories__read(args: { line_offset?: number; max_lines?: number; path: string; }): Promise<{ content: string; path: string; start_line_number: number; truncated: boolean; }>; };
```

#### `memories__search`

> 在 Codex memory 文件中做子串匹配搜索，可选地归一化分隔符，或要求所有查询子串出现在同一行、或落在同一个行窗口之内。

```ts
declare const tools: { memories__search(args: { case_sensitive?: boolean; context_lines?: number; cursor?: string; match_mode?: { type: "any"; } | { type: "all_on_same_line"; } | { line_count: number; type: "all_within_lines"; }; max_results?: number; normalized?: boolean; path?: string; queries: Array<string>; }): Promise<{ match_mode: { type: "any"; } | { type: "all_on_same_line"; } | { line_count: number; type: "all_within_lines"; }; matches: Array<{ content: string; content_start_line_number: number; match_line_number: number; matched_queries: Array<string>; path: string; }>; next_cursor?: string | null; path?: string | null; queries: Array<string>; truncated: boolean; }>; };
```

#### `memories__add_ad_hoc_note`

> 在用户明确要求 Codex 记住、忘记或更新某件事之后，创建一条只追加的临时 memory 笔记。

```ts
declare const tools: { memories__add_ad_hoc_note(args: {
  // 要创建的笔记文件名，格式为 YYYY-MM-DDTHH-MM-SS-<slug>.md。其中 slug 只能使用小写 ASCII 字母、数字和连字符。
  filename: string;
  // 要原样追加到临时 memory 笔记中的 Markdown 文本。
  note: string;
}): Promise<{}>; };
```

### 子代理 v1

> [!NOTE]
> **平时不用直接碰这一组**——prelude 那一层就是为了盖住两代之间的差异。
> 列在这里是为了阅读 prelude 源码时能对上号，以及需要 prelude 没封装的能力时知道去调什么。

#### `multi_agent_v1__spawn_agent`

> 为一个范围明确的任务派生一个子代理。返回被派生代理的 id，以及面向用户的昵称（可用时）。
> 被派生的代理默认继承你当前的模型。省略 `model` 即使用这个首选默认值。

```ts
declare const tools: { multi_agent_v1__spawn_agent(args: {
  // true 把当前 thread 的历史 fork 给新代理；false 或省略则只用初始 prompt 起步。
  fork_context?: boolean;
  // 结构化输入条目。用它来传显式的 mention（例如 app:// 形式的 connector 路径）。
  items?: Array<{
  // type 为 audio 时的音频 data URL。
  audio_url?: string;
  // type 为 image 时的图片 URL。
  image_url?: string;
  // type 为 skill 或 mention 时的显示名。
  name?: string;
  // type 为 local_image/local_audio/skill 时的路径；type 为 mention 时则是结构化的 mention 目标。
  path?: string;
  // type 为 text 时的文本内容。
  text?: string;
  // 输入条目类型：text、image、local_image、audio、local_audio、skill 或 mention。
  type?: string;
}>;
  // 给新代理的初始纯文本任务。message 与 items 二选一。
  message?: string;
  // 新代理的模型覆盖项。除非确需显式覆盖，否则省略。
  model?: string;
  // 新代理的推理强度覆盖项。省略则继承父级的推理强度。
  reasoning_effort?: string;
  // 新代理的服务层级覆盖项。除非明确要求，否则省略。
  service_tier?: string;
}): Promise<{
  // 被派生代理的 thread 标识。
  agent_id: string;
  // 被派生代理的、面向用户的昵称（可用时）。
  nickname: string | null;
}>; };
```

> **没有 `system_prompt` 这一项**——子 agent 的系统提示词设不了，`message` 是程序唯一的控制面。

#### `multi_agent_v1__wait_agent`

> 等待代理进入终态。completed 状态里可能带上代理的最终消息。超时时返回空的 status。

```ts
declare const tools: { multi_agent_v1__wait_agent(args: {
  // 要等待的代理 id。传入多个 id 表示等最先结束的那一个。
  targets: Array<string>;
  // 超时时间，单位毫秒。默认 30000，最小 10000，最大 3600000。优先用较长的等待以避免忙轮询。
  timeout_ms?: number;
}): Promise<{
  // 以代理 id 为键的终态。
  status: { [key: string]: "pending_init" | "running" | "interrupted" | "shutdown" | "not_found" | { completed: string | null; } | { errored: string; }; };
  // 本次等待调用是否是在任何代理进入终态之前因超时而返回的。
  timed_out: boolean;
}>; };
```

> [!WARNING]
> **一次等多个是陷阱**——它会反复返回最先完成的那个。所以收 N 个结果需要 N 次单目标等待，
> 这正是 `collectAll` 在 v1 分支写成串行 for 循环的原因。

#### `multi_agent_v1__send_input`

> 给一个已存在的代理发送消息。用 interrupt=true 可以立刻改变它的工作方向。

```ts
declare const tools: { multi_agent_v1__send_input(args: {
  // true 会打断当前任务并立即处理这条消息；false 或省略则把它排队。
  interrupt?: boolean;
  // 结构化输入条目。用它来传显式的 mention。
  items?: Array<{
  audio_url?: string;
  image_url?: string;
  name?: string;
  path?: string;
  text?: string;
  type?: string;
}>;
  // 发给代理的旧式纯文本消息。message 与 items 二选一。
  message?: string;
  // 要发消息的代理 id（来自 spawn_agent）。
  target: string;
}): Promise<{
  // 已排队的输入提交的标识。
  submission_id: string;
}>; };
```

#### `multi_agent_v1__resume_agent`

> 按 id 恢复一个此前已关闭的代理，使它能重新接收 send_input 与 wait_agent 调用。

```ts
declare const tools: { multi_agent_v1__resume_agent(args: {
  // 要恢复的代理 id。
  id: string;
}): Promise<{ status: "pending_init" | "running" | "interrupted" | "shutdown" | "not_found" | { completed: string | null; } | { errored: string; }; }>; };
```

#### `multi_agent_v1__close_agent`

> 在不再需要时关闭一个代理及其所有仍处于打开状态的后代，并返回目标代理在收到关闭请求之前的状态。
> **已完成的代理在被关闭前仍保持打开，并计入并发上限。**

```ts
declare const tools: { multi_agent_v1__close_agent(args: {
  // 要关闭的代理 id（来自 spawn_agent）。
  target: string;
}): Promise<{
  // 在收到关闭请求之前观测到的代理状态。
  previous_status: "pending_init" | "running" | "interrupted" | "shutdown" | "not_found" | { completed: string | null; } | { errored: string; };
}>; };
```

### 子代理 v2

#### `collaboration__spawn_agent`

> 派生一个代理去处理指定的任务。如果你当前的任务是 `/root/task1`，而你用 task_name "task_3" 调用它，
> 那么该代理的规范任务名就是 `/root/task1/task_3`。
> 被派生的代理会拥有和你一样的工具，**也能派生它自己的子代理**。
> 传 `fork_turns="none"` 不会把任何周边上下文传给子代理；`fork_turns="all"` 会全部提供。

```ts
declare const tools: { collaboration__spawn_agent(args: {
  // 可选的 fork 轮数。默认 `all`。可用 `none`、`all`，或像 `3` 这样的正整数字符串。
  fork_turns?: string;
  // 给新代理的初始纯文本任务。
  message: string;
  // 新代理的模型覆盖项。除非确需显式覆盖，否则省略。
  model?: string;
  // 新代理的推理强度覆盖项。省略则继承父级的推理强度。
  reasoning_effort?: string;
  // 新代理的任务名。只使用小写字母、数字和下划线。
  task_name: string;
}): Promise<{
  // 被派生代理的规范任务名。
  task_name: string;
}>; };
```

> [!CAUTION]
> **返回的 `task_name` 已经是全限定路径**（形如 `/root/scan_0`）。
> 如果按直觉再拼一次前缀去和 `list_agents` 比对，**结果是全部 agent 完成、一个都收不到，
> 而且不报任何错**——只是超时后返回一堆 `null`。
> 这也是 `collectAll` 的 v2 分支用 `endsWith` 匹配的原因。
>
> 另外注意这里的参数结构会**拒绝未知字段**：自作主张塞一个不存在的参数会直接抛异常。

#### `collaboration__wait_agent`

> 等待任意存活代理的信箱更新。**它不返回内容本身**；返回的是哪些代理有更新的摘要。

```ts
declare const tools: { collaboration__wait_agent(args: {
  // 超时时间，单位毫秒。默认 30000，最小 10000，最大 3600000。
  timeout_ms?: number;
}): Promise<{
  // 简短的等待摘要，不含代理的最终内容。
  message: string;
  // 本次等待调用是否因为超时前没有任何信箱更新而返回。
  timed_out: boolean;
}>; };
```

> **v2 下拿最终答复要靠 `list_agents`**，这个只是个"有动静了"的信号。

#### `collaboration__send_message`

> 给一个已存在的代理发送消息。消息会被及时投递。**不会触发新的一轮。**

```ts
declare const tools: { collaboration__send_message(args: {
  // 要在目标代理上排队的消息文本。
  message: string;
  // 要发消息的相对任务名或规范任务名（来自 spawn_agent）。
  target: string;
}): Promise<unknown>; };
```

#### `collaboration__followup_task`

> 给一个已存在的非 root 目标代理下发后续任务，**若它处于空闲状态则触发一轮**。

```ts
declare const tools: { collaboration__followup_task(args: {
  // 要发送给目标代理的消息文本。
  message: string;
  // 要下发后续任务的代理 id 或规范任务名（来自 spawn_agent）。
  target: string;
}): Promise<unknown>; };
```

> 与 `send_message` 的区别只在"触不触发新一轮"。`sendAndWait` 在 v2 下用的是这一个。

#### `collaboration__interrupt_agent`

> 打断某个代理当前的轮次（如果有），并返回它之前的状态。**该代理仍可继续接收消息与后续任务。**

```ts
declare const tools: { collaboration__interrupt_agent(args: {
  // 要打断的代理 id 或规范任务名（来自 spawn_agent）。
  target: string;
}): Promise<{
  // 在打断请求被处理之前观测到的代理状态。
  previous_status: "pending_init" | "running" | "interrupted" | "shutdown" | "not_found" | { completed: string | null; } | { errored: string; };
}>; };
```

> **prelude 没有封装这个。** 需要打断能力时直接调它。

#### `collaboration__list_agents`

> 列出当前 root thread 树中存活的代理。可选地按任务路径前缀过滤。

```ts
declare const tools: { collaboration__list_agents(args: {
  // 任务路径前缀过滤器，末尾不带斜杠。省略则列出所有存活的代理。
  path_prefix?: string;
}): Promise<{
  // 当前 root thread 树中可见的存活代理。
  agents: Array<{
  // 代理的规范任务名（可用时），否则为代理 id。
  agent_name: string;
  // 代理最后已知的状态。
  agent_status: "pending_init" | "running" | "interrupted" | "shutdown" | "not_found" | { completed: string | null; } | { errored: string; };
}>;
}>; };
```

### MCP

#### `list_mcp_resources`

> 列出各 MCP server 提供的资源。可能的话，优先使用资源而不是网页搜索。

```ts
declare const tools: { list_mcp_resources(args: {
  // 上一次调用返回的不透明游标；取第一页时省略。
  cursor?: string;
  // MCP server 名称。省略则列出所有已配置 server 的资源。
  server?: string;
}): Promise<unknown>; };
```

#### `list_mcp_resource_templates`

> 列出各 MCP server 提供的资源模板。

```ts
declare const tools: { list_mcp_resource_templates(args: {
  cursor?: string;
  server?: string;
}): Promise<unknown>; };
```

#### `read_mcp_resource`

> 给定 server 名称与资源 URI，从某个 MCP server 读取指定的资源。

```ts
declare const tools: { read_mcp_resource(args: {
  // 与配置中完全一致的 MCP server 名称。必须与 list_mcp_resources 返回的 'server' 字段匹配。
  server: string;
  // 要读取的资源 URI。必须是 list_mcp_resources 返回的 URI 之一。
  uri: string;
}): Promise<unknown>; };
```

#### `mcp__probe__echo`

> 示例工具：按给定的重复次数回显一条消息。**展示声明了 `outputSchema` 时的渲染结果。**

```ts
declare const tools: { mcp__probe__echo(args: {
  // 要回显的文本。
  message: string;
  // 重复多少次。
  times?: number;
}): Promise<CallToolResult<{
  // 实际使用的重复次数。
  count: number;
  // 被回显的文本。
  echoed: string;
}>>; };
```

#### `mcp__probe__no_output_schema`

> 示例工具：**没有声明 `outputSchema`**，返回的是裸 `CallToolResult`。

```ts
declare const tools: { mcp__probe__no_output_schema(args: { q: string; }): Promise<CallToolResult>; };
```

---

[← 模式库](./06-patterns.md) · [返回目录](./index.md) · 下一篇：[边界与限制](./08-limits.md)
