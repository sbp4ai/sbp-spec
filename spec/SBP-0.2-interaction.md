# SBP-0.2 · 交互提示扩展（Interaction Prompts）

> 本文是 SBP-0.1 的扩展，不改变 0.1 的信封、seq、懒加载与能力位规则。
> 它只解决一件事：**agent 把「请你选一个」摆到终端上时，远端 UI 不该让用户去敲键盘。**

## 0. 为什么需要单独一篇

SBP-0.1 只定义了「屏幕字节流」和「文本输入」。Claude Code 的

```
Do you want to proceed?
❯ 1. Yes
  2. Yes, and don't ask again for this project
  3. No, and tell Claude what to do differently (esc)
```

在 0.1 里只是一段 `output.screen`。手机端拿到它只能原样显示 ASCII，用户必须在小键盘上敲 `2\r`——这是终端时代的交互，不是协议。

0.2 把这类提示提升为**一等公民**：bridge 负责把屏幕（或结构化接口）解析成 `PromptSpec`，server 负责分发，UI 只渲染按钮。

---

## 1. 术语

| 术语 | 含义 |
|---|---|
| **Prompt** | agent 等待用户决策的一次交互。有唯一 `pid`，有生命周期。 |
| **Option** | Prompt 里可选的一项，对应终端上一个序号或热键。 |
| **Field** | Prompt 里需要填空的输入项（API key、路径、理由文本等）。 |
| **pid** | prompt id，bridge 侧生成，= `sha1(agent + 归一化提示文本)[:10]` |
| **指纹** | 生成 pid 用的归一化文本：去掉 ANSI、压缩空白、截到首行提示 + 所有选项行 |

---

## 2. 数据结构

### 2.1 PromptSpec

```jsonc
{
  "pid":  "p_9f3a1c7b02",
  "kind": "choice",              // choice | confirm | form | text | info
  "state": "open",               // open | answered | stale | gone
  "title": "Do you want to proceed?",
  "detail": "Bash: rm -rf node_modules && npm ci",   // 可选，风险决策的上下文
  "risk":  "warn",               // safe | warn | danger
  "options": [ OptionSpec, ... ],
  "fields":  [ FieldSpec, ... ],
  "allow_free_text": true,       // 选了 3 之后是否还要等一段自由文本
  "ts": 1730000000000,
  "raw":  "原始屏幕尾部（<=800 字符，供 UI 兜底显示）",
  "source": "screen"             // screen | hook | api
}
```

`kind` 取值语义：

| kind | 含义 | UI 必须渲染 |
|---|---|---|
| `choice` | 编号/热键选项 | 按钮组，支持点按 + 数字热键 |
| `confirm` | 是/否，等价于 options 只有两项 | 两个大按钮；`danger` 时次要按钮在右 |
| `form` | 需要填字段 | 输入框列表 + 提交 |
| `text` | 只要一段自由文本 | 单行输入 + 提交 |
| `info` | 只展示，不需要输入（如"按任意键继续"） | 一条提示 + 「继续」按钮 |

### 2.2 OptionSpec

```jsonc
{
  "index": 1,                    // 终端上的序号，没有序号时可为 0
  "key":   "y",                  // 热键；没有则为 null
  "label": "Yes",
  "desc":  "允许本次执行",
  "action": "input.choice",      // 作答时走哪条 semantics
  "payload": "1",                // 实际写入终端的内容（数字或热键）
  "risk":  "safe",               // safe | warn | danger，驱动按钮配色
  "default": false,              // 是否被终端光标高亮（UI 可加粗）
  "terminal": false              // 选了它会结束当前交互（如 "No (esc)"）
}
```

### 2.3 FieldSpec（`kind=form` 时才出现）

```jsonc
{
  "name": "api_key",
  "label": "API Key",
  "type": "text",                // text | secret | path | number | multiline
  "placeholder": "sk-...",
  "required": true,
  "submit_seq": "\r"             // 每填完一个字段按什么；最后一个字段用 \r
}
```

---

## 3. 帧

### 3.1 `session.prompt`（evt）

bridge → server → 所有 attach 了该 sid 的 client。

```jsonc
{"v":"sbp/0.2","type":"evt","name":"session.prompt","sid":"...","seq":42,
 "payload":{"pid":"p_9f3a1c7b02","kind":"choice","state":"open","title":"...","options":[...]}}
```

**幂等规则**：同一 `pid` 的 `state=open` 只允许出现一次。bridge 每轮 poll 重新计算指纹：

| 前后变化 | 动作 |
|---|---|
| 无 → 有 | 发 `state=open` |
| pid 变了（旧提示被新提示替代） | 发旧的 `state=gone`，再发新的 `state=open` |
| 提示消失（屏幕上正则不再命中） | 发 `state=gone` |
| 用户作答成功 | 发 `state=answered` |

### 3.2 `session.answer`（req）

client → server → 命中该 sid 的 bridge。

```jsonc
{"v":"sbp/0.2","type":"req","id":"c17","name":"session.answer","sid":"...",
 "payload":{"pid":"p_9f3a1c7b02","choice":2}}
```

payload 四种形态，互斥：

| 形态 | 语义 | bridge 动作 |
|---|---|---|
| `{"choice": <index>}` | 选第 N 项 | 按 option.action 写 `payload` + submit |
| `{"option": "<key>"}` | 按热键 | 写该 key + submit |
| `{"fields": {"k":"v"}}` | 填表 | 逐个写 value + 各自 submit_seq |
| `{"dismiss": true}` | 不作答，只让 UI 收起卡片 | 什么都不写，下次 pid 变才重开 |

响应：

```jsonc
{"type":"res","ref":"c17","name":"session.answer","ok":true,"payload":{"pid":"...","state":"answered"}}
```

错误码：

| code | 触发 |
|---|---|
| `E_PROMPT_STALE` | pid 已 gone/answered（用户对着过期卡片点了按钮） |
| `E_PROMPT_NOT_FOUND` | 该 sid 当前没有这个 pid |
| `E_NO_TMUX` | 会话不可写（只读历史会话） |

**UI 收到 `E_PROMPT_STALE` 必须收起卡片并提示「该选项已过期」**，不能保留一个按下去没反应的按钮。

### 3.3 附加在其它帧上的 `prompt` 字段

- `session.attach` 的响应、`pane.snapshot` 的响应、`session.state` 事件，都可以带一个可选 `prompt` 字段（当前 open 的 PromptSpec，没有则为 `null`）。
- 目的：UI 打开会话/重连后能立刻拿到卡片，不必等下一轮 poll。

---

## 4. 三级来源（降级链）

同一个 `PromptSpec` 可以由三种途径产出，bridge 按可用性自动选：

```
L2 结构化  agent 官方接口 / hook（如 PreToolUse 输出 JSON）   → 权威，字段全
L1 屏幕解析 capture-pane + adapter.prompts 正则             → 通用，见 §5
L0 无      解析不出，只发 output.screen                       → UI 隐藏选项卡片
```

`source` 字段标明实际来源。UI 对 L2/L1 一视同仁，对 L0 不渲染卡片（宁可不画，也不猜）。

---

## 5. adapter 的 `prompts` 段（替代 0.1 里零散的 `parse.choice`）

```yaml
prompts:
  - id: permission
    kind: choice
    match: {screen: "Do you want to (proceed|allow)"}
    title: {group: 0}                      # 取匹配行整个文本
    risk: warn
    options:
      - {index: 1, key: "y", label: "Yes",                     risk: safe}
      - {index: 2, key: "a", label: "Yes, don't ask again",    risk: safe}
      - {index: 3, key: "n", label: "No, tell Claude what to do differently", risk: danger, terminal: true}

  - id: numbered
    kind: choice
    match: {screen: "^\\s*(\\d+)\\.\\s+\\S", multiline: true}
    options_from: {regex: "^\\s*(\\d+)\\.\\s+(.+?)\\s*$", group_index: 1, group_label: 2}
    select: {action: "input.choice"}       # 写序号 + \r
```

约束（与 0.1 一致，**YAML 里禁止脚本**）：

- `options` 与 `options_from` 二选一
- 每条 `prompts[].options` 最多 9 项（终端只认个位数序号）
- `key` 必须是单字符
- 正则必须锚定行首 `^`，避免正文里的编号被误抓

**屏幕解析只对屏幕尾部生效**：bridge 只取最后 20 行做匹配，历史输出里的编号列表不算提示。

---

## 6. 风险分级（决定按钮配色与二次确认）

| risk | 判定 | UI 处理 |
|---|---|---|
| `danger` | 命中 `guards.confirm_regex`（rm -rf / DROP TABLE / 强制推送 / 生产部署）或选项含 `No`/`reject`/`cancel` | 红色描边，点按需长按或二次确认 |
| `warn` | 权限类提示（执行工具、写文件） | 橙色描边，直接可点 |
| `safe` | 其它 | 默认色 |

`guards.confirm_regex` 在 0.1 已有，0.2 只是把它的输出从「弹个对话框」变成 PromptSpec 上的 `risk` 字段，让所有端（网页/小程序/IM 卡片）共用同一套判定。

---

## 7. 时序

```
bridge                        server                     UI
  |  capture-pane (1.2s)         |                        |
  |--extract → pid 变化--------->|                        |
  |                              |--evt session.prompt--->|  画出卡片
  |                              |                        |  用户点「2」
  |                              |<--req session.answer---|
  |<---- 转发 -------------------|                        |
  |  send-keys "2" + \r          |                        |
  |--state=answered------------->|--evt session.prompt--->|  卡片收起
```

---

## 8. 与 0.1 的兼容

- 信封不变；0.2 只新增两个 `name`，老 client 收到 `session.prompt` 按 0.1 规则忽略未知事件即可
- 0.1 的 `parse.choice` 保留为 `prompts[].options_from` 的别名，旧 adapter 无需改
- 帧名别名（server 必须同时接受）：`sessions.index` ≡ `session.list`

---

## 9. 实现清单

| 组件 | 必须实现 |
|---|---|
| bridge | `extract(screen) -> PromptSpec?`、指纹缓存、`session.answer` 处理、`E_PROMPT_STALE` |
| server | 原样转发 `session.prompt`；`session.answer` 路由到持有该 sid 的 bridge；attach/snapshot 响应带 `prompt` |
| sdk/UI | 卡片渲染、`dismiss`、`E_PROMPT_STALE` 收起、`source=L0` 时不画 |
