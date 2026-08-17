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

## 4. 七条已知的坑（全量）

这是踩出来的完整清单，每条都花过一个调试周期。

| # | 现象 | 已在哪里处理 | 后果 |
| ---: | --- | --- | --- |
| 1 | v1 一次等多个目标时**反复报告第一个完成者** | `collectAll` 的 v1 分支改用串行单目标等待 | 收不齐结果 |
| 2 | v2 派发返回的 `task_name` **已经是全限定路径** | `collectAll` 的 v2 分支用 `endsWith` 匹配 | **静默零收集**，无报错 |
| 3 | **shell 失败的诊断信息与正常输出同通道** | `shellLines` 的 `validate` 过滤器 | **静默产出错误结果** |
| 4 | `exec` 默认让出时间仅 10 秒 | 首行 `// @exec:` pragma，由 runner 自动提到第 1 行 | 长程序被截断 |
| 5 | v2 并发上限**含 root 线程** | `v2` profile 把它调到 8 | 默认值 4 实际只剩 3 个名额 |
| 6 | 嵌套工具调用默认串行 | 无法规避，只能改设计 | `Promise.race` 不流式 |
| 7 | 模型清单配置项必须写在所有 `[table]` 之前 | `v1-forced` profile 里有行注释标出 | TOML 解析失败 |

**第 2 条和第 3 条是这七条里最危险的**——它们不报错。其余五条都会以可见的方式失败。

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

**③ 把规模压到 1，超时压到 60000。** 让一次失败只花一个 agent 的时间。

**④ 把 `log` 数组交出去。** 在 `catch` / `finally` 里也要 `text()`，否则那次运行的全部信息都没了。

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

**⑤ 用 `requireAgents()` 早点失败。** 它的异常消息里会列出当前全部可用工具名——**这是最快的排错入口**。

---

## 7. 三条不是 bug 的现状

下面这三种情况不必排查，这就是当前的行为：

1. **没有续跑**：程序一旦超时返回，剩下的部分回不来了。长任务只能靠把超时调大硬扛
2. **没有取消**：跑飞了只能杀进程
3. **无人值守**：程序中途没法向人提问

完整说明见[边界与限制](./08-limits.md)。

---

[← 边界与限制](./08-limits.md) · [返回目录](./index.md)
