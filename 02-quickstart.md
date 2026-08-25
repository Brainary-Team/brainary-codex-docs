---
title: 快速开始
description: 环境准备 → 探针 → 跑通第一个程序 → 写出自己的第一个
---

# 快速开始

[← 概览](./01-overview.md) · [返回目录](./index.md) · 下一篇：[核心概念](./03-concepts.md)

本篇覆盖从环境准备到跑通第一个自建程序的完整动线，全程约 20 分钟，其中大部分时间花在第 0 步的构建上。

---

## 0. 环境准备

### 0.1 二进制：要建两个，`codex-code-mode-host` 需手工喂 V8

> [!IMPORTANT]
> **这一步最容易卡住半天，先做完再往下。**

要建两个二进制，前提并不一样：

| 二进制 | 命令 | 额外前提 |
|---|---|---|
| `codex` | `cargo build -p codex-cli --bin codex` | **无**。`codex-rs/cli/Cargo.toml` 里没有 code-mode 依赖，常规构建即可 |
| `codex-code-mode-host` | `cargo build -p codex-code-mode-host` | **要一份开启 sandbox 特性的预编译 V8** |

卡点全在第二个。它依赖 `codex-code-mode-runtime`，后者声明 `v8 = { features = ["v8_enable_sandbox"] }`（`codex-rs/code-mode-runtime/Cargo.toml:24`）；而 v8 crate 的 `build.rs` 默认从 denoland 拉预编译包，本 workspace 需要的 `ptrcomp_sandbox_release` profile **只发布在 openai/codex 的 release 上**——那是 code mode 迁到独立沙箱宿主进程之后才有的产物。

做法是先手工下载这份沙箱版预编译库，把 `RUSTY_V8_ARCHIVE` 与 `RUSTY_V8_SRC_BINDING_PATH` 指过去，再构建。完整命令见 `workflow-demos/README.md` 的 *Building the binaries* 一节。

> [!CAUTION]
> **不带这两个环境变量时，失败发生在构建阶段，而不是运行阶段。** `build.rs` 会因下载 404 直接 panic（`.github/workflows/ci.yml:88-92` 的注释即为此事），你**拿不到**那个二进制——不是拿到一个建好了却跑不起 code mode 的二进制。所以这个坑在构建期就叫得很响、容易定位，不会留到跑 demo 才暴露；与下面 0.2 的 `CODEX_BIN` 正相反，那个是运行期静默拿错二进制。

> [!WARNING]
> **两个二进制必须在同一个目录里。** codex 是按自己所在目录去找 `codex-code-mode-host` 的。正常 `cargo build` 天然满足这一点，但把 `codex` 单独拷贝到别处就会失败。

### 0.2 连接配置：`.env`

在仓库根建一个 `.env`（已被 gitignore），照 `.env.example` 填：

```bash
OPENAI_API_URL=https://your-provider/v1     # 必须支持 Responses API
OPENAI_API_KEY=sk-...
OPENAI_API_MODEL=gpt-5.6-luna

# 强烈建议显式指定，理由见下方 WARNING
CODEX_BIN=/absolute/path/to/codex-rs/target/debug/codex
```

> [!CAUTION]
> **`CODEX_BIN` 不指定时的失败方式很有迷惑性。**
> 二进制的查找顺序是 `CODEX_BIN` → `cargo metadata` 的 target 目录 → `PATH`。
> **`cargo` 不在 `PATH` 上时**（用 rustup 装但没在 shell 启动文件里引入就是这种情况），
> 中间那档整个失效，直接落到 `PATH` 上的 `codex`——而那通常是上游安装版，
> **根本没有 PoA 用的那个 RPC 方法**。报错会出现在协议层，跟真正的原因隔着好几层。

> [!NOTE]
> `.env` 是被 `set -a; . .env` **当 shell 脚本 source** 的，不是按 properties 文件解析。所以 `=` 两边不能有空格，含空格或 `#` 的值要加引号。另外它会**覆盖**命令行里导出的同名变量，`VAR=x ./run.sh ...` 这种临时覆盖是无效的。

---

## 1. 第一步：跑探针

**这永远是第一条命令。** 它不派 agent、不调模型、不花钱，0.3 秒出结果。

```bash
cd workflow-demos
./run.sh demos/00_probe.js v1-forced
```

看输出里的两个字段：

| 字段 | 怎么读 |
| --- | --- |
| `backend` | `"v1"` 或 `"v2"` → 可以往下走。**`null` → 停下**，环境没配好 |
| `all_tools` | 当前这套配置下**实际能调的工具全集**。写程序时用到的每个工具都该在里面 |

> [!WARNING]
> **`./run.sh` 不给 profile 时默认是 `v1`，而那恰好是个坏的默认值。**
>
> 这个 profile 名字写着 v1，但多数模型会自报 v2，而 v2 的 agent 工具默认进不了 code mode，
> 结果是**一个 agent 工具都没有**，`backend` 为 `null`。
>
> **永远显式写 `v1-forced` 或 `v2`。**

四个 profile 的取舍：

> **⚠ 快照，不是契约。** profile 是 `workflow-demos/config/` 下的配置模板，**不是 brainary 的产品概念**；下表的工具数来自某一次实测，换模型 / provider 就不同。

| profile | 结果 |
| --- | --- |
| `v1` | **陷阱**，多数模型下拿到 0 个 agent 工具 |
| `v1-forced` | 可用，**推荐默认用这个**。它把全部模型钉到 v1 后端 |
| `v2` | 可用，需要更新的模型；并发名额算法与 v1 不同（见[编写指南 §6](./05-writing.md#63-并发度怎么定)） |
| `full` | 后端同 `v1-forced`，但**把可选工具组全打开**（memories / clock / token 预算 / 权限），工具数 13 → 20。demo 06 和 09 要它 |

> `backend` 为 `null` 时不要往下走。后面每一个程序都会以难以定位的方式失败。

**换模型、换 provider、换 profile 之后都要重跑探针。**

---

## 2. 第二步：跑一个现成的程序

```bash
./run.sh demos/01_fanout_summary.js v1-forced --cwd ../sdk
```

这个程序会列出 `sdk/` 下的子目录，给每个派一个 agent 去分析，然后把结果汇总成 JSON。

输出大致长这样：

```json
{
  "succeeded": 3,
  "total": 3,
  "results": [
    {
      "folder": "typescript",
      "summary": {
        "purpose": "codex 的 TypeScript SDK",
        "deps": ["zod", "tsup"],
        "evidence": ["typescript/package.json", "typescript/src/index.ts"]
      },
      "parse_error": null
    }
  ]
}
```

### `--cwd` 是最容易忘、后果又最迷惑的一个参数

> [!TIP]
> **不给 `--cwd`，程序的工作目录就是执行 `./run.sh` 时所在的目录**——按上面的步骤先 `cd workflow-demos` 的话，那就是 `workflow-demos/` 本身。
> 于是 `ls -d */` 列出来的是 `demos/`、`lib/`、`runner/`，而不是待扫描的仓库。
> **程序不会报错，只会认认真真地分析错的东西。**

十一个示例程序对工作目录和 profile 的要求不同：

| demo | 推荐 `--cwd` | profile | 为什么 |
| --- | --- | --- | --- |
| `00_probe.js` | 不需要 | 任意 | 不碰文件系统 |
| `01_fanout_summary.js` | `../sdk` | `v1-forced` | 恰好 3 个子目录，各自带清单文件 |
| `02_map_reduce.js` | `../codex-rs` | `v1-forced` | 前 4 个子目录都是小 crate 且共享内部依赖，跨报告比对才有结果 |
| `03_adversarial_review.js` | `../sdk/typescript` | `v1-forced` | 小而自足，断言容易被 reviewer 真正核验 |
| `04_stateful_dialogue.js` | `../sdk/typescript` | `v1-forced` | 根目录必须有 `Cargo.toml` / `package.json` / `pyproject.toml` / `README.md` 之一，否则程序直接退出 |
| `05_streaming_select.js` | 不需要 | `v1-forced` | 只 sleep |
| `06_capability_matrix.js` | 不需要 | `full`（别的也能跑，只是缺席项更多） | 把每个只读工具各调一次，报告实际拿到的返回形状 |
| `07_exec_session.js` | 不需要 | 任意 | `exec_command` 会话 + `write_stdin`；并行安全的工具与串行跑法对照计时 |
| `08_file_and_image.js` | 不需要（写在 `mktemp -d` 的临时目录，不碰仓库） | 任意 + `CODEX_DEMO_SANDBOX=workspace-write` | `apply_patch` / `view_image` / `image()`，以及写操作要过的三道独立的门 |
| `09_memory_clock_goal.js` | 不需要 | `full` | 带状态的那些工具：磁盘上的 memories、goal 状态机、plan、上下文预算、权限 |
| `10_agent_lifecycle.js` | 不需要 | `v1-forced` / `full` 走 v1 分支，`v2` 走 v2 分支 | prelude 盖住的那层原始工具：`resume_agent`、`interrupt_agent`、多目标等待的陷阱、递归深度 |

> [!TIP]
> **demo 00 和 06-09 一个 agent 都不派，零模型调用，可以随便重跑。**
> 这不是"因为它们跑得快"推出来的——把 provider 指到一个不通的地址
> （`CODEX_DEMO_BASE_URL=http://127.0.0.1:1/v1`）它们照样跑完，输出不变。
> 根线程从来不走模型回合，`thread/codeMode/exec` 直接跑程序，**任何 demo 产生的模型流量都来自它派出去的子 agent**。
>
> **换模型、换 provider、换 profile 之后跑 demo 06**：它会报告哪些工具在、哪些不在、不在的那些卡在哪道门上。

各自证明了什么，见[模式库](./06-patterns.md)。

---

### 2.1 跑一个自带能力的包（可跳过，不影响后面的步骤）

demo 是一个"用宿主给什么算什么"的散装 `.js`；**包**把能力带在身上。以下都在 `workflow-demos/` 下执行：

```bash
./run.sh      poas/00_echo v1-forced   # 目录直接跑，不用先打包
poas/build.sh poas/00_echo             # → ./00_echo.poa，给别人一个文件时才需要
./run.sh      ./00_echo.poa v1-forced
```

`poas/` 下目前只有 `00_echo` 这一个样例，也是最小的那个：manifest 申报一个随包分发的 stdio MCP server，`main.js` 从 `ALL_TOOLS` 里把它找出来调一次。**它不需要 provider**——包本体不采样，`run.sh` 会为包目标兜底一套占位配置。自己写的包放 `workflow-demos/poas/<名字>/`。

> [!WARNING]
> **绕开 `run.sh` 直接敲 `codex exec --poa <包>` 有两个前提，缺一个都跑不起来。**
>
> **① `codex` 必须是你自建的那个。** PATH 上通常是上游安装版，会报 `unexpected argument '--poa'`——就是 [§0.2](#02-连接配置env) 那个坑的第二次发作。用 `"$CODEX_BIN" exec --poa ...`。
>
> **② 当前 `CODEX_HOME` 的 `config.toml` 里得有 `[features.code_mode] enabled = true`。** 这个 feature **默认是关的**，而 `--poa` 不会替你打开它；关着的话 `exec` 工具压根不在工具面上，cell 起不来。`run.sh` 渲染出来的四个 profile 都带这一行，**它的价值不只是兜底 provider**，还包括这个开关和按 profile 隔离的 `CODEX_HOME`。
>
> 换句话说：`codex exec --poa` 是包的**分发形态**（拿到包的人只要环境配好就能跑），不是本仓开发时的便捷方式。开发时用 `run.sh`。

包的格式、边界和两个必踩的坑，见[核心概念 §9](./03-concepts.md#9-poa-包自带能力的那条路)。

---

## 3. 第三步：写出第一个自建程序

新建 `workflow-demos/demos/90_hello.js`，两行：

```js
// @exec: {"yield_time_ms": 60000, "max_output_tokens": 10000}
text(JSON.stringify({ backend: AGENT_BACKEND, tool_count: ALL_TOOLS.length }, null, 2));
```

```bash
./run.sh demos/90_hello.js v1-forced
```

这一步只验证一件事：**程序里写下的字符串能原样回到终端**。回不来就是环境问题，不是程序问题——不要带着一条坏掉的链路去调后面的并行派发逻辑。

> [!NOTE]
> **首行那个 `// @exec:` 必须真的在第 1 行。** 它是这次运行的配置，两个字段的默认值都只有 10000（10 秒 / 10000 token）。派了 agent 的程序**必须调大** `yield_time_ms`，否则程序还没等到第一个 agent 就被截断。详见[编写指南 §1](./05-writing.md#1-首行-pragma)。

### 接上第一个 agent

```js
// @exec: {"yield_time_ms": 900000, "max_output_tokens": 30000}
requireAgents();                                    // ① 先失败得响亮

const [{ reply }] = await runBatch(
  [{ message: '用一句话说明当前目录是做什么的。只回一行 JSON：{"purpose":"..."}' }],
  { concurrency: 1 },
);

const { value, error } = parseJsonReply(reply);      // ② 收口必须防御性解析
text(JSON.stringify({
  value,
  error,
  raw: value ? undefined : String(reply ?? "").slice(0, 200),
}));
```

```bash
./run.sh demos/90_hello.js v1-forced --cwd ../codex-rs
```

三个要点：

1. **`requireAgents()` 放在最前面。** 没有它，无后端时会在半路撞上一个 `tools.xxx is not a function`；有它，异常消息里直接列出当前可用的全部工具名。
2. **收口一定要防御性解析。** 没有任何机制能强制 agent 按格式回答，prompt 里的要求只是要求。
3. **解析失败时把原文留下来**——否则只能知道"失败了"，无从判断它到底回了什么。

---

## 4. 读懂一个完整的程序

下面是 `01_fanout_summary.js` 的骨架（略作简化）。**先整体扫一眼，下面逐段拆。**

```js
// @exec: {"yield_time_ms": 900000, "max_output_tokens": 30000}

// ① 用 shell 找出要处理的目录，最多取 3 个
const folders = (await shellLines("ls -d */ | sed 's#/$##' | sort", {
  validate: (line) => SAFE_NAME.test(line),
})).slice(0, 3);

// ② 给每个目录写一段任务说明
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

// ③ 一次派出去，等全部完成
const batch = await runBatch(
  folders.map((folder, i) => ({
    message: prompt(folder),
    name: `scan_${i}`,
    meta: { folder },
  })),
  { concurrency: 3 },
);

// ④ 收口：解析每个助手的回答，失败的留下原文
const results = batch.map(({ handle, reply }) => {
  const { value, error } = parseJsonReply(reply);
  return { folder: handle.meta.folder, summary: value, parse_error: error };
});

// ⑤ 交出结果
text(JSON.stringify({
  succeeded: results.filter((r) => r.summary).length,
  total: results.length,
  results,
}, null, 2));
```

### 逐段拆

**① `shellLines`** 跑一条 shell 命令、把输出按行拿回来。

**注意那个 `validate`：** shell 出错时，报错信息和正常输出走同一个通道。不过滤的话 `No such file or directory` 会被当成目录名喂给 agent，而 agent 会认认真真地去分析它。**这是所有坑里唯一一类会静默产出错误结果的**，`validate` 几乎必传。

**② 任务说明**（给 agent 的 prompt）写得很刻意，三件事缺一不可：

- **逼它真干活**：列出具体要跑的命令，并要求报告"你实际读了哪两个文件"。不这么写，agent 看见 `codex-protocol` 这种名字就能编出一段听着很对的描述
- **锁死输出格式**：`只回一行 JSON`。因为没有任何机制能强制它照做，只能这么要求，然后在 ④ 防御性解析
- **划定范围**：不然它会跑去读整个仓库，又慢又贵

> 程序能给 agent 的**只有这段任务文字**——它的系统提示词改不了。所以这段要写扎实。

**③ `runBatch`** 是"派出去 + 等全部完成"的合并写法。`concurrency` 是同时最多几个在派。

**`meta` 看着不起眼但不能省**——agent 的回答里不包含"我是谁"，哪个结果对应哪个目录只能由程序自己记。

**④ 收口。** `parseJsonReply` 从自由文本里抠出第一个 JSON 对象，返回 `{value, error}`。**永远要处理 error 分支。**

**⑤ `text()`** 是唯一可靠的输出手段——只有写进去的内容才会回到终端。

---

## 5. 调试协议层：`--raw`

`text()` 写进去的内容最终落在 RPC 返回值里，runner 逐项取出打印，认得出 JSON 就再格式化一遍。想看**原始返回值**加 `--raw`：

```bash
./run.sh demos/90_hello.js v1-forced --raw
```

> [!NOTE]
> **跑包时这个开关叫 `--json`。** `--raw` 是 `run_workflow.py` 的参数，而包这条路不经过它——额外参数被原样转给 `codex exec`，那边对应的是 `--json`。同理 `--cwd` 在包这条路上叫 `--cd` / `-C`。对照表见[编写指南 §13 ⑤](./05-writing.md#13-从-js-到-poa要改的五件事)。

调试协议层问题时这是唯一手段。

---

## 6. 本篇的产出

- [x] 环境跑通，探针的 `backend` 不是 `null`
- [x] 跑通一个现成的并行派发程序
- [x] 写出一个最小的自建程序，并接上第一个 agent
- [x] 了解 `--cwd` 和 profile 不给会怎样

**接下来**：

- 理解刚才那些名字（`shellLines` / `runBatch` 从哪来）→ [核心概念](./03-concepts.md)
- 着手写一个真正的程序 → [编写指南](./05-writing.md)
- 查看更多现成形状 → [模式库](./06-patterns.md)

---

[← 概览](./01-overview.md) · [返回目录](./index.md) · 下一篇：[核心概念](./03-concepts.md)
