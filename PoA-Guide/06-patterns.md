# 模式库

以下四种形状是 PoA 程序的基本组织方式。它们是一条递进链，不是四个平行样例：

```mermaid
graph TD
    D1["<b>① 并行派发</b><br/>程序能并行驱动 N 个 agent<br/>→ 加速"]
    D2["<b>② 先分后总</b><br/>程序能在 agent 之间路由信息<br/>→ 产生了新信息，不只是加速"]
    D3["<b>③ 交叉验证</b><br/>程序能强制独立性并裁决<br/>→ 单个 agent 结构上做不到的结论"]
    D4["<b>④ 长驻助手</b><br/>agent 是有状态实体<br/>→ 不是无状态函数"]
    D5["<b>边界指针</b><br/>不作为模式展开<br/>→ 见 04 §4"]

    D1 --> D2 --> D3 --> D4 --> D5

    style D1 fill:#dae8fc,stroke:#6c8ebf
    style D2 fill:#dae8fc,stroke:#6c8ebf
    style D3 fill:#fff2cc,stroke:#d6b656,stroke-width:3px
    style D4 fill:#dae8fc,stroke:#6c8ebf
    style D5 fill:#f5f5f5,stroke:#666666,stroke-dasharray: 5 5
```

③ 产出的是单个 agent 在结构上无法产生的结论；末尾只保留一个边界指针。

以下都是节选，为了突出结构省去了配置与容错，可直接运行的完整程序见《05-writing.md》§12。代码使用 prelude，规则见《03-concepts.md》§2；每种模式末尾标出了纯内置替代。

---

## 模式 ① 并行派发（fan-out / fan-in）

**用于**：N 个互不相干的活，一次派出去，一次收齐。

```js
const items = (await shellLines("ls -d */ | sed 's#/$##'", {
  validate: (line) => SAFE_NAME.test(line),
})).slice(0, MAX_ITEMS);

const batch = await runBatch(
  items.map((x, i) => ({ message: prompt(x), name: `scan_${i}`, meta: { x } })),
  { concurrency: CONCURRENCY },
);

const results = batch.map(({ handle, reply }) => {
  const { value, error } = parseJsonReply(reply);
  return {
    item: handle.meta.x,
    summary: value,
    parse_error: error,
    raw: value ? undefined : String(reply ?? "").slice(0, 200),
  };
});
```

上面的 `MAX_ITEMS` / `CONCURRENCY` / `prompt` 在完整程序里定义，完整版见《05-writing.md》§12.1。

**要点**：

- `meta` 把结果映射回输入，agent 的回答里不含"我是谁"
- 规模边界统一见《05-writing.md》§6.3
- 收口必须防御性解析，并保留解析失败的原文

**成功判据**：看 `succeeded / total`。这个比值衡量的不是 agent 答得好不好，而是"靠 prompt 约束 JSON 格式"这个办法的可靠程度——分母是派出去的数量，分子是成功解析出 JSON 的数量。

**适用的需求形态**：

```text
把 sdk/ 下每个子包分析一遍，各自的用途、依赖、入口文件，汇成一张表。
给这 30 个测试文件各写一句话说明它在测什么。
```

**只用内置时**：`shellLines` → `tools.exec_command` + 手写行过滤；`runBatch` → `mapLimit` + `spawn` + `collect`；`handle.meta` → 数组下标。

---

## 模式 ② 先分后总（map-reduce）

**用于**：需要跨来源比对的问题，单个 worker 只看得见自己那份，答不出来。

在模式 ① 上加第二阶段：N 个 worker 各读一个目录，再派一个汇总者，只向它提供前面那批产出。

```js
const facts = (await runBatch(mapSpecs, { concurrency: 4 }))
  .map((b) => parseJsonReply(b.reply).value)
  .filter(Boolean);

const [{ reply }] = await runBatch([{
  message: [
    `你是汇总者。下面是几份报告。`,
    `不要读任何文件，只根据这些数据推理。`,        // ← 关键
    JSON.stringify(facts, null, 2),
    `回答：哪些依赖在 2 份以上报告里都出现？`,      // ← 单个 worker 答不出的问题
    `只回一行 JSON：{"shared_dependencies":[...],"observation":"<一句话>"}`,
  ].join("\n"),
}]);
```

"不要读任何文件"这一句是整个模式的支点。它把汇总者的信息来源限定为只有程序传入的那份 JSON，于是它的任何结论都必然来自前一轮派发。程序在这里扮演的是信息路由器，决定谁看得见什么。这是单个 agent 结构上做不到的事：一个 agent 没法假装自己没读过某个文件。

**成功判据**：看那个需要跨报告比对才能填的字段（例子里是 `shared_dependencies`）。它非空，即为"派发确实产生了新信息，而不只是加速"的证据。

**适用的需求形态**：

```text
这个 workspace 里哪些内部依赖被 3 个以上的 crate 共用？
把这 10 份会议纪要读一遍，找出反复出现但一直没解决的问题。
```

**只用内置时**：和 ① 一样，两阶段都用 `mapLimit` + `spawn` + `collect`，第二阶段 keys 长度为 1。

---

## 模式 ③ 交叉验证（对抗评审）

**用于**：真正要的不是一个答案，而是这个答案的可信度。

三阶段：一个 agent 提出可检验的断言 → 三个 agent 各带一个视角去反驳 → 程序数票。

```js
const REVIEWERS = [
  { lens: "存在性", ask: "断言里提到的文件和符号真的存在吗？去查。" },
  { lens: "语义",   ask: "就算存在，代码行为真的和断言说的一致吗？" },
  { lens: "分寸",   ask: "断言是不是说过头了——只在很窄的情况下成立？" },
];

const verdicts = await runBatch(
  REVIEWERS.map((r, i) => ({
    message: [
      `断言：${claim}`,
      r.ask,
      `证据不清楚时默认判定为被反驳。`,               // ← 默认有罪
      `只回一行 JSON：{"refuted":true|false,"why":"<一句话>"}`,
    ].join("\n"),
    name: `review_${i}`,
    meta: r,
  })),
  { concurrency: 3 },
);

const usable = verdicts
  .map((v) => ({ ...v, verdict: parseJsonReply(v.reply).value }))
  .filter((v) => v.verdict);

const refutedCount = usable.filter((p) => p.verdict.refuted === true).length;
const survives = usable.length > 0 && refutedCount < Math.ceil(usable.length / 2);
// 多数反驳即判死；平票也算反驳
```

### 三个设计要点，缺一条模式即不成立

**①** 视角是分化的，不是冗余的。三种不同的出错方式：符号不存在 / 存在但行为不符 / 行为对但说过头了。冗余检查抓不到多样化的错误。

**②** 默认判定为被反驳。prompt 里写死"证据不清楚时默认判定为被反驳"。模型倾向于附和，所以默认值要反转过来。

**③** 判决在程序里，不在模型里。就是最后那两行代码。

这是 PoA 核心论点的实证：单个 agent 给出的置信度是自我报告；三个互相隔离的 agent 分别尝试反驳后结论仍成立，才构成可信度。而"互相隔离"与"数票"只能由程序保证。

**成功判据**：看 `survives_review`。它是这几种模式里唯一一个单个 agent 在结构上无法产生的输出。

**适用的需求形态**：

```text
这份迁移方案里的技术断言，哪几条经得起查？
这个 bug 报告说的根因是对的吗，找三个角度各自去推翻它。
```

**只用内置时**：三个 reviewer 就是三次 `spawn` + 一次 `collect`；数票那两行是纯 JS，原样保留。

---

## 模式 ④ 长驻助手（有状态多轮）

**用于**：一次昂贵的读取要摊到多次提问上。

前三种模式里 agent 都是一次性函数调用，这一种把它当作能持续对话的实体：读一次文件，然后跨多轮追问，且不许重读。

```js
const [h] = await spawnMany([{
  name: "analyst",          // v2 认这个名字，v1 忽略它
  message: `读一遍 ${target}，报告行数和主题。读完记住内容——之后不许再读，凭记忆回答。`,
}]);

// 每一轮都要锁死 JSON 契约，否则下面的断言取不到字段
const a1 = await sendAndWait(h,
  '不要重读。凭记忆列出你看到的 5 个条目。只回一行 JSON：{"names":[...]}');
const a2 = await sendAndWait(h,
  '从你刚才列的那几个里挑最重要的一个，说明理由。只回一行 JSON：'
  + '{"picked":"<必须是你上一轮列过的名字>","why":"<一句话>","reread":false}');
const a3 = await sendAndWait(h,
  '我一共问了你几个问题？第一条消息里的文件路径是什么？只回一行 JSON：'
  + '{"questions_seen":<整数>,"first_file_read":"<第一条消息里的那个路径>"}');

await closeAll([h]);
```

**成功判据**：程序侧写两个断言，两个都为 `true` 才算成立。

```js
const a1Parsed = parseJsonReply(a1).value;
const a2Parsed = parseJsonReply(a2).value;
const a3Parsed = parseJsonReply(a3).value;

const contextRetained = Boolean(a3Parsed?.first_file_read?.includes(target));
const builtOnOwnAnswer = Boolean(
  a2Parsed && a1Parsed && (a1Parsed.names || []).includes(a2Parsed.picked),
);
```

三段追问里那几行 JSON 契约不能省——断言取的就是 `names` / `picked` / `first_file_read` 这几个字段。

`builtOnOwnAnswer` 检查的是：它挑的那个名字是否确实出自它自己上一轮列的名单。如果它重读文件重新生成，列出的名字未必与上一轮一致，这条断言即不成立。

`sendAndWait` 负责两代后端的状态衔接；恢复条件与两段超时预算见《07-api-reference.md》§2.1。

**实际价值**：文件读一遍进上下文，之后每次追问不再花 token 重读。

**适用的需求形态**：

```text
把这份 4000 行的规格读进去，然后我要连着问十几个问题，每次都重读太贵。
让它通读这个 crate，之后逐个模块追问设计意图。
```

**只用内置时**：需要自行实现追问与状态轮询；参考实现见
[《07 API 参考》§2.4](07-api-reference.md#24-prelude-全文)。

---

## 边界指针：原模式 ⑤

这一项不是可复用模式。本篇不再复述实验、机制或后果，统一以《04-how-it-works.md》§4 为准。

---

## 八条反模式

| 反模式 | 后果 | 正解 |
| --- | --- | --- |
| 按工具调用是否 `ok` 判断 agent 干成了 | 一个不通的 provider 看起来像行为符合文档的后端 | 先判终态种类（`{completed}` / `{errored}`），再看内容 |
| 硬编码 `mcp__x__y` 这类工具名 | 命名规则一变就调不到或调错工具 | 短名字可从 `ALL_TOOLS` 按后缀匹配并断言唯一；它只能扛前缀变化，扛不住哈希后缀与 64 字符截断，详见《07-api-reference.md》§3.1 |
| 用 `notify()` 输出进度 | 看不见 | 攒 `log` 数组，最后 `text()` |
| 用 `yield_control()` 分段交付 | 后半段回不来 | 调大 `yield_time_ms` |
| 派 agent 去 `ls` | 耗时、耗 token、结果不确定 | `tools.exec_command` 自己执行 |
| 直接信任 agent 回的 JSON | 偶发解析失败，且查不出原因 | `parseJsonReply` + 留原文 |
| 一上来就跑全量 | 一次错等 15 分钟 | 规模常量压到 1，`yield_time_ms` 压到 60000 |
| 在 `try` 块里调 `exit()` | 哨兵异常被自己的 `catch` 吃掉，程序继续往下跑 | 把提前退出的分支挪到 `try` 外，或改成 `throw` |
