# SBP — Session Bridge Protocol v0.1

**一句话定义**：SBP 是一套让「IM 里的一个网页卡片」能远程接管「你电脑上任意一个正在跑的 Agent 会话」的开放协议。它把会话抽象成 `sessionId + 标题 + 单向追加的 JSONL 事件流 + 能力声明`，手机端只按需拉取，不在 IM 里同步全量糊屏。

- 状态：Draft / 0.1（可跑通参考实现）
- 定位：协议层，不绑定具体 IM、不绑定具体 Agent
- 参考实现：`sbp-server` + `sbp-bridge` + `sbp-sdk`
- 首批 IM：飞书（Lark）→ 钉钉 → 企业微信 → Telegram

---

## 0. 名词

| 名词 | 含义 |
|---|---|
| `device` | 一台被控电脑（跑着 agent 的机器），有唯一 `device_id` |
| `bridge` | 电脑侧的连接器进程，向 server **反向**建立 WS，负责把本地 agent 会话翻译成 SBP |
| `agent` | 被接管的程序：claude code / codex / hermes / openclaw / dsh / workbuddy / 千问办公 |
| `session` | 一个 agent 的一次会话，等于一条 `sessionId` + 一个 jsonl 文件 |
| `client` | 手机/PC 上 IM 内嵌的网页（sbp-sdk），或任何 SBP 客户端 |
| `adapter` | 声明式 YAML，把某版本 agent 的输入输出形式化映射到 SBP 语义 |
| `cap` | 能力位，如 `input.choice`、`input.interrupt`，决定网页上出现哪些可勾选控件 |
| `seq` | 会话内单调递增序号，全协议唯一的一致性锚点 |

---

## 1. 设计原则

1. **反向连接（CS 模式）**：bridge 主动连 server，永不要求用户家里/公司电脑开公网端口。server 是唯一有公网入口的一方。
2. **一个页面，不做 App**：client 永远是网页。IM 只负责「容器 + 鉴权 + 入口」，不负责渲染会话。
3. **懒加载是一等公民**：首屏只允许传 `sessionId + 标题 + 状态 + 时间`。历史必须由用户点击/上滑触发，按 `seq` 倒序分页拉取。
4. **能力先声明后渲染**：bridge 说自己能干什么，网页才画什么按钮。协议不允许客户端假设任何输入方式存在。
5. **文本控制面 + 结构化数据面**：控制用 JSON 帧，终端内容用「行模型」而非裸 ANSI（手机可读、可 diff、可缓存）。
6. **可替换的 agent**：新增一个 agent = 新增一个 YAML，不改 server、不改 sdk。
7. **弱网优先**：seq 断点续传、ack 背压、离线指令队列。地铁里点一下「继续」，出站后自动补发。

---

## 2. 拓扑

```
┌──────────────┐   HTTPS 网页(IM webview)   ┌────────────────┐
│  飞书/钉钉    │ ─────────────────────────▶ │  sbp-sdk 网页  │
│  机器人卡片   │                             └───────┬────────┘
└──────────────┘                                     │ WSS /sbp   (role=client)
                                                     ▼
                                            ┌──────────────────┐
                                            │    sbp-server    │
                                            │  hub / jsonl /   │
                                            │  bind / auth     │
                                            └────────┬─────────┘
                                                     ▲ WSS /sbp (role=device)
                                                     │ 反向长连接，NAT 穿透
                                            ┌────────┴─────────┐
                                            │   sbp-bridge     │
                                            │ adapter + source │
                                            └────────┬─────────┘
                                       ┌─────────────┼──────────────┐
                                       ▼             ▼              ▼
                                    tmux/pty      HTTP API      WS API
                                  claude/codex   hermes/dsh   openclaw/qwen
```

三个模块职责边界（必须严格，否则协议会烂）：

| 模块 | 允许做 | 禁止做 |
|---|---|---|
| `sbp-server` | 鉴权、路由、持久化、分页、限流、审计 | 解析 agent 输出语义、执行本地命令 |
| `sbp-bridge` | 采集会话、翻译语义、注入输入、声明能力 | 存全量历史、对外暴露端口、决定 UI |
| `sbp-sdk` | 渲染、懒加载、输入采集、本地缓存 | 直连设备、持有设备密钥 |

---

## 3. 身份与绑定（设备验证码）

### 3.1 流程

```
bridge                          server                      IM 用户
  │  POST /api/device/register    │                            │
  │  {name, agent, fingerprint}   │                            │
  ├──────────────────────────────▶│                            │
  │  {device_id, device_secret,   │                            │
  │   bind_code:"K7QM-3P9X",      │                            │
  │   bind_url, expires_in:600}   │                            │
  │◀──────────────────────────────┤                            │
  │  终端打印二维码 + 6~8位码       │                            │
  │                               │  ① IM 里发 /bind K7QM-3P9X │
  │                               │◀───────────────────────────┤
  │                               │  ② 或打开卡片网页输入验证码  │
  │                               │  ③ server 校验 code，写绑定  │
  │                               ├───────────────────────────▶│
  │  WS: hello(device_secret)     │   (同时推送 bind.ok)        │
  │◀─────────────────────────────▶│                            │
```

### 3.2 规则

- `bind_code`：`ABCD-EFGH` 形式（去掉易混字符 `0O1I`），TTL 600s，一次性，绑定成功即焚毁。
- 同一 `device_id` 可被多个 IM 账号绑定（家庭共享），但每个绑定关系独立可撤销。
- `device_secret` 只在注册响应里出现一次，bridge 落盘 `~/.sbp/device.json`（0600）。
- client 拿到的不是 `device_secret`，而是 `client_token`（HMAC 签名，默认 12h，作用域 `device_id + caps`）。
- 解绑：`POST /api/device/{device_id}/unbind`，服务端立即断开该 device 的全部 WS 并吊销 client_token。

### 3.3 Token 格式（零依赖）

```
token = base64url(json_payload) + "." + base64url(HMAC-SHA256(secret, payload))
payload = {"sub":"u_123","role":"client","did":"dev_x","cap":["input.text"],"exp":1730000000,"jti":"..."}
```
server 只验签 + 验 exp + 查 `jti` 黑名单（撤销用）。不做 JWT 库依赖。

---

## 4. 传输层

- 协议：WebSocket，子协议名 `sbp.v0.1`，单端点 `wss://<host>/sbp`。
- 编码：文本帧 = UTF-8 JSON；v0.1 **不定义**二进制帧（终端字节走 `enc:"b64"`，见 §8.2）。
- 心跳：服务端每 25s 发 `ping`，30s 无 `pong` 断开。客户端不必自己发心跳。
- 首帧必须是 `hello`，5s 内未收到则关闭（code 4401）。
- 关闭码：`4401` 未认证 / `4403` 无权限 / `4408` 首帧超时 / `4429` 限流 / `4500` 服务端错误。
- 重连：指数退避 1s→2s→4s→…→30s，带 ±20% 抖动。重连后必须重新 `hello` + `session.attach(from_seq=last_seq+1)`。
- 单连接订阅上限：client 8 个 session，device 不限（受内存限制）。

---

## 5. 帧格式

### 5.1 Envelope

```json
{
  "v": "sbp/0.1",
  "id": "c-8f3a1b",          // 发送方自增，用于 request/response 配对
  "type": "req",             // req | res | evt
  "name": "session.history",
  "ts": 1730000000123,       // 毫秒，仅供参考
  "ref": "c-8f3a1b",         // type=res 时指向请求 id
  "sid": "s_9f21",           // 会话相关消息必填
  "seq": 42,                 // 会话事件必填，单调递增
  "ok": true,                // 仅 type=res
  "err": {"code":"E_NO_SESSION","msg":"...","retryable":false},
  "payload": {}
}
```

约束：
- `v` 必填且必须匹配服务端主版本，否则 `E_VERSION`。
- `type=req` 必须带 `id`；`type=res` 必须带 `ref`。
- 未知 `name` 且 `type=req` → 返回 `E_UNKNOWN_NAME`；未知 `type=evt` → **静默丢弃**（前向兼容，这是可以加新事件的原因）。
- 未知 `payload` 字段必须忽略，不得报错。

### 5.2 错误码

| code | 含义 | retryable |
|---|---|---|
| `E_VERSION` | 版本不匹配 | false |
| `E_AUTH` | token 无效/过期 | false（需重绑） |
| `E_NO_DEVICE` | 设备离线 | true（排队） |
| `E_NO_SESSION` | 会话不存在或已归档 | false |
| `E_NOT_ALLOWED` | 能力未授权（例如客户端没 `input.interrupt`） | false |
| `E_BACKPRESSURE` | 发送过快，需降速 | true |
| `E_RATE_LIMIT` | 超限 | true |
| `E_INTERNAL` | 服务端异常 | true |

---

## 6. 会话模型

### 6.1 会话索引（首屏唯一允许的内容）

```json
{
  "sid": "s_9f21",
  "title": "重构 auth 模块",
  "agent": "claude-code",
  "adapter": "claude-code@1",
  "status": "running",        // running | idle | waiting_input | exited
  "updated_at": 1730000000123,
  "last_seq": 812,
  "msg_count": 214,
  "caps": ["input.text","input.choice","input.interrupt"],
  "preview": "第 3 个测试失败，正在看堆栈"
}
```

**硬性约束：`session.list` 响应不得包含任何 message 正文。** 违反即为不合规实现。首屏预算：50 个会话 < 8KB。

### 6.2 jsonl 存储

每条消息一行，append-only，文件路径 `data/sessions/<device_id>/<sid>.jsonl`：

```json
{"seq":811,"role":"agent","kind":"text","ts":...,"text":"...","meta":{"tool":"Edit"}}
{"seq":812,"role":"user","kind":"text","ts":...,"text":"继续"}
```

- `seq` 从 1 开始，严格 +1，不允许空洞（便于 `has_more` 与断点续传判断）。
- `role`: `user | agent | system | tool`
- `kind`: `text | choice | tool_call | tool_result | screen | error`
- 大字段（超过 32KB）截断为 `{"truncated":true,"ref":"blob_xx"}`，blob 单独存。

### 6.3 倒序懒加载（核心）

```
session.history  req  {sid, dir:"back", before_seq:812, limit:20}
                 res  {items:[{seq:812..793}], has_more:true, next_before_seq:793}
```
- `dir:"back"`：取 `seq < before_seq` 的最近 N 条，**按 seq 升序返回**（渲染时前端自行 prepend）。
- 省略 `before_seq` = 从尾部取（即"最新一页"）。
- 前端策略：进入会话 → 拉一页（20 条）→ 滚到顶部再拉 → 直到 `has_more=false`。端侧用 IndexedDB/内存 Map 按 `sid` 缓存已拉分页，30 分钟内不重复请求。
- 「10 万行会话」场景：首屏 1 次请求，约 6KB。

### 6.4 增量

`session.attach` 成功后，服务端从 `from_seq` 开始推 `output.delta`。客户端每收到 16 条或 200ms 发一次 `ack{sid,seq}`，服务端据此裁剪重放缓冲（保留最近 500 条或 5 分钟）。

---

## 7. 能力协商

### 7.1 能力位（v0.1 全集）

| cap | 语义 | 网页表现 |
|---|---|---|
| `session.list` | 可列会话 | 列表页 |
| `session.spawn` | 可新建会话 | 「+ 新会话」按钮 |
| `input.text` | 可发文本 | 输入框 |
| `input.key` | 可发特殊键 | ↑ ↓ ⏎ Esc 键位条 |
| `input.choice` | 可选择项 | 数字选项 chips（勾选） |
| `input.approve` | 可 y/n 确认 | 「允许 / 拒绝」两个按钮 |
| `input.interrupt` | 可打断 | 「停止」按钮（红色，二次确认） |
| `output.screen` | 屏幕快照 | 行渲染 |
| `output.stream` | 原始流 | xterm 渲染 |
| `file.read` | 可读文件 | 附件气泡 |
| `screen.snapshot` | 可主动要屏 | 下拉刷新 |

### 7.2 协商规则

- `hello` 时 bridge 上报 `caps`；server 与 client_token 的 `cap` 取交集；交集结果回给 client。
- client 渲染前必须按交集裁剪 UI。**没有 `input.choice` 就绝不能出现选项 chips**。
- 会话级 `caps` 可以比设备级更小（例如某个 dsh 会话只读）。
- 客户端发起未授权 `name` → `E_NOT_ALLOWED`，不得静默降级。

---

## 8. 输入 / 输出

### 8.1 输入

```json
{"name":"session.input","payload":{"sid":"s_9f21","kind":"text","text":"继续"}}
{"name":"session.input","payload":{"sid":"s_9f21","kind":"key","key":"UP"}}
{"name":"session.input","payload":{"sid":"s_9f21","kind":"choice","index":2,"label":"只跑失败用例"}}
{"name":"session.input","payload":{"sid":"s_9f21","kind":"approve","value":true}}
{"name":"session.input","payload":{"sid":"s_9f21","kind":"interrupt"}}
```
- 每条输入带 `cid`（客户端生成 uuid），bridge 侧去重 5 分钟，保证弱网重发不重复执行。
- `interrupt` 必须要求客户端二次确认，且必须记审计日志。
- 输入在设备离线时进入 server 队列（默认上限 20 条 / 10 分钟），设备重连后按序补发；超时返回 `E_NO_DEVICE`。

### 8.2 输出：行模型（默认）

```json
{"type":"evt","name":"output.delta","sid":"s_9f21","seq":812,
 "payload":{"mode":"screen","rows":40,"cols":100,
   "lines":["● 已完成 3 处修改","","❯ "],
   "cursor":{"row":2,"col":2},
   "full":true}}
```
- `full:true` 为整屏快照（首次/滚动/清屏），`full:false` 时 `lines` 是 `[起始行, ...]` 的脏行补丁。
- `mode:"stream"` 时改为 `payload:{data:"...",enc:"utf8"|"b64"}`，供 xterm 使用。
- 频率上限：bridge 侧 500ms 合并一次，禁止逐字节推送（手机流量 + 电量的硬约束）。

### 8.3 状态

```json
{"name":"session.state","sid":"s_9f21","payload":{"status":"waiting_input","hint":"需要确认是否继续"}}
```
`waiting_input` 时 IM 侧应推送一条可点的卡片消息（这是「手机管控电脑」的关键体验点）。

---

## 9. Adapter 映射（形式化）

新增 agent 只写一个 YAML。文件放 `spec/adapters/<agent>.yaml`，运行时由 bridge 加载。

### 9.1 结构

```yaml
adapter: dsh@2                       # 唯一标识 <name>@<major>
agent: dsh
match:                               # 自动识别规则
  cmd: ["dsh", "dsh-agent"]
  version: ">=2.0 <3"

transport:                           # 怎么连上它
  type: tmux | pty | http | ws
  target: "dsh-{{sid}}"
  capture: {mode: screen, rows: 40, cols: 100, interval_ms: 600}

state:                               # 怎么判断它在干嘛
  busy:    {regex: "Thinking|running\\.\\.\\."}
  idle:    {regex: "❯\\s*$|^\\$\\s*$"}
  waiting: {regex: "\\[y/n\\]|Do you want to proceed"}

parse:                               # 从屏幕提取结构化选项
  choice:  {regex: "^\\s*(\\d+)\\.\\s+(.+)$", group_index: 1, group_label: 2}
  prompt:  {regex: "^❯\\s*$"}

semantics:                           # 抽象动作 → 具体写法（模板）
  input.text:      {write: "{{text}}\r"}
  input.key:       {keys: {"UP":"\u001b[A","DOWN":"\u001b[B","ENTER":"\r","ESC":"\u001b","TAB":"\t"}}
  input.choice:    {write: "{{index}}\r"}
  input.approve:   {write: "{{yes_no}}\r"}
  input.interrupt: {write: "\u0003"}
  session.spawn:   {exec: 'tmux new-session -d -s dsh-{{sid}} -x {{cols}} -y {{rows}} "dsh"'}
  session.list:    {exec: 'tmux list-sessions -F "#{session_name}"'}

caps: [session.list, input.text, input.key, input.choice, input.approve, input.interrupt, output.screen]
```

### 9.2 模板变量

`{{text}} {{key}} {{index}} {{label}} {{yes_no}} {{sid}} {{pid}} {{cols}} {{rows}} {{cwd}}`

### 9.3 映射原则

- **只允许声明式**。YAML 里不能写任意脚本（否则变成远程代码执行面）。需要复杂逻辑 → 写 `transport.type: http` 的 sidecar，把复杂度关在进程里。
- 同一 agent 多版本 → 多个 YAML，用 `match.version` 区间选择；bridge 加载时按 `match` 打分，取最高分。
- 找不到匹配 adapter 时降级为 `generic@0`（纯 pty + 全文透传 + 只有 `input.text`）。

首批映射见 `spec/adapters/`：`claude-code@1`、`codex@1`、`hermes@1`、`openclaw@1`、`dsh@2`、`workbuddy@1`、`qwen-office@1`。

---

## 10. IM 接入层

网页是同一份，差异全在容器。**IM 适配器只解决三件事：入口、鉴权、webview 限制。**

| IM | 入口 | 免登方式 | webview 限制 | 首版状态 |
|---|---|---|---|---|
| 飞书 | 机器人卡片 / 网页应用 | `authen/v1/jsapi_ticket` + `window.h5sdk` | 需 HTTPS + 域名白名单，localStorage 可用 | v0.1 首选 |
| 钉钉 | 工作台 / 消息卡片 | `dd.runtime.permission` / jsapi authCode | 缓存更严，首屏建议 inline token | v0.1 |
| 企业微信 | 应用消息 | OAuth2 `snsapi_privateinfo` | 部分机型禁 localStorage → 用 URL fragment 传 token | v0.2 |
| Telegram | Inline Keyboard → WebApp | `initData` 校验 | 无特殊限制 | v0.2 |
| 纯浏览器 | 直链 | 验证码 | 无 | v0.1 |

三条硬约束（踩过坑的）：
1. **token 不进 Cookie**，走 URL fragment 或 `postMessage`，IM webview 跨域 Cookie 政策不可控。
2. **首屏必须 < 100KB**（SDK ~30KB gzip + 一个骨架屏），IM 内置浏览器首屏超过 1.5s 用户就走了。
3. **不能用 `alert/confirm`**，各 IM webview 表现不一致，全部自绘。

---

## 11. 安全

| 面 | 措施 |
|---|---|
| 传输 | 强制 WSS/TLS1.2+；HSTS |
| 设备 | `device_secret` 仅本地 0600；泄露可单设备吊销 |
| 客户端 | `client_token` 12h + 绑定关系撤销 + jti 黑名单 |
| 权限 | 能力位最小授权；`input.interrupt`/`input.approve` 单独审计 |
| 危险动作 | `rm -rf`、`git push --force`、`sudo` 等命中 bridge 侧正则 → 强制回落到 `waiting_input` 要求网页二次确认 |
| 数据 | jsonl 落盘即加密可选（AES-GCM，密钥来自 server env）；默认 30 天归档 |
| 审计 | 每次 input 记 `{who, when, device, sid, kind, hash(text)}`，保留 90 天 |
| 隔离 | adapter YAML 禁止脚本；transport 只能起白名单命令 |

明确**不做**的：v0.1 不做端到端加密（bridge↔server 之间明文于内存）。理由：多设备漫游 + 服务端分页需要可读历史；E2E 放到 v0.3 作为企业模式选项。

---

## 12. 一致性与弱网

- **至少一次**：输入侧靠 `cid` 去重；输出侧靠 `seq` 去重。
- **断点续传**：client 记 `lastAckSeq`，重连 `attach(from_seq=lastAckSeq+1)`；服务端缓冲不足时返回 `E_GAP`（→ 客户端改走 `session.history` 补洞）。
- **离线指令队列**：sdk 侧 IndexedDB 存 `{cid,sid,kind,payload,ts}`，上线后按 ts 顺序发；UI 显示「待发送 1 条」。
- **多端同看**：同一 sid 允许多 client 订阅，输入互不阻塞，屏幕以最后 ack 的 seq 为准。

---

## 13. 消息总表

| name | type | 方向 | 说明 |
|---|---|---|---|
| `hello` / `hello.ok` | req/res | 双向 | 握手，带 role 与 caps |
| `ping` / `pong` | evt | S↔C | 心跳 |
| `device.status` | evt | S→C | 设备上下线 |
| `session.list` | req/res | C→S | 只返回索引 |
| `session.history` | req/res | C→S | 倒序分页 |
| `session.attach` / `detach` | req/res | C→S | 订阅增量 |
| `session.input` | req/res | C→S→D | 输入注入 |
| `session.resize` | req | C→S→D | 终端尺寸 |
| `session.snapshot` | req | C→S→D | 主动拉屏 |
| `session.spawn` | req | C→S→D | 新建会话 |
| `session.created` / `closed` | evt | D→S→C | 会话生命周期 |
| `output.delta` | evt | D→S→C | 行补丁/流片段 |
| `session.state` | evt | D→S→C | running/idle/waiting |
| `ack` | evt | C→S | 消费确认 |
| `error` | res | S→C | 见 §5.2 |

完整 JSON Schema：`spec/schema/envelope.schema.json`。

---

## 14. 时序：手机点开会话到看到最新输出

```
client                server                 bridge(agent)
  │ session.list ────────▶│                      │
  │◀── 12 条索引(6KB) ────│                      │
  │ 用户点 s_9f21          │                      │
  │ session.attach ───────▶│── attach ───────────▶│
  │                       │◀── output.delta(812) ─│  ← 立即要一帧全屏
  │◀── screen(812) ───────│                      │
  │ 上滑                  │                      │
  │ session.history(back)─▶│── 读 jsonl 尾部 ─────│
  │◀── 20 条 ─────────────│                      │
  │ 输入「继续」           │                      │
  │ session.input ────────▶│── write ────────────▶│
  │                       │◀── delta(813..) ─────│
  │◀── delta ─────────────│                      │
  │ ack(815) ─────────────▶│                      │
```

---

## 15. 版本演进

- 主版本不同 → 拒绝连接（`E_VERSION`）。
- 次版本只允许「加字段 / 加事件 / 加 cap」，不允许改语义。
- 每个 evt 都可能有未知 `name`，客户端必须忽略而不是报错 —— 这是全套协议能平滑升级的唯一保障。
- 预留：`v0.2` 加二进制帧与断线补洞 `E_GAP` 语义；`v0.3` 加 E2E 与企业审计导出。

---

## 16. 合规检查清单（实现自检）

- [ ] `session.list` 里没有正文
- [ ] `session.history` 支持 `before_seq` 与 `has_more`
- [ ] 未知 `evt` 被静默丢弃
- [ ] 未知 `payload` 字段被忽略
- [ ] 无 `input.choice` 能力时不渲染选项
- [ ] `interrupt` 有二次确认与审计
- [ ] 弱网重连能 `from_seq` 续传且不重复渲染
- [ ] `bind_code` 一次性 + TTL
- [ ] adapter 为纯声明式，无脚本
