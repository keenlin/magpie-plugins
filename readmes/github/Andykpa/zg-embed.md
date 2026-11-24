# opencode-zg-embed-auth · 百炼 zg Embedding

一个 [magpie](https://usemagpie.ai) / OpenCode provider 插件：让 [zvec-grep（zg）](https://www.npmjs.com/package/@zvec/zvec-grep) 通过 magpie 网关直接使用阿里云百炼（DashScope）的 embedding 模型，**改名在 magpie 内部完成，不需要外部代理**。

## 为什么需要它

zg 的远程 embedding 模型名是写死的：它只会发送 `qwen3.7-text-embedding`。而百炼上性价比更高的模型是 `qwen3.7-text-embedding-flash`——两者**向量不兼容**（余弦相似度约 0.01–0.06），价格差 4 倍（base ¥0.5/M vs flash ¥0.125/M）。zg 的 CLI、配置文件、授权三层都拒绝 flash 名，所以必须有一层改名。

本插件把名为 `qwen3.7-text-embedding` 的模型的 **API id 声明为 flash**，magpie 网关自动按 API id 发给上游。zg 照常发裸名，上游实际跑 flash，用量记账也在 magpie 里。

## 模型桥接

| zg 发送的名（写死） | 默认上游 API id | 默认维度 | 单行上限 | 批量上限 |
| --- | --- | --- | --- | --- |
| `qwen3.7-text-embedding` | `qwen3.7-text-embedding-flash` | 1024 | 128,000 token | 20 行 |
| `text-embedding-v4` | `text-embedding-v4` | 1024 | 8,192 token | 10 行 |
| `qwen3-vl-embedding` | `qwen3-vl-embedding` | 2560 | 32,000 token | 多模态 |

- 前两个走 OpenAI 兼容接口（`/compatible-mode/v1/embeddings`）。
- `qwen3-vl-embedding` 走 DashScope **原生多模态接口**（请求/响应形状都不同）：zg 发来的原生请求体原样转发；其他 agent 发来的 OpenAI 形状请求由插件双向翻译。
- 上游名可用 options 覆盖（见下）。

## 安装

```sh
magpie plugin add github:Andykpa/zg-embed
```

也可以装本地文件夹：`magpie plugin add ./zg-embed`。首次添加插件时 magpie 会自动下载 Bun。在 OpenCode 里同样可用（OpenCode v1 provider 插件形状）。

## 签名

```sh
magpie plugin login zg-bailian-embed
```

或在 magpie 应用里：Plugins → 该供应商行 → 签名。会问两件事：

1. **端点**：
   - *Public DashScope / 公共百炼*：`https://dashscope.aliyuncs.com/compatible-mode/v1`，任何百炼 key 可用；
   - *Custom workspace / 自定义业务空间*：形如 `https://<workspace>.<region>.maas.aliyuncs.com/compatible-mode/v1`，配该业务空间的专属 key。
2. **百炼 API key**（`sk-…`）。

签名保存在 `~/.config/magpie/plugin-auth.json`（与 OpenCode 的 `auth.json` 同形，mode 600）。换 key：重新 login 即可。

## zg 侧配置（每台机器）

`~/.zvec-grep/config.json`：

```json
{
  "version": 1,
  "defaults": { "embedding": "qwen/qwen3.7-text-embedding" },
  "providers": { "qwen": { "apiKey": "magpie" } },
  "models": {
    "qwen/qwen3.7-text-embedding": {
      "endpoint": "http://127.0.0.1:3425/v1/embeddings"
    }
  }
}
```

- endpoint 指向你那台 magpie 网关（默认端口 3425）。回环地址不校验入站 key；跨机器访问时用网关 key。
- 每个已索引工作区还要授权，**endpoint 必须与上面一致**：

```sh
zg --auth grant --capability embedding --scope workspace \
  --embedding qwen/qwen3.7-text-embedding \
  --endpoint http://127.0.0.1:3425/v1/embeddings
```

## 三个坑（实测得出，务必读）

1. **同一上游不要再挂一个 key 供应商，且其模型目录里含 `qwen3.7-text-embedding`。**
   magpie 的 embeddings 通道按上游目录匹配裸名，那个 key 供应商会抢先截走裸名、按裸名发上游——结果跑的是 base：向量与 flash 索引不兼容，价格 4 倍。禁用模型、改供应商显示名都拦不住。
   *症状*：路由日志里该请求的 provider 不是 `zg-bailian-embed`。*修复*：移除那个 key 供应商（本插件就是它的替代品）。
2. **zg 的授权 grant 是追加，不是替换。** 换 endpoint 时先 `zg --auth revoke` 再 grant，否则旧条目排在前面被优先使用。
3. **endpoint 还存在索引的 `manifest.json` 里**（`embeddingConfigs`）。换 endpoint 后旧索引要 `zg --index --rebuild`，否则检索仍按旧 endpoint 校验授权。

## Options（可选）

```sh
magpie plugin options zg-embed '{"upstream":{"qwen3.7-text-embedding":"qwen3.7-text-embedding"}}'
magpie plugin options zg-embed off   # 恢复默认
```

（`zg-embed` 换成 `magpie plugin` 列表里显示的名字。）

- `upstream`：覆盖各 zg 名对应的上游模型名。上例让裸名跑 base（默认是 flash）。
- `endpoint`：覆盖所有账号的 base URL，优先于签名时选的端点。

## 用量显示说明

- 用量详情里的「发送模型」记的是**请求名**（裸名），不是实际发送名；实际发送名以 API id 为准（可用向量比对验证）。
- 费用默认按请求名查公开价：裸名 `qwen3.7-text-embedding` 在 models.dev 上对应 base 价，会高估。本插件已为桥接模型声明 flash 的公共价格；若你的业务空间计价不同，用下面命令修正显示（百炼账单按实际跑的模型计，不受影响）：

```sh
magpie model price zg-bailian-embed/qwen3.7-text-embedding 0.018682,0,0,0
```

## 验证它真的跑了 flash

对网关发裸名请求，与直连百炼 flash 的向量比余弦，应 ≈ 1（base 的话约 0.05）：

```sh
curl -s http://127.0.0.1:3425/v1/embeddings \
  -H 'content-type: application/json' -H 'authorization: Bearer magpie' \
  -d '{"model":"qwen3.7-text-embedding","input":"你好世界"}'
```

## 开发

```sh
node --test test/helpers.test.mjs   # 纯函数单元测试
```

沙箱测试（官方推荐，绝不碰生产 magpie）：独立 `HOME`/`XDG_CONFIG_HOME`/`XDG_CACHE_HOME` + `MAGPIE_ADDR=127.0.0.1:3499`，然后 `magpie web --addr 127.0.0.1:3499 --gateway --no-open`、`magpie plugin add ./zg-embed`、`magpie plugin login zg-bailian-embed`。详见 [magpie 插件文档](https://usemagpie.ai/docs/plugins)。

## 许可证

MIT

---

# English

A [magpie](https://usemagpie.ai) / OpenCode provider plugin that lets [zvec-grep (zg)](https://www.npmjs.com/package/@zvec/zvec-grep) use Alibaba Bailian (DashScope) embedding models through a magpie gateway — **the model rename happens inside magpie, no external proxy**.

**Why.** zg hardcodes the remote embedding name `qwen3.7-text-embedding`, but the cost-effective Bailian model is `qwen3.7-text-embedding-flash`. Their vectors are incompatible (cosine ≈ 0.01–0.06) and prices differ 4×. This plugin declares the model's **API id** as flash; the gateway sends flash upstream while zg keeps sending the bare name.

**Bridges.** `qwen3.7-text-embedding` → `qwen3.7-text-embedding-flash` (1024-d, 128K); `text-embedding-v4` → itself (1024-d, 8K); `qwen3-vl-embedding` → itself via DashScope's native multimodal API (2560-d; native bodies pass through, OpenAI-shaped bodies are translated both ways). Upstream ids are overridable via options.

**Install & sign in.**

```sh
magpie plugin add github:Andykpa/zg-embed
magpie plugin login zg-bailian-embed   # pick public DashScope or a custom workspace URL, then the sk- key
```

**zg config.** Point `~/.zvec-grep/config.json` at your gateway (`models."qwen/qwen3.7-text-embedding".endpoint = http://127.0.0.1:3425/v1/embeddings`, `providers.qwen.apiKey`), then per workspace: `zg --auth grant --capability embedding --scope workspace --embedding qwen/qwen3.7-text-embedding --endpoint <same URL>`.

**Gotchas (field-tested).**
1. Do **not** keep another key provider on the same upstream whose catalog contains the bare name — magpie's embeddings channel resolves bare names against upstream catalogs first, so it would intercept the request and run base. Remove it; this plugin replaces it.
2. zg auth grants are **appended**, not replaced: `zg --auth revoke` before re-granting with a new endpoint.
3. The endpoint is also stored in each index's `manifest.json`; after changing it, `zg --index --rebuild` old indexes.

**Options.** `magpie plugin options <name> '{"upstream":{…},"endpoint":"…"}'` overrides upstream ids / base URL; `off` resets.

**Usage display.** The "sent model" column logs the request name, not the wire name; costs are looked up by request name (the bare name maps to base pricing on models.dev). The plugin declares flash's public price; adjust with `magpie model price zg-bailian-embed/qwen3.7-text-embedding <in>,<out>,<cacheRead>,<cacheWrite>` if your workspace prices differ. Bailian bills the model actually run.

MIT licensed. See the Chinese section above for full details.