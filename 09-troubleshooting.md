# 故障排查

本篇按可观察到的现象排列。

---

## 0. 不报错的失败

多数问题会以可见的方式失败。以下三种不会——它们产出的结果形式完全正常，因此先排除它们。（§4 的清单里另有三条静默项，第 2、13、14 条。）

### ① shell 的报错文本被当成数据传给了 agent

**现象**：结果形式正常，但内容不对。或者 agent 分析了一个名为 `No such file or directory` 的"目录"。

**原因**：shell 失败时诊断信息和正常输出走同一个通道。`shellLines` 不传 `validate` 就会把它们当数据传下去。

**处置**：

```js
const folders = await shellLines("ls -d */ | sed 's#/$##'", {
  validate: (line) => SAFE_NAME.test(line),   // ← 必须包一层箭头函数
});
```

**自查**：确认程序里每一处 `shellLines` 都传了 `validate`。

### ② 忘了 `await`

**现象**：某段代码未执行，且不报错。或者结果里少了一部分，但没有任何异常。

**原因**：脚本求值结束时沙箱生命周期即终止，未 `await` 的 promise 被静默丢弃。不报错、不警告，只是那部分工作没有发生。

**处置**：收口前检查每个异步调用前面都有 `await`。`runBatch` / `collectAll` / `spawnAgent` / `spawnMany` / `sendAndWait` / `closeAll` / `shellLines` / `mapLimit` / `callTool` / `timed` 与全部 `tools.` 调用都是 async；`requireAgents` / `parseJsonReply` / `hasTool` / `shapeOf` 是同步的。

### ③ v2 下批量超过驻留名额，先完成的结果被丢弃

**现象**：只在 v2 后端出现。派出去的数量对，收回来的结果里有一部分是 `null`，没有异常也没有超时迹象；重跑时变成 `null` 的还不一定是同几个。

**原因**：v2 的上限计的是同时驻留数，驻留位满时宿主会驱逐一个已完成的 agent 来腾位置。被驱逐的 agent 随即从 `list_agents` 里消失，而 v2 的收口正是靠它读状态和答复，于是那一项被填成 `null`。`runBatch` 是先全部派完再一次收齐的，派发阶段发生的驱逐无法补救。

**处置**：条目数超过可用名额时改成分批，每批收齐再派下一批，见《05-writing.md》§6.3。

---

## 1. 症状速查表

| 现象 | 为什么 | 怎么办 |
| --- | --- | --- |
| 探针输出 `backend: null` | 模型自报 v2，而 v2 的 agent 工具默认进不了 code mode，于是一个都没有 | 加 `[features.multi_agent_v2] enabled = true` 且 `non_code_mode_only = false`，或换一个 v1 的模型。见《02-quickstart.md》§2 |
| `tools.xxx is not a function` | 那个工具在这套配置下不存在 | 跑探针看 `all_tools`；开头加 `requireAgents()` 让它早点失败 |
| `unexpected argument '--poa'`，或报错出现在 JSON-RPC 方法层说没有这个方法 | 调到的是 `PATH` 上的上游安装版 codex | 用完整路径调本项目那个二进制，见下方 §2 |
| 程序长时间无输出 | 派 agent 的任务本身耗时较长，且中途没法取消 | 调试时把 `yield_time_ms` 改成 60000、规模常量改成 1，跑通后再放大 |
| 程序没等到第一个 agent 就返回了 | 首行 pragma 没生效，沿用了 10 秒默认值 | 确认 `// @exec:` 在第 1 行 |
| `ls` 的结果里混进了报错文本 | 见 [§0 ①](#0-不报错的失败) | `shellLines` 传 `validate` |
| agent 回的 JSON 偶尔解析失败 | 没有任何机制能强制它按格式回答。一个高频成因是它把格式模板复述了一遍，两段 `{}` 被贪婪匹配连成一块 | prompt 里加上"不要复述格式说明"；收口用 `parseJsonReply` 并留下解析失败的原文 |
| 结果全是 `null` | 超时未达终态，或者 v2 下标识匹配不上 | v1 是逐个串行等待，确认 N × `timeoutMs` 仍在 `yield_time_ms` 之内；v2 下注意 `task_name` 是全限定路径 |
| 结果里只有一部分是 `null`，没有异常 | v2 下超过驻留名额，先完成的被驱逐 | 见 [§0 ③](#0-不报错的失败)，改成分批 |
| `Promise.race` 不按完成先后返回 | 程序侧的等待调用是串行的（agent 是真并行的） | 不要设计成流水线，改成"全部派出去 → 一次收齐" |
| 派到第 N 个 agent 就失败 | 超过可用名额 | 分批派发，v1 每批之后 `closeAll`，见 §3 |
| 中途 `notify()` 打的进度看不见 | 那个函数走的是另一条通道，客户端收不到 | 攒一个 `log` 数组，最后随结果一起 `text()` 出去 |
| 用了 `yield_control()`，只拿到前半段 | 没有续跑手段，后半段回不来 | 不要用它做流式，把 `yield_time_ms` 调大 |
| shell 命令找错了目录 | 忘了 `-C` | `codex exec --poa <包> -C <目标目录>` |
| 结果里少了一部分，也没报错 | 见 [§0 ②](#0-不报错的失败) | 检查 `await` |
| `refusing to run: required MCP server(s) unavailable: X` | 包申报的 server 没起来，或起来了但一个工具都没报 | 先手动跑一遍 `shell` 那条命令行；注意解出来的文件权限固定 `0644`，命令行必须自带解释器 |
| 报 `runBatch is not defined` | `entry` 是原样提交的，prelude 那一层不抄进来就不存在 | 把《07-api-reference.md》§2.5 整段抄在首行 pragma 之后 |
| 包里的工具调用报 `tools.mcp__x__y is not a function` | 工具名被硬编码了；前缀会被清洗、重名时加哈希后缀 | 从 `ALL_TOOLS` 按后缀匹配 + 断言只命中一个 |
| 整条命令卡死，`yield_time_ms` 也不救 | 某个 MCP 工具没写 annotations，于是被判定为需要审批。审批回执要登记在一次模型回合上，而 cell 从不建立回合，回执无处投递，那条 RPC 永不返回 | 只能杀进程。处置是把标注补齐，见《10-packaging.md》§4.2；它同时解决挂死和并行安全两件事 |
| 工具在 `ALL_TOOLS` 里，调用却像是被人拒绝了 | 同一个成因的另一种表现，取决于调用发生在 cell 里还是子 agent 里 | 处置同上：把三条 annotations 补齐 |
| `codex exec --poa` 报错并 `exit(2)` | manifest 在本地就被校验掉了，一个会话都不会起 | 照错误文案改；`network` 传输、非 `.js/.mjs` 的 `entry`、`poa_api_version != 1`、重名 server 都是这里拒的。⚠️ 报错文案里可能出现 `packageBase64` 这个字段名，实际的字段名是 `package` |
| `codex exec --poa` 报 `--poa cannot be combined with a subcommand.` | `--poa` 与 `resume` / `review` 子命令互斥，也与位置参数 prompt 互斥 | 包是一次 cell，不是一轮采样循环，两者不能叠 |
| 报没有 `exec` 工具 / cell 起不来 | `code_mode` feature 默认关闭，`--poa` 不会自动打开它 | 当前 `CODEX_HOME` 的 `config.toml` 里加 `[features.code_mode] enabled = true` |
| `Not inside a trusted directory`，`exit(1)` | `codex exec` 默认拒绝在 git 仓库之外运行，判的是 `-C` 给的工作目录 | 加 `--skip-git-repo-check`，或把 `-C` 指到一个仓库里 |

---

## 2. 调错了二进制

`PATH` 上的 `codex` 通常是上游安装版，它没有 `--poa`，也不提供那个 RPC 方法。报错落在参数解析或协议层，跟真正的原因隔着好几层。

**处置**：用完整路径调本项目那个二进制，并确认同一目录下有 `codex-code-mode-host`——codex 是按自己所在目录找它的。

```bash
bin=~/codex-poa/codex
LC_ALL=C grep -aqm1 thread/codeMode/exec "$bin" && echo "OK" || echo "这不是本项目的 codex"
ls "$(dirname "$bin")/codex-code-mode-host"
```

---

## 3. 并发相关

| 现象 | 原因 | 怎么办 |
| --- | --- | --- |
| 派发到第 N 个抛 `AgentLimitReached` | 超过可用名额，超订不会排队 | 分批派发 |
| v1 下跑了几轮之后开始派不出 agent | v1 计的是累计派出数，已完成的 agent 不关掉仍占名额 | 每批之后 `closeAll` |
| 收回来的结果有一部分是 `null`，无异常 | v2 驻留位满时驱逐已完成的 agent | 每批收齐再派下一批 |
| 把 `concurrency` 调小仍然失败 | 它限的是同时在途的派发调用数，不影响累计数与驻留数 | 改分批结构，不是改这个参数 |

可用名额：v1 默认 6 且 root 不占；v2 是 `max_concurrent_threads_per_session` 减 1（默认 4 → 3）。语义差异与分批写法见《05-writing.md》§6.3。

---

## 4. 十五条已知问题（全量）

以下是已确认的完整清单。"怎么处理"一列里凡是写着 `collectAll` / `shellLines` 的，都由 prelude 那一层代为处理（《07-api-reference.md》§2.5）；不抄那一层就要自己实现一遍，写法见《05-writing.md》§12.2。

| # | 现象 | 怎么处理 | 后果 |
| ---: | --- | --- | --- |
| 1 | v1 一次等多个目标时反复报告第一个完成者 | 改用串行单目标等待，`collectAll` 的 v1 分支即如此 | 收不齐结果 |
| 2 | v2 派发返回的 `task_name` 已经是全限定路径 | 用 `endsWith` 匹配，不要再拼一次前缀 | 静默零收集，无报错 |
| 3 | shell 失败的诊断信息与正常输出同通道 | `shellLines` 的 `validate` 过滤器，或手写一遍行过滤 | 静默产出错误结果 |
| 4 | `exec` 默认让出时间仅 10 秒 | 首行 `// @exec:` pragma 调大 `yield_time_ms`，且它必须在第 1 行 | 长程序被截断 |
| 5 | v2 上限含 root 线程，且计的是同时驻留数而非累计数 | 把 `max_concurrent_threads_per_session` 调大，并写成分批 | 默认配置 4 只剩 3；驻留位满时已完成的 agent 被驱逐，结果静默变 `null` |
| 6 | 嵌套工具调用默认串行 | 改设计：全派出去、一次 join。唯一的替代是把耗时任务放进标了只读的 MCP 工具 | `Promise.race` 不流式 |
| 7 | 顶层配置键必须写在所有 `[table]` 之前 | `config.toml` 里 `model` / `model_provider` 这些顶层键放在 `[model_providers.x]` 上面 | 被折进上一个表，TOML 解析失败或静默失效 |
| 8 | `exec_command` 给不给 `session_id`，取决于进程在 `yield_time_ms` 内有没有结束 | 判据看有没有 `session_id`，不要看 `tty` | 判据不是 `tty`，是"还活着就给会话、已退出就只给 `exit_code`"。`tty` 之所以相关是间接的：不带 PTY 时 stdin 是个空管道，读 stdin 的命令（比如一个交互式 shell）立刻拿到 EOF 就退出了，于是没有会话可给。`sleep 60` 不带 tty 照样返回 `session_id` |
| 9 | 空写轮询的 `yield_time_ms` 被钳进 `[5000, 300000]` | 按钳制后的值排期，别指望快速返回 | 两个方向都不报错、不提示。省略该参数时默认值 250 会被抬到 5000，所以空轮询没有"快速返回"这个选项。非空写是另一套：默认 250ms，上限 30000ms |
| 10 | 命令失败 ≠ 工具失败 | 一律看 `exit_code` | `exec_command` 对一个非零退出的命令返回 `ok`，诊断信息和真数据走同一个 `output` 字段。判据永远是 `exit_code`，不是文本 |
| 11 | exec 策略按命令内容拒绝，与沙箱无关 | 被拒时改用同语义的另一种写法（`rm -rf` → `rm -r`） | `rm -rf` 即便在 `danger-full-access` 下也被拒（"rm -f style commands are not permitted"），同路径的 `rm -r` 却放行。"与沙箱无关"是准确的，但"拒绝"这个结果依赖 `approval_policy = never`——`codex exec` 无条件钉成 never，所以观测到的就是拒绝 |
| 12 | `memories__list` 在从没写过的库上直接失败 | 先写一条再列 | 根节点在第一条笔记之前不存在，于是报 `path '' was not found` 而不是返回空列表。临时笔记落在 `extensions/ad_hoc/notes/` 下，不在根上 |
| 13 | 工具描述里的约束不是服务端校验 | 需要哪个不变量就自己校验 | `update_plan` 文档写着"最多一个 `in_progress`"，实际接受两个且不给任何提示。描述是写给模型看的 |
| 14 | 出错的 agent 仍然是一次成功的工具调用 | 先判终态种类（`{completed}` / `{errored}`），再看内容 | 终态是个 tagged union，`wait_agent` 两种都返回 `ok`，provider 的报错文本就坐在答复该在的位置。按工具状态判会让一个不通的 provider 看起来像行为完全符合文档的后端 |
| 15 | `request_permissions` 在 `approval_policy = never` 下自问自答 | 仍然给它套超时 | 既不挂起也不报错：毫秒级返回一个 JSON 字符串（不是对象），里面每项权限都是 `null`。套超时是因为这是宿主行为，换个客户端可能真的阻塞，而一个阻塞的非并行工具会一直攥着那把共享写锁，把后面每一次调用都拖住 |

第 2、3、5、13、14 条不报错，产出的结果形式完全正常。其中第 14 条后果最重：它让"什么都没发生"呈现为"经过验证的否定结论"。任何从子 agent 答复得出的结论都要先确认终态种类；结论为"没有"（没有孙 agent、没有匹配项）时同样要确认，不能因为是否定结论就跳过。

---

## 5. 配置类问题

| # | 问题 | 怎么办 |
| --- | --- | --- |
| 1 | feature 有两种写法且不通用：`[features] memories = true` 是布尔形式，`[features.current_time_reminder] enabled = true` 是表形式。写错形式会报"期望布尔却拿到 map"，且报错行号指向的是别的表 | 按《02-quickstart.md》§4 那段配置逐项核对写法，不要照着相邻的另一个 feature 类推 |
| 2 | 记忆工具需要两个开关 | `[features] memories = true` 和 `[memories] dedicated_tools = true` 同时给，只给一个得到 0 个工具 |
| 3 | 往一个已有表的配置尾部追加顶层键，会被折进那个表 | `model` / `model_provider` 这类顶层键写在所有 `[table]` 之前 |
| 4 | 开了 `[features.multi_agent_v2] enabled` 但工具面上一个 agent 工具都没有 | 同一段里还要写 `non_code_mode_only = false`，默认为真时整组只给模型直接调 |

---

## 6. 诊断动作清单

按以下顺序排查：

**①** 跑探针。它不派 agent、不调模型，写法见《02-quickstart.md》§2。

```bash
codex exec --poa ~/poa-probe --skip-git-repo-check
```

看 `backend`（不能是 `null`）和 `all_tools`（程序用到的工具是否都在里面）。

**②** 加 `--json` 看原始返回值。调试协议层问题时这是唯一手段。

```bash
codex exec --poa ~/poa-probe --json
```

**③** 把规模压到 1，超时压到 60000。使一次失败只消耗一个 agent 的时间。

**④** 把 `log` 数组交出去。在 `catch` / `finally` 里也要 `text()`。

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

**⑤** 用 `requireAgents()` 提前失败。它的异常消息里会列出当前全部可用工具名。

---

## 7. 三条不是 bug 的现状

以下三种情况不必排查，这就是当前的行为：

1. **没有续跑**：程序一旦超时返回，剩下的部分回不来了。长任务只能靠把超时调大
2. **没有取消**：程序失控时只能杀进程
3. **不能问人**：程序中途没法向人提问

完整说明见《08-limits.md》文档。
