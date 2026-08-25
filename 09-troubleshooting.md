---
title: 故障排查
description: 症状 → 原因 → 处置；两个不报错的坑单独标出
---

# 故障排查

[← 边界与限制](./08-limits.md) · [返回目录](./index.md)

**这一篇按可观察到的现象排列。** 遇到问题从这儿查。

---

## 0. 先看这两个：它们不报错

其余所有问题都会以可见的方式失败。**只有这两个会安静地产出一个错误的结果**，所以先排除它们。

> [!CAUTION]
> ### ① shell 的报错文本被当成数据喂给了 agent
>
> **现象**：结果看起来完全正常，但内容不对。或者 agent 一本正经地分析了一个叫 `No such file or directory` 的"目录"。
>
> **原因**：shell 失败时诊断信息和正常输出**走同一个通道**。`shellLines` 不传 `validate` 就会把它们当数据传下去。
>
> **处置**：
> ```js
> const folders = await shellLines("ls -d */ | sed 's#/$##'", {
>   validate: (line) => SAFE_NAME.test(line),   // ← 必须包一层箭头函数
> });
> ```
> **自查**：程序里每一处 `shellLines` 都传了 `validate` 吗？

> [!CAUTION]
> ### ② 忘了 `await`
>
> **现象**：某段代码好像没执行，也不报错。或者结果里少了一部分，但没有任何异常。
>
> **原因**：脚本求值结束时沙箱生命周期即终止，**未 `await` 的 promise 被静默丢弃**。不报错、不警告，只是那部分工作没有发生。
>
> **处置**：收口前检查**每个 `tools.` 调用**与**每个 prelude 函数调用**前面都有 `await`。注意 `runBatch` / `collectAll` / `sendAndWait` / `closeAll` / `shellLines` 全都是 async。

---

## 1. 症状速查表

| 现象 | 为什么 | 怎么办 |
| --- | --- | --- |
| 探针输出 `backend: null` | profile 给错了 | 显式用 `v1-forced` 或 `v2`，**别用默认值** |
| `tools.xxx is not a function` | 那个工具在这套配置下不存在 | 跑探针看 `all_tools`；开头加 `requireAgents()` 让它早点失败 |
| 报错出现在 JSON-RPC 方法层，说没有这个方法 | 用到的 codex 二进制是上游安装版，没有 PoA 那个入口 | 在 `.env` 里**显式设 `CODEX_BIN`**，见下方 §2 |
| 程序跑起来就没动静，等很久 | 派了 agent 的活本来就慢，而且**中途没法取消** | 调试时把 `yield_time_ms` 改成 60000、规模常量改成 1，跑通后再放大 |
| 程序没等到第一个 agent 就返回了 | 首行 pragma 没生效，沿用了 10 秒默认值 | 确认 `// @exec:` **真的在第 1 行** |
| `ls` 的结果里混进了报错文本 | 见 [§0 ①](#0-先看这两个它们不报错) | `shellLines` 传 `validate` |
| agent 回的 JSON 偶尔解析失败 | **没有任何机制能强制它按格式回答** | `parseJsonReply` + 把解析失败的原文留下来 |
| 结果全是 `null` | 超时未达终态，或者 v2 下标识匹配不上 | 先看 `timeoutMs` 是否小于 `yield_time_ms`；v2 下注意 `task_name` 是全限定路径 |
| `Promise.race` 不按完成先后返回 | **程序侧的等待调用是串行的**（agent 是真并行的） | 别设计成流水线，改成"全部派出去 → 一次收齐" |
| 派到第 4 个 agent 就失败 | 有并发上限，且 **v2 把 root 也算一个** | `concurrency` 按"上限减 1"估，见 §3 |
| 中途 `notify()` 打的进度看不见 | 那个函数走的是另一条通道，客户端收不到 | 攒一个 `log` 数组，最后随结果一起 `text()` 出去 |
| 用了 `yield_control()`，只拿到前半段 | 没有续跑手段，后半段回不来 | 别用它做流式，把 `yield_time_ms` 调大 |
| shell 命令找错了目录 | 忘了 `--cwd` | `./run.sh <程序> <profile> --cwd <目标目录>` |
| 改了 `.home/` 下的配置，没生效 | 那是**每次运行都重新生成的**产物 | 改 `config/*.toml` 模板 |
| 命令行前缀 `VAR=x ./run.sh ...` 不起作用 | `.env` 是无条件赋值，会**覆盖**已导出的同名变量 | 改 `.env`，或把那一行注释掉 |
| 结果里少了一部分，也没报错 | 见 [§0 ②](#0-先看这两个它们不报错) | 检查 `await` |
| `refusing to run: required MCP server(s) unavailable: X` | 包申报的 server 没起来，**或起来了但一个工具都没报** | 先手动跑一遍 `shell` 那条命令行；注意解出来的文件权限固定 `0644`，命令行必须自带解释器 |
| 包跑起来报 `runBatch is not defined` | 包的 `entry` 是**原样提交**的，不拼 prelude | 把 prelude 抄进包里，见[编写指南 §13](./05-writing.md#13-从-js-到-poa要改的五件事) |
| 包里的工具调用报 `tools.mcp__x__y is not a function` | 工具名被硬编码了；前缀会被清洗、重名时加哈希后缀 | 从 `ALL_TOOLS` 按后缀匹配 + 断言只命中一个 |
| 包内某个工具调用被拒，但工具明明在 `ALL_TOOLS` 里 | 那个工具**没标注**→ 需要审批 → 而四个 profile 都是 `approval_policy = "never"`，于是毫秒级自动 decline。**客户端全程收不到请求**，看起来像"人拒绝了"，其实人没被问 | 先用 `CODEX_RPC_DEBUG=1` 确认客户端**确实一条请求都没收到**（见 [§6 ③](#6-诊断动作清单)），坐实是"没被问"而不是"被拒了"；然后在 server 的 `tools/list` 里给每个工具写 `annotations: { readOnlyHint: true, destructiveHint: false, openWorldHint: false }`，三个都写 |
| **跑包时整条命令卡死，`yield_time_ms` 也不救**（issue #32） | 同样是工具没标注，但 `approval_policy` **不是** `never`。审批回执登记在 `active_turn` 上，而 cell 从不建 `active_turn` → 客户端答了也没用，答案被 core 丢弃，`rx_response` 永不返回 | 只能杀进程。根治见 issue #32；**包作者侧的唯一解是把标注补齐**——它同时买到"躲开这个死锁"和"并行安全"两件事 |
| `codex exec --poa` 报错并 `exit(2)` | manifest 在**本地**就被校验掉了，一个会话都不会起 | 照错误文案改；`network` 传输、非 `.js/.mjs` 的 `entry`、`poa_api_version != 1`、重名 server 都是这里拒的。⚠️ 报错文案里可能出现 `packageBase64` 这个字段名——**线上并不存在这个字段**（实际是 `package`），是上游没改干净的残留 |
| `codex exec --poa` 报 `--poa cannot be combined with a subcommand.` | `--poa` 与 `resume` / `review` 子命令互斥，也与位置参数 prompt 互斥 | 包是一次 cell，不是一轮采样循环，两者不能叠 |
| 绕开 `run.sh` 跑包，报没有 `exec` 工具 / cell 起不来 | `code_mode` feature **默认关闭**，`--poa` 不会替你打开 | 当前 `CODEX_HOME` 的 `config.toml` 里加 `[features.code_mode] enabled = true`，或直接用 `run.sh` |

---

## 2. `CODEX_BIN` 相关的两个失败

这两个都会指向错误的层次，值得单独说。

> [!CAUTION]
> **① `CODEX_BIN` 指向一个不存在的路径时，解析看起来是成功的。**
> 二进制解析拿到 `CODEX_BIN` 就**原样返回，不校验文件是否存在**。
> 于是 `.env` 里一个过时路径——重建到 `release/`、换了克隆位置、删过 `target/`——会一路通过，
> 最后报出的却是启动进程失败。

> [!CAUTION]
> **② `cargo` 不在 `PATH` 上时，会静默落到上游安装版。**
> 查找顺序是 `CODEX_BIN` → `cargo metadata` 的 target 目录 → `PATH`。
> 用 rustup 装 cargo 但没在 shell 启动文件里引入时，中间那档整个失效，
> 直接落到 `PATH` 上的 `codex`——而那通常**没有 PoA 用的那个 RPC 方法**。
> **报错会出现在协议层，跟真正的原因隔着好几层。**

**处置**：在 `.env` 里**显式写 `CODEX_BIN`**，并确认那个路径下**同时有 `codex-code-mode-host`**（codex 是按自己所在目录找它的）。

---

## 3. 并发相关

| 现象 | 原因 |
| --- | --- |
| 派发到第 N 个失败 | 超过可用名额。**超订不会排队，直接失败** |
| v1 下跑了几轮之后开始派不出 agent | **已完成的 agent 不关掉仍占名额**，`closeAll` 漏了 |
| 同一段程序在 `v2` profile 下比 `v1-forced` 少派几个 | v2 的名额要**减掉 root 线程** |

**可用名额**：

| 后端 | 默认可用 |
| --- | --- |
| v1 | 6 |
| v2 | 上限减 1；默认上限 4 → **实际 3** |

`concurrency` 按「可用名额减 1」取最保险。

---

## 4. 十五条已知的坑（全量）

这是踩出来的完整清单，每条都花过一个调试周期。**8–15 来自 demo 06–10 那批能力样本**，各自的复现程序写在「已在哪里处理」一列。

| # | 现象 | 已在哪里处理 | 后果 |
| ---: | --- | --- | --- |
| 1 | v1 一次等多个目标时**反复报告第一个完成者** | `collectAll` 的 v1 分支改用串行单目标等待 | 收不齐结果 |
| 2 | v2 派发返回的 `task_name` **已经是全限定路径** | `collectAll` 的 v2 分支用 `endsWith` 匹配 | **静默零收集**，无报错 |
| 3 | **shell 失败的诊断信息与正常输出同通道** | `shellLines` 的 `validate` 过滤器 | **静默产出错误结果** |
| 4 | `exec` 默认让出时间仅 10 秒 | 首行 `// @exec:` pragma，由 runner 自动提到第 1 行 | 长程序被截断 |
| 5 | v2 并发上限**含 root 线程** | `v2` profile 把它调到 8 | 默认值 4 实际只剩 3 个名额 |
| 6 | 嵌套工具调用默认串行 | 无法规避，只能改设计 | `Promise.race` 不流式 |
| 7 | 模型清单配置项必须写在所有 `[table]` 之前 | `v1-forced` profile 里有行注释标出 | TOML 解析失败 |
| 8 | **`exec_command` 给不给 `session_id`，取决于进程在 `yield_time_ms` 内有没有结束** | demo 07 开 PTY 起交互式 shell | **判据不是 `tty`**，是"还活着就给会话、已退出就只给 `exit_code`"。`tty` 之所以相关是间接的：不带 PTY 时 stdin 是个空管道，**读 stdin 的命令**（比如一个交互式 shell）立刻拿到 EOF 就退出了，于是没有会话可给。`sleep 60` 不带 tty 照样返回 `session_id` |
| 9 | **空写轮询会把 `yield_time_ms` 向上钳到 5000ms** | demo 07 按这个值排期 | 要更小**不报错也不提示**，调用就是隔 5 秒才落地一次。非空写默认 250ms |
| 10 | **命令失败 ≠ 工具失败** | demo 07 一律看 `exit_code` | `exec_command` 对一个非零退出的命令返回 `ok`，诊断信息和真数据走同一个 `output` 字段。**判据永远是 `exit_code`，不是文本** |
| 11 | **exec 策略按命令内容拒绝，与沙箱无关** | demo 08 把三道门分开报告 | `rm -rf` 即便在 `danger-full-access` 下也被拒（"rm -f style commands are not permitted"），同路径的 `rm -r` 却放行。**"与沙箱无关"是准确的，但"拒绝"这个结果依赖 `approval_policy = never`**——四个 profile 都是 never 所以观测到的就是拒绝；换成 `on-request` 会变成弹审批 |
| 12 | **`memories__list` 在从没写过的库上直接失败** | demo 09 先写一条再列 | 根节点在第一条笔记之前不存在，于是报 `path '' was not found` 而不是返回空列表。临时笔记落在 `extensions/ad_hoc/notes/` 下，不在根上 |
| 13 | **工具描述里的约束不是服务端校验** | demo 09 自己校验 | `update_plan` 文档写着"最多一个 `in_progress`"，然后**照收两个不吭声**。描述是写给模型看的，程序需要哪个不变量就自己保证 |
| 14 | **出错的 agent 仍然是一次成功的工具调用** | demo 10 先判终态种类 | 终态是个 tagged union（`{completed}` / `{errored}`），`wait_agent` 两种都返回 `ok`，provider 的报错文本就坐在答复该在的位置。按工具状态判会让**一个不通的 provider 看起来像行为完全符合文档的后端**——demo 10 就曾对着死端点跑完全程，拿两个空列表报出 `same_agent_twice: true` |
| 15 | **`request_permissions` 在 `approval_policy = never` 下自问自答** | demo 09 仍然给它套超时 | 既不挂起也不报错：毫秒级返回一个 **JSON 字符串**（不是对象），里面每项权限都是 `null`。套超时是因为这是宿主行为，**换个客户端完全可能真的阻塞——而一个阻塞的非并行工具会一直攥着那把共享写锁，把后面每一次调用都拖住** |

**最危险的是第 2、3、13、14 条**——它们不报错，产出的东西看起来完全正常。第 14 条后果最重：**它让"什么都没发生"长得像"经过验证的否定结论"**。任何一条从子 agent 答复里得出的结论，都要先确认终态种类；从"没有"（没有孙 agent、没有匹配项）得出的结论，更要确认，而不是更不用。

---

## 5. 配置踩坑

| # | 坑 |
| --- | --- |
| 1 | **feature 有两种写法，且不通用。** `[features] memories = true` 是布尔形式；`[features.current_time_reminder] enabled = true` 是表形式。写错形式会报「期望布尔却拿到 map」，**而且报错行号指向的是别的表** |
| 2 | **记忆工具需要两个开关。** `[features] memories = true` 和 `[memories] dedicated_tools = true` 必须同时给，**只给一个得到 0 个工具** |
| 3 | **TOML 表边界。** 往一个已有表的配置尾部追加顶层键，会被折进那个表 |
| 4 | **`v1` profile 是个陷阱。** 名字写着 v1，多数模型下拿到 0 个 agent 工具 |

---

## 6. 诊断动作清单

按这个顺序做，能定位绝大多数问题：

**① 跑探针。** 0.3 秒，不花钱。

```bash
cd workflow-demos
./run.sh demos/00_probe.js v1-forced
```

看 `backend`（不能是 `null`）和 `all_tools`（程序用到的工具是否都在里面）。

**② 加 `--raw` 看原始返回值。** 调试协议层问题时这是唯一手段。

```bash
./run.sh demos/90_hello.js v1-forced --raw
```

**③ 怀疑是审批问题时，用 `CODEX_RPC_DEBUG=1` 看服务端到底问没问。**

```bash
CODEX_RPC_DEBUG=1 ./run.sh demos/00_probe.js v1-forced
```

运行器会把**每一条服务端发回来的请求**连同参数打到 stderr。这是区分"人拒绝了"和"人根本没被问"的唯一手段——`approval_policy = never` 下 core 在毫秒级自己 decline，客户端**一条请求都收不到**，屏幕上什么都不会出现；而如果请求确实到了客户端，这里就能看见它的 `_meta` 里带没带 `codex_approval_kind`（带的是工具审批，运行器 `accept`；不带的是 server 真的在问人，运行器 `decline`）。

**④ 把规模压到 1，超时压到 60000。** 让一次失败只花一个 agent 的时间。

**⑤ 把 `log` 数组交出去。** 在 `catch` / `finally` 里也要 `text()`，否则那次运行的全部信息都没了。

```js
const log = [];
try {
  // …
} catch (err) {
  log.push(`FATAL: ${String(err)}`);
} finally {
  text(JSON.stringify({ result, log, elapsed_ms: Date.now() - t0 }, null, 2));
}
```

**⑥ 用 `requireAgents()` 早点失败。** 它的异常消息里会列出当前全部可用工具名——**这是最快的排错入口**。

---

## 7. 三条不是 bug 的现状

下面这三种情况不必排查，这就是当前的行为：

1. **没有续跑**：程序一旦超时返回，剩下的部分回不来了。长任务只能靠把超时调大硬扛
2. **没有取消**：跑飞了只能杀进程
3. **无人值守**：程序中途没法向人提问

完整说明见[边界与限制](./08-limits.md)。

---

[← 边界与限制](./08-limits.md) · [返回目录](./index.md)
