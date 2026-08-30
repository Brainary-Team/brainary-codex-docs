# 快速开始

本篇覆盖从拿到二进制到跑通第一个自建程序的完整动线。

## 前置条件

| 事项 | 当前状态 |
| --- | --- |
| 从哪个前端提交 PoA | `codex exec --poa` 与 `codex app-server` 的 `thread/codeMode/exec` 两条，均已可用。TUI、SDK 与 `codex mcp-server` 暂不支持 |
| 二进制从哪来 | 本项目发布的 release 制品。上游安装版的 `codex` 没有 `--poa`，也不提供那个 RPC 方法 |
| 程序以什么形态提交 | 一个包目录（`manifest.toml` + `main.js`）。`codex exec --poa <目录>` 每次现打包现跑，没有"先 build 再跑"这一步 |
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

> macOS 与 arm64 当前没有制品，只能从源码构建。`codex` 常规 `cargo build` 即可，但 `codex-code-mode-host` 要求一份开启 `v8_enable_sandbox` 特性的预编译 V8，crates.io 上的产物不带这个特性，需要另行准备。

---

## 1. 配置 `CODEX_HOME/config.toml`

`CODEX_HOME` 默认是 `~/.codex`。整份配置如下，逐项说明在下面：

```toml
model = "gpt-5.6-luna"
model_provider = "poa"
approval_policy = "never"
sandbox_mode = "read-only"
suppress_unstable_features_warning = true

[model_providers.poa]
name = "poa"
base_url = "https://your-provider/v1"
env_key = "BRAINARY_API_KEY"
wire_api = "responses"

[features.code_mode]
enabled = true
```

设置环境变量：

```bash
export BRAINARY_API_KEY=sk-...
```

| 项 | 为什么需要 |
| --- | --- |
| `[features.code_mode] enabled = true` | 这个 feature 默认关闭，`--poa` 不会自动打开它。未打开时 `exec` 工具不在工具面上，cell 起不来 |
| `model` | 必须是 codex 自带模型清单里的 slug，否则解析不出模型。它同时决定用哪代 agent 后端，见 §2 |
| `model_provider` | 指向下面那张表的名字，两处必须一致（这里是 `poa`）。表里的 `name` 只是显示名 |
| `base_url` | provider 的 API 根地址。codex 在它后面拼路径，所以要带到 `/v1` 那一层 |
| `wire_api` | 使用哪种请求协议。只有两个值：`responses`（默认，`/v1/responses`）和 `anthropic`（`/v1/messages`）。其余值在解析配置时就报错，不会退回默认值 |
| `env_key` | 只是环境变量的**名字**，取什么名都行，key 本身写在环境变量里，不写进配置文件 |
| `suppress_unstable_features_warning` | 抑制 `code_mode` 每次运行时的警告 |
| `sandbox_mode` | 默认就是 `read-only`。程序要写文件时改成 `workspace-write` |
| `approval_policy` | `codex exec` 无论如何都会把它钉成 `never`，这里只是为了让 `app-server` 那条路一致 |

用别家 provider 时，`model` 仍然沿用清单里的 slug，真正指向哪个模型由 `base_url` 决定。

### 换用 Anthropic 协议

把 `wire_api` 换成 `anthropic`，就改打 `/v1/messages`：

```toml
[model_providers.poa]
name = "poa"
base_url = "https://api.moonshot.cn/anthropic/v1"
env_key = "BRAINARY_API_KEY"
wire_api = "anthropic"
```

接入 Moonshot、DeepSeek 等不提供 `/v1/responses` 的 provider 时用这一种。

| Provider | `base_url` | 模型 |
|---|---|---|
| Anthropic | `https://api.anthropic.com/v1` | `claude-opus-5` |
| Moonshot | `https://api.moonshot.cn/anthropic/v1` | `kimi-k3` |
| DeepSeek | `https://api.deepseek.com/anthropic/v1` | `deepseek-v4-pro` |

---

## 2. 第一步：跑探针

这永远是第一条命令。它不派 agent、不调模型，不产生模型费用。

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
text(JSON.stringify({
  backend: names.includes("multi_agent_v1__spawn_agent") ? "v1"
         : names.includes("collaboration__spawn_agent") ? "v2" : null,
  tool_count: names.length,
  all_tools: names,
}, null, 2));
```

跑：

```bash
~/codex-poa/codex exec --poa ~/poa-probe --skip-git-repo-check
```

看输出里的两个字段：

| 字段 | 怎么读 |
| --- | --- |
| `backend` | `"v1"` 或 `"v2"` → 可以往下走。`null` 则表示环境没配好 |
| `all_tools` | 当前这套配置下实际能调的工具全集。写程序时用到的每个工具都该在里面 |

### `--skip-git-repo-check`

`codex exec` 默认拒绝在 git 仓库之外运行，直接 `exit(1)` 并打印 `Not inside a trusted directory`。使用这个参数可以绕过检查。

### `backend` 是 `null` 怎么办

后端由模型元数据决定，程序改不了，也没有任何 feature 开关能强制回到 v1。自带清单里只有 `gpt-5.6-sol` 和 `gpt-5.6-terra` 两个模型标了 v2，其余落到 v1。

| 现象 | 原因 | 处置 |
| --- | --- | --- |
| `backend: null`，模型是 sol / terra | v2 那组工具默认只给模型直接调，进不了 code mode | 打开下面那个 v2 开关，或换成 v1 的模型 |
| `backend: null`，模型是 v1 的 | code mode 没打开，或模型名没解析出来 | 检查 `[features.code_mode] enabled = true` 与 `model` 拼写 |

打开 v2：

```toml
[features.multi_agent_v2]
enabled = true
non_code_mode_only = false          # 默认为 true，此时整组工具 PoA 完全拿不到
max_concurrent_threads_per_session = 8   # 默认 4，而 root 线程占一个，所以默认只剩 3 个可用
```

注意 `[features.multi_agent_v2] enabled = true` 是一个**强制到 v2** 的开关，它压过模型元数据：打开之后 v1 的模型也会走 v2。两代的名额语义不同，会影响程序结构，见《05-writing.md》§6.3。

换模型、换 provider、改配置之后都要重跑探针。

---

## 3. 第二步：接第一个 agent

把 `main.js` 换成下面这段，其余不动。这里用的是**只有内置 primitive** 的写法，包里能直接跑：

```js
// @exec: {"yield_time_ms": 900000, "max_output_tokens": 30000}

const NAMES = new Set(ALL_TOOLS.map((t) => t.name));
const BACKEND = NAMES.has("multi_agent_v1__spawn_agent") ? "v1"
  : NAMES.has("collaboration__spawn_agent") ? "v2" : null;

// ① 无后端时立即失败，并把当前可用的工具名列出来
if (!BACKEND) throw new Error("no agent tools: " + [...NAMES].sort().join(", "));

const message = '用一句话说明当前目录是做什么的。只回一行 JSON：{"purpose":"..."}';

let reply = null;
if (BACKEND === "v1") {
  const { agent_id } = await tools.multi_agent_v1__spawn_agent({ message });
  const out = await tools.multi_agent_v1__wait_agent({ targets: [agent_id], timeout_ms: 300000 });
  reply = ((out.status || {})[agent_id] || {}).completed ?? null;
} else {
  // 返回的 task_name 已经是全限定路径（形如 /root/probe_1），不要再拼一次前缀
  const { task_name } = await tools.collaboration__spawn_agent({ task_name: "probe_1", message });
  const deadline = Date.now() + 300000;
  let done = false;
  while (!done && Date.now() < deadline) {
    const listed = await tools.collaboration__list_agents({});
    for (const e of listed.agents || []) {
      if (e.agent_name !== task_name && !e.agent_name.endsWith(`/${task_name}`)) continue;
      const st = e.agent_status;
      if (st && typeof st === "object" && ("completed" in st || "errored" in st)) {
        reply = st.completed ?? null;   // errored 时是 null，下面按失败处理
        done = true;
      }
    }
    if (done) break;
    // v2 的 wait 只是"有动静了"的信号，内容仍要去 list_agents 里捞
    await tools.collaboration__wait_agent({ timeout_ms: 15000 });
  }
}

// ② 收口必须防御性解析，③ 失败时把原文留下来
const m = typeof reply === "string" ? reply.match(/\{[\s\S]*\}/) : null;
let value = null, error = m ? null : "no JSON object in reply";
if (m) { try { value = JSON.parse(m[0]); } catch (err) { error = String(err); } }

text(JSON.stringify({
  backend: BACKEND,
  value,
  error,
  raw: value ? undefined : String(reply ?? "").slice(0, 200),
}, null, 2));
```

```bash
~/codex-poa/codex exec --poa ~/poa-probe -C /path/to/some/repo
```

---

## 4. 打开可选的工具组

默认配置下这几组工具不在工具面上。需要在 `config.toml` 中开启：

```toml
# 布尔形式的 feature，必须和下面的表形式分开写
[features]
memories = true                    # 记忆工具，还需要下面的 [memories]
token_budget = true                # get_context_remaining
request_permissions_tool = true    # request_permissions，且需存在一个 environment

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

