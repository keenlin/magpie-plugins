# APIConnect for Magpie

在 Magpie 里调用 APIConnect 模型、查看套餐用量或钱包余额。先安装插件，再按手里的 Key 选择登录方式。

```shell
magpie plugin add https://github.com/heyduanping/magpie-apiconnect
magpie plugin
magpie plugin login apiconnect
```

首次安装后看到 `on`、`0 models` 和 `not signed in` 属于正常状态。

| 我的情况 | 登录时怎么选 | 登录后会看到什么 |
| --- | --- | --- |
| 我只有充值 Key | 选「API Key」，粘贴 Key | 这把 Key 可见的模型。用量接口提供钱包余额或 Key 限额时，卡片会显示对应读数 |
| 我买了套餐 | 打开官网「我的订阅 → 查看可使用套餐的 Key」，复制对应分组的有效 Key，先选「API Key」 | Key 返回的套餐、周额度和本期额度。想查看官网账号下其他套餐和钱包时，选「官网账号与套餐 API Key」 |
| 我已经手动配过 APIConnect | 保留原供应商，按上面的命令安装并登录 | 插件供应商可能名为 `apiconnect-plugin`。后续选择模型、测试、查额度都用界面显示的实际供应商 ID |

启用官网二次验证的账号可以直接使用 API Key。登录时可输入线路名，例如 `daily` 或 `backup`。留空会根据 Key 自动生成名称。**使用已有线路名登录会替换该线路的凭据**，添加备用 Key 时请使用新名称。

## 第一次调用

1. 打开「供应商 → APIConnect」，点击「重新获取模型」，勾选一个模型并点「保存」。将鼠标移到模型按钮上可看到确切 ID。有线路名的选项会标出线路和登录时的套餐。记下实际的供应商 ID 和模型 ID。
2. 运行 `magpie provider test <供应商ID> <模型ID>`。将两个占位符替换为上一步的值。这个操作会发送真实模型请求，可能产生费用。看到成功结果后，在 Agent 中选择 `<供应商ID>/<模型ID>`。
3. 打开「观测 → 用量 → 概览」，切换到「剩余」。也可运行 `magpie quota refresh <供应商ID>`。模型列表和额度来自两个独立接口，额度读取失败时仍可测试已列出的模型。

下图用演示账号选择了 `daily` 的 Haiku。右键点击单个模型也可以测试它。窗口底部的「测试模型」会测试多项，按次计费时留意所选范围。

![Magpie 中勾选 daily 线路的 Haiku 并保存](docs/images/models.png)

## 管理多把 Key

- **添加、切换和轮换。** 在供应商窗口点「添加另一个 APIConnect 账号」，或再次运行登录命令。用不同名称添加第二把 Key。选择带线路名的模型，即可指定使用哪把 Key。只选普通模型名时，由 Magpie 按供应商的账号路由设置选择。要轮换失效 Key，输入新 Key 和原线路名，再刷新模型。
- **自动命名。** 留空时使用 `Key-` 加短摘要标识这把 Key，模型列表保留普通模型名。需要固定这条线路时，可在 Magpie 路由组按账号选择模型。换一把 Key 会生成新的自动名称；轮换时可以手工填写旧名称。
- **从旧版升级。** 更新后原绑定和普通模型 ID 继续可用，读取官网额度时会重新核对分组。若想给旧绑定建立具名线路，先用新名称登录并测试，确认后可在 Magpie 的账号设置中移除旧绑定。插件更新可在「插件 → 已安装」操作，命令 `magpie plugin update` 会检查全部已安装插件。

## 读懂额度

- 每日、每周和本期窗口可能记录同一笔消费，各自对照上限看剩余。同一笔用量不要重复相加。
- 官网账号模式会核对当前 Key 的分组。标有「其他订阅」的窗口仅供查看。暂缺的窗口会标出原因，已取得的额度继续显示。
- 钱包显示 APIConnect 返回的美元余额。在 Magpie 中可使用 `magpie quota alert --balance 5` 开启余额提醒，此设置作用于 Magpie 支持的余额卡。套餐额度耗尽后，后续请求可能继续消耗钱包余额。本地提醒不拦截上游扣费，购买前请核对当前套餐的结算说明。[APIConnect 充值与订阅](https://apiconnect.lol/purchase)
- 有明确重置时间的窗口显示下次重置。当前分组只有一个有效套餐时会显示其到期时间。

下图中的两把 Key 共用 Demo Pro。两张卡每周都剩 $80，整个套餐也只剩 $80。APIConnect 返回订阅记录 ID 时，同一订阅会显示相同的短标识；只返回套餐名称时，请到官网核对是否共用同一份订阅。钱包按账号或 Key 可查询到的读数显示。

![两把 Key 共享周额度，本期额度与钱包余额单独显示](docs/images/quota.png)

## 排查问题

| 现象 | 怎么处理 |
| --- | --- |
| 登录后模型列表为空 | 重新获取模型，核对 Key 在官网所属的分组以及该分组可用的模型 |
| 模型测试提示 401、403 或模型不存在 | 核对正在使用的线路和 Key，重新获取模型，再测试列表中的确切 ID |
| 提示 429 或网络连接失败 | 限流后稍候重试。连接失败时检查 Magpie 的供应商代理和本机网络 |
| 模型可用，额度显示暂不可查 | 点击额度卡刷新，或稍后运行 `magpie quota refresh <供应商ID>`。仅返回 Key 限额时，卡片会标为「Key 额度」 |
| 官网会话过期 | 用「官网账号与套餐 API Key」重新登录，或选择只需要 Key 的登录方式 |
| 插件显示已登录，供应商和用量列表却为空 | 检查 Magpie「设置 → 隐私」中的订阅账号设置，以及启动环境是否设置了 `MAGPIE_NO_ACCOUNTS=1` |

提交 issue 时可复制用量卡中的「接入诊断」，另附 Magpie 版本、操作步骤和 HTTP 状态码。诊断文本只含插件版本、登录方式、已知模型数量和额度读取状态。请勿附完整命令输出、API Key、邮箱、密码或会话。

## 凭据和协议

只有官网账号模式会将邮箱和密码提交给 `https://apiconnect.lol/api/v1/auth/login`。密码只用于登录，Magpie 在本机保存模型 Key 和可续期的会话。模型列表来自 `GET /v1/models`。插件按该 Key 的模型列表中声明的协议启用 Messages、token 计数或 Responses，模型请求均发往 `https://apiconnect.lol/v1`。旧列表缺少协议字段时，对 Claude 模型沿用 Messages。插件不依赖运行时 npm 包。

## 版本验证

v0.4.0 已在 Windows 的 Magpie 0.1.4479 上验证双线路登录、同名换 Key、共享额度显示和模型测试。README 截图使用隔离的演示数据。线上验证另外使用一条已授权线路，读取了真实模型列表、订阅和钱包，并对 `claude-haiku-4-5` 发出一次 Responses 流式请求，收到文本和完成事件。其他模型及套餐的可用性以各自的 Key 为准。

开发者可运行 `npm test`。宿主集成测试使用 `node scripts/host-smoke.mjs <Magpie的Bun路径> <Magpie的host.js路径>`，测试凭据和上游响应都来自离线样例。

[APIConnect](https://apiconnect.lol) · [Magpie 插件文档](https://usemagpie.ai/docs/zh/plugins) · [MIT 许可证](LICENSE)
