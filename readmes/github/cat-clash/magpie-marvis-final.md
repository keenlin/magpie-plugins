# 腾讯 Marvis · magpie 供应商插件（WS / AG-UI 版）

> **⭐ 这是 final 版（0.4.0）——先读 [`README-FINAL.md`](./README-FINAL.md)**：
> 4100404 → 429+Retry-After 与熔断冷却、官方中文兜底首行、4100406 U+2060 护栏、
> AK/SK 休眠签名、dsh-go 手机端面板，PC + 手机双端通用。以下为 WS 传输层的原始说明。

> 原版 `MYCF711/magpie-marvis` 已停更。2026-11-22 起腾讯对 **PC 端** `/v1/chat/completions`
> 做了**全局 4100404 风控**：4 个账号 × 2 个槽位实测全部失败，`uskey` / AK-SK / UA / IP
> 全试过，连官方 PC 客户端自己的标题改写也返回 4100404。
> **HTTP 模型网关这条路在服务端已经死了。**
>
> 本插件改走 WebSocket 通道（文档 [`dsh-magpie-marvis`](https://github.com/lyx2839272-coder/dsh-magpie-marvis)
> 路线 11.3 的本地版），并附上两条**实测确认**的帧协议。

## 它和原版的区别

| | 原版 / fix | 本版 |
|---|---|---|
| 推理传输 | HTTP `/v1/chat/completions`（4100404 全挂） | 本地网关 AG-UI / 远端 openclaw WS |
| 设备指纹 | Beacon SDK 现签 uskey | 不需要 uskey |
| 多轮对话 | 真多轮 | 展平成一段 prompt（WS 限制） |
| 流式输出 | 逐 token | 一次性文本包成 SSE 两帧 |
| usage | 真实 token | 恒 0 |

## 三条通道

### 通道 0：bridge —— 借用手机 App（唯一能真正出内容的，实测可用）

PC 端 HTTP 被 4100404 封、手机端 agent 又独占，所以最后走的是「**让手机 App 自己发消息**」：

```
magpie → http://127.0.0.1:3502/v1 → tools/marvis-bridge.py
                                      ├─ adb input 注入文本到 App
                                      ├─ frida hook SSL_read 抓回复
                                      └─ 包成 OpenAI 格式返回
```

**实测结果**（2026-11-23）：

```
POST /v1/chat/completions {"content":"say HELLO"}
→ 200 {"choices":[{"message":{"content":"HELLO"}}]}

stream:true → text/event-stream，两帧 + [DONE]
```

启动：

```bash
# 1) 模拟器里 Marvis App 已登录、保持前台
# 2) frida-server 跑起来（x86_64 版；arm64 版在 MuMu 转译层加载不了 libart）
adb root && adb shell "nohup /data/local/tmp/frida-server -l 0.0.0.0:27042 &"
adb forward tcp:27042 tcp:27042

# 3) 起桥
python tools/marvis-bridge.py
```

插件配置：

```jsonc
// native/openclaw.json
{ "mode": "bridge", "bridgeUrl": "http://127.0.0.1:3502/v1" }
```

**代价（务必知道）**：

- App 必须**保持前台**（后台会被系统限制，注入点不到）
- 每轮延迟 **3～10 秒**（注入 + 等待模型回复）
- **多轮会被拍平**：每次只发最后一条 user 消息，不带历史
- 流式是「拿到完整回复后一次性吐出两帧」，不是逐 token
- usage 恒 0
- 并发差：同一时刻只能跑一条（UI 只有一个输入框）

### 通道一：relay —— 本地 xiaobao 网关（免抓包，链路通、上游被风控）

Marvis PC 客户端自带一个本地网关，前端就是通过它对话的。

```
连接   ws://127.0.0.1:10123/?token=<gateway token>
首帧   ← {"type":"event","event":"gateway.connected","data":{"agentConnectionStatus":"Ready"}}
请求   → {"type":"req","requestId":"1","event":"agent.run","payload":{
           "conversation_id":"conv_<32hex>",
           "message":"...",
           "response_id":"resp_<32hex>",
           "attachments":[],
           "mcp_manage_tools_enabled":true }}
响应   ← {"type":"res","requestId":"1","ok":true,...}
       ← {"type":"event","event":"ag_ui_event","data":{"event":"TEXT_MESSAGE_CONTENT","delta":"..."}}
       ← {"type":"event","event":"ag_ui_event","data":{"event":"RUN_FINISHED"}}
```

**关键细节（踩过的坑，别改）**：

- `token` 是 PC 客户端**每次启动随机生成**的，不带会被鉴权中间件拒（`url_token 缺失或与网关 token 不匹配`）。
  插件自动从 `%TEMP%\Tencent\Marvis\logs\Marvis.log` 的
  `AiStarter.GetGatewayToken ... {"token":"<uuid>","port":10123}` 行取最后一条。
- 帧里用 **`requestId`**（不是 `id`），缺了回 `missing field requestId`。
- 事件参数必须放 **`payload`** 字段；放 `data` / `params` / `body` 会回
  `agent.run payload 必须为对象`。
- 事件名是 **`agent.run`**（不是 `chat.send`，也不是 openclaw 的 `method:connect`）。

**边界**：这条通道只是把请求交给 PC 客户端的本地 agent，agent 最终仍要调 Marvis 的
HTTP 模型网关 —— 而 PC 端目前被 4100404 全局风控，回复通常表现为
「当前环境有误，暂时无法回答你的问题」。**链路可用、上游受限**，这是环境问题不是插件问题。

### 通道二：remote —— 手机会话域（真正能出内容）

直连 openclaw 远端，复用**手机 App 的会话**（手机端是另一个风控域，不受 PC 风控影响）。
需要从手机上抓一次握手：

```text
# handshake.log（要 Host 头 + 两行凭据）
Host: 124.223.136.123:20017
sessionId: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
token: xxxxxxxxxxxx

# frames.log（\n-----\n 分隔；connect 帧必需，MVT1 hello 可选）
{"type":"req","id":"...","method":"connect","params":{...}}
```

```jsonc
// native/openclaw.json
{
  "mode": "remote",
  "url": "wss://124.223.136.123:20017/ws",   // 留空则从 handshake.log 的 Host 解析
  "handshakeFile": "C:/path/to/handshake.log",
  "framesFile": "C:/path/to/frames.log",
  "sessionKey": "agent:main:main",
  "killCmd": ""                              // 可选：每请求前抢占手机独占会话
}
```

## 配置

环境变量优先于 `native/openclaw.json`：

| 变量 | 说明 |
|---|---|
| `MARVIS_WS_MODE` | `bridge`（手机 App）/ `relay`（本地网关）/ `remote`（手机会话）/ `off`（退回原版 HTTP） |
| `MARVIS_BRIDGE_URL` | bridge 模式地址，默认 `http://127.0.0.1:3502/v1` |
| `MARVIS_GW_URL` | relay 网关地址，默认 `ws://127.0.0.1:10123/` |
| `MARVIS_GATEWAY_TOKEN` | 手动指定网关 token（默认自动从日志读） |
| `MARVIS_WS_URL` | remote 的 wss 地址 |
| `MARVIS_HANDSHAKE` / `MARVIS_FRAMES` | remote 的抓包文件 |
| `MARVIS_SESSION_KEY` | 默认 `agent:main:main` |
| `MARVIS_WS_TIMEOUT` / `MARVIS_GW_TIMEOUT` | 单轮超时（默认 180s） |

自检：

```bash
node scripts/probe-openclaw.mjs
```

## 配置上限（社区实测值）

已按实测上限落地到插件：

| 项 | 旧值 | 新值 | 位置 |
|---|---|---|---|
| `maxConcurrency` | 2 | **8** | `src/auth.mjs` + `package.json` |
| 默认输出 `DEFAULT_OUTPUT` | 32000 | **131072** | `src/context.mjs` |
| 上下文 | 1,000,000 | 1,000,000（已是上限） | 原样 |
| `maxAccounts` | 10 | 10（已是全部账号） | 原样 |

以下属于宿主/网关层，不在插件包内：

| 项 | 旧值 | 新值 | 实测依据 |
|---|---|---|---|
| `maxRPM` | 20 | **120** | 8 并发 9 秒跑完 8 发 |
| `minIntervalMs` | 8000 | **1000** | 1 秒间隔连发 15 发全 200 |
| 冷却 `cooldownMs` / 上限 | 600s / 3600s | **120s / 600s** | 已有自动换号兜底 |

## 安装

```bash
magpie plugin add cat-clash/magpie-marvis-ws
```

provider id 保持 `marvis`，**已有的登录账号和路由直接复用**，不用重新登录。

## 开发

```bash
node --test          # 194 项离线测试（含两个 mock WS 端到端）
node scripts/check.mjs
```

## 许可

MIT。不含任何账号凭证或二进制；握手模板、token、qimei36 均需你自己的设备产出。
