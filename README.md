# SBP · Session Bridge Protocol — 规范仓库

> 让「IM 里的一个网页 / 小程序」接管「你电脑上正在跑的 Agent 会话」
> （claude-code / codex / dsh / hermes / openclaw / workbuddy / qwen-office）。
> 先定协议，再落地实现。本仓库只放**协议规范本身**，不含任何运行时代码。

## 仓库划分

SBP 拆成 6 个独立仓库，本仓库是「协议契约」，其余都依赖它：

| 仓库 | 角色 | 说明 |
|---|---|---|
| [sbp-spec](https://github.com/sbp4ai/sbp-spec) | 协议规范（本仓库） | 角色/帧/消息表/懒加载/安全/演进 + agent adapter 映射 |
| [sbp-server](https://github.com/sbp4ai/sbp-server) | 公网中继 | 鉴权、设备码绑定、路由、JSONL 持久化、倒序分页、seq 分配；以 submodule 引入 sbp-sdk |
| [sbp-sdk](https://github.com/sbp4ai/sbp-sdk) | 网页客户端库 | 单文件 `sbp-sdk.js`，被 sbp-server 作为静态文件提供给 IM 卡片 |
| [sbp-agent](https://github.com/sbp4ai/sbp-agent) | 本机守护 | 装在跑 tmux+claude 的机器上：panel(tmux-claude 面板)+bridge(PC 侧桥接)+demo |
| [sbp-minis](https://github.com/sbp4ai/sbp-minis) | 微信小程序 | 手机端驱动会话（REST 轮询默认，WS 可选） |
| [sbp-im-embed](https://github.com/sbp4ai/sbp-im-embed) | IM 内嵌技能 | 让智能体在 IM 里懒加载 SBP 网页的 Skill |

> 划分原则：spec 是唯一契约（版本化）；server 与 agent 是两端的运行时；sdk 是共享网页库
> （被 server 依赖，故以 submodule 引入而非复制）；minis 与 im-embed 是两种不同形态的消费端。

## 架构与仓库调用关系

```mermaid
flowchart TB
    classDef c_spec fill:#E8F0FE,stroke:#4285F4,stroke-width:2px,color:#10243f
    classDef c_srv  fill:#FCE8E6,stroke:#EA4335,stroke-width:2px,color:#3a0d0a
    classDef c_ag   fill:#E6F4EA,stroke:#34A853,stroke-width:2px,color:#0c2e18
    classDef c_sdk  fill:#FEF7E0,stroke:#F9AB00,stroke-width:2px,color:#3a2c00
    classDef c_mini fill:#F3E8FD,stroke:#A142F4,stroke-width:2px,color:#2e0a47
    classDef c_emb  fill:#E0F7FA,stroke:#00ACC1,stroke-width:2px,color:#00363d
    classDef c_note fill:#FFFFFF,stroke:#9AA0A6,stroke-width:1px,stroke-dasharray:4 4,color:#444

    SPEC["<b>📜 sbp-spec</b><br/>协议单一事实来源<br/>SBP-0.1 基础 / SBP-0.2 交互<br/>REST↔WS 双模 · HMAC · 错误码"]

    subgraph SRV["🔧 sbp-server · 公共枢纽 (公网 IP)"]
        direction TB
        SRV_H["WS Hub + JSONL Store<br/>seq 分配 (行号即 seq)"]
        SRV_D["Device Binding<br/>bind code / HMAC"]
        SRV_R["Router<br/>GET /api/index·state·history<br/>POST /api/rpc · /ws"]
        SRV_S["serve /sbp-sdk.js<br/>(submodule → sdk/)"]
        SRV_H --> SRV_R
        SRV_D --> SRV_R
        SRV_R --> SRV_S
    end

    subgraph SDK["📦 sbp-sdk · 单文件客户端 sbp-sdk.js"]
        direction TB
        SDK_T["双通道 REST / WS<br/>HMAC 签名 (ts=毫秒)"]
        SDK_L["懒加载 · 倒序分页<br/>字节偏移索引 O(1)"]
        SDK_T --> SDK_L
    end

    subgraph AG["🖥️ sbp-agent · PC 侧 (无入站端口)"]
        direction TB
        AG_B["sbp-bridge<br/>pty/tmux source + adapter DSL"]
        AG_A["active 判定<br/>claude L0/L1/L2 · dsh 进程级"]
        AG_P["panel (tmux-claude-panel)<br/>FastAPI + 无构建 Web UI<br/>读 ~/.claude/projects/*/*.jsonl"]
        AG_B --> AG_A --> AG_P
    end

    subgraph MINI["📱 sbp-minis · 微信小程序"]
        direction TB
        M_S["pages/settings<br/>传输方式 REST/WS 选择"]
        M_U["utils/sbp.js · hmac.js<br/>(手写, 无 WebCrypto)"]
        M_H["应用层心跳 ping"]
        M_S --> M_U --> M_H
    end

    subgraph EMB["💬 sbp-im-embed · IM 卡片懒加载页"]
        direction TB
        E_C["飞书 / 钉钉 卡片"]
        E_L["懒加载 web 页 → /ws"]
        E_C --> E_L
    end

    NOTE["<b>核心不变量</b>: bridge 只拨出 (PC 永不开放端口, 无需 NAT 穿透).<br/>所有客户端经 sbp-sdk 调 server API: GET /api/index·state·history, POST /api/rpc, /ws."]

    %% 依赖关系
    SPEC -. "定义" .-> SRV
    SPEC -. "定义" .-> SDK
    SPEC -. "定义" .-> AG
    SPEC -. "定义" .-> MINI
    SPEC -. "定义" .-> EMB

    SRV_S -. "submodule" .-> SDK

    SDK -. "嵌入/适配" .-> AG_P
    SDK -. "适配" .-> M_U
    SDK -. "嵌入" .-> E_L

    AG_B == "outbound WS 反向连接" ==> SRV_H
    AG_P == "WS / REST" ==> SRV_R
    M_H  == "wss / REST" ==> SRV_R
    E_L  == "WS / REST" ==> SRV_R

    SRV_H -. "fanout 事件" .-> AG_P
    SRV_H -. "fanout 事件" .-> M_H
    SRV_H -. "fanout 事件" .-> E_L

    class SPEC c_spec
    class SRV,SRV_H,SRV_D,SRV_R,SRV_S c_srv
    class AG,AG_B,AG_A,AG_P c_ag
    class SDK,SDK_T,SDK_L c_sdk
    class MINI,M_S,M_U,M_H c_mini
    class EMB,E_C,E_L c_emb
    class NOTE c_note
```

> 边样式速读：**虚线「定义」** = sbp-spec 是协议权威；**虚线「submodule」** = sbp-server 以 submodule 挂载 sbp-sdk；
> **虚线「嵌入/适配」** = 客户端内嵌 sbp-sdk；**实线粗箭头「调用」** = 运行时调用（bridge 仅出站反向连接，其余经 WS/REST 调 server）。
> 完整可编辑源见 [`architecture.mmd`](./architecture.mmd)。

## 协议版本

规范按**版本化契约**演进，实现方应固定到某个版本：

- 当前基线：**SBP/0.2**（`SBP-0.1.md` 角色/帧/懒加载 + `SBP-0.2-interaction.md` 选项卡片 + `SBP-transport.md` 传输选型）
- 每个实现在握手时通告自己的协议版本（如 `sbp-server` 的 `VERSION = "sbp/0.1"`、
  `sbp-sdk` 的 `V = "sbp/0.1"`、`sbp-agent` panel 的 `version="0.1.0"`），跨仓漂移时据此降级或报错
- 本仓库打有轻量 tag `sbp/0.2` 标记当前基线；实现方 `git checkout sbp/0.2` 即可锁定契约

> 各实现目前仍偏 `sbp/0.1`（握手子协议 `sbp.v0.1`），升级到 0.2 前请先对齐 `SBP-0.2-interaction.md` 的 `session.prompt/answer` 帧。

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
