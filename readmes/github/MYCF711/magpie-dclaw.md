# DClaw 龙虾 for magpie

**DClaw 1.3.30 原生 Anthropic Messages 直通：无需启动客户端，不经过 Claude Code，不注入 DClaw 客户端系统提示词。**

这是非官方的 Magpie / OpenCode 供应商插件。短信登录只用于取得账号的 `model_api_key`；聊天请求直接发送到 `model-router.edu-aliyun.com/v1/messages`。已有原生 Key 时可完全跳过 DClaw 站点，模型目录使用随包快照。

```text
11 个当前静态模型快照 · 在线目录自动更新
1M / 256K / 200K 实际上下文元数据
3 种登录方式 · 0 个客户端进程 · 0 条 DClaw 注入提示词
```

---

## 功能

- **脱离客户端** —— 不安装、不启动、不读取 `DClaw.exe`，不连接本地 `26899` WebSocket，不依赖内置 Claude Code。
- **原生 Messages 直通** —— 使用 `@ai-sdk/anthropic`，保留流式 text/thinking/tool use/usage 事件，成功 `Response` 原样返回。
- **剥离的是 DClaw，不是你的指令** —— 不注入模型目录里的 `prompt`、内置身份句、全局/项目 `CLAUDE.md`、技能商店和端口规则；调用方自己的 `system` 与 `messages` 原样保留。
- **三种登录** —— 手机号 + 短信验证码、DClaw 会话令牌、仅模型 API Key。纯 Key 模式不访问任何 DClaw 站点。
- **完整模型元数据** —— 在线读取上下文、最大输出、图片输入、推理能力与倍率；接口失败时回退最近目录或随包快照，不清空模型。
- **账号隔离与余额** —— 会话令牌只用于换取并缓存原生 Key，多账号不串号；卡片显示账号余额和套餐。
- **每日签到** —— 复刻客户端「腾讯签到」：读状态、按需领取、报出本次积分；不合格账号如实显示「不符合条件」，纯 Key 模式跳过。
- **原生错误语义** —— 登录失效、模型不存在、参数错误、限流、超时和上游故障分别映射，避免一次模型错误误杀全部账号。
- **密钥外发防护** —— 模型网关固定 HTTPS 白名单，拒绝重定向；自定义头不能覆盖 Authorization、Host 或 `x-api-key`。

## 模型

**DeepSeek / Qwen / Kimi / GLM** —— 登录后跟随平台目录上下架。下表只列当前代表项，完整列表以 Magpie 插件目录为准。

| 发行商 | 代表模型 | 上下文 | 最大输出 | 图片输入 |
|---|---|---:|---:|---|
| DeepSeek | `qwen/deepseek-v4.1-flash-1017` | 1,000,000 | 384,000 | ✓ |
| Qwen | `qwen/qwen4-max` | 1,000,000 | 128,000 | ✓ |
| Kimi | `qwen/kimi-k2.7-code` | 256,000 | 16,000 | ✓ |
| GLM | `qwen/glm-5.3` | 1,000,000 | 128,000 | ✗ |

`custom/deepseek^deepseek-v4.1-flash@default` 这类特殊 ID 会原样交给网关，不拆分 `/ ^ @`。模型倍率来自 DClaw 目录；插件不把“未声明免费”的模型标成免费。

## 登录与计费

在 Magpie 的 **插件 → DClaw 龙虾 → 登录** 中选择：

1. **手机号 + 短信验证码（推荐）**：不需要客户端；插件复刻 1.3.30 的 `/api/auth/send-code` 与 `/api/auth/login`。
2. **粘贴 DClaw 会话令牌**：插件通过 `/api/v1/user/info` 换取 `model_api_key`。
3. **仅模型 API Key**：直接使用已有 `model_api_key`，不访问 DClaw 站点，模型目录使用随包快照，不显示个人余额。

模型调用仍消耗 DClaw 账户余额。插件不伪造免费额度，也不把会话令牌当成原生模型 Key。

### 每日签到

Magpie 会在账号那一行画「立即签到」，也可在设置里开启自动签到。插件逻辑：

```text
GET  /api/v1/user/tencent-signin/status   → eligible / canClaim / claimable / claimCount
POST /api/v1/user/tencent-signin/claim    → 仅在 canClaim=true 时调用
```

结果按宿主契约报 `claimed` / `done` / `ineligible` / `inactive` / `failed`：今天已领也报 `claimed`（`credit` 为 0），这样宿主才会盖当天时间戳。签到奖励与次数上限由 DClaw 活动决定（当前接口给出 `maxClaims: 10`）。

> 实测当前账号 `eligible=false`、`reason="非活动用户"`，所以卡片显示「该账号不符合条件」——这是账号资格问题，不是插件没实现。换一个参加活动的账号即可正常领取。

DClaw 1.3.30 没有「每日任务」类接口；客户端里与任务相关的只有提醒红包（`/api/v1/client/reminder/reward/*`，需微信授权后按 `rewardRecordId` 领取）、卸载挽留奖励和邀请统计，都不属于每日任务体系，插件均未接入。

## 安装

源码已发布到 [GitHub](https://github.com/MYCF711/magpie-dclaw)；npm 尚未发布，用 git 源或本地目录安装。

### 从 GitHub（推荐）

```bash
magpie plugin add github:MYCF711/magpie-dclaw
magpie plugin login dclaw
```

更新时 `magpie plugin update`；git 源跟随默认分支 `main`，改代码前请先看 [`UPDATING.md`](./UPDATING.md)。

### 本地目录（改代码时）

```bash
cd plugin
npm test
node scripts/local-install.mjs
magpie plugin list
```

脚本只向 `~/.config/magpie/plugins.json` 添加当前插件目录，不重启 Magpie、不修改全局代理或其他供应商。执行前自动备份到 `~/.config/magpie/backups/magpie-dclaw/<时间>/plugins.json`；互斥锁防止两个安装流程并发，写入前再次比对原文，检测到宿主并发变更时放弃覆盖。安装后用 `magpie plugin list` 和本机网关确认热加载状态。

移除：

```bash
node scripts/local-install.mjs --remove
```

### 本地 tarball

```bash
npm pack --pack-destination ./dist
```

将生成的 `dist/magpie-dclaw-0.1.1.tgz` 复制到 `~/.config/magpie/plugins/dist/`，按脚手架的 `plugins.json + plugins/package.json` 方式登记后运行 Magpie 自带 Bun install。**不要**直接把 tgz 路径传给 `magpie plugin add`；该命令对本地 tgz 有已知问题。

## 验证

```bash
# 契约、协议、安全与全量测试
npm run check
npm run check:ua
npm run check:errors
npm test
npm pack --dry-run

# 真实网关：自动读取 Magpie 的 dclaw 登录槽，不打印密钥
node scripts/live-probe.mjs --rounds 3

# 也可只经环境变量传入，报告不会保存这些值
DCLAW_MODEL_API_KEY=<原生Key> node scripts/live-probe.mjs --rounds 3
```

实网脚本对同一请求交替测量原生 `fetch` 与插件 `runtime.forward` 热路径的 TTFB、首个 SSE 事件和完整流耗时，并检查 canary、上游消息形状及客户端提示词零注入。验收线：插件中位总耗时 ≤ 原生直连的 `1.20× + 100ms`，不达标时退出非零。完整 Magpie/SDK 链路另通过本机网关 smoke 验证。报告写入 `artifacts/`，不包含 token、Key 或手机号。

## 常见问题

**会删除 DSH 自己的系统提示词吗？** 不会。插件只不添加 DClaw 的提示词；DSH / Magpie 调用方提供的 system、developer、user、assistant 和 tool 内容都保留。

**为什么官方客户端回答风格不同？** 官方客户端还会注入身份、营销、技能、记忆、端口和文件规则，并启动 Claude Code。绕开这些层后，模型行为更接近同模型原生 API。

**是否需要重新登录？** DClaw 1.3.30 没有公开 refresh_token 端点。会话失效后重新走短信登录即可；仅模型 Key 模式不需要会话刷新。

**错误码含义？** 见 [`ERROR-CODES.md`](./ERROR-CODES.md)。协议细节和逆向证据见 [`docs/internals.md`](./docs/internals.md)，真实账号与性能结果见 [`docs/validation.md`](./docs/validation.md)。

## 参考源

| 来源 | 用在哪 |
|---|---|
| [MYCF711/magpie-plugin-scaffold v0.3.14](https://github.com/MYCF711/magpie-plugin-scaffold)（MIT） | 九钩结构、通用协议原语、测试与发布门禁 |
| [MYCF711/magpie-loomy-auth](https://github.com/MYCF711/magpie-loomy-auth)（MIT） | 手机号 + 短信 OAuth 交互与挂载测试结构 |
| 用户提供的 `DClaw_Setup_1.3.30.exe` | 认证字段、Anthropic Messages 网关、模型目录、提示词注入链与官方图标 |

> 更新记录：[`CHANGELOG.md`](./CHANGELOG.md)

## 许可

插件代码 MIT。DClaw 的名称、图标与服务归原厂商所有；本插件不是官方发布物。完整声明见 [`NOTICE.md`](./NOTICE.md)。
