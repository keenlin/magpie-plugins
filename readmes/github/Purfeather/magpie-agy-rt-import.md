# AGY 令牌导入 · Magpie 页面插件

在 Magpie 窗口里把 Antigravity Refresh Token 转成 `agy-cli` 的账号导入 JSON。

## 安装

```sh
magpie plugin add github:Purfeather/magpie-agy-rt-import
```

也可以先克隆仓库，再从仓库目录运行 `magpie plugin add .`。

## 使用

1. 在 Magpie 侧栏的插件分组打开 **AGY 令牌导入**。
2. 粘贴一个 Antigravity Refresh Token，点击 **复制导入 JSON**。如果窗口限制自动复制，页面会选中 JSON 供手动复制。
3. 打开 **供应商 → Antigravity（本地 agy CLI）→ 从文件导入账号**，粘贴 JSON 后导入。

此页面只做本地转换，无插件权限、后端或遥测，也不会保存令牌。复制内容包含完整 RT，导入后请清理剪贴板。账号验证和续期由已安装的 `agy-cli` 插件和本机 `agy` 完成。RT 须来自可被 Antigravity OAuth 客户端续期的有效授权。

## 兼容性

已知 `@magpie-community/opencode-agy-cli-auth` 从 `0.1.68` 起支持此导入格式。插件采用 Magpie 的静态页面接口，无依赖或构建步骤。

- [Magpie 页面插件文档](https://usemagpie.ai/docs/plugins#pages)
- [AGY-CLI 导入支持](https://github.com/magpie-community/plugins/commit/9907c187b212c55680acae4d46f5d2c868cda5f2)
