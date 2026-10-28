# SBP · Session Bridge Protocol — 规范仓库

> 让「IM 里的一个网页 / 小程序」接管「你电脑上正在跑的 Agent 会话」
> （claude-code / codex / dsh / hermes / openclaw / workbuddy / qwen-office）。
> 先定协议，再落地实现。本仓库只放**协议规范本身**，不含任何运行时代码。

## 仓库划分

SBP 被拆成 4 个独立仓库，本仓库是「协议契约」，其余三者都依赖它：

| 仓库 | 角色 | 说明 |
|---|---|---|
| [sbp-spec](https://github.com/slamKunTech/sbp-spec) | 协议规范（本仓库） | 角色/帧/消息表/懒加载/安全/演进 + agent adapter 映射 |
| [sbp-server](https://github.com/slamKunTech/sbp-server) | 公网中继 | 鉴权、设备码绑定、路由、JSONL 持久化、倒序分页、seq 分配 |
| [sbp-agent](https://github.com/slamKunTech/sbp-agent) | 本机守护 | 装在跑 tmux+claude 的机器上：panel(tmux-claude 面板)+bridge(PC 侧桥接)+demo |
| [sbp-client](https://github.com/slamKunTech/sbp-client) | 客户端 | 微信小程序 + 单文件网页 SDK + IM 内嵌 skill |

## 为什么是 CS 而不是 P2P

被控电脑永远只**出站**连 server。不开端口、不配内网穿透、不依赖同一局域网。

## 目录

```
spec/
  SBP-0.1.md              协议正文（角色/帧/消息表/懒加载/安全/演进）
  SBP-0.2-interaction.md  交互提示：session.prompt / session.answer、选项卡片、三级降级
  SBP-transport.md        传输选型：WebSocket 推送 vs HTTP 轮询+ETag vs MQTT
  schema/*.json           envelope 与 session 索引的 JSON Schema
  adapters/*.yaml         8 个 agent 的形式化映射（纯声明式，禁止脚本）
```

## 传输默认

**REST 轮询 + ETag 作为主传输**，WebSocket 降级为可选升级（详见 `SBP-transport.md`）。
理由：在「省服务器资源 > 实时性、且不要 WS 长连接」的取舍下，轮询关闭连接后服务端几乎零成本，
未变化的轮询只是一次 304 比对；WS 则要始终持有 socket 并在每次变化时被推。
