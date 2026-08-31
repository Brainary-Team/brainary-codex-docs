# API 参考

书写 PoA 时，能使用的工具有三类不同的来源：

| 类别 | 来源 | 数量 |
| --- | --- | --- |
| [全局 primitive](#1-全局-primitive12-个) | codex 内置，直接调用 | 12（不含全局对象 `tools` 本身） |
| [prelude primitive](#2-prelude-primitive16-个) | 不是内置的，是一层普通 JS 封装。用之前要把 [§2.5](#25-prelude-全文) 的全文抄进程序 | 16 |
| [内置工具](#3-内置工具) | codex 内置，挂在 `tools` 上 | 30 + MCP + provider 相关 |

---

## 1. 全局 primitive（12 个）

不在 `tools` 上，直接调用。

| primitive | 干什么 | PoA 下 |
| --- | --- | --- |
| `text(v)` | 往本次运行的返回值里追加一条文本 | 唯一可靠的输出手段 |
| `image(v)` / `audio(v)` / `generatedImage(v)` | 追加图片 / 音频 / 生成图，与 `text()` 同族 | 少见 |
| `store(k, v)` / `load(k)` | 会话级 KV。程序私有，模型看不见 | 偶尔 |
| `ALL_TOOLS` | 当前可用工具的 `{ name, description }` 数组；每项带 per-tool TypeScript declaration | 探测环境和运行时查声明用 |
| `exit()` | 结束整段脚本。实现方式是抛一个内部哨兵异常，只有抛到顶层才被识别为正常退出 | ⚠️ 不能放在带 `catch` 的 `try` 里；只有 `finally` 时安全 |
| `setTimeout` / `clearTimeout` | 沙箱里全部的定时器能力就这两个 | 偶尔 |
| `notify(v)` | 不等程序结束，立刻额外送出一条内容 | ❌ 实测 PoA 客户端收不到，不要用它输出进度 |
| `yield_control()` | 先把已攒的输出交出去，程序继续跑 | ❌ 实测没有可用的续跑手段，约等于提前结束 |

---

## 2. prelude primitive（16 个）

> **这 16 个不是内置的。** 它们是一层普通 JavaScript 封装，全文在 [§2.5](#25-prelude-全文)（341 行）。
>
> 存放的位置：首行 pragma 之后。

### 2.1 派 agent / 收结果

| 名字 | 签名（含默认值） | 干什么 | 要注意 |
| --- | --- | --- | --- |
| `runBatch` | `runBatch(specs, { concurrency = 3, timeoutMs = 300000 })` | 最常用的一个。派一批 + 等全部完成。`specs` 是 `[{ message, name?, meta? }]`，返回 `[{ handle, reply }]`，顺序与输入一致 | `reply` 是字符串或 `null`；失败来源见《05-writing.md》§7。v1 的总等待预算最坏为 N × `timeoutMs`，必须留在首行 pragma 的 `yield_time_ms` 内 |
| `spawnAgent` | `spawnAgent(message, { name?, meta? })` | 派一个，立刻返回 handle `{ key, label, meta }` | 用 `meta` 关联输入；`name` 只在 v2 生效，命名与重试规则见表后。helper 不继承父线程历史，也不暴露 `fork_turns` |
| `spawnMany` | `spawnMany(specs, concurrency = 3)` | 派一批，只派不等，返回 handle 数组 | 想边派边干别的时用它，否则直接用 `runBatch` |
| `collectAll` | `collectAll(handles, timeoutMs = 300000, pollMs = 15000)` | 等一批 handle 全部到终态，返回 `Map`：`handle.key → 最终答复 \| null` | 没在超时内完成的会被填成 `null`，不会抛错，要自己检查。等待下限默认 10000 ms，但 v2 宿主配置可改为 0；prelude 会从当前后端的工具声明读取 min/max，解析失败才回落默认值，并把每次原生 wait 钳进该范围。v2 max 为 0 时只取一次列表快照，不忙轮询。`pollMs` 只在 v2 生效 |
| `sendAndWait` | `sendAndWait(handle, message, { timeoutMs = 180000, pollMs = 3000 })` | 对一个还活着的 agent 追问，等它给出新答复 | 两代的追问前置状态和超时预算不同，见表后 |
| `closeAll` | `closeAll(handles)` | 关掉这批 agent。吞掉所有错误，不会抛出 | 只对 v1 后端有效（v2 那组工具里没有"关闭"）；生命周期用法见《05-writing.md》§6.3 |

### 2.2 工具函数

| 名字 | 签名 | 干什么 | 要注意 |
| --- | --- | --- | --- |
| `shellLines` | `shellLines(cmd, { validate = null })` | 跑一条 shell 命令，把输出按行拿回来：逐行 trim、丢掉空行 | `validate` 几乎必传。shell 出错时报错信息和正常输出走同一个通道。`exec_command` 默认约 10 秒就让出；长命令可能只返回 `session_id`，而当前封装只取这次的 `output`、不会续接，后续输出会丢失 |
| `parseJsonReply` | `parseJsonReply(raw)` | 从自由文本里抠 JSON，返回 `{ value, error }` | 匹配是贪婪的：取第一个 `{` 到最后一个 `}` 的整段。回复里出现两段 `{}` 时整体解析失败，不会退回第一段——详见下方 WARNING。三种失败各有不同 `error` 文案：不是字符串 / 没找到 `{}` / 解析异常。永远处理 error 分支 |
| `mapLimit` | `mapLimit(items, limit, fn)` | 限流并发 map，最多 `limit` 个同时在飞，结果保持输入顺序 | 非正 `limit` 按 1 处理；空输入仍启动 0 个 worker。`runBatch` / `spawnMany` 内部就是用它限流的 |
| `SAFE_NAME` | 常量 `/^[A-Za-z0-9._-]+$/` | 现成的白名单正则，配合 `validate` 用 | 只放行字母数字和 `. _ -`，带空格、斜杠、中文的路径会被一并滤掉 |

### 2.3 环境探测

| 名字 | 是什么 | 干什么 |
| --- | --- | --- |
| `AGENT_BACKEND` | 常量，`"v1"` / `"v2"` / `null` | 启动时扫一遍 `ALL_TOOLS` 自动判定后端。`null` 就是这套配置下一个 agent 工具都没有 |
| `requireAgents()` | 函数，返回后端名 | 断言有可用后端，没有就抛错——且错误信息里带上当前全部可用工具名，这是最快的排错入口 |

### 2.4 prelude 全文

整段抄进 `main.js`，抄在首行 `// @exec:` pragma 之后。它只用到内置的 `tools` 与 `ALL_TOOLS`，没有任何外部依赖。

```js
// ---- 以下 341 行是 prelude，程序本体从它下面开始 ----

const _TOOL_NAMES = new Set(ALL_TOOLS.map((t) => t.name));
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
  return Object.values(names).every((name) => _TOOL_NAMES.has(name)) ? names : null;
})();
const AGENT_BACKEND = _TOOL_NAMES.has("multi_agent_v1__spawn_agent")
  ? "v1"
  : _V2_TOOLS ? "v2" : null;

/** 从当前后端的 wait_agent 声明读取动态 min/max；读不到时回落到宿主默认值。 */
function _readWaitLimits() {
  const waitName =
    AGENT_BACKEND === "v1" ? "multi_agent_v1__wait_agent" : _V2_TOOLS?.wait_agent;
  const description = ALL_TOOLS.find((t) => t.name === waitName)?.description || "";
  const min = Number(description.match(/\bmin(?:imum)?\b[^0-9]{0,24}(\d+)/i)?.[1]);
  const max = Number(description.match(/\bmax(?:imum)?\b[^0-9]{0,24}(\d+)/i)?.[1]);
  if (!Number.isFinite(min) || !Number.isFinite(max) || min < 0 || max < min) {
    return { min: 10000, max: 3600000 };
  }
  return { min, max };
}

const _WAIT_LIMITS = _readWaitLimits();
const _clampWaitMs = (ms) => Math.min(_WAIT_LIMITS.max, Math.max(_WAIT_LIMITS.min, ms));
/** v2 sibling 名必须唯一；nonce + 序号覆盖当前 cell，宿主冲突再有限重试。 */
const _agentNonce = Date.now().toString(36);
let _agentSeq = 0;
const _SPAWN_ATTEMPTS = 3;

/** 没有任何多 agent 工具进到 code mode 时抛错，并列出当前可用的工具名。 */
function requireAgents() {
  if (AGENT_BACKEND) return AGENT_BACKEND;
  throw new Error(
    "no multi-agent tools exposed to code mode; available: " +
      ALL_TOOLS.map((t) => t.name).sort().join(", "),
  );
}

/** 对 items 跑 fn，最多 limit 个同时在飞；非正 limit 按 1 处理。结果保持输入顺序。 */
async function mapLimit(items, limit, fn) {
  const results = new Array(items.length);
  let next = 0;
  const worker = async () => {
    for (;;) {
      const index = next++;
      if (index >= items.length) return;
      results[index] = await fn(items[index], index);
    }
  };
  await Promise.all(
    Array.from({ length: Math.min(Math.max(1, limit), items.length) }, worker),
  );
  return results;
}

/**
 * 派一个 agent，返回与后端无关的 handle：{ key, label, meta }。
 * key 是后端后续 wait / send / close 认的那个标识。
 */
async function spawnAgent(message, opts = {}) {
  const backend = requireAgents();
  if (backend === "v1") {
    const a = await tools.multi_agent_v1__spawn_agent({ message });
    return { key: a.agent_id, label: a.nickname || a.agent_id, meta: opts.meta };
  }
  const cleaned = String(opts.name ?? "task")
    .toLowerCase()
    .replace(/[^a-z0-9_]/g, "_");
  const base = !cleaned || cleaned === "root" || cleaned === "_" ? "task" : cleaned;
  for (let attempt = 0; attempt < _SPAWN_ATTEMPTS; attempt++) {
    const seq = _agentSeq++;
    const taskName = attempt === 0 ? `${base}_${seq}` : `${base}_${_agentNonce}_${seq}_${attempt}`;
    try {
      const a = await tools[_V2_TOOLS.spawn_agent]({
        task_name: taskName,
        message,
        fork_turns: "none",
      });
      return { key: a.task_name, label: a.task_name, meta: opts.meta };
    } catch (err) {
      if (!/already exists/i.test(String(err)) || attempt + 1 === _SPAWN_ATTEMPTS) throw err;
    }
  }
  throw new Error("unreachable spawn retry state");
}

/** 限流派一批。specs 是 [{message, name?, meta?}]。 */
async function spawnMany(specs, concurrency = 3) {
  return mapLimit(specs, concurrency, (spec) =>
    spawnAgent(spec.message, { name: spec.name, meta: spec.meta }),
  );
}

const _isFinalStatus = (status) =>
  status === "shutdown" ||
  status === "not_found" ||
  (status && typeof status === "object" && ("completed" in status || "errored" in status));

const _replyOf = (status) =>
  status && typeof status === "object" && "completed" in status
    ? status.completed ?? null
    : null;

/**
 * 只收已确认到达终态的 handle；Map 中存在 key 但值为 null，表示终态没有答复。
 * previous 非空时忽略与上一轮相同的终态答复；allowInterrupted 只供追问前确认可恢复状态。
 */
async function _collectTerminal(
  handles,
  timeoutMs,
  pollMs,
  previous = null,
  allowInterrupted = false,
) {
  const backend = requireAgents();
  const replies = new Map();
  timeoutMs = Math.max(_WAIT_LIMITS.min, timeoutMs);
  pollMs = Math.max(_WAIT_LIMITS.min, pollMs);

  const record = (handle, status) => {
    if (
      !_isFinalStatus(status) &&
      !(backend === "v2" && allowInterrupted && status === "interrupted")
    ) {
      return;
    }
    const reply = _replyOf(status);
    if (previous?.has(handle.key) && previous.get(handle.key) === reply) return;
    replies.set(handle.key, reply);
  };

  if (handles.length === 0) return replies;

  if (backend === "v1") {
    // 一次只等一个：多目标等待会反复返回最先完成的那个。
    for (const h of handles) {
      const deadline = Date.now() + timeoutMs;
      let firstWait = true;
      while (!replies.has(h.key)) {
        const remainingMs = deadline - Date.now();
        if (!firstWait && (remainingMs <= 0 || remainingMs < _WAIT_LIMITS.min)) break;
        const outcome = await tools.multi_agent_v1__wait_agent({
          targets: [h.key],
          timeout_ms: _clampWaitMs(remainingMs),
        });
        firstWait = false;
        record(h, (outcome.status || {})[h.key]);
        if (_WAIT_LIMITS.max === 0) break;
      }
    }
    return replies;
  }

  const deadline = Date.now() + timeoutMs;
  let firstWait = true;
  for (;;) {
    const listed = await tools[_V2_TOOLS.list_agents]({});
    for (const entry of listed.agents || []) {
      const match = handles.find(
        (h) => entry.agent_name === h.key || entry.agent_name.endsWith(`/${h.key}`),
      );
      if (match && !replies.has(match.key)) record(match, entry.agent_status);
    }
    if (replies.size >= handles.length) break;

    // 每次 wait 后都会再做一次 list；第一次允许按 min 等待，避免 timeout === min 时零 wait。
    const remainingMs = deadline - Date.now();
    if (_WAIT_LIMITS.max === 0) break;
    if (!firstWait && (remainingMs <= 0 || remainingMs < _WAIT_LIMITS.min)) break;
    await tools[_V2_TOOLS.wait_agent]({
      timeout_ms: _clampWaitMs(Math.min(pollMs, remainingMs)),
    });
    firstWait = false;
  }
  return replies;
}

/** 对一个 v2 handle 只取一次当前列表快照；列表缺席与非终态分开返回。 */
async function _snapshotV2(handle) {
  const listed = await tools[_V2_TOOLS.list_agents]({});
  const entry = (listed.agents || []).find(
    (item) => item.agent_name === handle.key || item.agent_name.endsWith(`/${handle.key}`),
  );
  return entry ? { found: true, status: entry.agent_status } : { found: false, status: null };
}

/**
 * 等每个 handle 到终态。
 * 返回 Map：handle.key → 最终答复字符串（或 null）。
 */
async function collectAll(handles, timeoutMs = 300000, pollMs = 15000) {
  const replies = await _collectTerminal(handles, timeoutMs, pollMs);
  for (const h of handles) if (!replies.has(h.key)) replies.set(h.key, null);
  return replies;
}

/** 派一批 + 等全部完成，返回 [{handle, reply}]，顺序与输入一致。 */
async function runBatch(specs, { concurrency = 3, timeoutMs = 300000 } = {}) {
  const handles = await spawnMany(specs, concurrency);
  const replies = await collectAll(handles, timeoutMs);
  return handles.map((h) => ({ handle: h, reply: replies.get(h.key) ?? null }));
}

/**
 * 对一个还活着的 agent 追问，等它给出**新**答复。
 * v1 必须先取得终态基准；v2 对 final / interrupted 直接追问，列表缺席也直接尝试 followup。
 * 明确仍在运行时才先等上一轮。两阶段各有 timeoutMs 预算，另有工具调用开销。
 */
async function sendAndWait(handle, message, { timeoutMs = 180000, pollMs = 3000 } = {}) {
  const backend = requireAgents();
  timeoutMs = Math.max(_WAIT_LIMITS.min, timeoutMs);
  pollMs = Math.max(_WAIT_LIMITS.min, pollMs);
  if (backend === "v1") {
    const before = await _collectTerminal([handle], timeoutMs, pollMs);
    if (!before.has(handle.key)) return null;
    await tools.multi_agent_v1__send_input({ target: handle.key, message });
    const after = await _collectTerminal([handle], timeoutMs, pollMs, before);
    return after.get(handle.key) ?? null;
  }

  let before = new Map();
  const snapshot = await _snapshotV2(handle);
  if (snapshot.found) {
    if (_isFinalStatus(snapshot.status) || snapshot.status === "interrupted") {
      before.set(handle.key, _replyOf(snapshot.status));
    } else {
      before = await _collectTerminal([handle], timeoutMs, pollMs, null, true);
      if (!before.has(handle.key)) return null;
    }
  }
  await tools[_V2_TOOLS.followup_task]({ target: handle.key, message });
  const after = await _collectTerminal([handle], timeoutMs, pollMs, before);
  return after.get(handle.key) ?? null;
}

/** 尽力关闭，永不抛出。v2 那组工具里没有"关闭"，所以对 v2 是空操作。 */
async function closeAll(handles) {
  if (AGENT_BACKEND !== "v1") return;
  await Promise.all(
    handles.map((h) => tools.multi_agent_v1__close_agent({ target: h.key }).catch(() => null)),
  );
}

/** 从 agent 的自由文本里抠出第一个 JSON 对象，返回 {value, error}。 */
function parseJsonReply(raw) {
  if (typeof raw !== "string") return { value: null, error: "agent produced no final message" };
  const match = raw.match(/\{[\s\S]*\}/);
  if (!match) return { value: null, error: "no JSON object in reply" };
  try {
    return { value: JSON.parse(match[0]), error: null };
  } catch (err) {
    return { value: null, error: String(err) };
  }
}

/** 跑一条 shell 命令，把输出按行取回：逐行 trim、丢掉空行。 */
async function shellLines(cmd, { validate = null } = {}) {
  const res = await tools.exec_command({ cmd });
  const lines = String(res.output || "")
    .split("\n")
    .map((l) => l.trim())
    .filter((l) => l.length > 0);
  // shell 失败时诊断信息与正常输出走同一个通道；不过滤就会把它们当数据送进 agent 的 prompt。
  return validate ? lines.filter(validate) : lines;
}

const SAFE_NAME = /^[A-Za-z0-9._-]+$/;

// ---------------------------------------------------------------------------
// 能力探测
//
// 以上都假定自己要调的工具存在。可选工具组（memories、clock、MCP、provider 决定的那几个）
// "这套配置下没有"是正常结果，不是 bug。没有下面这几个函数，碰一下可选工具要么在
// `tools.x is not a function` 上炸掉，要么每个调用点都手写一遍 try/catch。
// ---------------------------------------------------------------------------

/** name 在这套配置、模型与 provider 下有没有进到 code mode。 */
function hasTool(name) {
  return _TOOL_NAMES.has(name);
}

/**
 * 调一个工具，不让"缺席"或"调失败"结束整个程序。永不抛。返回：
 *   { name, status: "ok" | "absent" | "error", value, error, ms }
 *
 * args 原样透传，所以 freeform 工具直接给字符串
 * （`callTool("apply_patch", "*** Begin Patch\n...")`）。
 */
async function callTool(name, args = {}) {
  if (!hasTool(name)) {
    return { name, status: "absent", value: null, error: "not in ALL_TOOLS", ms: 0 };
  }
  const t0 = Date.now();
  try {
    return { name, status: "ok", value: await tools[name](args), error: null, ms: Date.now() - t0 };
  } catch (err) {
    // 工具失败在 JS 里是一个抛出的异常，不是一个错误返回值。
    return { name, status: "error", value: null, error: String(err), ms: Date.now() - t0 };
  }
}

/**
 * 把一个运行时值描述成结构串，如 `object{output: string, exit_code: number}`。
 * 声明为 `Promise<unknown>` 的那些工具对返回形状没有任何承诺，唯一的办法就是看实际回来的是什么。
 */
function shapeOf(value, depth = 1) {
  if (value === null) return "null";
  if (Array.isArray(value)) {
    if (value.length === 0) return "array(0)";
    return depth <= 0
      ? `array(${value.length})`
      : `array(${value.length}) of ${shapeOf(value[0], depth - 1)}`;
  }
  const type = typeof value;
  if (type !== "object") return type;
  const keys = Object.keys(value).sort();
  if (keys.length === 0) return "object{}";
  if (depth <= 0) return `object{${keys.join(", ")}}`;
  return `object{${keys.map((k) => `${k}: ${shapeOf(value[k], depth - 1)}`).join(", ")}}`;
}

/** 跑 fn 并带上墙钟耗时，永不抛。返回 { ok, value, error, ms }。 */
async function timed(fn) {
  const t0 = Date.now();
  try {
    return { ok: true, value: await fn(), error: null, ms: Date.now() - t0 };
  } catch (err) {
    return { ok: false, value: null, error: String(err), ms: Date.now() - t0 };
  }
}

// ---- prelude 到此为止 ----
```

---

## 3. 内置工具

挂在全局对象 `tools` 上，能不能用取决于配置与 provider。

### 3.1 按名字查一个工具在不在

"默认可得"一列：

| 记号 | 含义 |
| :---: | --- |
| ✅ | 默认配置下就在 |
| ⚙ | 需打开对应 feature 或配置 |
| ⚠ | 默认可调用，但声明不进 CodeModeOnly 的 `exec` 说明正文（`Deferred` 档）；运行时仍可从 `ALL_TOOLS[].description` 读取完整 per-tool 声明 |
| 探针 | 外挂 MCP server 提供，不是 codex 内建 |

#### 执行与文件（5）

| 工具 | 作用 | 默认可得 | 并行安全 |
| --- | --- | --- | :---: |
| [`exec_command`](#exec_command) | 在 PTY 中跑一条命令；进程未结束时返回可续接的 `session_id` | ✅ ConPTY 可用时 | ✅ |
| [`write_stdin`](#write_stdin) | 向已有 exec 会话写 stdin 并取回最近输出，也可空写用作轮询 | ✅ 同上 | ✅ |
| [`shell_command`](#shell_command) | 跑一条 shell 脚本，无会话概念的一次性执行 | ⚙ 仅回落路径可见；走 `exec_command` 路径时以 `Hidden` 注册，code mode 看不见 | ✅ |
| [`apply_patch`](#apply_patch) | 编辑文件。入参是裸字符串不是对象 | ✅ 需有 environment | ❌ |
| [`view_image`](#view_image) | 把磁盘上已存在的本地图片读进上下文 | ✅ 需有 environment | ✅ |

> **shell 工具二选一，但依据不是操作系统。** unified exec feature 开启且 ConPTY 可用时使用 `exec_command` + `write_stdin`，否则回落到 `shell_command`；现代 Windows 通常也走前者。回落还要求存在单一本地环境。走 unified exec 路径时，为兼容 legacy 而注册的 `shell_command` 以 `Hidden` 曝光，不会进入 `ALL_TOOLS`，PoA 也无法调用。两条路径的命令参数分别是 `cmd` 和 `command`。

#### 计划与目标（4）

| 工具 | 作用 | 默认可得 |
| --- | --- | --- |
| [`update_plan`](#update_plan) | 覆写任务计划列表，同一时刻至多一步 `in_progress` | ✅ |
| [`create_goal`](#create_goal) | 开启一个活跃目标，可带 token 预算 | ✅ |
| [`get_goal`](#get_goal) | 读当前目标的状态、预算、用量 | ✅ |
| [`update_goal`](#update_goal) | 把目标标记为 `complete` 或 `blocked` | ✅ |

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

> 这 4 条要 `[features] memories = true` 且 `[memories] dedicated_tools = true`，两个默认都是假，只开一个得到 0 个记忆工具。

#### 子代理 v1（5）

| 工具 | 作用 | 默认可得 |
| --- | --- | --- |
| [`multi_agent_v1__spawn_agent`](#multi_agent_v1__spawn_agent) | 派生子代理，返回 `agent_id` | ⚠ |
| [`multi_agent_v1__wait_agent`](#multi_agent_v1__wait_agent) | 阻塞等待代理进入终态；多 id 时等最先结束的那个 | ⚠ |
| [`multi_agent_v1__send_input`](#multi_agent_v1__send_input) | 给已有代理发消息，`interrupt=true` 可立刻改变方向 | ⚠ |
| [`multi_agent_v1__resume_agent`](#multi_agent_v1__resume_agent) | 恢复已关闭的代理 | ⚠ |
| [`multi_agent_v1__close_agent`](#multi_agent_v1__close_agent) | 关闭代理及其后代，释放并发额度 | ⚠ |

#### 子代理 v2（6）

> 下表与 §4 的 `collaboration__*` 名称是默认 namespace 的静态快照，不是可执行代码应依赖的固定前缀。程序应按 §2.5 的方式从 `ALL_TOOLS` 找到带 `task_name` / `fork_turns` 的 `*spawn_agent`，再验证并保存同前缀的实际工具名。

| 工具 | 作用 | 默认可得 |
| --- | --- | --- |
| [`collaboration__spawn_agent`](#collaboration__spawn_agent) | 按 `task_name` 派生代理，`fork_turns` 控制带多少上下文 | ⚙ 见下 |
| [`collaboration__wait_agent`](#collaboration__wait_agent) | 等任意存活代理的信箱更新，只返回摘要而非内容 | ⚙ 见下 |
| [`collaboration__send_message`](#collaboration__send_message) | 投递一条消息，不触发新一轮 | ⚙ 见下 |
| [`collaboration__followup_task`](#collaboration__followup_task) | 下发后续任务，空闲时触发一轮 | ⚙ 见下 |
| [`collaboration__interrupt_agent`](#collaboration__interrupt_agent) | 打断当前轮次，代理本身还活着 | ⚙ 见下 |
| [`collaboration__list_agents`](#collaboration__list_agents) | 列出存活代理及其状态。v2 下拿最终答复要靠它 | ⚙ 见下 |

> 这 6 条要 `[features.multi_agent_v2] enabled = true`（默认关）且 `non_code_mode_only = false`。后者默认为真，此时整组只给模型直接调，PoA 完全拿不到，两行必须一起写。

#### MCP（5）

| 工具 | 作用 | 默认可得 | 并行安全 |
| --- | --- | --- | :---: |
| [`list_mcp_resources`](#list_mcp_resources) | 列出各 MCP server 提供的资源 | ⚙ 配了任意 MCP server | ✅ |
| [`list_mcp_resource_templates`](#list_mcp_resource_templates) | 列出带参数的资源模板 | ⚙ 同上 | ✅ |
| [`read_mcp_resource`](#read_mcp_resource) | 按 server 名 + URI 读取资源 | ⚙ 同上 | ✅ |
| [`mcp__probe__echo`](#mcp__probe__echo) | 示例：有 `outputSchema` 时的渲染结果 | 探针 | 有条件 |
| [`mcp__probe__no_output_schema`](#mcp__probe__no_output_schema) | 示例：没有 `outputSchema` 时的渲染结果 | 探针 | 有条件 |

> 除这 3 个 resource 工具外，每个 MCP server 的每个工具还会各渲染出一个 `mcp__<server>__<tool>`，数量取决于挂载了哪些 server——包括包自带的那些。

`mcp__<server>__<tool>` 这个形状不是契约，不要硬编码。前缀由宿主的命名规则决定，命名空间会被清洗，重名时还会加哈希后缀。从 `ALL_TOOLS` 按后缀匹配只能扛住前缀变化，扛不住工具名触发 64 字符截断并追加 SHA-1 的 12 位哈希后缀；那时原来的 `<server>__<tool>` 尾段可能已经不存在。

下面以一个包自带的、名为 `echo` 的 server 上那个 `echo` 工具为例，它在当前配置下渲染成 `mcp__echo__echo`：

```js
// 只规避前缀变化；哈希后缀或 64 字符截断仍会让匹配失败
const matches = ALL_TOOLS.map((t) => t.name).filter((n) => /(^|__)echo__echo$/.test(n));
if (matches.length !== 1) throw new Error(`expected exactly one echo tool, found ${matches.length}`);
const echo = matches[0];
const r = await tools[echo]({ text: "hi" });
```

那句断言不是冗余：它的价值是把命名规则变化显式暴露成错误，而不是静默调错工具；它并没有把后缀本身变成稳定契约。

一般 MCP 工具的并行安全是"有条件"的，而这是唯一可用的手段。判据是"工具自己声明并行安全，或带只读标注"，所以一个把自己标成只读的 MCP 工具就是并行安全的。要让派发出去的耗时任务真正并行，把它放进标了只读的 MCP server。

包自带的 server 也走这条判据，但只走得通后半条：包起的 server 那份配置里 `supports_parallel_tool_calls` 固定为 `false`，服务器级豁免用不了，只能逐个工具标 `readOnlyHint`。

### 3.2 由 provider 决定的三类

**这三类跟 feature 开关无关，是 provider 的能力决定的**，所以在具体环境里可能有也可能没有。跑探针看 `all_tools` 是唯一可靠的判断方式。

| 工具 | 作用 | 门槛 |
| --- | --- | --- |
| `web__run` | 联网搜索。`supports_parallel_tool_calls` 为真，是少数几个能让并行派发真正并行的工具 | provider 是 OpenAI 系或显式声明支持独立 web search |
| `skills__list` / `skills__read` | 列举与读取 skill | orchestrator skill provider 可用 |
| `image_gen__imagegen` | 生成图片 | provider 能力位与 feature 同时为真 |

### 3.3 PoA 拿不到的

| 工具 | 挡它的机制 | 能否恢复 |
| --- | --- | --- |
| `request_user_input` | 曝光度 `DirectModelOnly`。只给模型，不给程序——这就是"PoA 程序问不了人"的技术根源 | ❌ 无开关。子 agent 也问不了，它拒绝非根线程的调用 |
| `new_context` | 曝光度 `DirectModelOnly` | ❌ |
| `clock` 的 `sleep` | 曝光度 `DirectModelOnly` | ❌ 要等就用 `setTimeout`，或 `exec_command` 跑 `sleep` |
| `collaboration__*`（默认配置下） | 曝光度 `DirectModelOnly`，但由配置驱动：`non_code_mode_only` 为真才降级 | ✅ `[features.multi_agent_v2] non_code_mode_only = false`，见《02-quickstart.md》§2 |
| `tool_search` | 按 spec 种类整体丢弃，与曝光度无关 | ❌ 改任何配置都没用 |
| hosted 版联网搜索 | 同上，同一处判定。code mode 会话里根本不会下发，取而代之的是 `web__run` | ❌ 同上 |

### 3.4 用之前先探一次返回形状

下表的计数是一次快照，与 §4 的声明同源；工具面变了它不会自动更新，以 §4 的声明和探针输出为准。

| 返回类型 | 个数 | 是哪些 |
| --- | ---: | --- |
| 结构化（`Promise<{...}>`，字段带注释） | 18 | `exec_command`、`write_stdin`、`view_image`、`clock__curr_time`、`get_context_remaining`、memories 全部 4 个、multi_agent_v1 全部 5 个、collaboration 的 `spawn_agent` / `wait_agent` / `interrupt_agent` / `list_agents` |
| `Promise<unknown>` | 12 | `apply_patch`、`shell_command`、`update_plan`、`create_goal`、`get_goal`、`update_goal`、`request_permissions`、3 个 mcp resource 工具、collaboration 的 `send_message` / `followup_task` |
| `CallToolResult`（MCP 探针示例，不计入上述 30 个内置工具） | 2 | `mcp__probe__echo`、`mcp__probe__no_output_schema` |

这些 TypeScript 声明只是运行时文档，不提供 V8 静态类型检查。调用失败时 Promise 可能直接 reject；成功后，结构化返回仍要防御可选字段和动态 map 缺项。对 `Promise<unknown>`，先用 `text(JSON.stringify(result))` 探测实际形状，再写解析。

MCP per-tool declaration 会引用共享类型块。CodeModeOnly 的 `exec` 说明会给出一次，ordinary CodeMode 的 `exec` 说明不会；这里恢复一份紧凑的离线定义，供两种模式查阅，也让下面两个探针声明自洽：

```ts
type Role = "user" | "assistant";
type MetaObject = Record<string, unknown>;
type Annotations = {
  audience?: Role[];
  priority?: number;
  lastModified?: string;
};
type Icon = {
  src: string;
  mimeType?: string;
  sizes?: string[];
  theme?: "light" | "dark";
};
type TextResourceContents = {
  uri: string;
  mimeType?: string;
  _meta?: MetaObject;
  text: string;
};
type BlobResourceContents = {
  uri: string;
  mimeType?: string;
  _meta?: MetaObject;
  blob: string;
};
type ContentBlock =
  | { type: "text"; text: string; annotations?: Annotations; _meta?: MetaObject }
  | { type: "image"; data: string; mimeType: string; annotations?: Annotations; _meta?: MetaObject }
  | { type: "audio"; data: string; mimeType: string; annotations?: Annotations; _meta?: MetaObject }
  | {
      type: "resource_link";
      name: string;
      uri: string;
      title?: string;
      description?: string;
      mimeType?: string;
      icons?: Icon[];
      annotations?: Annotations;
      size?: number;
      _meta?: MetaObject;
    }
  | {
      type: "resource";
      resource: TextResourceContents | BlobResourceContents;
      annotations?: Annotations;
      _meta?: MetaObject;
    };
type CallToolResult<TStructured = Record<string, unknown>> = {
  _meta?: MetaObject;
  content: ContentBlock[];
  isError?: boolean;
  structuredContent?: TStructured;
  [key: string]: unknown;
};
```

---

## 4. 查一个工具收什么参数

### 执行与文件

#### `exec_command`

> 在 PTY 中运行一条命令，返回输出，或返回一个可用于后续交互的会话 ID。

```ts
declare const tools: { exec_command(args: {
  // 要执行的 shell 命令。
  cmd: string;
  // 仅连接了多个 environment 的构建会出现；省略则使用主 environment。
  environment_id?: string;
  // 仅启用 `features.exec_permission_approvals` 时出现；须配合 `with_additional_permissions`。
  additional_permissions?: {
    file_system?: {
      read?: Array<string>;
      write?: Array<string>;
    };
    network?: {
      enabled?: boolean;
    };
  };
  // 用于 `require_escalated` 的、面向用户的审批问题；其余情况省略。
  justification?: string;
  // 仅 `allow_login_shell` 开启时出现。true 让 shell 以 -l/-i 语义运行，false 则关闭。默认 true。
  login?: boolean;
  // 输出 token 预算。默认 10000 tokens；更大的请求可能被策略封顶。
  max_output_tokens?: number;
  // 针对 `cmd` 的可复用审批前缀，仅在 `sandbox_permissions: "require_escalated"` 时有效；例如 ["git", "pull"]。
  prefix_rule?: Array<string>;
  // 单条命令级别的沙箱覆盖。`with_additional_permissions` 仅在对应 feature 开启时出现。
  sandbox_permissions?: "use_default" | "with_additional_permissions" | "require_escalated";
  // 仅 `include_shell_parameter` 开启时出现。要启动的 shell 可执行文件；默认为用户的默认 shell。
  shell?: string;
  // true 为该命令分配一个 PTY；false 或省略则使用普通管道。
  tty?: boolean;
  // 命令的工作目录。默认为本轮的 cwd。
  workdir?: string;
  // 返回输出前的等待时长。默认 10000 ms；非 Windows 有效范围 250-30000 ms，Windows 为 10000-30000 ms。
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

返回的是对象，不是字符串，要拿命令的 stdout 得取 `.output`。另外注意它自己也有一个 `yield_time_ms`（默认 10 秒）：到点时进程仍在运行的，返回的是被截断的中途输出加一个 `session_id`，跑得久的命令要显式调大。

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
  // 返回输出前的等待时长，两种写入用两套规则，且越界一律静默钳制、不报错：
  //   非空写入：默认 250 ms，上限 30000 ms
  //   空轮询（省略 chars）：钳进 [5000, 300000]，上界可由宿主配置调整。
  //     默认值 250 会被抬到 5000，所以空轮询没有"快速返回"这个选项
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
  // 进程仍在运行时，用于传给后续 write_stdin 的会话标识。
  session_id?: number;
  // 等待输出所耗的墙钟时间，单位为秒。
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
  // 仅启用 `features.exec_permission_approvals` 时出现；须配合 `with_additional_permissions`。
  additional_permissions?: {
    file_system?: {
      read?: Array<string>;
      write?: Array<string>;
    };
    network?: {
      enabled?: boolean;
    };
  };
  // 用于 `require_escalated` 的、面向用户的审批问题；其余情况省略。
  justification?: string;
  // 仅 `allow_login_shell` 开启时出现。true 以 login shell 语义运行，false 则关闭。默认 true。
  login?: boolean;
  // 针对 `cmd` 的可复用审批前缀，仅在 `sandbox_permissions: "require_escalated"` 时有效。
  prefix_rule?: Array<string>;
  // 单条命令级别的沙箱覆盖。`with_additional_permissions` 仅在对应 feature 开启时出现。
  sandbox_permissions?: "use_default" | "with_additional_permissions" | "require_escalated";
  // 命令的最长运行时间。默认 10000 ms。
  timeout_ms?: number;
  // 命令的工作目录。默认为本轮的 cwd。
  workdir?: string;
}): Promise<unknown>; };
```

`timeout_ms` 与 `exec_command` 的 `yield_time_ms` 长得像，语义相反。两者都默认 10000 ms，但 `yield_time_ms` 到点只是让出已有输出、进程继续跑（所以才回一个 `session_id`）；`timeout_ms` 到点是杀进程，退出码 124。同一条 `sleep 30`，前者返回一个可续接的句柄，后者返回一次被杀的失败。

#### `apply_patch`

> `apply_patch` 工具可用于编辑文件。这是一个 FREEFORM 工具，因此不要把补丁包进 JSON 里。

```ts
declare const tools: { apply_patch(input: string): Promise<unknown>; };
```

**入参是裸字符串，不是对象**，跟这里其余所有工具的调用形式都不一样。

#### `view_image`

> 需要目视检查时，查看文件系统上的一个本地图片文件。用于磁盘上已经存在的图片。

```ts
declare const tools: { view_image(args: {
  // 仅模型声明支持请求原图细节时出现。默认 `high`；需要保留精确分辨率时用 `original`。
  detail?: "high" | "original";
  // 仅多 environment 构建出现；省略则使用 primary environment。
  environment_id?: string;
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

使用 §2.5 那一层时通常不用直接碰这一组。列在这里有两个用处：读实现时对上号，以及需要它没封装的能力（如 `resume_agent`、`interrupt_agent`）时知道去调什么。纯内置版本见《05-writing.md》§12.2。

#### `multi_agent_v1__spawn_agent`

> 为一个范围明确的任务派生一个子代理。返回被派生代理的 id，以及面向用户的昵称（可用时）。
> 被派生的代理默认继承你当前的模型。省略 `model` 即使用这个首选默认值。

```ts
declare const tools: { multi_agent_v1__spawn_agent(args: {
  // 仅宿主构建/角色配置暴露该覆盖项时出现。
  agent_type?: string;
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
  // 仅 `hide_spawn_agent_metadata = false` 时出现。新代理的服务层级覆盖项。
  service_tier?: string;
}): Promise<{
  // 被派生代理的 thread 标识。
  agent_id: string;
  // 被派生代理的、面向用户的昵称（可用时）。
  nickname: string | null;
}>; };
```

> **没有 `system_prompt` 这一项。** 子 agent 的系统提示词设不了，`message` 是程序唯一的控制面。

#### `multi_agent_v1__wait_agent`

> 等待代理进入终态。completed 状态里可能带上代理的最终消息。超时时返回空的 status。
> 代理到达终态时，还会收到一条包含同一 completed 状态的通知消息。

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

**一次等多个是陷阱**：它会反复返回最先完成的那个。所以收 N 个结果需要 N 次单目标等待，这正是 `collectAll` 在 v1 分支写成串行 for 循环的原因。

schema 把 `status` 定义为按 agent id 索引。当前 v1 没有 `agent_path`，因此实际返回的 key 恰好等于 `agent_id`；这是当前实现的巧合，不应当作跨版本契约。

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
> 被派生的代理会拥有和你一样的工具，也能派生它自己的子代理。
> 传 `fork_turns="none"` 不会把任何周边上下文传给子代理；`fork_turns="all"` 会全部提供。
> §2.5 的跨后端 helper 固定传 `fork_turns="none"`；需要继承时直接调用这里的原生 v2 API。

```ts
declare const tools: { collaboration__spawn_agent(args: {
  // 仅宿主构建/角色配置暴露该覆盖项时出现；显式覆盖时 fork_turns 不能用 all。
  agent_type?: string;
  // 可选的 fork 轮数。默认 `all`。可用 `none`、`all`，或像 `3` 这样的正整数字符串。
  fork_turns?: string;
  // 给新代理的初始纯文本任务。
  message: string;
  // 仅 `expose_spawn_agent_model_overrides` 开启时出现。除非确需显式覆盖，否则省略。
  model?: string;
  // 仅 `expose_spawn_agent_model_overrides` 开启时出现。省略则继承父级的推理强度。
  reasoning_effort?: string;
  // 仅 `hide_spawn_agent_metadata = false` 时出现。
  service_tier?: string;
  // 新代理的任务名。只使用小写字母、数字和下划线。
  task_name: string;
}): Promise<{
  // 被派生代理的规范任务名。
  task_name: string;
  // 仅 `hide_spawn_agent_metadata = false` 时返回；隐藏 metadata 时返回值只有 task_name。
  nickname?: string | null;
}>; };
```

**返回的 `task_name` 已经是全限定路径**（形如 `/root/scan_0`）。再拼一次前缀去和 `list_agents` 比对，结果是全部 agent 完成、一个都收不到，而且不报任何错，只是超时后返回一堆 `null`。这也是 `collectAll` 的 v2 分支用 `endsWith` 匹配的原因。

`task_name` 只允许小写字母、数字和下划线，而且同一个父任务下的 sibling 名必须唯一。§2.5 的 prelude 首次使用清洗后的 `${base}_${seq}` 稳定名；只有宿主返回 `already exists` 后，回退名才追加当前 cell 的小写时间 nonce、后续序号与 attempt。nonce 不承诺绝对随机；总共尝试 3 次（最多重试 2 次），其他错误立即抛出。

另外注意这里的参数结构会拒绝未知字段：自作主张塞一个不存在的参数会直接抛异常。

#### `collaboration__wait_agent`

> 等待任意存活代理的信箱更新。它不返回内容本身；返回的是哪些代理有更新的摘要。

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

> 给一个已存在的代理发送消息。消息会被及时投递。不会触发新的一轮。

```ts
declare const tools: { collaboration__send_message(args: {
  // 要在目标代理上排队的消息文本。
  message: string;
  // 要发消息的相对任务名或规范任务名（来自 spawn_agent）。
  target: string;
}): Promise<unknown>; };
```

#### `collaboration__followup_task`

> 给一个已存在的非 root 目标代理下发后续任务，若它处于空闲状态则触发一轮。

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

> 打断某个代理当前的轮次（如果有），并返回它之前的状态。该代理仍可继续接收消息与后续任务。

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
  // 上一次调用返回的不透明游标；取第一页时省略。
  cursor?: string;
  // MCP server 名称。省略则列出所有已配置 server 的资源模板。
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

> 示例工具：按给定的重复次数回显一条消息。展示声明了 `outputSchema` 时的渲染结果。

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

> 示例工具：没有声明 `outputSchema`，返回的是裸 `CallToolResult`。

```ts
declare const tools: { mcp__probe__no_output_schema(args: { q: string; }): Promise<CallToolResult>; };
```
