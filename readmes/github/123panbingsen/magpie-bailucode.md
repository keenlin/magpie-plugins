# magpie-bailucode

把 **白鹿（BAILU AI）Open Platform** 接进 [magpie](https://github.com/yetone/magpie)：粘贴一枚 API key，白鹿的模型就出现在 magpie 的模型选择器里，Claude Code / Codex / OpenCode 等 agent 都能直接选。

> BAILU AI (白鹿) provider for magpie — one API key, models discovered from `GET /models`, OpenAI Chat Completions / Anthropic Messages / OpenAI Responses faces on the same gateway.

- key 页面：<https://bailucode.com/api_platform/#api-keys>
- 网关：`https://bailucode.com/openapi/v1`（一个 base，一个 `Authorization: Bearer sk-…`）

## 安装

```bash
# 从 GitHub 安装（推荐）
magpie plugin add github:123panbingsen/magpie-bailucode

# 确认装上了
magpie plugin list
```

改代码时直接装本地目录，magpie 会照原样加载：

```bash
magpie plugin add C:/path/to/magpie-bailucode
```

装完后 magpie 的 Providers 里会多出 **BAILU（白鹿）**。

## 添加 key

任一方式：

1. GUI：magpie → Providers → **BAILU（白鹿）** → 添加账号 → 粘贴 `sk-…`，地址提示直接回车（默认 `https://bailucode.com/openapi/v1`）。
2. CLI：`magpie plugin login bailucode`

卡片会显示网关是否收下这枚 key；不接受时会直接给出重新生成 key 的链接。

## 协议

网关同一个 base 下同时说三种协议（见下表的实测结果）。默认走 OpenAI Chat Completions，白鹿的 Anthropic 风格（thinking / 工具调用）请切到 `anthropic`：

```bash
magpie plugin options magpie-bailucode '{"protocol":"anthropic"}'
```

可选值：`openai`（默认，`@ai-sdk/openai-compatible`）、`anthropic`（`@ai-sdk/anthropic`）。

其它可注入的选项：`baseURL`（自建/镜像网关）、`models`（追加模型 id）、`headers`（附加请求头）。

## 模型列表从哪来

1. 有 key：`GET {base}/models`，白鹿返回什么就用什么（`{data:[…]}`、`{models:[…]}`、裸数组都认）。
2. 拉不到（key 没配、网关 5xx、被 Cloudflare 拦）：回落到内置清单（`bailu-2.9-preview`、`bailu-2.8`、`bailu-2.7`、`bailu-apex-2.7`、`yuyu-air`），保证 provider 至少可用，不会把模型选择器清空。
3. 你在 magpie 配置里自己写的 `models` 永远最优先，不会被覆盖。

> 内置清单是白鹿公开的模型家族；确切 id、上下文长度一律以 `GET /models` 为准。

## 网关实测（写这个插件时的取证）

用无效 key 直连 `https://bailucode.com`，把「路由存在」和「路由不存在」区分开：

| 路由 | 结果 | 结论 |
| --- | --- | --- |
| `GET /openapi/v1/models` | `401 Missing Authorization header` / `401 {"error":{"code":"invalid_api_key"}}` | 存在，认 Bearer |
| `POST /openapi/v1/chat/completions` | `401 {"error":{"message":"Invalid API key",…}}` | OpenAI Chat 面存在 |
| `POST /openapi/v1/messages` | `401` | Anthropic Messages 面存在 |
| `POST /openapi/v1/responses` | `401` | OpenAI Responses 面存在 |
| `POST /openapi/v1/embeddings` | `401` | Embeddings 面存在 |
| `GET /openapi/v1/{usage,balance,key,credits,…}` | `404` | 网关没有按 key 查额度/余额的路由 |
| `api.bailucode.com/*` | `410 service_retired` | 旧域名已下线，别再填它 |

因此这张卡片只报能证明的事：**key 是否被接受**。没有余额/用量接口，就不编一个。

## 请求身份（重要，实测）

同一个请求，只有「客户端身份」不同，Cloudflare 的判定就不同：

| 客户端 | 结果 |
| --- | --- |
| `user-agent: magpie-bailucode/0.1.0`（本插件） | `401 {"code":"invalid_api_key"}` — 网关正常应答 |
| `user-agent: undici` / Bun 默认 | `403` Cloudflare 「Just a moment…」HTML |
| 伪装的 `Chrome/131` UA，但 TLS 不是 Chrome | `403` — **声称是浏览器但指纹不符，必被拦** |
| `node:https` 原始请求 | `401` — 裸 TLS 也放行 |

结论：别伪装浏览器。插件在 `auth.loader` 的 headers 和 `chat.headers` 两处都把 UA 钉成 `magpie-bailucode/<version>`，所以推理请求不会被 Cloudflare 拦。上面三条协议（chat / messages / responses）已用这个 UA 穿过 `HTTP(S)_PROXY=127.0.0.1:7890` 验证，全部返回 `401 invalid_api_key` 而不是 403。

## 已知限制

- **额度、余额、按 key 的用量**：网关没有对应路由，magpie 里看不到，请到 <https://bailucode.com/api_platform/> 查。
- 没有实现 OAuth/邮箱登录——白鹿的 key 页面本身要登录，key 是唯一凭证。
- 模型清单里的上下文/输出长度：白鹿 `GET /models` 不返回时用内置估值（只影响客户端预算，不影响可用性）。

## 排障

```bash
magpie plugin list --json          # 有没有装上、provides 里有没有 bailucode
magpie plugin on bailucode         # 被关掉了就打开
tail -f "$LOCALAPPDATA/Magpie/logs/app.log"   # 插件日志：service=bailucode
```

- 模型列表空着：key 没配或网关拒绝，看卡片上的错误。
- 403 + HTML：Cloudflare 拦截出口，见上。
- `401 invalid_api_key`：key 失效/删除了，去 key 页面重新生成。

## 卸载

```bash
magpie plugin rm magpie-bailucode
```

插件是插卡式的：删掉它，magpie 的其它 provider 和配置一概不动。

## 开发

```bash
node --test test/plugin.test.mjs      # 16 个用例，全部 mock fetch
node ../tools/smoke.mjs               # 对真网关做一次冒烟（无效 key，直连）
```

## License

MIT