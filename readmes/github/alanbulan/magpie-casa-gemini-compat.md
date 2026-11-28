<p align="center">
  <img src="./assets/casa-logo.jpg" width="112" height="112" alt="CASA 厂商标识" />
</p>

<h1 align="center">CASA Gemini 工具与缓存兼容</h1>

<p align="center">面向 MagPie 的第三方供应商插件与网关中间件</p>
<p align="center">
  <a href="https://usemagpie.ai/docs/plugins">MagPie 插件规范</a> ·
  <a href="./docs/架构设计.md">架构设计</a> ·
  <a href="./docs/配置说明.md">配置说明</a> ·
  <a href="./docs/客户端接入.md">客户端接入</a> ·
  <a href="./docs/规范核对.md">规范核对</a> ·
  <a href="./docs/实测与AB.md">测试与实测</a>
</p>

CASA 的 Gemini 流式响应可能把工具名称和参数拆成不同调用，导致客户端收到空参数或无名称调用。本插件默认向 CASA 的 `/v1/chat/completions` 发送 `stream:false`，完整校验工具参数后，再按客户端请求的协议返回 JSON 或 SSE。等待期间只发送 SSE 注释心跳。

**仅处理 `casa/gemini-*`。** 其他供应商与 CASA 的 GPT、Claude 模型保持原路由。没有 GPT 降级通道，也不修改原供应商配置。兼容对象是当前 CASA（`apikey.casa`），厂商 LOGO 来自其官网，来源见 [资源说明](./assets/来源.json)。本项目与 MagPie、CASA 均无官方隶属关系。

## 能做什么

- Responses、Chat Completions、Anthropic Messages、Gemini Native 四种入口，共用协议无关的 IR 与缓存引擎。
- 支持声明式函数工具、并行工具、MCP 暴露的函数工具、Responses 命名空间、客户端工具搜索，以及自由文本工具的参数包装。
- 工具 Schema 深度排序、原始参数字符串透传、确定性历史调用 ID、可配置动态头冻结与后置。
- 会话指纹、短请求旁路隔离、粘性密钥与通道、TTL、有界状态和同会话串行保护。
- OpenAI 用量采用包含式，Anthropic 用量采用分离式；原生缓存计数优先，缺失时按字节 LCP 估算。
- 首业务帧前支持 502/503/404 备用 Gemini 通道切换；400 只裁剪配置匹配的末尾辅助块。
- 脱敏 Prefix Diff 日志、独立缓存 A/B 脚本与不启动 MagPie 的插件契约测试。

“工具通用”指客户端声明并执行的工具。厂商托管的 `web_search`、`file_search`、Google Search、代码执行与文档检索等需要独立执行器；插件遇到不支持的声明会明确报错。MagPie 自身提供的搜索与工具编排仍由 MagPie 负责。

## 当前验证状态

| 检查 | 结果 |
| --- | --- |
| 本地 Node.js 单元与集成测试 | 70 项通过 |
| Bun 中的 MagPie 插件契约测试 | 10 项通过；包含在上述 70 项内 |
| CASA 真实工具两轮闭环 | Carbon / Hydrogen / Neon × 四种协议，12 项通过 |
| 同步缓存 A/B | 观察到尾部输入量从约 11.7 万降至 500 tokens |
| MagPie 实际安装、登录、客户端联调 | 待用户自行安装验证 |

真实工具测试通过插件入口与 `loader.fetch` 闭包直连上游，未完成 MagPie 安装后的全客户端验收。测试结论有版本与时间边界，详见 [实测与 A/B](./docs/实测与AB.md)。仓库不会自动安装、启用或启动 MagPie。

## 安装与登录

以下命令供你决定测试时手动执行。插件适配核对基线是 MagPie `0.1.4572`；较早版本的中间件、模型级协议或免密引用能力可能不同。

先在 MagPie 中保留已有 `casa` 供应商，确认它具有 Gemini 模型、Chat Completions 地址及名为 `gemini` 的密钥。插件从本地 `providers.json` 引用它，不要求把真实密钥再填一遍。

```sh
magpie plugin add github:alanbulan/magpie-casa-gemini-compat
magpie plugin login casa-gemini-compat
```

登录时选择“引用现有 CASA 配置”。支持 `keyless` 的宿主不要求填写密钥；旧宿主若仍显示输入框，升级至核对基线后再测。保存到 `plugin-auth.json` 的是 `local-reference:casa` 引用标识，真实厂商密钥仍在原 `providers.json` 中。

客户端继续选原来的 `casa/gemini-4-carbon`、`casa/gemini-4-hydrogen` 或 `casa/gemini-4-neon`。中间件按入口协议路由到插件内部模型，例如 `casa-gemini-compat/gemini-4-carbon__responses`，响应中恢复原模型名。原有路由组若先选择其他模型，不会因此自动变成 Gemini；验证时请直接选 CASA Gemini 模型。

从本地源码测试时，先安装项目依赖，再由你手动添加文件夹：

```sh
npm ci --ignore-scripts
npm test
# 需要安装到 MagPie 时才手动执行下一行
magpie plugin add /你的项目路径/magpie-casa-gemini-compat
```

关闭插件后，中间件不再重定向，客户端原来的 `casa/gemini-*` 名称可以继续使用。插件不会自动清理诊断目录或用户原供应商配置。移除可在 MagPie 的“插件 → 已安装”中完成。

## 配置与运行数据

默认配置位于 [config.json](./config.json)，严格结构见 [config.schema.json](./config.schema.json)。需要自定义时，将配置复制到插件目录之外，通过 MagPie `plugins.json` 中该插件条目的 `options.configPath` 指向它；不要改动其他插件条目。完整示例见 [配置说明](./docs/配置说明.md)。

默认是轨道 A（同步整包），每 8 秒发送 `: keep-alive` 注释。轨道 B 需要显式 `streamVerified:true`；当前 CASA 工具拆分故障下不建议启用。

默认读取 `$XDG_CONFIG_HOME/magpie/providers.json`，未设置时读取 `~/.config/magpie/providers.json`。日志位于该配置目录下的 `plugin-work/casa-gemini-compat/diagnostics.jsonl`；`options.runtimeDir` 可以改位置。日志记录哈希、计数、结构掩码与重试轨迹，不记录真实密钥、用户正文或工具参数。会话默认只在内存保存，重启后会重新冷校准；可选 `sessions.statePath` 会把会话正文写入权限受限的本地状态文件，需自行管理保留时间。

缓存 token 缺少原生计数时是**估算值**，用于维持客户端合理的上下文口径，不能替代供应商账单，也不能保证避免所有自动压缩。旁路请求识别是基于延续性和前缀锚点的启发式规则；相同前缀的无显式键会话可能共享槽位，建议客户端提供会话键。

## 开发与诊断

```sh
npm ci --ignore-scripts
npm test
npm run check:plugin
# 已有 Bun 时可核对宿主所用运行时，不需要启动 MagPie
bun test test/plugin.test.mjs
```

`npm run probe`、`npm run smoke`、`npm run ab` 会发起真实 CASA 请求，可能产生用量，均只使用合成测试内容。它们不会安装或启动 MagPie；具体参数、结果解读与客户端配置见 [文档](./docs/实测与AB.md)。另提供可选 [独立本地 HTTP 入口](./docs/客户端接入.md)，插件本身无需额外监听端口。

反馈问题时请附 MagPie 版本、模型名、客户端与入口协议、是否使用 SSE、脱敏错误码及诊断中的重试轨迹。不要上传 `providers.json`、`plugin-auth.json`、真实对话或密钥。维护者可以先用离线夹具复现。

## 许可与鸣谢

代码采用 [MIT 许可](./LICENSE)。CASA LOGO 为第三方厂商标识，不在本项目代码许可授权范围内。感谢 [MagPie](https://github.com/yetone/magpie) 的插件与中间件接口。
