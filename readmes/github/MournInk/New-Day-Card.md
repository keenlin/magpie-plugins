# 每日用量卡

`@magpie-community/page-day-card`，[magpie](https://usemagpie.ai) 的一个页面插件。

它把某一天（或一周、一个月、全部时间）的 AI 用量打印成一张热敏小票，放在你选的背景上。小票上有：

- 按 API 价格折算的总费用
- 连续使用天数、请求数、错误数，以及这一天和此前日均的对比
- 每个 Agent 和它用过的模型，带图标和费用，按费用排序（最多列 8 个 Agent，每个最多 5 个模型，其余的合并成「还有 N 个」）
- Token 总量、输入、输出和缓存命中率

小票可以复制成图片，或者存成 PNG。

## 安装

在 magpie 的 **插件 › 发现** 里安装，或者运行：

```sh
magpie plugin add @magpie-community/page-day-card
```

装好后，侧边栏的「插件」下面会多一个 **每日用量卡** 标签页，标题跟随窗口语言（Day card、每日用量卡、1日の使用量カード、Tageskarte），magpie 应用和 `magpie web` 里都有。

它需要支持 `usage:day` 读取和 `magpie.image` 的 magpie（版本见 `package.json` 的 `engines.magpie`）。太旧的 magpie 会拒绝这次读取，页面会提示更新。如果 magpie 早于周、月、全部区间（`usage:day` 不支持 `from`），页面只显示单日，切到其他区间时会提示更新。

## 用法

- **日 / 周 / 月 / 全部**：左上角切换统计区间。周是截至所选日期的 7 天，月是 30 天，全部是到今天为止每一个有调用的日子。周、月和全部的小票抬头写起止日期，统计里多了活跃天数和活跃日均费用。
- **选日期**：用箭头前后翻（周和月一次翻一整段），或者点日期打开菜单，里面列出每一个有调用的日子。今天的小票每分钟刷新一次。
- **背景**：小票下面垫什么。可以不要背景（只有纸，导出透明 PNG），也可以选 15 种图案（默认的青绿，以及野餐格、方格纸、牛皮纸、蓝晒、青海波、宣纸等）、4 张风景照片，或者自己的图片（选择、拖进页面或粘贴都行，角上的 × 可以删掉）。边距可调，照片还能调模糊和压暗。
- **纸张色调**：0 是刚出纸的冷白热敏纸，100 是在钱包里放过一个夏天的旧纸，纸越旧印字越淡一点。默认 25。
- **打印失真**：模拟热敏打印头的瑕疵：笔画边缘的锯齿和缺点、实心笔画里的掉点、偏弱或断线的打印头列、走纸时错开一个像素的点行、行间的浓淡起伏、欠热发浅的点，以及粗体字周围轻微的洇墨。0 是干净的打印，默认 50，100 是两倍。
- **模拟折痕**：小票像是折起来放进口袋又展开了，谷折和山折交替，长票三道，短票两道。每道折痕有一条细线，两侧有明暗过渡，印字上能看到涂层裂开的白纹，纸边也有小缺口。开关旁边的滑块调折痕深浅：0 是平的，默认 50，100 最深。关掉折痕时，这个滑块不可用。
- **复制图片 / 保存 PNG**：复制到剪贴板，或者存到「下载」文件夹。页面上的小票会缩放到工具栏下面一屏放得下的大小，复制和保存的是原尺寸（屏幕分辨率的 2 到 3 倍）。

同一天、同一区间的小票每次画出来都是同一张纸：纸纹、撕口、折痕和打印瑕疵都由日期决定。背景和纸张的设置保存在插件选项里。

## 读取了什么

只读取 `package.json` 里声明的唯一一项权限 `usage:day`（`magpie.permissions`），内容是某一天或某个区间的总量、每个 Agent 和模型的用量、此前有调用的日子，以及 Agent 和服务商的图标。其中没有密钥、账号、会话或提示词。

和所有插件页面一样，它运行在沙盒 frame 里，没有网络（`connect-src 'none'`），也不允许内联脚本。小票画在 canvas 上，浅色和深色主题下都一样，PNG 通过 `_magpie/bridge.js` 交给 magpie。在 `magpie web` 里，由浏览器自己复制或下载。

## 文件

- `page/index.html`、`page/style.css`：页面和工具栏。
- `page/view.js`：小票上的文字，有 English、简体中文、繁體中文、日本語和 Deutsch 五种语言，不依赖 DOM，测试直接加载它。
- `page/app.js`：通过 bridge 读取数据，画出纸张、印字、光照和折痕，复制和保存图片，并每分钟刷新今天的小票。
- `page/backgrounds.js`：背景的绘制，以及对保存下来的背景设置的校验。
- `page/bg/`：四张照片和它们的缩略图，以 JPEG 数据的形式放在脚本里，用到时才加载。frame 从 URL 加载的图片会污染 canvas，小票就没法复制和保存了。
- `page/fonts.js`：小票的字体，[Inconsolata](https://github.com/googlefonts/Inconsolata) SemiCondensed 的子集（400、700、800 三种字重），以 woff2 数据保存，许可见 `page/fonts.LICENSE.txt`。frame 不能下载字体文件，所以页面用 `FontFace` 加载这些字节。中文和日文用系统字体。
- `page/icons.js`：内置的 Amp 官方标志，说明见 `NOTICE`。

## 测试

```sh
bun test
```

两个界面测试用 Playwright 在 Chromium 和 WebKit 里驱动真实页面：

```sh
NODE_PATH=<playwright 所在的 node_modules> node day-menu.gui.mjs
```

```sh
NODE_PATH=<playwright 所在的 node_modules> node bg-delete.gui.mjs
```

## 许可

本项目以 [GNU Affero 通用公共许可证第 3 版](LICENSE)（AGPL-3.0-only）发布。

它源自 [magpie-community/plugins](https://github.com/magpie-community/plugins) 里以 MIT 许可发布的 day-card 插件。按 MIT 许可的要求，原作者的版权和许可声明保留在 `NOTICE` 里。字体 Inconsolata 采用 SIL Open Font License 1.1（`page/fonts.LICENSE.txt`）。Amp 标志属于它的所有者，不在本项目的许可范围内。

小票的版式参考了 [nowdex](https://nowdex.app) 的分享卡片。
