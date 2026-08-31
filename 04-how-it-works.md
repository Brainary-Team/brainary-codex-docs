# 工作原理

本篇只讲会影响写法的机制。

---

## 1. 三种工具投送形态

codex 如何把工具送到模型手里，取决于模型运行形态。

### Direct 模式

每个工具是请求体里的一项，带完整的参数 schema。模型直接发一次函数调用，服务端执行，结果写回上下文，模型据此决定下一步。

调 10 个工具就是 10 次采样、10 份原始输出进入上下文。

### ordinary CodeMode

fallback 模型显式打开 `[features.code_mode]` 后进入 ordinary CodeMode。模型仍看到原来的直接工具，另外多一个 `exec`；`exec` 不展开 nested per-tool declarations，也不附共享 MCP 类型块，但存在 `Deferred` nested tools 时，会在运行时基础说明后附通用的 `ALL_TOOLS` 查询提示。

### CodeModeOnly

`gpt-5.6-luna` / `sol` / `terra` 的模型元数据会进入 CodeModeOnly。可进入 code mode 的 nested tools 被折叠进 `exec`，其说明只展开非 `Deferred` 工具的 per-tool declarations，存在 MCP 时再附一次共享 MCP 类型块；存在 `Deferred` 时，正文只提供查 `ALL_TOOLS` 的提示。code-mode `wait` 与 `DirectModelOnly` 工具仍独立对模型可见，不受折叠。两种 CodeMode 的程序调用语法相同：

```js
const r = await tools.exec_command({ cmd: "ls -la" });
```

模型只发一次 `exec`，那段 JS 在沙箱里循环、分支、并发地调那 10 个工具，只把整理好的结果交回去。

### 差别在哪

| | Direct | ordinary CodeMode | CodeModeOnly |
| --- | --- | --- | --- |
| 模型侧工具面 | 各个直接工具 | 直接工具 + `exec` | `exec` + 独立的 `wait` / `DirectModelOnly` 工具 |
| `exec` 说明 | — | 运行时基础说明；有 `Deferred` 时提示查 `ALL_TOOLS`；不展开 declarations 或共享 MCP 类型 | 非 `Deferred` nested declarations；有 MCP 时附共享类型块；有 `Deferred` 时提示查 `ALL_TOOLS` |
| `exec` 内的编排循环 | — | V8 程序 | V8 程序 |
| 一次 `exec` 的采样次数 | — | O(1) | O(1) |
| 模型调用 `exec` 时的上下文压力 | — | O(主动交给模型的输出总量) | O(主动交给模型的输出总量) |

看一个具体对比：

```js
// 一次 exec；内部 N+1 次工具调用 + 一次并发 join；模型只被采样一次

// ① 第 1 次工具调用：列出所有子目录
const listed = await tools.exec_command({ cmd: "ls -d */" });
const dirs = listed.output.split("\n").filter(Boolean);

// ② 第 2..N+1 次：并发统计每个目录的 .rs 文件数
const counts = await Promise.all(
  dirs.map((d) => tools.exec_command({ cmd: `find ${d} -name '*.rs' | wc -l` })),
);

// ③ 唯一进模型上下文的一步
//    上面那些 ls / find 的原始输出全都留在沙箱里，模型一个字都看不到
text(JSON.stringify(dirs.map((d, i) => ({ dir: d, rs: counts[i].output.trim() }))));
```

---

## 2. PoA 走的是 code mode 这条路

模型走 code mode 时，那段 JavaScript 是它自己写的，是它一只更灵巧的手。

PoA 用的是同一条路上的另一个提交入口，让模型以外的调用方也能提交 JS。

```
thread/codeMode/exec   { threadId, package }  →  { output }
```

`package` 是 base64 编码的 `.poa` 包（zip，根上 `manifest.toml`）。服务端解包 → 读 manifest → 起它申报的 MCP server → 等这些 server 就位 → 把 `entry` 那个文件的内容当源码送进沙箱。送进去的就是那段 JS 源码字符串，和模型走的是同一条路：同一个沙箱、同一批工具、同一条分发路径与记录。

直接的推论有三个：

**①** 编排全程零模型调用。程序不是模型写的，是客户端直接提交的。模型只出现在被派出去的子 agent 内部，它们是被程序调用的对象，不是控制者。

**②** 模型侧有的能力，程序侧不一定有。这个入口只提供"提交"这一件事。最典型的是续跑：模型可以让一个中途让出的 cell 接着跑，程序没有对应的手段。

**③** 工具面在 cell 起跑前就定死了。包申报的 server 必须在捕获工具面之前起完，起不来直接拒跑，而不是少个工具照跑。所以 `ALL_TOOLS` 在程序里读一次就够，它在这次运行中不会再变。

---

## 3. `tools.foo()` 实际发生了什么

`tools.foo(x)` 不在 V8 里执行任何东西。

调用发生时，沙箱当场造一个未完成的 Promise 返回给 JS，同时把这次调用作为一个事件交给宿主。宿主侧走的是正常的工具分发路径，与模型直接调这个工具完全同一条路，同样的策略判定、同样的记录。执行完之后：

- 成功 → resolve 那个 Promise
- 失败 → reject 它

JS 那边的 `await` 在此时恢复。

### 两个推论

**①** 工具失败在 JS 里表现为一个抛出的异常。

```js
try {
  const res = await tools.exec_command({ cmd: "some-command-that-fails" });
} catch (err) {
  // 可以兜住，不必让整段程序一并失败
}
```

**②** `Promise.all([...])` 是不是真并发，完全由宿主决定。

V8 是单线程的，它只负责等。多个工具调用是多条独立事件，并发度由宿主侧的策略决定，而默认策略是串行，详见下一节。

---

## 4. 并发不等于流式

在 PoA 下，并发与流式是两回事。

### 默认串行

工具执行时会取一把共享的锁。声明了并行安全的工具取读锁（可以共存），没声明的取写锁（独占）。而这个 guard 被持有到整个调用执行结束。

**子 agent 那两代工具一个都没有声明并行安全。**

### 后果：`Promise.race` 拿不到最先完成的那个

有一个专门测量这件事的实验：三个 agent 分别 sleep 12 / 2 / 7 秒，故意让完成顺序不同于启动顺序。理想情况下结果应该按 `1 → 2 → 0` 到达。

实际不会。第一个 wait（12 秒那个）拿到写锁后独占 12 秒，期间另外两个 wait 根本发不出去；等它放锁，另外两个 agent 早已完成，wait 立即返回。

到达顺序 = 启动顺序 = 0, 1, 2。

### 要分清丢的是哪一半

agent 本身是并发的。三个同时在 sleep，总耗时约 12 秒而不是 21 秒。串行的是程序侧的等待调用。

所以丢掉的具体是这三件事：

| 做不到 | 具体表现 |
| --- | --- |
| 流式输出 | 没法边收边处理 |
| 早停 | 拿到满意结果后没法提前结束剩下的 |
| 动态调度 | 等待期间连再派一个新 agent 都做不到 |

写程序时按"全部派出去 → 一次 join → 统一收口"来组织，不要设计成流水线。

### 哪些工具是并行安全的

| 工具 | 并行安全 |
| --- | --- |
| `exec_command` | ✅ |
| `write_stdin` | ✅ |
| `shell_command` | ✅ |
| `view_image` | ✅ |
| `web__run`（联网搜索） | ✅ |
| MCP 资源工具 | ✅ |
| 一般 MCP 工具 | 有条件——见下文 |
| 子 agent 工具（v1 与 v2 全部） | ❌ |
| 其余内置工具 | ❌ |

最后那条"有条件"是唯一可用的手段。判据是"工具自己声明并行安全或带只读标注"，所以一个标成只读的 MCP 工具就是并行安全的。要让派发出去的耗时任务真正并行，把它放进一个标了只读的 MCP server。

这条路不需要宿主配合：包自带的 stdio server 走同一条判据，但只满足得了后半条——包起的 server 那份配置里 `supports_parallel_tool_calls` 固定为 `false`，服务器级的整体豁免用不了，只能逐个工具标 `readOnlyHint`。

并行与审批使用不同判据；包内 MCP 的标注规则见《08-packaging.md》§4.2。

---

## 5. `tools.` 上有哪些工具

这里用的是黑名单而不是白名单：工具注册表里的东西默认全部暴露给 code mode，只有明确掉出去的才没有。

一个工具要能被 PoA 程序调到，要连过四道条件：

| # | 判什么 | 谁会被挡 |
| --- | --- | --- |
| 1 | 工具的曝光度档位允许 code mode 使用 | 那些"只给模型直接调"和"对谁都不露面"的工具 |
| 2 | 命名空间没有被配置点名排除 | 配置里排除掉的整个命名空间 |
| 3 | spec 的种类 | 由客户端或模型服务执行的特殊工具（`tool_search`、hosted 版联网搜索） |
| 4 | 不是 `exec` / `wait` 自身 | 这两个 |

第 3 道按种类筛，与曝光度无关。`tool_search` 的曝光度看起来是允许的，但它由客户端执行，所以 PoA 依然调不到。

### 一档特殊状态：不进 CodeModeOnly 的 `exec` 正文，但运行时仍能查形状

有一档曝光度叫 `Deferred`：工具能调通、名字也出现在 `ALL_TOOLS` 里，但它的声明不会进入 CodeModeOnly 模型看到的 `exec` 说明正文。

v1 那 5 个子 agent 工具的默认状态就是这一档。

这不妨碍程序在运行时检查形状：无论是 ordinary CodeMode 还是 CodeModeOnly，`ALL_TOOLS` 中每个条目的 `description` 都内嵌完整的 per-tool exec TypeScript 声明，包括该工具的入参、必填性与返回声明。

普通工具的声明可独立阅读。MCP 工具的返回声明会引用 `CallToolResult`、`ContentBlock`、`MetaObject` 等共享别名；CodeModeOnly 的 `exec` 说明会附一次共享类型块，ordinary CodeMode 不会，因此后者需查《07-api-reference.md》§3.4 的离线定义。

### `ALL_TOOLS` 的 `description` 带完整声明

```js
const execTool = ALL_TOOLS.find((tool) => tool.name === "exec_command");
if (!execTool) throw new Error("exec_command is unavailable");
text(execTool.description); // 内含 declare const tools: { exec_command(...): ... }
```

《07-api-reference.md》§4 的价值是提供离线可查的中文版和实测坑位注解，不是填补运行时拿不到 schema 的缺口。

即便能读声明，也仍要在运行时先检查工具是否存在，并为缺席路径降级：

```js
const has = new Set(ALL_TOOLS.map((t) => t.name));
if (!has.has("web__run")) { /* 走降级路径 */ }
```

---

## 6. 输出是怎么回到客户端的

只有一条路：往本次运行的返回值里追加条目，跑完一次性交给客户端打印。

| primitive | 它做什么 | PoA 下 |
| --- | --- | --- |
| `text(v)` | 追加一条文本 | ✅ 唯一可靠的文本通道 |
| `image(v)` / `audio(v)` / `generatedImage(v)` | 追加一条非文本内容 | ⚠️ 会进返回值，但条目里没有 `text` 字段，只被渲染成 `[input_image] path=…` 这样的一行摘要，不是给人读正文用的 |
| `notify(v)` | 立刻额外送出一条内容 | ❌ 走的是另一条通道，客户端收不到 |
| `yield_control()` | 把已累积的输出先交出去，程序继续跑 | ⚠️ 约等于提前结束，只能拿到前半段，后半段回不来 |

`yield_control()` 的问题在于：它交出前半段之后程序还在跑，但没有配套的手段去接后半段。模型侧有（可以用 `wait` 工具续跑），程序侧没有。

长程序的正解不是 yield，而是把 `yield_time_ms` 调大。

---

## 7. 机制如何决定写法

| 机制 | 写法上的结论 |
| --- | --- |
| 一次提交 = 一个 cell = 一次跑完 | 整个流程写在同一段代码里；未 `await` 的活会消失 |
| 工具调用往返宿主，失败表现为异常 | 可以 `try/catch` 兜住单点失败 |
| 等待调用默认串行 | "全派出去 → 一次 join"，不要设计成流水线 |
| `ALL_TOOLS` 的 `description` 内嵌完整 per-tool 声明 | 可在运行时查参数与返回；MCP 共享别名从 CodeModeOnly 的 `exec` 共享块或《07-api-reference.md》§3.4 查；仍要探测工具是否存在 |
| 只有 `text()` 可靠 | 进度信息攒成数组，最后一次性交出去 |
| 没有续跑手段 | 长任务靠调大 `yield_time_ms`，不靠 yield |
