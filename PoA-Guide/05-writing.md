# 编写指南

本篇讲从一个空文件开始写出一个新 PoA 程序的完整路线。

## 前置条件

| 事项 | 当前状态 |
| --- | --- |
| 后端是 v1 还是 v2 | 默认由模型元数据决定；本地 `[features.multi_agent_v2]` 可强制 v2，但不能强制回 v1 |
| 用不用 prelude 那一层 | 来源与使用规则见《03-concepts.md》§2；prelude 版与纯内置版范例见 §12 |
| 程序能不能中途问人 | 不能 |
| 程序失控时如何处理 | 只能杀进程。没有取消，也没有续跑 |

---

## 0. 一个 PoA 程序的六段形状

PoA 程序的基本骨架如下：

```js
// @exec: {...}                        ① 配置，必须在第 1 行

const MAX_ITEMS = 3;                    ② 规模常量提到顶部
const CONCURRENCY = 3;

requireAgents();                        ③ 前置断言：无后端时立即失败

const items = await shellLines(...);    ④ 程序自己发现工作
const batch = await runBatch(...);      ⑤ 派发
text(JSON.stringify(...));              ⑥ 收口并交出结果
```

---

## 1. 写首行 pragma

```js
// @exec: {"yield_time_ms": 900000, "max_output_tokens": 30000}
```

必须在第 1 行。两个字段的默认值都只有 10000（10 秒 / 10000 token）。

| 字段 | 计量的是什么 | 默认 |
| --- | --- | --- |
| `yield_time_ms` | 本次运行在脚本仍未跑完时提前交还之前，最多等多久 | 10000 ms |
| `max_output_tokens` | 本次运行返回值的 token 预算，即 `text()` 累积下来准备交回的那份内容 | 10000 |

这两个数管的都是"这一次 PoA 运行"，不是"每个子 agent"，也不是"所有子 agent 的消耗总和"。

---

## 2. 把结果交出去：只有 `text()` 可靠

现成示例代码中的部分输出写法在 PoA 下不生效，逐项如下：

| primitive | PoA 客户端看得见吗 |
| --- | --- |
| `text(v)` | ✅ 唯一可靠的文本通道 |
| `image(v)` / `audio(v)` / `generatedImage(v)` | ⚠️ 会进返回值，但只渲染成 `[input_image] …` 一行摘要。能证明"东西产出来了"，但读不到内容 |
| `notify(v)` | ❌ 走另一条通道，收不到 |
| `yield_control()` | ⚠️ 交出前半段之后没法接后半段，约等于提前结束 |
| `exit()` | ⚠️ 可用，但不能放在 `try` 块里，见下 |

> **`exit()` 是靠抛一个内部哨兵异常实现的，被 `catch` 拦住就失效。**
> 只有这个异常一路抛到顶层才会被识别为正常退出。带 `catch` 的 `try` 会把它接住
> （`__codex_code_mode_exit__` 这个字符串随之混进错误日志），退出不生效，脚本一路跑到自然结尾。
>
> `try { … exit(); } finally { … }`（只有 `finally`、没有 `catch`）是安全的，`finally` 不吞异常。
> 需要 `catch` 时，把可能 `exit()` 的分支挪到 `try` 外面。

不要用 `notify()` 输出进度，PoA 客户端收不到。正确做法是攒一个数组，最后随结果一起交出去：

```js
const log = [];
log.push(`backend=${AGENT_BACKEND} items=${items.length}`);
// …
text(JSON.stringify({ result, log, elapsed_ms: Date.now() - t0 }, null, 2));
```

`yield_control()` 不能用来分段交付 PoA 的最终结果。长程序应调大 `yield_time_ms`。

---

## 3. 把外部输入送进程序

PoA 没有启动参数。提交接口收的是线程 id 和一个包（`{ threadId, package }`），仍然没有一个放参数的地方。

于是实际可用的只有三条路：

| 想传什么 | 怎么传 |
| --- | --- |
| 少量常量（规模、并发度、目标名） | 直接写死在 `entry` 顶部，或由自研客户端每次现拼一个包再提交 |
| 程序的工作目录 | `-C` / `--cd`，它决定线程的工作目录，也就决定了 `exec_command` 里相对路径的基准 |
| 用哪个模型 | `-m` / `--model` |

```bash
codex exec --poa ./my-poa -C /path/to/target
```

不给 `-C`，工作目录就是执行命令时所在的目录。

"把数据文件打进包里"这条路目前走不通。包确实会被完整解出来，但解在服务端的一个临时目录里，而这个路径不会以任何形式交给程序。症结是"拿不到路径"而不是"没有读的手段"——`exec_command` 是宿主进程执行的，能读整个文件系统，但它不知道该读哪儿（它的相对路径基准是 `-C`）。靠 `ls -dt /tmp/...` 推测那个临时目录不是接口，不应依赖。拿得到那个目录的只有包自带的 MCP server（它的 cwd 就是包根，所以 `node mcp/echo.mjs` 才解析得了）。所以"随包带数据"当前只有两种写法：写进 `entry` 的源码里，或者由包自带的 server 去读、程序调它拿。

`store()` / `load()` 在 PoA 里基本等于全局变量。它们跨多次提交存活，但一次运行只发一次提交，结束就关掉进程。除非改用自研客户端、在同一个线程上连发多次，否则用普通 `const` 就够了。

---

## 4. 让程序自己去发现工作

```js
// ls -d */ 列出所有子目录（带尾部 /），sed 把那个 / 去掉，得到干净的目录名
const folders = await shellLines("ls -d */ | sed 's#/$##'", {
  // 逐行过滤器：返回 true 的行才留下
  validate: (line) => SAFE_NAME.test(line),
});
```

`validate` 必须包一层箭头函数：

```js
validate: (line) => SAFE_NAME.test(line)     // ✅
validate: SAFE_NAME.test                     // ❌ 方法被从正则上摘下来，this 丢失
```

`SAFE_NAME` 是 `/^[A-Za-z0-9._-]+$/`，只放行字母数字和 `. _ -`。带空格、斜杠、中文的路径会被它一并滤掉，需要的话自己写一个更宽的正则。

---

## 5. 写子 agent 的任务文字

子 agent 的系统提示词改不了。派发接口的参数面上没有这一项。v2 的入参是严格校验的，多余字段抛异常，不是静默忽略。

程序能控制的与不能控制的，是这样分的：

| 层 | 来源 | 程序能否控制 |
| --- | --- | --- |
| 基础指令 | 从父会话继承 | ❌ |
| 角色层（模型、推理强度、工具集） | `agent_type` 指向的角色配置 | ⚠️ 只能选宿主已装好的，不能现场定义 |
| 任务内容 | `message` | ✅ 完全控制 |

> `agent_type` 是一个角色名，不是一段 prompt。它只在宿主配置过 agent role 时才出现在派发工具的参数里；一个 role 都没配时，这个字段不会出现，那一层就只剩宿主的默认值。

所以 `message` 是程序唯一的控制面，要写扎实。它必须同时做三件事：

| 要素 | 写法 | 防的是什么 |
| --- | --- | --- |
| 强制执行 | 列出要跑的具体命令，并要求回报证据字段（"你实际读了哪两个文件"） | agent 凭常识编一段看似合理的答案 |
| 强制格式 | `只回一行 JSON，不要代码块，不要多余的话，不要复述格式说明` | 回一段散文正则捞不出来；复述了模板则整段解析失败，见 §7 |
| 划定作用域 | `不要跑到 ./<folder> 外面去` | 读整个仓库，消耗 token 并导致超时 |

完整模板：

```js
const prompt = (folder) => [
  `分析这一个目录：./${folder}`,
  ``,
  `回答前必须真的动手：`,
  `1. 跑 ls -R ${folder} | head -60`,
  `2. 读它的清单文件（Cargo.toml / package.json 之类）`,
  `3. 真的打开主入口和另一个像样的源文件读一遍`,
  ``,
  `只回一行 JSON，不要代码块，不要多余的话：`,
  `{"purpose":"<一句话>","deps":["<最多3个>"],"evidence":["<你真的读了的两个文件>"]}`,
  ``,
  `不要跑到 ./${folder} 外面去。`,
].join("\n");
```

这段文字是程序侧唯一能施加的约束。程序无法进入 agent 内部检查它是否实际执行，要求它报出证据字段是唯一的手段。而且这只是"要求"不是"保证"，收口时仍必须防御性解析。

---

## 6. 并行派发

```js
const batch = await runBatch(
  folders.map((folder, i) => ({
    message: prompt(folder),
    name: `scan_${i}`,
    meta: { folder },              // ← 没有它就无法把结果映射回输入
  })),
  { concurrency: 3 },
);
```

`runBatch` 是"按并发度派出去 + 等全部到终态"的合并写法，返回 `[{ handle, reply }]`，顺序与输入一致。

v2 的原生 `task_name` 只允许小写字母、数字和下划线，而且同一个父任务下的 sibling 名必须唯一。prelude 首次使用清洗后的 `${base}_${seq}` 稳定名；仅遇到 `already exists` 后，回退名才追加当前 cell 的小写时间 nonce。总共尝试 3 次（最多重试 2 次），nonce 不承诺绝对随机，其他错误也不会被吞掉。

为与 v1 默认不 fork 的语义一致，prelude 和本篇纯内置 helper 的 v2 派发都显式传 `fork_turns: "none"`，默认不继承父线程历史。需要继承时直接调用从 `ALL_TOOLS` 发现的原生 v2 `spawn_agent` 并显式选择 `fork_turns`；本次 helper API 不跨后端扩展这个参数。

### 6.1 在 `meta` 里记下"谁是谁"

派出一个子 agent，拿到的是一个 handle：

```js
{ key, label, meta }
```

| 字段 | 是什么 |
| --- | --- |
| `key` | 后端认的标识，后续等待 / 追问 / 关闭都靠它 |
| `label` | 宿主分配的面向人昵称或任务名，不是模型自己起的名字 |
| `meta` | 留给调用方塞上下文的口袋。prelude 不解释它，只负责原样带回来 |

`meta` 不是可选项。agent 的终稿文本里不包含"我是谁"，程序必须自己记，收口时靠 `handle.meta.folder` 这类字段把结果对回输入。写包时没有 handle 这层封装，`spawn` 只返回一个 key，对应关系要自己用数组下标维持，见 §12 的包版范例。

### 6.2 `concurrency` 限的是什么

限的是同时在途的派发调用数，不是同时运行的 agent 数。派发一返回，worker 就去派下一个，已派出的 agent 全都还在跑。

因此 `concurrency` 守不住名额：它既不影响 v1 同时在册的 agent 数，也不影响 v2 同时驻留的 agent 数。名额要靠 §6.3 的分批结构来守。

### 6.3 名额：两代语义不同，且都会决定程序结构

这一条决定的是程序的形状，不是一个参数的取值。

| | v1 | v2 |
| --- | --- | --- |
| 上限计的是什么 | 同时在册的 agent 数（已完成但未 `close` 的仍在册） | 同时驻留的 agent 数 |
| 默认上限 | 6，root 不占名额 | 配置值减 1（root 占一个）：默认 4 → 3 |
| 跑完的 agent 释放名额吗 | 不释放，必须显式 `closeAll` | 需要腾名额时可自动驱逐 |
| 超限时 | 派发抛异常 | 派发抛异常，另有一种静默后果，见下 |

配置里把 `max_concurrent_threads_per_session` 写成 8，可用数就是 7。

要处理的条目数超过名额时，必须写成分批。两代分批的理由不同：

#### v1：close 之后名额才回来

超限时派发抛 `AgentLimitReached`，而已派出的那几个 token 已经花掉。

```js
const BATCH = 5;                                    // 上限 6，留一格余量
const results = [];
for (let i = 0; i < items.length; i += BATCH) {
  const batch = await runBatch(specsOf(items.slice(i, i + BATCH)), { concurrency: BATCH });
  results.push(...batch);
  await closeAll(batch.map((b) => b.handle));       // 释放名额，必须在下一批派发之前
}
```

`closeAll` 要在下一批派发之前调，不能挪到循环外面。

#### v2：收齐之后才能派下一批

`runBatch` 是"先全部派完，再一次收齐"。v2 在派发阶段腾驻留位时，较早完成的结果可能因自动驱逐在收口前消失，并被填成 `null`。

所以 v2 下分批不是为了争名额，是为了在驱逐发生之前把结果读走：

```js
const BATCH = 6;                                    // 配置值 8 时可用 7，留一格；默认配置只有 3，取 2
for (let i = 0; i < items.length; i += BATCH) {
  results.push(...await runBatch(specsOf(items.slice(i, i + BATCH)), { concurrency: BATCH }));
}
```

条目数在名额以内时两代都可以一次派完，`closeAll` 放在最后。

> 上表里"减不减 root""释不释放"是当前实现规则；6 和 4 是当前配置默认值，宿主改了就不是这个数。当前实际能开几个，以自己那份 `config.toml` 为准。

### 6.4 v2 下自行实现递归深度限制

子 agent 能不能再派孙 agent，两代行为相反：

| 后端 | 行为 |
| --- | --- |
| v1 | 深度上限默认为 1，且拦两道——超限的子 agent 在 JS 里根本看不到那组工具，即便看到运行时也会再拦一次。递归会自己停 |
| v2 | 两道都没有，无条件放行 |

v2 下的限制只能在 JS 里自行实现：传一个深度计数进 `message`，或只在顶层派发。后端默认由模型元数据决定；本地 `[features.multi_agent_v2]` 可强制 v2，但不能强制回 v1；程序 cell 启动后不能改变。

### 6.5 收口顺序

收口顺序的边界统一见《04-how-it-works.md》§4，本篇不重复展开。

---

## 7. 收口

```js
const results = batch.map(({ handle, reply }) => {
  const { value, error } = parseJsonReply(reply);
  return {
    folder: handle.meta.folder,
    summary: value,
    parse_error: error,
    raw: value ? undefined : String(reply ?? "").slice(0, 200),   // ← 失败时留原文
  };
});
```

`parseJsonReply` 从自由文本里抠 JSON，返回 `{ value, error }`。三种失败各有不同的 error 文案：不是字符串 / 没找到 `{}` / 解析异常。

它的匹配是贪婪的，从第一个 `{` 吃到最后一个 `}`。所以 agent 只要在正式答案前把 [§5](#5-写子-agent-的任务文字) 那段格式模板复述一遍，两段 `{}` 会被连成一整块，整体解析失败，而不是退回第一段。这就是 §5 那句 `不要复述格式说明` 要写进 prompt 的原因。

`reply` 是 `null` 和是空字符串不是一回事。`null` 有四个来源：超时未达终态、agent 正常结束但没给终稿、agent 以 `{ errored }` 到达终态、v2 下结果在收口前被驱逐（§6.3）。四种都要当失败处理，但排查方向不同：第一种调 `timeoutMs`，第二种调 prompt；第三种的具体错误没有保留在 prelude 的 `reply` 中，需要直接查看原生 agent 状态；第四种要改分批结构。

`runBatch` / `collectAll` 的 `timeoutMs` 默认 300000 ms。v1 的收口是逐个 handle 串行等待的，每个都可能等满这个值，所以极端情况下总耗时是 N × `timeoutMs`——它必须留在首行 pragma 的 `yield_time_ms` 之内。调试期把 `yield_time_ms` 压到 60000 时，`timeoutMs` 要跟着一起压。

---

## 8. 容错：四个必须写的地方

工具失败在 JS 里表现为一个抛出的异常，可以 `try/catch` 兜住，不必让整段程序一并失败。并行派发程序至少要处理这四处：

| # | 处理什么 | 怎么做 | 详见 |
| --- | --- | --- | --- |
| ① | 单个 agent 失败拖垮整批 | 把每个 worker 的调用包进 `try/catch`，失败的记成 `null` 继续走 | — |
| ② | shell 的报错文本被当成数据 | `shellLines` 传 `validate` | [§4](#4-让程序自己去发现工作) |
| ③ | 解析失败之后查不出原因 | 保留原始回复的前 200 字 | [§7](#7-收口) |
| ④ | v1 下名额耗尽 | `closeAll` 放在每批之后。它吞掉所有错误，不会抛出；v2 下是空操作 | [§6.3](#63-名额两代语义不同且都会决定程序结构) |

---

## 9. 调试：在没有取消、没有续跑的前提下

没有任何手段能中止一个正在跑的程序，yield 之后也没法续跑。两条合起来意味着程序失控时只能杀进程。

由此有四条做法。

### ① `yield_time_ms` 分两档写

调试期 `60000`，跑通了再改成 `900000`。压 `yield_time_ms` 时，`runBatch` 的 `timeoutMs`（默认 300000）要一起压下来——它必须留在 `yield_time_ms` 之内。

### ② 先把规模压到 1

顶部把常量提出来，调试时全设成 1：

```js
const MAX_FOLDERS = 3;      // 调试时改 1
const CONCURRENCY = 3;      // 调试时改 1
```

目的是让一次失败只消耗一个 agent 的时间。

### ③ 观测点靠攒，不靠打印

沙箱里没有 `console`，`notify()` 又看不见。可行的写法是攒一个数组：

```js
const log = [];
const t0 = Date.now();
log.push(`backend=${AGENT_BACKEND} folders=${folders.length}`);

try {
  // …主流程…
} catch (err) {
  log.push(`FATAL: ${String(err)}`);
} finally {
  // 即使中途抛异常也要把 log 交出去——否则那次运行的全部信息都没了
  text(JSON.stringify({ result, log, elapsed_ms: Date.now() - t0 }, null, 2));
}
```

### ④ 收口前检查每个 `await`

> **未 `await` 的 promise 会被静默丢弃。**
> 脚本求值结束时沙箱生命周期即终止，未 `await` 的 promise 被静默丢弃。
> 不报错、不警告，只是那部分工作没有发生。
>
> 提交前逐个检查每个异步调用前面都有 `await`：全部 `tools.` 调用，以及 prelude 那一层的
> `runBatch` / `collectAll` / `spawnAgent` / `spawnMany` / `sendAndWait` / `closeAll` / `shellLines` / `mapLimit` / `callTool` / `timed`。
> `requireAgents` / `parseJsonReply` / `hasTool` / `shapeOf` 是同步函数，不需要 `await`。

---

## 10. 什么活派给 agent，什么活自己干

判据只有一条：这件事要不要读懂内容才能做。列目录、数文件、grep、跑测试都不需要，交给 `tools.exec_command` 更快、零 token、结果可复现；"这个模块的用途是什么"需要，才派 agent。

各种形状的写法见《06-patterns.md》文档。

---

## 11. 提交前检查清单

- [ ] 探针跑过，`backend` 不是 `null`
- [ ] `-C` 指向了真正要扫的目录
- [ ] 首行是 `// @exec:` pragma，且在第 1 行
- [ ] 调试期 `yield_time_ms` = 60000，规模常量 = 1
- [ ] `requireAgents()` 在最前面
- [ ] 每个异步调用前有 `await`（`runBatch` / `collectAll` / `sendAndWait` / `closeAll` / `shellLines` 与全部 `tools.` 调用都是 async；`requireAgents` / `parseJsonReply` / `hasTool` / `shapeOf` 是同步的）
- [ ] `shellLines` 传了 `validate`
- [ ] 每个 agent 回复都过 `parseJsonReply`，失败留原文
- [ ] handle 上挂了 `meta`
- [ ] 条目数超过可用名额时写成了分批，且 v1 下每批之后调了 `closeAll`（§6.3）
- [ ] 收口的 `text()` 在 `catch` / `finally` 里也会执行
- [ ] 用到的每个工具都在探针输出的 `all_tools` 里

---

## 12. 完整范例

同一个程序有 prelude 版与纯内置版，骨架完全一样。

两版共用同一个包目录和同一份 manifest：

```
my-poa/
├── manifest.toml
└── main.js
```

```toml
[poa]
name = "scan_folders"
version = "0.1.0"
runtime = "codex-v8"
poa_api_version = 1
entry = "main.js"
```

没有 `[[capabilities.mcp]]`，因为这个程序不需要自带能力。

### 12.1 使用 prelude

`main.js` 的顺序是：首行 pragma → [《07 API 参考》§2.4](07-api-reference.md#24-prelude-全文)全文 → 下面这段。

```js
// @exec: {"yield_time_ms": 900000, "max_output_tokens": 30000}

// ---- 调试期把这两个都改成 1 ----
const MAX_FOLDERS = 3;      // 目录数超过可用名额时不能这样一次派完，改成 §6.3 的分批结构
const CONCURRENCY = 3;

const log = [];
const t0 = Date.now();
let payload = null;

try {
  requireAgents();
  log.push(`backend=${AGENT_BACKEND}`);

  // ① 程序自己发现工作
  const folders = (await shellLines("ls -d */ | sed 's#/$##' | sort", {
    validate: (line) => SAFE_NAME.test(line),
  })).slice(0, MAX_FOLDERS);
  log.push(`folders=${folders.join(",")}`);

  // 这里不能用 exit()：它抛的哨兵异常会被下面那个 catch 接住，程序不会停
  if (folders.length === 0) throw new Error("cwd 下没有子目录，检查 -C");

  // ② prompt 同时做三件事：强制执行、锁定格式、划定范围
  const prompt = (folder) => [
    `分析这一个目录：./${folder}`,
    `回答前必须真的动手：跑 ls -R ${folder} | head -60，读清单文件，真的打开两个源文件。`,
    `只回一行 JSON，不要代码块：`,
    `{"purpose":"<一句话>","deps":["<最多3个>"],"evidence":["<你真的读了的两个文件>"]}`,
    `不要跑到 ./${folder} 外面去。`,
  ].join("\n");

  // ③ 全部派出去，一次 join
  const batch = await runBatch(
    folders.map((folder, i) => ({
      message: prompt(folder),
      name: `scan_${i}`,
      meta: { folder },
    })),
    { concurrency: CONCURRENCY, timeoutMs: 240000 },
  );

  // ④ 防御性收口，失败留原文
  const results = batch.map(({ handle, reply }) => {
    const { value, error } = parseJsonReply(reply);
    return {
      folder: handle.meta.folder,
      summary: value,
      parse_error: error,
      raw: value ? undefined : String(reply ?? "").slice(0, 200),
    };
  });

  // ⑤ 释放名额（v1 下必须，v2 下是空操作）
  await closeAll(batch.map((b) => b.handle));

  payload = {
    succeeded: results.filter((r) => r.summary).length,
    total: results.length,
    results,
  };
} catch (err) {
  log.push(`FATAL: ${String(err)}`);
} finally {
  // 无论成败都把 log 交出去
  text(JSON.stringify({ ...payload, log, elapsed_ms: Date.now() - t0 }, null, 2));
}
```

这里显式把 `timeoutMs` 设为 240000。v1 会串行等待 3 个目标，最坏是 `3 × 240000 = 720000` ms，给首行的 `yield_time_ms: 900000` 留出 180000 ms 余量。

```bash
~/codex-poa/codex exec --poa ./my-poa -C /path/to/target
```

### 12.2 只用内置 primitive

同一份 manifest，`main.js` 自带一段最小替代：

```js
// @exec: {"yield_time_ms": 900000, "max_output_tokens": 30000}

// ===== 纯内置版：自带最小替代（133 行）=====

const _NAMES = new Set(ALL_TOOLS.map((t) => t.name));
const _V2_TOOLS = (() => {
  const spawn = ALL_TOOLS.find((t) =>
    t.name.endsWith("spawn_agent") &&
    String(t.description).includes("task_name") &&
    String(t.description).includes("fork_turns"));
  if (!spawn) return null;
  const prefix = spawn.name.slice(0, -"spawn_agent".length);
  const names = Object.fromEntries(
    ["spawn_agent", "wait_agent", "list_agents", "followup_task"]
      .map((key) => [key, `${prefix}${key}`]),
  );
  return Object.values(names).every((name) => _NAMES.has(name)) ? names : null;
})();
const BACKEND = _NAMES.has("multi_agent_v1__spawn_agent") ? "v1" : _V2_TOOLS ? "v2" : null;
let agentSeq = 0;

function _readV2WaitLimits() {
  const description =
    ALL_TOOLS.find((t) => t.name === _V2_TOOLS?.wait_agent)?.description || "";
  const min = Number(description.match(/\bmin(?:imum)?\b[^0-9]{0,24}(\d+)/i)?.[1]);
  const max = Number(description.match(/\bmax(?:imum)?\b[^0-9]{0,24}(\d+)/i)?.[1]);
  if (!Number.isFinite(min) || !Number.isFinite(max) || min < 0 || max < min) {
    return { min: 10000, max: 3600000 };
  }
  return { min, max };
}

const _V2_WAIT_LIMITS = _readV2WaitLimits();
const _clampV2WaitMs = (ms) =>
  Math.min(_V2_WAIT_LIMITS.max, Math.max(_V2_WAIT_LIMITS.min, ms));
const _agentNonce = Date.now().toString(36);
const _SPAWN_ATTEMPTS = 3;

// 非正 limit 按 1 处理；空输入仍启动 0 个 worker。
async function mapLimit(items, limit, fn) {
  const out = new Array(items.length);
  let next = 0;
  const worker = async () => {
    for (;;) {
      const i = next++;
      if (i >= items.length) return;
      out[i] = await fn(items[i], i);
    }
  };
  await Promise.all(
    Array.from({ length: Math.min(Math.max(1, limit), items.length) }, worker),
  );
  return out;
}

async function spawn(message, name) {
  if (BACKEND === "v1") {
    const a = await tools.multi_agent_v1__spawn_agent({ message });
    return a.agent_id;
  }
  const cleaned = String(name ?? "task").toLowerCase().replace(/[^a-z0-9_]/g, "_");
  const base = !cleaned || cleaned === "root" || cleaned === "_" ? "task" : cleaned;
  for (let attempt = 0; attempt < _SPAWN_ATTEMPTS; attempt++) {
    const seq = agentSeq++;
    const taskName = attempt === 0 ? `${base}_${seq}` : `${base}_${_agentNonce}_${seq}_${attempt}`;
    try {
      const a = await tools[_V2_TOOLS.spawn_agent]({
        task_name: taskName,
        message,
        fork_turns: "none",
      });
      return a.task_name;   // v2 认的是调用方给的 task_name
    } catch (err) {
      if (!/already exists/i.test(String(err)) || attempt + 1 === _SPAWN_ATTEMPTS) throw err;
    }
  }
  throw new Error("unreachable spawn retry state");
}

const _isFinalStatus = (status) =>
  status === "shutdown" ||
  status === "not_found" ||
  (status && typeof status === "object" && ("completed" in status || "errored" in status));

const _replyOf = (status) =>
  status && typeof status === "object" && "completed" in status
    ? status.completed ?? null
    : null;

// 收齐 N 个结果——两代形状差得最远的就是这一处
async function collect(keys, timeoutMs = 300000) {
  const replies = new Map();
  if (keys.length === 0) return replies;
  const minWaitMs = BACKEND === "v2" ? _V2_WAIT_LIMITS.min : 10000;
  timeoutMs = Math.max(timeoutMs, minWaitMs);
  if (BACKEND === "v1") {
    // 一次只等一个：多目标等待会反复返回最先完成的那个
    for (const k of keys) {
      const out = await tools.multi_agent_v1__wait_agent({ targets: [k], timeout_ms: timeoutMs });
      replies.set(k, _replyOf((out.status || {})[k]));
    }
    return replies;
  }
  // v2：wait 只是"有动静了"的信号，内容要另外去 list_agents 里捞
  const deadline = Date.now() + timeoutMs;
  let firstWait = true;
  for (;;) {
    const listed = await tools[_V2_TOOLS.list_agents]({});
    for (const e of listed.agents || []) {
      const k = keys.find((x) => e.agent_name === x || e.agent_name.endsWith(`/${x}`));
      const st = e.agent_status;
      if (k && !replies.has(k) && _isFinalStatus(st)) {
        replies.set(k, _replyOf(st));
      }
    }
    if (replies.size >= keys.length) break;
    const remainingMs = deadline - Date.now();
    if (_V2_WAIT_LIMITS.max === 0) break;
    if (!firstWait && (remainingMs <= 0 || remainingMs < _V2_WAIT_LIMITS.min)) break;
    await tools[_V2_TOOLS.wait_agent]({
      timeout_ms: _clampV2WaitMs(Math.min(15000, remainingMs)),
    });
    firstWait = false;
  }
  for (const k of keys) if (!replies.has(k)) replies.set(k, null);
  return replies;
}

function parseJson(raw) {
  if (typeof raw !== "string") return { value: null, error: "agent produced no final message" };
  const m = raw.match(/\{[\s\S]*\}/);
  if (!m) return { value: null, error: "no JSON object in reply" };
  try { return { value: JSON.parse(m[0]), error: null }; }
  catch (err) { return { value: null, error: String(err) }; }
}

// ===== 以上是替代 prelude 的部分，程序本体从这里开始 =====

const MAX_FOLDERS = 3;
const CONCURRENCY = 3;

const log = [];
const t0 = Date.now();
let payload = null;

try {
  if (!BACKEND) throw new Error("code mode 里没有多 agent 工具：" + [..._NAMES].sort().join(", "));
  log.push(`backend=${BACKEND}`);

  // ① 程序自己发现工作。那个 filter 等价于 shellLines 的 validate，不能省
  const res = await tools.exec_command({ cmd: "ls -d */ | sed 's#/$##' | sort" });
  const folders = String(res.output || "")
    .split("\n")
    .map((l) => l.trim())
    .filter((l) => /^[A-Za-z0-9._-]+$/.test(l))
    .slice(0, MAX_FOLDERS);
  log.push(`folders=${folders.join(",")}`);

  // 同上：try 里不能用 exit()
  if (folders.length === 0) throw new Error("cwd 下没有子目录，检查 -C");

  // ② prompt 与 demo 版一字不差
  const prompt = (folder) => [
    `分析这一个目录：./${folder}`,
    `回答前必须真的动手：跑 ls -R ${folder} | head -60，读清单文件，真的打开两个源文件。`,
    `只回一行 JSON，不要代码块：`,
    `{"purpose":"<一句话>","deps":["<最多3个>"],"evidence":["<你真的读了的两个文件>"]}`,
    `不要跑到 ./${folder} 外面去。`,
  ].join("\n");

  // ③ 全部派出去，一次 join。没有 handle.meta，靠数组下标维持对应关系
  const keys = await mapLimit(folders, CONCURRENCY, (folder, i) =>
    spawn(prompt(folder), `scan_${i}`),
  );
  // 与 prelude 版一致：v1 三个目标的等待预算最坏为 3 × 240000 = 720000 ms。
  const replies = await collect(keys, 240000);

  // ④ 防御性收口，失败留原文
  const results = folders.map((folder, i) => {
    const reply = replies.get(keys[i]) ?? null;
    const { value, error } = parseJson(reply);
    return {
      folder,
      summary: value,
      parse_error: error,
      raw: value ? undefined : String(reply ?? "").slice(0, 200),
    };
  });

  // ⑤ 释放名额（v1 下必须，v2 下没有这个工具）
  if (BACKEND === "v1") {
    await Promise.all(
      keys.map((k) => tools.multi_agent_v1__close_agent({ target: k }).catch(() => null)),
    );
  }

  payload = {
    succeeded: results.filter((r) => r.summary).length,
    total: results.length,
    results,
  };
} catch (err) {
  log.push(`FATAL: ${String(err)}`);
} finally {
  text(JSON.stringify({ ...payload, log, elapsed_ms: Date.now() - t0 }, null, 2));
}
```

```bash
codex exec --poa ./my-poa -C /path/to/target
```

### 12.3 两版的差异一览

| | 使用 prelude | 只用内置 |
| --- | --- | --- |
| 后端探测 | `AGENT_BACKEND` / `requireAgents()` | 自己从 `ALL_TOOLS` 里认 |
| 派发 + 等待 | `runBatch` 一行 | `mapLimit` + `spawn` + `collect`，两代分支自己写 |
| 结果对回输入 | `handle.meta.folder` | 数组下标 |
| shell 取行 | `shellLines(cmd, {validate})` | `tools.exec_command` + `split` + `filter` |
| 解析回复 | `parseJsonReply` | 自带一个同名实现 |
| 释放名额 | `closeAll(handles)` | `if (BACKEND === "v1")` + `close_agent` |
| 行数 | prelude 341 行 + 本体 | 自备 133 行 + 本体 |

两版的 pragma 都在第 1 行；完整组装规则见《03-concepts.md》§2。

---

## 13. 包这条路上另外两件事

**①** 工具名从 `ALL_TOOLS` 里找，不能写死。尤其是包自带的那些，`mcp__<server>__<tool>` 的前缀会被清洗、重名时加哈希后缀。写法与理由见《07-api-reference.md》§3.1。

**②** 包自带的能力与宿主提供的能力，可靠性不是一回事。包里 `[[capabilities.mcp]]` 申报的 server 起不来会直接拒跑，所以这些工具不用探测；宿主侧的工具（`memories__*`、`clock__curr_time` 之类）仍然要照 [§8](#8-容错四个必须写的地方) 那样探测 + 写降级。两者在 `ALL_TOOLS` 里混在一起，但可靠性完全不同。

改常量要重新打包。`codex exec --poa <目录>` 每次现打包，所以本地改完直接重跑即可；但一份已经发出去的 `.poa` 是定稿，改不了。[§3](#3-把外部输入送进程序) 那三条路里，只有"写死在源码顶部"对包成立。

包的完整格式、边界与打包发布，见《08-packaging.md》文档。
