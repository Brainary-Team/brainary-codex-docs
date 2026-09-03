# 打包与分发

本篇讲把一段能跑的 JS 打包成一个可分发的 `.poa` 包。

---

## 1. 代码是怎么进到沙箱里的

排障的第一步是判断失败落在哪一侧，所以先看这条链。

```mermaid
graph LR
    YOU["开发者"] --> EXEC["codex exec --poa<br/>目录现打包 + 本地校验 manifest"]
    EXEC --> RPC["{ threadId, package }"]
    RPC --> CODEX["app-server<br/>解包 → 起包内 MCP → 等就位"]
    CODEX --> CELL["V8 里的一个 cell"]
    CELL --> SUB["子 agent × N"]

    style EXEC fill:#d5e8d4,stroke:#82b366
    style CELL fill:#dae8fc,stroke:#6c8ebf,stroke-width:2px
    style SUB fill:#fff2cc,stroke:#d6b656
```

**①** `codex exec --poa <目录>` 每次现打包，`entry` 是运行时被读成字符串送进去的。

**②** 协议层只有一种形状：`{ threadId, package }`。自研客户端走 `codex app-server` 的 `thread/codeMode/exec`，发的是同一个东西，只是包由它自己造、base64 之后过 JSON-RPC。只持有一段 JS 时也要现场封装成一个申报零能力的包，接口没有另一个只收源码的入口。

**③** `entry` 原样提交，前面不会拼入任何内容，所以首行 pragma 在第 1 行。

**④** 报错落在哪一侧决定去哪查。坏包在本地就被拒，`codex` 自己报错并 `exit(2)`；V8 侧的失败在返回的内容里。

---

## 2. 程序必须能独自跑完

当前 PoA 机制上没有任何人工介入点，因此书写的 PoA 程序必须能独立运行。

---

## 3. 写一个包

程序可以像一个安装包那样，把自己要用的 MCP server 带在身上。

能力申报写在包的 `manifest.toml` 里，服务端解包后按 thread 把它们起起来，不需要事先在宿主的 `config.toml` 里配好。

### 3.1 目录结构

```
my-poa/
├── manifest.toml       [poa] 元数据 + [[capabilities.mcp]] 能力申报
├── main.js             程序本体，原样提交
└── mcp/echo.mjs        随包分发的 stdio MCP server，零依赖
```

`.poa` 就是这个目录打成的 zip，`manifest.toml` 必须在根上。

### 3.2 manifest 字段

```toml
[poa]
name = "echo_expert"              # 必填
version = "0.1.0"                 # 必填，格式不校验
runtime = "codex-v8"              # 必填，只认这一个值，其余值直接反序列化失败
poa_api_version = 1               # 必填，只认 1
entry = "main.js"                 # 必填，必须 .js/.mjs，必须在包内
description = "..."               # 可选

# 整段可以没有（不自带能力时），也可以重复多次（带多个 server）
[[capabilities.mcp]]
name = "echo"                     # 必填，只能用字母数字和 _ -，且不能与同包内其他 server 重名
type = "stdio"                    # 必填，只有这一种；写 network 会在解析阶段报错
shell = "node mcp/echo.mjs"       # 必填，按 shell 语法切成 argv，直接派生，不经 sh -c
description = "..."               # 可选

[[capabilities.mcp]]
name = "lookup"
type = "stdio"
shell = "node mcp/lookup.mjs"
```

### 3.3 提交之后发生了什么

两条路解包的地方不一样，排障时要分清：

```mermaid
graph TD
    subgraph P1["① codex exec --poa（目录或 .poa）"]
      A1["目录现打包 / .poa 直接读"] --> A2["校验 manifest + 解包到临时目录"]
      A2 --> A3["进程内把 PoaPackage 交给 app-server<br/>同进程，不过 JSON"]
    end

    subgraph P2["② 自研客户端"]
      B1["自己造包<br/>现成 JS 就包成零能力包"] --> B2["base64 → JSON-RPC<br/>{ threadId, package }"]
      B2 --> B3["codex app-server<br/>解 base64、校验、解包"]
    end

    A2 -.->|"坏包在这里就是 exit(2)"| BAD(["一个会话都不会起"])

    A3 --> M(["两路在此汇合"])
    B3 --> M

    M --> S1["每条 [[capabilities.mcp]] 注册成<br/>thread 级 MCP server（cwd = 包根）"]
    S1 -->|"顺序不能反，见下方 ③"| S2["等这些 server 全部就位<br/>再捕获工具面"]
    S2 --> S3{"每个申报的 server<br/>都在工具面上留下工具了吗"}
    S3 -->|"否"| REFUSE(["拒跑"])
    S3 -->|"是"| S4["把 entry 的内容当源码送进 V8"]

    style A3 fill:#d5e8d4,stroke:#82b366
    style B3 fill:#dae8fc,stroke:#6c8ebf
    style M fill:#f5f5f5,stroke:#666666
    style S4 fill:#dae8fc,stroke:#6c8ebf,stroke-width:2px
    style BAD fill:#f8cecc,stroke:#b85450
    style REFUSE fill:#f8cecc,stroke:#b85450
```

**①** "组装在服务端"指的是起 server，不是解包。`codex exec --poa` 走的是进程内调用，包在客户端就打好、校验完、解压好了，`PoaPackage` 作为一个 Rust 值直接传进去，全程没有 base64、没有 JSON 序列化。base64 只发生在真正跨进程的 stdio JSON-RPC 上。

真正在服务端的是起 MCP server 这一段，它挂在 thread 级的扩展点上，所以任何前端只要能把包递进去就能复用。

**②** server 随 thread 生死，不写全局配置。包起的 server 只对这一个 thread 可见，`config.toml` 一个字都不改，thread 关掉进程就回收。

**③** 顺序不能反。包必须在"捕获工具面"之前发布，因为捕获工具面这一步才是建工具计划的时候。晚一步注册的 server，对这个 cell 来说等于不存在。

**④** 起不来即拒跑。判据是"这个 server 有没有在工具面上留下工具"，所以起来了但一个工具都不报的 server 同样算缺席。这是刻意与采样循环相反的选择：模型少一个工具可以改走别的路径，程序不能。启动超时 30 秒。

---

## 4. 两项必做处理

### 4.1 从 `ALL_TOOLS` 里找工具名，不要硬编码

`mcp__echo__echo` 是当前配置下的产物，前缀由宿主的命名规则决定，命名空间会被清洗，长名或冲突还会触发哈希后缀与 64 字符截断。短名字可以从 `ALL_TOOLS` 里按后缀找并断言唯一来扛前缀变化，但这不是稳定契约，也扛不住上述哈希与截断；完整边界见《07-api-reference.md》§3.1。

### 4.2 给每个 MCP 工具写三条 annotations

标注写在 MCP server `tools/list` 返回里每个工具的 `annotations` 上：

```js
annotations: { readOnlyHint: true, destructiveHint: false, openWorldHint: false }
```

三个都要写，只写 `readOnlyHint` 不够：判定顺序是 `destructiveHint === true` 先短路（直接要审批），才轮到看 `readOnlyHint`。
标注还有一项附带作用：只读同时也是并行安全的判据，所以包自带的只读 server 是可以真并行的（包起的 server 那份配置里 `supports_parallel_tool_calls` 固定为 `false`，服务器级的整体豁免用不了，只能逐个工具标只读）。

一个零依赖、手写 stdio JSON-RPC 的最小 server 是这样的（stdin 逐行读请求，stdout 逐行写响应）：

```js
// mcp/echo.mjs —— 零依赖 stdio MCP server 的最小骨架
const TOOLS = [{
  name: "echo",
  description: "回显传入的文本",
  inputSchema: { type: "object", properties: { text: { type: "string" } }, required: ["text"] },
  annotations: { readOnlyHint: true, destructiveHint: false, openWorldHint: false },
}];

const send = (msg) => process.stdout.write(JSON.stringify(msg) + "\n");

let buf = "";
process.stdin.on("data", (chunk) => {
  buf += chunk;
  let i;
  while ((i = buf.indexOf("\n")) >= 0) {
    const line = buf.slice(0, i).trim();
    buf = buf.slice(i + 1);
    if (!line) continue;
    const req = JSON.parse(line);
    if (req.method === "initialize") {
      send({ jsonrpc: "2.0", id: req.id, result: {
        protocolVersion: "2024-11-05",
        capabilities: { tools: {} },
        serverInfo: { name: "echo", version: "0.1.0" },
      }});
    } else if (req.method === "tools/list") {
      send({ jsonrpc: "2.0", id: req.id, result: { tools: TOOLS } });
    } else if (req.method === "tools/call") {
      const text = String(req.params?.arguments?.text ?? "");
      send({ jsonrpc: "2.0", id: req.id, result: {
        content: [{ type: "text", text }],
        isError: false,
      }});
    } else if (req.id !== undefined) {
      send({ jsonrpc: "2.0", id: req.id, error: { code: -32601, message: "unknown method" } });
    }
    // 没有 id 的是通知（如 notifications/initialized），不回
  }
});
```

要点：`tools/list` 里每个工具都要带上面那三条 annotations；`tools/call` 的返回必须有 `content` 数组；带 `id` 的请求都要回，宿主会一直等到响应到达。`shell` 写成 `node mcp/echo.mjs`，因为解出来的文件权限固定 `0644`，可执行位不保留。

---

## 5. 包能带什么，不能带什么

| 能 | 不能 |
| --- | --- |
| `stdio` MCP server | `network` 传输（解析阶段拒，不是静默忽略） |
| 任意随包文件 | `[build]` 段、resources |
| 根线程的工具面 | 子 agent 的工具面——它跑在自己的 thread 上，拿不到包里的 server |

包的大小上限 64 MiB。

---

## 6. 打成一个文件发给别人

```bash
cd my-poa && zip -r ../my-poa.poa . && cd ..
codex exec --poa ./my-poa.poa          # 打好的包也照跑
```

`.poa` 就是一个普通 zip，唯一的要求是 `manifest.toml` 在根上。

解包时文件权限被无条件覆盖成 `0644`。
