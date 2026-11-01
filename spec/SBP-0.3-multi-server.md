# SBP-0.3 · 一个客户端，N 台服务器

> 依赖：[SBP-0.1](./SBP-0.1.md) 的信封与 seq 规则，[SBP-0.2](./SBP-0.2-interaction.md)
> 的交互提示。本文只描述**多服务器聚合**新增的部分。

## 1 要解决什么

0.1 里一台客户端握着的 token 写死了 `did`（一台设备）。于是一台机器 = 一个网址，
N 台机器就要 N 个网址、N 份登录态，手机上没法在一处切换。

0.3 引入 **server 级客户端**：一次握手拿到的是「这台服务器上登记的全部设备」，
客户端保存 N 条「地址 + 凭据」，用下拉切机器。

```
机器 A： 面板 --bridge ──► sbp-server A ─┐
机器 B： 面板 --bridge ──► sbp-server B ─┴─► 一个客户端（网页 / 小程序）
```

设备侧（面板）仍然只往外连，机器不需要对外开放新端口。

## 2 握手

```jsonc
// req
{"v":"sbp/0.1","id":"1","type":"req","name":"hello",
 "payload":{
   "role":"client",
   "auth":{"scheme":"hmac-sha256","keyId":"ecs",
           "ts":1793498584620,"nonce":"n1793498584620","sig":"<hex>"}
 }}
// res
{"v":"sbp/0.1","type":"res","ref":"1","name":"hello.ok","ok":true,
 "payload":{"server":{"v":"sbp/0.1","ts":...,"hmac":true,"devices":2},
            "device":null,"scope":"server","caps":["*"]}}
```

签名内容（与 tmux 面板、微信小程序逐字一致，同一份 keyId/secret 三端通用）：

```
sig = HMAC-SHA256(secret, "sbp1\n{keyId}\n{ts}\n{nonce}")   // 十六进制
```

`ts` 与服务端时间偏差超过 **5 分钟** 拒收（防重放）。

`scope` 决定这个连接能看到什么：

| scope | 准入方式 | did |
|---|---|---|
| `server` | HMAC 签名；或 `SBP_OPEN=1` 且未配密钥（仅调试） | `*` |
| `device` | 绑定码换来的 token（`POST /api/bind`） | token 里的 `did` |

失败一律 `4401`（`bad token`）；设备已吊销 `4403`。

## 3 会话列表

`session.list` 的返回在 0.1 基础上增加 `did` 与 `devices`：

```jsonc
{"ok":true,"payload":{
  "sessions":[{"sid":"...","did":"dev_a1","title":"...","kind":"claude",
               "agent":"claude-code","count":82,"updated_at":1793498584620,"status":"running"}],
  "groups":[{"kind":"device","did":"dev_a1","name":"ecs-host","cwd":"sbp-panel",
             "live":true,"items":[...],"count":26}],
  "devices":[{"did":"dev_a1","name":"ecs-host","agent":"sbp-panel","live":true,"count":26}]}}
```

device 级客户端拿到的结构与 0.1 完全一致（只有一组），旧客户端不需要改。

## 4 请求路由：sid → 设备

server 级客户端只知道 `sid`，归哪台设备由服务端反查：

1. `hub.sid_did`（设备上报 `session.index` 时记下）；
2. 兜底扫一遍 store 的索引（服务端重启后设备还没重报的情况）；
3. 整台服务器只有一台在线设备时，直接认定它。

| 请求 | 行为 |
|---|---|
| `session.history` | **转发到设备，等它的 res 回传**；设备离线才回服务端自己的 jsonl |
| `session.attach` | 本地登记订阅 + 转发到设备（res 里的 `tmux/kind/editable/prompt` 由设备给） |
| `session.answer` | 转发 + 等 res（UI 要能收到 `E_PROMPT_STALE`） |
| `pane.snapshot`、`session.spawn` | 转发 + 等 res |
| `session.input`、`session.snapshot`、`session.resize`、`session.subtitle`、`session.detach` | fire-and-forget，立刻回 `{"queued": true}` |
| `ping` | 应用层心跳，服务端自己回 |

转发时服务端登记 `pending[ref] = 发起的客户端`；设备的 res 回来后**只**转交给
发起者，不广播（30 秒未回包则丢弃）。

错误码：`E_NO_DEVICE`（设备离线，`retryable`）、`E_NO_SESSION`、`E_UNKNOWN_NAME`。

## 5 事件

| 名字 | 载荷 | 说明 |
|---|---|---|
| `device.state` | `{did, name, agent, online}` | 设备上下线；客户端据此重拉列表 |
| `session.index` | `{sessions:[...]}` | 设备上报的会话索引（每项带 `did`） |
| `output.delta` | `{mode:"screen", lines:[...]}` | 实时屏幕 |
| `session.prompt` | SBP-0.2 的 PromptSpec | 终端选项变按钮 |
| `session.message` | `{role, kind, text, ts, meta}` | 增量消息 |

扇出规则：帧属于某台设备时，发给 `cl.did == did` **或** `cl.all` 的客户端；
带 `sid` 的帧还要该客户端订阅过这个 `sid`。

## 6 设备侧（面板 `--bridge`）

面板注册成一台 device（`agent: sbp-panel`），caps：

```
session.index session.list session.history session.attach session.detach
session.input session.answer pane.snapshot session.subtitle ping
```

出方向帧与面板自己的 WS 通道语义一致，只是名字对齐服务端
（`sessions.index` → `session.index`）；`session.message` 的正文在
`payload.{role,kind,text,ts,meta}`，而不是面板内部的 `{message:{...}}`。

**历史必须由设备回答**：面板现场读 CLI 的 jsonl（claude 的
`~/.claude/projects/*/*.jsonl`、qwen 的 `~/.qwen/projects/*/chats/*.jsonl`），
内容与在本机面板里看到的完全一致；服务端那份只是设备离线时的兜底。

## 7 降级路径

| 情况 | 表现 |
|---|---|
| 设备离线 | 列表仍在（store 里有），历史翻服务端那份；写操作回 `E_NO_DEVICE` |
| 客户端不支持 0.3 | 旧客户端用 token 握手，`scope=device`，行为与 0.1 一致 |
| 设备不回答某个请求 | 客户端 30s 后超时；UI 提示重试，不阻塞其它会话 |
| 服务端未配 HMAC 且非 `SBP_OPEN` | 只能走绑定码（token） |
| nginx 少一层斜杠导致 301 | WebSocket 握手失败；客户端按「重连」处理（配置期用「测试连接」先暴露） |
