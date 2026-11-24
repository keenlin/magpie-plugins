# 腾讯 Marvis · magpie 供应商插件（final 0.7.0）

> **⭐ 使用文档请读 [`README-FINAL.md`](./README-FINAL.md)。**
>
> 本插件把腾讯 Marvis（应用宝 AI 助手）背后的 DeepSeek V4.1 Flash / V4 Pro、
> Claude Sonnet/Opus 4.6、GPT-5.4、Hy3、HY-Vision 2.0 接进 magpie / OpenCode，
> QQ / 微信登录，不用粘贴 key。**PC 与手机通用（手机通过本地 OpenAI 网关）。**

## 传输通道（0.7.0）

| 模式 | 说明 | 适用 |
|---|---|---|
| `gateway`（默认推荐） | 本地 OpenAI 兼容网关 `http://127.0.0.1:8088/v1`，跨平台，PC 与手机共用；手机在 Termux / PRoot 里跑 Python 网关即可 | PC + 手机，默认 |
| `off`（PC HTTP 直连） | `https://yybadaccess.3g.qq.com/v1/chat/completions`，Beacon SDK 按请求上下文现签 uskey | PC（2026-11-24 实测 200） |
| `openclaw`（WS） | 手机会话域 WebSocket，凭据来自 `native/handshake.log` | 备选；凭据短命（≈15-20 分钟），失效重抓 |

模式选择：环境变量 `MARVIS_TRANSPORT=gateway|off|openclaw`，默认 `gateway`（自动探测本地 8088）。
旧变量 `MARVIS_WS_MODE` 兼容保留。网关地址 / 鉴权 / 超时分别用
`MARVIS_GATEWAY_URL` / `MARVIS_GATEWAY_KEY` / `MARVIS_GATEWAY_TIMEOUT_MS` 配置。
出票与配置资产放**稳定目录**（用户环境变量 `MARVIS_NATIVE_DIR`，建议
`C:\Users\<你的用户名>\.magpie-marvis-native`），在插件目录之外，`magpie plugin update` 清不掉 —— 跨更新免疫；详见 README-FINAL 第二节。

**流式透传（0.7.0 新增）**：`gateway`（PC AG-UI WS）与 `openclaw`（手机会话域 WS）两条通道改为流式 ——
网关侧 `TEXT_MESSAGE_CONTENT` / openclaw 侧 `event:chat` 增量实时转成 OpenAI `chat.completion.chunk` SSE，
首字延迟从「整轮结束」降到秒级；**非流式行为不变**。flatten 改进：单条 user 消息原样直发，
system 独立置顶保留；`agent.run` 的 requestId 已唯一化。

**风控与节奏（0.7.0）**：出站补全请求受请求节奏闸 pacing 约束——
`MARVIS_MIN_REQUEST_INTERVAL_MS`（默认 `800`）控制最小间隔，`MARVIS_REQUEST_JITTER_MS`（默认 `400`）叠加随机抖动；
熔断器退避改为指数退避并持久化到 `<稳定资产目录>/breaker-state.json`（重启不丢）。
**随机 uskey 兜底默认关闭（0.7.0 行为变更）**：没有 `native/`、也没有 `MARVIS_USKEY_URL` 时直接 fail-fast
报可操作错误，不再静默放行随机票（线上必 `4100404`）；显式设 `MARVIS_ALLOW_RANDOM_USKEY=1` 可找回旧兜底（仅开发排查用）。
主动续期：token 剩余时效不足 `MARVIS_REFRESH_LEAD_MS`（默认 30 天，clamp 微信 TTL 1/3）时提前续期，
同账号最多每 `MARVIS_REFRESH_MIN_INTERVAL_MS`（默认 1 小时）续一次；微信 2h token 不再周期性掉线。
完整环境变量见 [README-FINAL](./README-FINAL.md)。

## 手机怎么用（Termux / PRoot / magpie 安卓壳）

手机端走 `gateway` 通道（**0.7.0 起也可以直接点 QQ / 微信登录**，详见 README-FINAL 第四节；
gateway 仍是推荐路径，账号池在网关侧维护）：登录 / 加号账号池由本地 Python 网关维护，插件用「免登录」占位账号出模型
（占位账号现已排在真登录之后）。

> **0.7.2 修好了手机端的两个真问题**（用户实测反馈）：上一版手机端点 QQ / 微信登录只剩一段文字 ——
> 既没有粘贴输入框、也没有「打开浏览器」按钮，三个登录入口只剩两个。实测宿主契约后已改回
> `method:"code"`（拿回「Code:」粘贴输入框）+ 不设 `opened:true`（拿回唤起按钮），
> 并恢复 **QQ 登录 / 微信登录 / 浏览器·粘贴回调地址** 三个入口；QQ 另用 `mqqapi://forward/url`
> 真正唤起手机 QQ App。

简要：① Termux 先 `pkg install python curl -y`，跑参考仓库 `yy158358/Marvis-proxy-public` 的 `sh run.sh start` 并加号登录；
② 插件目录通过 magpie 控制后台 `127.0.0.1:3430` → 插件 → 导入；
③ 复制 `plugin/gateway.example.json` 到 `~/.magpie-marvis-native/gateway.json` 开启网关模式
   （用 phone-kit 附件时直接 `sh phone-setup.sh` 会自动写这份配置）；
④ 加账号选「本地网关（手机推荐，免登录）」→ `magpie provider test`。
完整步骤见 [README-FINAL 第四节](./README-FINAL.md)。

## 历史

- 原版 `MYCF711/magpie-marvis`（HTTP-only）2026-11-18 起停更；
  2026-11-17 上游收紧 uskey 校验、2026-11-22 PC 端全局风控——HTTP 路线一度全灭。
- 本插件经历 0.2.x（现签修复）→ 0.3.x（openclaw WS 传输）→ 0.4.x（断网重连抓包、
  熔断器、中文首行护栏、AK/SK 休眠）→ 0.5.x（精简，HTTP 直连恢复后回归主链路）
  → 0.6.x（正名为纯 magpie 插件，新增本地网关传输，手机可通过 Termux / PRoot 使用）
  → 0.7.x（流式透传、熔断指数退避持久化、请求节奏闸、主动续期、随机 uskey 默认 fail-fast、
  手机端可直接 QQ / 微信登录——自带本地粘贴页，不再依赖旧包名补丁）
  的完整演进，修复史见 [CHANGELOG](./CHANGELOG.md)。

## 帧协议参考（openclaw WS 模式）

- openclaw WS（手机会话域）：握手头 `sessionId` / `token` / `platform: mobile` / `role: operator`，
  `connect` 帧的 `auth` 必须带 `relaySessionId` / `relayToken`（0.4.1 定位）且**不带** accessToken（0.4.2 定位）；
  `chat.send` → `event: chat`（text 累积，`isAgentTextFinal` 终止）/ `event: agent`（phase end/error）。

以上协议细节均为 2026-11-21～24 真机抓包实测值，上游随时可能改。
