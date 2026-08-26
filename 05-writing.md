---
title: 编写指南
description: 从空文件到能跑的完整动线，含四处必写的容错与调试习惯，以及打成 `.poa` 包时要改的五件事
---

# 编写指南

[← 工作原理](./04-how-it-works.md) · [返回目录](./index.md) · 下一篇：[模式库](./06-patterns.md)

这是主体文档。前面几篇解释已经存在的东西，这一篇讲从一个空文件开始如何写出一个新程序。

---

## 0. 一个 PoA 程序的五段形状

绝大多数 PoA 程序都是这个骨架：

```js
// @exec: {...}                        ① 配置，必须在第 1 行

const MAX_ITEMS = 3;                    ② 规模常量提到顶部，方便调试时压到 1
const CONCURRENCY = 3;

requireAgents();                        ③ 前置断言：先失败得响亮

const items = await shellLines(...);    ④ 程序自己发现工作
const batch = await runBatch(...);      ⑤ 派发
text(JSON.stringify(...));              ⑥ 收口并交出结果
```

---

## 1. 首行 pragma

```js
// @exec: {"yield_time_ms": 900000, "max_output_tokens": 30000}
```

**必须真的在第 1 行。** 两个字段的默认值都只有 10000（10 秒 / 10000 token），派了 agent 的程序必须调大 `yield_time_ms`。

| 字段 | 计量的是什么 | 默认 |
| --- | --- | --- |
| `yield_time_ms` | 本次运行在脚本仍未跑完时提前交还之前，最多等多久 | 10000 ms |
| `max_output_tokens` | 本次运行返回值的 token 预算，即 `text()` 累积下来准备交回的那份内容 | 10000 |

> [!IMPORTANT]
> **这两个数管的都是「这一次运行」，不是「每个子 agent」，也不是「所有子 agent 的消耗总和」。**
>
> 子 agent 自己烧掉多少 token，这里一概不管，那走的是各自的回合，有各自的上下文窗口。
>
> 并行派发之所以必须调大，是因为另外两件事：
> 整个扇出必须在这一次运行的 `yield_time_ms` 内跑完（没有续跑手段）；
> N 份摘要最后合成一个返回值，一起挤在同一份 `max_output_tokens` 里。

> [!WARNING]
> **pragma 被挤下第 1 行时静默失效。**
> 用 `run.sh` 跑 demo 时 runner 会自动把它提回第 1 行，但自研客户端拼源码时没有任何环节会代劳——
> prelude 有 243 行，直接拼在前面就会把它挤到第 244 行。不报错，只是沿用 10 秒默认值，
> 程序还没等到第一个 agent 就被截断。
>
> 写成 `.poa` 包时这个坑反过来了：`entry` 是原样提交的，没人往前面拼东西，pragma 天然在第 1 行。
> 代价是 prelude 里那 16 个 primitive 一个都不存在，`runBatch`、`parseJsonReply`、`AGENT_BACKEND`
> 全都没有。要用就自己抄进包里（`lib/prelude.js` 本来就是仓库自己写的一层，不是内核的）。

---

## 2. 输出：只有 `text()` 可靠

这是最容易照着现成代码抄错的一节。

| primitive | PoA 客户端看得见吗 |
| --- | --- |
| `text(v)` | ✅ **唯一可靠的文本通道** |
| `image(v)` / `audio(v)` / `generatedImage(v)` | ⚠️ 会进返回值，runner 打成 `[input_image] …` 一行摘要。能证明"东西产出来了"，但读不到内容（写法见 `demos/08_file_and_image.js`） |
| `notify(v)` | ❌ 走另一条通道，收不到 |
| `yield_control()` | ⚠️ 交出前半段之后没法接后半段，约等于提前结束 |
| `exit()` | ✅ 顶层提前 return，可以正常用 |

> [!CAUTION]
> **不要用 `notify()` 打进度。** 现成的示例代码里有用到它的地方，那是给模型看的写法在 PoA 场景下的残留。
>
> 正确做法是攒一个数组，最后随结果一起交出去：
> ```js
> const log = [];
> log.push(`backend=${AGENT_BACKEND} items=${items.length}`);
> // …
> text(JSON.stringify({ result, log, elapsed_ms: Date.now() - t0 }, null, 2));
> ```

> [!NOTE]
> **`yield_control()` 别当"流式输出"用。** 它把已累积的输出交出去之后脚本还在跑，
> 但没有配套的手段去接后半段。长程序的正解是把 `yield_time_ms` 调大。

---

## 3. 外部输入怎么进到程序里

写第二个程序时会立刻撞上这个问题：**PoA 没有启动参数。** 提交接口收的是线程 id 和一个包（`{ threadId, package }`），仍然没有一个放参数的地方。

于是实际可用的只有三条路：

| 想传什么 | 怎么传 |
| --- | --- |
| 少量常量（规模、并发度、目标名） | 直接写死在 `entry` 顶部，或由自研客户端每次现拼一个包再提交 |
| **程序在哪个目录里干活** | `--cwd`，它决定线程的工作目录，也就决定了 `exec_command` 里相对路径的基准 |
| 用哪个模型 | `--model`，或环境变量 `CODEX_DEMO_MODEL` |

> [!WARNING]
> **"把数据文件打进包里"这条路目前走不通，别照直觉写。**
> 包确实会被完整解出来，但解在服务端的一个临时目录里，而这个路径不会以任何形式交给程序。
> 注意症结是"拿不到路径"而不是"没有读的手段"——`exec_command` 是宿主进程执行的，能读整个文件系统，
> 但它不知道该读哪儿（它的相对路径基准是 `--cwd`）。靠 `ls -dt /tmp/...` 去猜那个临时目录不是接口，别依赖。
> 拿得到那个目录的只有包自带的 MCP server（它的 cwd 就是包根，所以 `node mcp/echo.mjs` 才解析得了）。
> 所以"随包带数据"当前只有两种写法：写进 `entry` 的源码里，或者由包自带的 server 去读、程序调它拿。

```bash
./run.sh demos/90_hello.js v1-forced --cwd ../codex-rs
```

> [!TIP]
> **`--cwd` 是最容易忘、后果又最迷惑的一个。** 不给它，工作目录就是执行命令时所在的目录。
> 程序不会报错，只会对着错的东西认真分析一通。

> [!NOTE]
> **`store()` / `load()` 在 PoA 里基本等于全局变量。** 它们跨多次提交存活，
> 但一次运行只发一次提交，结束就关掉进程。除非改用自研客户端、在同一个线程上连发多次，
> 否则用普通 `const` 就够了。

---

## 4. 程序自己去发现工作

```js
// ls -d */ 列出所有子目录（带尾部 /），sed 把那个 / 去掉，得到干净的目录名
const folders = await shellLines("ls -d */ | sed 's#/$##'", {
  // 逐行过滤器：返回 true 的行才留下
  validate: (line) => SAFE_NAME.test(line),
});
```

> [!CAUTION]
> **`validate` 几乎必传，这是所有坑里唯一一类会静默产出错误结果的。**
>
> shell 失败时诊断信息和正常输出走同一个通道。不过滤的话 `No such file or directory`
> 会被当成目录名喂给子 agent，而子 agent 会照着它认真分析一通，
> 最终产出一份看起来完全正常的报告。

> [!WARNING]
> **`validate: SAFE_NAME.test` 这样写会报错。** 必须包一层箭头函数：
> ```js
> validate: (line) => SAFE_NAME.test(line)     // ✅
> validate: SAFE_NAME.test                     // ❌ 方法被从正则上摘下来，this 丢失
> ```

`SAFE_NAME` 是 `/^[A-Za-z0-9._-]+$/`，只放行字母数字和 `. _ -`。带空格、斜杠、中文的路径会被它一并滤掉，需要的话自己写一个更宽的正则。

---

## 5. 写子 agent 的任务文字

**子 agent 的系统提示词改不了。** 派发接口的参数面上根本没有这一项，而且 v2 那边会拒绝未知字段，自作主张塞一个不会被静默忽略，而是直接抛异常。

程序能控制的与不能控制的，是这样分的：

| 层 | 来源 | 程序能否控制 |
| --- | --- | --- |
| 基础指令 | 从父会话继承 | ❌ |
| 角色层（模型、推理强度、工具集） | `agent_type` 指向的角色配置 | ⚠️ 只能**选**宿主已装好的，不能现场定义 |
| **任务内容** | **`message`** | ✅ **完全控制** |

> `agent_type` 是最容易误读的一项：它是一个角色名，不是一段 prompt。传一个宿主没配过的名字会直接报错。

所以 `message` 是程序唯一的控制面，要写扎实。它必须同时做三件事：

| 要素 | 写法 | 防的是什么 |
| --- | --- | --- |
| **强制动手** | 列出要跑的具体命令，并要求回报证据字段（"你实际读了哪两个文件"） | agent 凭常识编一段听起来对的答案 |
| **强制格式** | `只回一行 JSON，不要代码块，不要多余的话，不要复述格式说明` | 回一段散文正则捞不出来；复述了模板则整段解析失败，见 §7 |
| **划定作用域** | `不要跑到 ./<folder> 外面去` | 跑去读整个仓库，烧 token 还超时 |

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

> [!IMPORTANT]
> **这段文字是程序侧唯一能施加的约束。**
> 程序无法进入 agent 内部检查它有没有真的干活，要求它报出证据字段是唯一的抓手。
> 而且这只是"要求"不是"保证"，收口时仍必须防御性解析。

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

`runBatch` 是「按并发度派出去 + 等全部到终态」的合并写法，返回 `[{ handle, reply }]`，顺序与输入一致。

### 6.1 `meta` 不是可选的

agent 的终稿文本里不包含"我是谁"，程序必须自己记。收口时靠 `handle.meta.folder` 把结果对回输入。

### 6.2 `concurrency` 限的是什么

限的是同时发出的派发调用数，不是同时运行的 agent 数。派发一返回 agent 就已经在跑了，限流只作用在"开"这个动作上。

### 6.3 并发度怎么定

两代后端的名额算法不一样，而且只有一代减掉 root：

| 后端 | 可用子 agent 名额 | 默认值 |
| --- | --- | --- |
| **v1** | 配置里那个数，不减 root | **6** |
| **v2** | 配置里那个数减 1（root 线程占一个名额） | 上限默认 4 → **实际只剩 3** |

同一段程序换个 profile，能开的并发数就从 6 变成 3。

> [!WARNING]
> **超订不会排队，派发直接失败。** 所以 `concurrency` 要按上表的「可用名额」估，
> 不是按配置里写的那个数。保险的做法是按「上限减 1」取。

`v2` profile 把上限调到了 8，为的就是把可用名额撑到 7。

### 6.4 递归深度：v2 下没有刹车

子 agent 能不能再派孙 agent，两代行为相反：

| 后端 | 行为 |
| --- | --- |
| **v1** | 深度上限默认为 1，且拦两道——超限的子 agent 在 JS 里根本看不到那组工具，即便看到运行时也会再拦一次。递归会自己停 |
| **v2** | **两道都没有，无条件放行** |

> [!WARNING]
> **写并行派发程序时这是实打实的风险。**
> v2 下刹车只能在 JS 里自己踩：传一个深度计数进 `message`，或干脆只在顶层派发。
>
> 唯一的兜底是 agent 总数上限，那是个数量闸不是深度闸，触发时已经起了一堆 agent。
>
> 而后端是 v1 还是 v2 不由本地配置决定，所以"我这边是 v1，不用管"这个假设不成立。

### 6.5 别设计成流水线

`Promise.race` 拿不到最先完成的那个，程序侧的等待调用是串行的（机制见[工作原理 §4](./04-how-it-works.md#4-并发不等于流式)）。

按「全部派出去 → 一次 join → 统一收口」来组织。

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

> [!WARNING]
> **它的匹配是贪婪的**，从第一个 `{` 吃到最后一个 `}`。
> 所以 agent 只要在正式答案前把 [§5](#5-写子-agent-的任务文字) 那段格式模板复述一遍，
> 两段 `{}` 会被连成一整块，整体解析失败，而不是退回第一段。
> 这就是 §5 那句 `不要复述格式说明` 要写进 prompt 的原因。

> [!TIP]
> **`reply` 是 `null` 和是空字符串不是一回事。**
> 超时未达终态的 handle 会被统一填成 `null`，而 agent 正常结束但没给终稿也可能是 `null`。
> 两者都要当失败处理，但排查方向不同：前者调 `timeoutMs`，后者调 prompt。

---

## 8. 容错：四个必须写的地方

工具失败在 JS 里表现为一个抛出的异常，可以 `try/catch` 兜住，不必让整段程序陪葬。并行派发程序至少要处理这四处：

**① 单个 agent 失败不该拖垮整批。** 把每个 worker 的调用包进 `try/catch`，失败的那个记成 `null` 继续走。

**② `shellLines` 必须传 `validate`。** 见 §4，这是唯一一类会静默产出错误结果的失败。

**③ 解析失败要留原文。** 见 §7。否则只能知道"失败了"，无从判断它到底回了什么。

**④ `closeAll` 放最后。** 它吞掉所有错误、绝不抛。

> [!NOTE]
> **`closeAll` 只对 v1 后端有效**，v2 那组工具里根本没有"关闭"这一个，所以它在 v2 下直接 return。
> 但在 v1 下不能省：已完成的 agent 不关掉仍然占着并发名额。

---

## 9. 调试循环：在没有取消、没有续跑的前提下

约束是硬的：没有任何手段能中止一个正在跑的程序，yield 之后也没法续跑。两条合起来意味着程序跑飞了只能杀进程。

由此有四条习惯。

### ① `yield_time_ms` 分两档写

调试期 `60000`，跑通了再改成 `900000`。用 15 分钟的超时去调一个写错的循环，一次要等 15 分钟。

### ② 先把规模压到 1

顶部把常量提出来，调试时全设成 1：

```js
const MAX_FOLDERS = 3;      // 调试时改 1
const CONCURRENCY = 3;      // 调试时改 1
```

这不是风格问题，是为了让一次失败只花一个 agent 的时间。

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

### ④ 忘了 `await` 会静默丢结果

脚本求值结束时沙箱生命周期即终止，**未 `await` 的 promise 被静默丢弃**。不报错、不警告，只是那部分工作没有发生。

收口前检查每个 `tools.` 调用与每个 prelude 函数调用前面都有 `await`。

---

## 10. 什么活派给 agent，什么活自己干

最常见的浪费是把确定性的活派给 agent。判据：

| 活的性质 | 怎么做 | 为什么 |
| --- | --- | --- |
| 确定性的、shell 能表达的（列目录、数文件、grep） | `tools.exec_command` 自己干 | 快、并行安全、零 token、结果可复现 |
| 需要阅读理解或判断 | 派 agent | 程序做不了 |
| 需要跨来源比对 | 程序做信息路由 + 一个 reducer agent | 单个 worker 只看得见自己那份 |
| 需要**可信度**而不只是答案 | 多个视角互相看不见地做，程序数票 | 一个 agent 的"我很确信"是自我报告 |
| 一次昂贵读取要摊到多轮追问 | 一个长驻 agent + `sendAndWait` | 文件读一遍进上下文，后续追问不再花 token 重读 |

> 最后两行是 PoA 真正的价值所在。前面几行只是并行加速，而强制独立性再数票这件事单个 agent 结构上做不到，它无法假装自己没看过某份材料。

各自的完整写法见[模式库](./06-patterns.md)。

---

## 11. 动手前的检查清单

- [ ] 探针跑过，`backend` 不是 `null`
- [ ] profile 显式给了 `v1-forced` 或 `v2`
- [ ] `--cwd` 指向了真正要扫的目录（忘了给不会报错，只会分析错东西）
- [ ] 首行是 `// @exec:` pragma，**且确实在第 1 行**
- [ ] 调试期 `yield_time_ms` = 60000，规模常量 = 1
- [ ] `requireAgents()` 在最前面
- [ ] 每个 `tools.` 调用和每个 prelude 函数调用前有 `await`
- [ ] `shellLines` 传了 `validate`
- [ ] 每个 agent 回复都过 `parseJsonReply`，失败留原文
- [ ] handle 上挂了 `meta`
- [ ] `concurrency` ≤ 可用名额（v1 按 6、v2 按 3 起算，再减 1）
- [ ] 收口的 `text()` 在 `catch` / `finally` 里也会执行
- [ ] 用到的每个工具都在探针输出的 `all_tools` 里

---

## 12. 完整范例

把上面所有要点合到一起：

```js
// @exec: {"yield_time_ms": 900000, "max_output_tokens": 30000}

// ---- 调试期把这两个都改成 1 ----
const MAX_FOLDERS = 3;
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

  if (folders.length === 0) {
    text(JSON.stringify({ error: "cwd 下没有子目录，检查 --cwd", log }));
    exit();
  }

  // ② prompt 同时做三件事：逼动手、锁格式、划范围
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
    { concurrency: CONCURRENCY },
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

---

## 13. 从 `.js` 到 `.poa`：要改的五件事

上面整篇讲的都是 `demos/*.js` 那种写法：散装一个文件，靠 runner 拼 prelude。要把它变成一个能交给别人跑的[包](./03-concepts.md#9-poa-包自带能力的那条路)，有五处必须动，其中前两处不动就是直接报错。

**① prelude 没了，自己补——但别把 pragma 挤下去。** 包的 `entry` 是原样提交的，`runBatch`、`parseJsonReply`、`AGENT_BACKEND` 这 16 个名字全部不存在。要用就把 `lib/prelude.js` 抄进来，抄在首行 pragma 之后：

```js
// @exec: {"yield_time_ms": 900000, "max_output_tokens": 30000}   ← 第 1 行永远是它
// ---- 以下抄自 workflow-demos/lib/prelude.js ----
async function mapLimit(...) { /* … */ }
// ---- 程序本体 ----
```

把 prelude 抄到文件开头是这一步最容易犯的错：包这条路上没有 runner 替你把 pragma 提回第 1 行（见下面 ④），抄反了就是 10 秒超时，而且不报错。

**② 工具名从 `ALL_TOOLS` 里找，不能写死。** 尤其是包自带的那些，`mcp__<server>__<tool>` 的前缀会被清洗、重名时加哈希后缀。写法与理由见 [API 参考 §3.1](./07-api-reference.md#31-速查表)。

**③ 常量的位置变了。** 原来靠"改文件再 `./run.sh`"调参，现在改完要重新打包（`codex exec --poa <目录>` 会现打包，所以这一步其实是自动的，但发出去的 `.poa` 一旦定稿就改不了了）。[§3](#3-外部输入怎么进到程序里) 那三条路里，只有"写死在源码顶部"对包成立。

**④ pragma 没人替你提回第 1 行了。** 好消息是不用再担心被 prelude 挤下去（因为没有 runner 拼 prelude），坏消息是手工抄 prelude 时会重新制造出同一个问题，见 ①。

**⑤ 命令行开关换了一套。** `--cwd` 和 `--raw` 是 `run_workflow.py` 的参数，`codex exec` 上没有：

| 想干什么 | demo 这条路 | 包这条路 |
| --- | --- | --- |
| 指定工作目录 | `--cwd <dir>` | `--cd <dir>` / `-C <dir>` |
| 看原始返回值 | `--raw` | `--json` |
| 指定模型 | `--model` | `--model` / `-m`（这个两边一样） |

`./run.sh <包> <profile> <额外参数>` 会把额外参数原样转给 `codex exec`，所以在 `run.sh` 后面写 `--cwd` 同样是错的。

> [!NOTE]
> **包自带的能力与宿主提供的能力，可靠性不是一回事。** 包里 `[[capabilities.mcp]]` 申报的 server 起不来会直接拒跑，
> 所以这些工具不用探测；宿主侧的工具（`memories__*`、`clock__curr_time` 之类）仍然要照
> [§8](#8-容错四个必须写的地方) 那样探测 + 写降级。两者在 `ALL_TOOLS` 里混在一起，但可靠性完全不同。

---

[← 工作原理](./04-how-it-works.md) · [返回目录](./index.md) · 下一篇：[模式库](./06-patterns.md)
