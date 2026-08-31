# 快速开始

本篇覆盖从拿到二进制到跑通第一个 PoA 程序的完整路线。

## 前置条件

| 事项 | 当前状态 |
| --- | --- |
| 从哪个前端提交 PoA | `codex exec --poa` 与 `codex app-server` 的 `thread/codeMode/exec` 两条，均已可用。TUI、SDK 与 `codex mcp-server` 暂不支持 |
| 二进制从哪来 | 本项目发布的 release 制品。 |
| 程序以什么形态提交 | 一个包目录（`manifest.toml` + `main.js`）。`codex exec --poa <目录>`  |
| 用哪代多 agent 后端 | 由模型元数据决定，另有一个 feature 可以强制到 v2。跑探针查，见 §2 |

---

## 0. 取得两个二进制

运行 PoA 需要两个二进制，且必须位于同一目录：

| 二进制 | 作用 |
| --- | --- |
| `codex` | 本项目的 CLI 与 app-server，提供 `--poa` 与 `thread/codeMode/exec` |
| `codex-code-mode-host` | 承载 V8 的独立沙箱进程，由 `codex` 拉起。`codex` 按自己所在目录去找它 |

两者一起打在 `codex-poa-<平台>.tar.gz` 里，发布在本项目的 GitHub release 页：

<https://github.com/Brainary-Team/brainary-codex/releases>

当前只发布 `x86_64-unknown-linux-musl` 一个平台，版本都是 `dev-<分支名>-<时间戳>` 形式的预发布，尚无正式 release。

两条硬约束：

- **两个文件必须放在同一目录。**
- **机器上装过 `codex` 时，注意不要调到那一个。** 本篇所有命令都写完整路径。

> macOS 与 arm64 当前没有制品。

---

## 1. 配置 `CODEX_HOME/config.toml`

`CODEX_HOME` 默认是 `~/.codex`。最小推荐配置：

```toml
model = "gpt-5.6-luna"
model_provider = "poa"
approval_policy = "never"
sandbox_mode = "read-only"

[model_providers.poa]
name = "poa"
base_url = "https://your-provider/v1"
env_key = "BRAINARY_API_KEY"
wire_api = "responses"
```

设置环境变量：

```bash
export BRAINARY_API_KEY=sk-...
```

| 项 | 为什么需要 |
| --- | --- |
| `[features.code_mode] enabled = true` | 上面的 luna 配置不用写；使用 fallback metadata 时必须显式打开 |
| `model` | 填 provider 接受的模型 ID；只有网关接受或映射内置 slug 时才能填 luna |
| `model_provider` | 指向下面那张表的名字，两处必须一致（这里是 `poa`）。表里的 `name` 只是显示名 |
| `base_url` | endpoint 名的父 URL；不硬性要求以 `/v1` 结尾 |
| `wire_api` | `responses`（默认）或 `anthropic`；其余值在解析配置时直接报错 |
| `env_key` | 只是环境变量的**名字**，取什么名都行，key 本身写在环境变量里，不写进配置文件 |
| `suppress_unstable_features_warning` | 不属于最小配置。只有显式启用了会告警的不稳定 feature 时，才按需设为 `true` |
| `sandbox_mode` | 默认就是 `read-only`。程序要写文件时改成 `workspace-write` |
| `approval_policy` | 写 `never`，让 app-server 路径与通常的 headless exec 路径一致 |

### 选择 Anthropic 请求协议

把 `wire_api` 设为 `anthropic`。

```toml
model = "kimi-k3"
model_provider = "poa"
approval_policy = "never"
sandbox_mode = "read-only"

[features.code_mode]
enabled = true

[model_providers.poa]
name = "poa"
base_url = "https://api.moonshot.cn/anthropic/v1"
env_key = "BRAINARY_API_KEY"
wire_api = "anthropic"
```

| Provider | `base_url` | 模型 |
|---|---|---|
| Anthropic | `https://api.anthropic.com/v1` | `claude-opus-5` |
| Moonshot | `https://api.moonshot.cn/anthropic/v1` | `kimi-k3` |

---

## 2. 第一步：跑探针

建一个目录，两个文件：

```
~/poa-probe/
├── manifest.toml
└── main.js
```

`manifest.toml`：

```toml
[poa]
name = "probe"
version = "0.1.0"
runtime = "codex-v8"
poa_api_version = 1
entry = "main.js"
```

`main.js`：

```js
// @exec: {"yield_time_ms": 60000, "max_output_tokens": 10000}
const names = ALL_TOOLS.map((t) => t.name).sort();
const v2Spawn = ALL_TOOLS.find((t) =>
  t.name.endsWith("spawn_agent") &&
  String(t.description).includes("task_name") &&
  String(t.description).includes("fork_turns"));
const v2Prefix = v2Spawn ? v2Spawn.name.slice(0, -"spawn_agent".length) : null;
const v2Tools = v2Prefix === null ? null : Object.fromEntries(
  ["spawn_agent", "wait_agent", "list_agents", "followup_task"]
    .map((key) => [key, `${v2Prefix}${key}`]),
);
const hasV2 = v2Tools && Object.values(v2Tools).every((name) => names.includes(name));
const agentTools = names.filter((name) =>
  name.startsWith("multi_agent_v1__") || (v2Prefix !== null && name.startsWith(v2Prefix)));
text(JSON.stringify({
  backend: names.includes("multi_agent_v1__spawn_agent") ? "v1"
         : hasV2 ? "v2" : null,
  tool_count: names.length,
  agent_tools: agentTools.length,
  all_tools: names,
}, null, 2));
```

跑：

```bash
~/codex-poa/codex exec --poa ~/poa-probe --skip-git-repo-check
```

输出是 JSON。数量和工具全集取决于当前配置；一次 v1 环境的关键字段摘录如下：

```json
{
  "backend": "v1",
  "agent_tools": 5,
  "all_tools": ["multi_agent_v1__spawn_agent", "..."]
}
```

| 字段 | 含义 |
| --- | --- |
| `backend` | `"v1"` 或 `"v2"` → 可以往下走。`null` 则表示环境没配好 |
| `tool_count` | 当前工具总数，只用于比较配置变化，不应与示例中的某个固定数字对齐 |
| `agent_tools` | 当前发现的 agent 工具数；v1 通常为 5，v2 取决于当前 namespace 实际暴露的工具 |
| `all_tools` | 当前这套配置下实际能调的工具全集。写程序时用到的每个工具都该在里面 |

### `--skip-git-repo-check`

`codex exec` 默认拒绝在 git 仓库之外运行，直接 `exit(1)` 并打印 `Not inside a trusted directory`。使用这个参数可以绕过检查。

### `backend` 是 `null` 怎么办

后端默认由模型元数据决定；不在内置清单里的真实模型 ID 使用 fallback metadata，默认走 v1。本地 `[features.multi_agent_v2]` 可以强制 v2，但不能强制回 v1；程序 cell 启动后也不能改变后端。

| 现象 | 原因 | 处置 |
| --- | --- | --- |
| `backend: null`，模型是 sol / terra | 模型已经由 `code_mode_only` 进入 code mode，但 v2 那组 agent 工具默认只给模型直接调，PoA 看不见 | 打开下面那个 v2 开关，或换成 luna 等 v1 模型 |
| `backend: null`，模型是 luna | 推荐模型会自动进入 code mode；此时更可能是模型名没解析出来或工具注册异常 | 检查 `model` 拼写，再看探针的完整 `all_tools` |
| `backend: null`，使用原生 provider 的真实模型 ID | 未命中内置清单时会走 fallback metadata，不会靠 `code_mode_only` 自动进入 code mode | 确认 `model` 是 provider 接受的真实 ID，并加 `[features.code_mode] enabled = true` |

打开 v2：

```toml
[features.multi_agent_v2]
enabled = true
non_code_mode_only = false          # 默认为 true，此时整组工具 PoA 完全拿不到
max_concurrent_threads_per_session = 8   # 默认 4，而 root 线程占一个，所以默认只剩 3 个可用
```

注意 `[features.multi_agent_v2] enabled = true` 是一个**强制到 v2** 的开关，它压过模型元数据：打开之后 v1 的模型也会走 v2。对应的程序结构见《05-writing.md》§6.3。

换模型、换 provider、改配置之后都要重跑探针。

---

## 3. 第二步：接第一个 agent

先按《03-concepts.md》§2 组装 `main.js`：第 1 行写 pragma，其后放
[《07 API 参考》§2.4](07-api-reference.md#24-prelude-全文)，最后追加下面这段。

```js
const message = '用一句话说明当前目录是做什么的。只回一行 JSON：{"purpose":"..."}';
const [{ handle, reply }] = await runBatch([{ message, name: "probe", meta: {} }], {
  concurrency: 1, timeoutMs: 300000,
});
const { value, error } = parseJsonReply(reply);
await closeAll([handle]);
text(JSON.stringify({ backend: AGENT_BACKEND, succeeded: value ? 1 : 0, total: 1,
  value, error, raw: value ? undefined : String(reply ?? "").slice(0, 200) }, null, 2));
```

```bash
~/codex-poa/codex exec --poa ~/poa-probe -C /path/to/some/repo
```

成功时输出应包含 `succeeded: 1`、`total: 1` 和非空 `value.purpose`。

```json
{
  "backend": "v1",
  "succeeded": 1,
  "total": 1,
  "value": { "purpose": "..." },
  "error": null
}
```

`runBatch` 会自动适配当前可用的 v1 或 v2 多 agent 后端；底层差异见[《03 核心概念》§2](03-concepts.md#2-分清-prelude-和内置)。

---

## 4. 打开可选的工具组

默认配置下这几组工具不在工具面上。需要在 `config.toml` 中开启：

```toml
# 布尔形式的 feature，必须和下面的表形式分开写
[features]
memories = true                    # 记忆工具，还需要下面的 [memories]
token_budget = true                # get_context_remaining
request_permissions_tool = true    # request_permissions，且需存在一个 environment；approval_policy = "never" 时立即返回空权限

# 表形式的 feature
[features.current_time_reminder]
enabled = true                     # clock__curr_time

# 记忆工具的另一半，少这一段是零个工具而不是一半
[memories]
dedicated_tools = true
```

---

## 5. 看原始返回值：`--json`

`text()` 写进去的内容最终落在 RPC 返回值里，`codex exec` 把里面的文本取出来打印。如果要查看未经处理的返回值加 `--json`：

```bash
~/codex-poa/codex exec --poa ~/poa-probe --json
```
