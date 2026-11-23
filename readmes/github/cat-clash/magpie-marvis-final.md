# 腾讯 Marvis · magpie 供应商插件（final 0.5.x）

> **⭐ 使用文档请读 [`README-FINAL.md`](./README-FINAL.md)。**
>
> 本插件把腾讯 Marvis（应用宝 AI 助手）背后的 DeepSeek V4.1 Flash / V4 Pro、
> Claude Sonnet/Opus 4.6、GPT-5.4、Hy3、HY-Vision 2.0 接进 magpie / OpenCode，
> QQ / 微信登录，不用粘贴 key。PC 与 dsh-go 手机端通用。

## 传输通道（0.5.x）

| 模式 | 说明 | 现状 |
|---|---|---|
| `off`（HTTP 直连） | `https://yybadaccess.3g.qq.com/v1/chat/completions`，Beacon SDK 按请求上下文现签 uskey | **2026-11-24 实测 200**（PC 风控解除），PC 端默认推荐 |
| `remote`（openclaw WS） | 手机会话域 WebSocket，凭据来自 `native/handshake.log` | 手机会话域不受 PC 风控影响；凭据短命（≈15-20 分钟），失效后重抓放回 |
| `relay`（本地 AG-UI） | PC 版 Marvis 客户端的本地网关 `ws://127.0.0.1:10123` | 需 PC 客户端在跑；上游是否可用取决于客户端自身 |

模式选择：`native/openclaw.json` 的 `"mode"` 字段或环境变量 `MARVIS_WS_MODE`；
无 handshake 文件时默认 relay，有则默认 remote，`{"mode":"off"}` 强制 HTTP 直连。
出票与配置资产放**稳定目录**（用户环境变量 `MARVIS_NATIVE_DIR`，建议
`C:\Users\c\.magpie-marvis-native`：Beacon 出票文件 + `openclaw.json {"mode":"off"}`），
在插件目录之外，`magpie plugin update` 清不掉 —— 跨更新免疫；详见 README-FINAL 第二节。

## 历史

- 原版 `MYCF711/magpie-marvis`（HTTP-only）2026-11-18 起停更；
  2026-11-17 上游收紧 uskey 校验、2026-11-22 PC 端全局风控——HTTP 路线一度全灭。
- 本插件经历了 0.2.x（现签修复）→ 0.3.x（openclaw WS 传输）→ 0.4.x（断网重连抓包、
  熔断器、中文首行护栏、AK/SK 休眠）→ 0.5.x（精简：删 bridge / AK-SK / 自动刷新，
  HTTP 直连恢复后回归主链路）的完整演进，完整修复史见 [CHANGELOG](./CHANGELOG.md)。

## 帧协议参考（remote 模式）

- openclaw WS（手机会话域）：握手头 `sessionId` / `token` / `platform: mobile` / `role: operator`，
  `connect` 帧的 `auth` 必须带 `relaySessionId` / `relayToken`（0.4.1 定位）且**不带** accessToken（0.4.2 定位）；
  `chat.send` → `event: chat`（text 累积，`isAgentTextFinal` 终止）/ `event: agent`（phase end/error）。
- PC 客户端本地网关（relay）：AG-UI 协议，`?token=` 鉴权（token 在
  `%TEMP%\Tencent\Marvis\logs\Marvis.log` 的 `AiStarter.GetGatewayToken` 行）、
  字段名 `requestId`（非 `id`）、参数必须放 `payload`。

以上协议细节均为 2026-11-21～24 真机抓包实测值，上游随时可能改。
