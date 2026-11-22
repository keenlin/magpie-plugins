# GemPT v0.3.0

[English Quick Start & Docs](./README.en.md) · [Antigravity Plugin README](./plugin/README.md) · [Third-Party Licenses](./THIRD_PARTY_LICENSES.txt)

GemPT 让 Gemini 通过本机 [Magpie](https://github.com/yetone/magpie) 咨询 GPT、委派 Codex 仓库任务，并在 **GemPT Live** 侧栏实时查看和介入运行。Magpie 插件、Antigravity stdio MCP、hooks、安装器和 CLI 共用一套 Node.js ESM 核心；运行时零第三方依赖（仅用 Node.js 内置模块），GemPT Live 随包提供预构建资源与完整第三方开源许可（[THIRD_PARTY_LICENSES.txt](./THIRD_PARTY_LICENSES.txt)）。

## 前置依赖与版本要求

| 组件 | 要求 | 说明 |
| --- | --- | --- |
| **Node.js** | `>= 20.11.0` | 运行 MCP server、hooks、CLI 与安装器 |
| **Magpie** | `>= 0.1.1340` | 本地网关（默认 `http://127.0.0.1:3425`），从 [yetone/magpie](https://github.com/yetone/magpie) / [usemagpie.ai](https://usemagpie.ai) 安装并登录 ChatGPT/Codex 订阅 |
| **Magpie 侧 GemPT 插件** | `0.3.0` | 提供 `Plugins → GemPT` 面板，并托管共享 Codex `app-server`；**仅单独安装 Codex CLI 无法替代** |
| **Codex CLI** | `@openai/codex`（已实测 `0.164.0`） | 仅 `gpt_run` 仓库执行需要；通过 `npm install -g @openai/codex` 安装（纯 `gpt_ask` 咨询可不装） |
| **Git** | 任意现代版本 | 仅并发隔离 `workspace="share"` 需要 |

## 从零安装与首次验证

### 1. 安装并登录 Magpie

安装 [Magpie](https://usemagpie.ai)（`>= 0.1.1340`），启动本地网关并在 Magpie 中登录你的 ChatGPT/Codex 订阅（如需 API 兜底，也可配置 `openai/*` 提供商）。

### 2. 安装 Magpie 侧插件

```bash
# 从 npm 安装（或锁定版本：gempt@0.3.0）
magpie plugin add gempt

# 或从 GitHub 安装（锁定 tag）
magpie plugin add github:yizhang-dream/gempt#v0.3.0
```

在 Magpie 左侧栏打开 **Plugins → GemPT**，确认网关连通、订阅模型就绪以及共享 Codex `app-server` 状态正常，点击 **“一键启用订阅配置”**（或运行 `gempt init`）。

> Magpie 默认向其他接入客户端暴露 HTTP MCP（`http://127.0.0.1:3425/mcp/magpie/tools/gempt`），且默认 `exec: false`（仅允许 `gempt_ask`、只读 `gempt_run(access="inspect")` 与 `gempt_ctl`）。如需允许 HTTP 客户端执行写入，在 Magpie GemPT 插件选项中显式开启 `exec: true`。

### 3. 安装 Antigravity 侧插件

```bash
# 推荐：锁定版本一键安装
npx --yes gempt@0.3.0 install-agy
# 或直接从 GitHub tag 安装
npx --yes github:yizhang-dream/gempt#v0.3.0 install-agy
```

从本地源码安装时：
- **Linux / macOS**：`bash install.sh`（支持 `--target <目录>`）
- **Windows PowerShell**：`powershell -ExecutionPolicy Bypass -File .\install.ps1`（避免默认执行策略拦截）
- **构建可搬移 Marketplace 插件包**：`npm run build:agy-plugin`（输出自包含目录 `dist/agy-plugin/`，使用 `${PLUGIN_ROOT}` 与相对路径 hook）

安装器会将运行时、MCP 配置、hooks、规则、skills、图标与 Live 资源复制到 `~/.gemini/config/plugins/gempt`，并在 `~/bin` 写入 `gempt-wait` shim。若 `~/bin` 不在 `PATH` 中，将其加入 `PATH` 即可运行裸命令 `gempt-wait`，或始终直接执行 `gpt_run` 返回的完整 `wait_command`（内置绝对路径与跨平台引号后备）。

> **双装去重**：同时装了 Magpie 插件和 Antigravity 插件时，可在 Magpie GemPT 插件选项设置 `mcpTools: false`，让 Antigravity 走专属 stdio `gempt_gempt`，同时保留 Magpie 控制面板与共享 `app-server`。独立 stdio 客户端可用 `gempt print-mcp --installed` 导出配置。
>
> **常驻规则提示**：Antigravity 启用本插件期间，`rules/AGENTS.md` 会持续生效；在 Customizations 禁用插件即可同步停用。

### 4. 首次连通性验收

```bash
gempt discover
gempt init --non-interactive
```

随后在 Antigravity 中：
1. 首次咨询：调用 `gpt_ask(goal="Reply with OK", tier="sonnet")` 验证 Magpie 网关连通。
2. 首次只读委派：调用 `gpt_run(goal="Inspect repository layout and report key files", cwd="<你的项目路径>", access="inspect")`，并按返回的 `wait_command` 后台等待唤醒。

### 从 Python 旧版升级

对原插件目录重新运行上述安装命令即可，无需 Python。旧目录存在 `work_split_hook.py` 时，安装器会自动生成兼容转发入口，让未重启的旧会话平滑过渡；共享配置与会话目录继续沿用。

## 三个 MCP 工具

| Antigravity stdio (`gempt_gempt`) | Magpie HTTP | 用途 |
| --- | --- | --- |
| `gpt_run` | `gempt_run` | 异步 Codex 仓库任务。传 `goal`、`cwd`，用 `access="inspect"` 只读调查或 `access="change"` 改代码。多路并发改同一 Git 仓库时传 `access="change"` + `workspace="share"`（基于已提交 `HEAD` 建独立 worktree；只读 `inspect` 禁止使用 `share`）。 |
| `gpt_ask` | `gempt_ask` | 同步咨询。传 `goal`，可选 `cwd`、`file_paths`、`tier`（`sonnet`/`opus`/`fable`/`chore`）与 `session_id`。 |
| `gpt_ctl` | `gempt_ctl` | 状态查询、转向、中止或回复 `ask_gemini`（`action="status"|"steer"|"interrupt"|"reply"`）。在 Antigravity 调用 `gpt_ctl({})` 打开 **GemPT Live** 侧栏。 |

`gpt_run` 返回 `run_id`、`session_id` 和 `wait_command`。请优先原样通过 `run_command(WaitMsBeforeAsync=500)` 后台执行返回的 `wait_command` 等待系统唤醒（终端手动等待可用 `gempt-wait <run_id>`、`gempt-wait id1,id2` 或 `gempt-wait --all id1,id2`）。续聊传 `session_id`；失败返工省略 `session_id` 开新线程。

## 分工档位与路由模型说明

GemPT 将「**何时派给 GPT**」与「**派给哪一档模型**」解耦：
- **5 档工作分工（Work Split，在 Magpie 面板切换）**：`0%` 关闭 / `15%` 重活提示 / `35%` 源码写转 GPT / `60%` 源码写 + 每回合 5 步核实止损 / `80%` 入口直派 + 2 步核实（80% 档计入 Gemini 前端编排约 15–20% 的占比下限）。
- **3 档执行模型 + `chore` 旁路**：
  - `sonnet`（~75% 日常开发）、`opus`（~20% 复杂工程）、`fable`（~5% 顶配攻坚）；`chore` 为免费高速旁路（仅 `gpt_ask`，用于摘要/压缩/分类）。
  - 模板中的 `Sol`（`codex/gpt-6.2-sol`）、`Astra`（`codex/gpt-6.2-astra`）均为 **Magpie 网关模型标识**，可在 `~/.config/gempt/config.json` 或 [templates/magpie-groups.template.json](./templates/magpie-groups.template.json) 中按你实际的订阅模型自由修改。
  - `/gempt-sonnet`、`/gempt-opus`、`/gempt-fable`、`/gempt-chore` 快捷技能用于引导对应档位咨询或调用 `gempt tier` 切换共享默认档位（`gpt_run` 的实际执行档位以共享配置与面板为准）。

## 命令行

```sh
gempt discover
gempt init --non-interactive
gempt tier opus
gempt sandbox workspace-write
gempt effort auto
gempt status --run-id <run_id> --wait
gempt prune-worktrees <repo_dir> --max-age-hours 168
gempt presets --preview
gempt presets --apply
gempt print-mcp --installed
```

`gempt prune-worktrees` 仅安全清理**无未提交改动且 `HEAD` 已合入主仓库**的闲置隔离 worktree，绝不删除未合并的工作成果。

## 权限、数据与安全边界

- **本地数据**：配置与会话默认位于 `~/.config/gempt/`（可用 `GEMPT_CONFIG`、`GEMPT_SESSIONS_DIR` 覆盖）。
- **工作区边界**：Antigravity stdio `gpt_run` 默认 `access="change"` 并沿用配置的 Codex 沙箱（推荐 `workspace-write`）。建议在外部/敏感环境中配置环境变量 `GEMPT_WORKSPACE_ROOT` 限定允许访问的根目录。
- **只读与 Worktree 隔离**：`access="inspect"` 严格禁止搭配 `workspace="share"`；`workspace="share"` 从已提交的 `HEAD` 检出（工作区未提交改动不会自动带入），创建时禁用目标仓库 Git hooks，并校验 worktree 归属。
- **回环鉴权**：Magpie 控制面板仅监听 `127.0.0.1:3498`，校验 Host/Origin、`x-gempt-key` 及本机身份文件；共享 Codex `app-server` 仅监听回环并启用 capability-token 认证。
- **Live 会话隔离**：GemPT Live 默认按当前绑定的宿主会话过滤（`scope="session"`），未绑定时默认不跨会话泄露历史运行，需显式切到 `scope="all"` 查看全部。

## 升级与卸载

- **升级**：重新运行 `magpie plugin add gempt@<新版本>` 与 `npx --yes gempt@<新版本> install-agy`。
- **卸载 Magpie 侧**：在 Magpie **Plugins** 页面移除 `gempt`，或执行 `magpie plugin remove gempt`。
- **卸载 Antigravity 侧**：在 Antigravity Customizations 卸载 `gempt` 插件（或删除 `~/.gemini/config/plugins/gempt`），并删除 `~/bin/gempt-wait*`。
- **清理个人配置（可选）**：删除 `~/.config/gempt/` 及本机隔离 worktree 目录（默认 `~/.cache/gempt-worktrees` 或 `%LOCALAPPDATA%\gempt-worktrees`）。

## 验证与开发

```sh
npm test
npm run build:agy-plugin
npm pack --dry-run
```
