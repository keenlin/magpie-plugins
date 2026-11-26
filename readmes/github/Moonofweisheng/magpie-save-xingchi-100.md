# magpie-save-xingchi-100

[简体中文](README.zh-CN.md)

Automatically request Codex context compaction before Xingchi's 100-round limit, then resume ordinary requests on the primary model without manually switching models.

Version `0.1.0` is a candidate distributed as GitHub source. It has not been published to npm. The configurable implementation was originally prepared as `magpie-context-guard`; this repository and its package now use `magpie-save-xingchi-100`.

Magpie gateway middleware that triggers a client's native context compaction before a configured model reaches its request-round limit. It uses exact model IDs, so it can be enabled for one routing group while other providers and models continue unchanged.

The first supported profile is **Codex Responses**. The trigger works by adjusting the successful response's reported `usage.total_tokens`; Magpie's recorded provider usage remains unchanged, while Codex sees a value above its configured automatic-compaction threshold. This depends on Codex and Magpie preserving the relevant Responses usage fields. Other clients and protocols are not claimed as supported until separately tested.

The package is inert until `enabled` is true, at least one exact model ID is configured, and `contextWindowTokens` is positive. It does not change Magpie routing groups or global compaction settings. Set the normal model and the `compact=true` model in the Magpie routing group yourself. All IDs in one `models` list share the same policy.

## Configure

Install this GitHub repository from the Magpie Plugins page, or with:

```sh
magpie plugin add github:Moonofweisheng/magpie-save-xingchi-100
```

For local development, clone the repository and install its folder:

```sh
git clone https://github.com/Moonofweisheng/magpie-save-xingchi-100.git
cd magpie-save-xingchi-100
magpie plugin add "$PWD"
```

The plugin appears as **Save Xingchi 100**. In its Options, save an object like:

```json
{
  "enabled": true,
  "agents": ["codex"],
  "protocols": ["responses"],
  "models": ["group/your-capped-model"],
  "roundLimit": 90,
  "overflowAt": 95,
  "overflowModel": "other-provider/rescue-model",
  "contextWindowTokens": 256000,
  "compactionRatio": 0.95,
  "summaryPreamble": "Another language model started to solve this problem and produced a summary of its thinking process.",
  "compactionPrompt": "You are performing a CONTEXT CHECKPOINT COMPACTION."
}
```

Or set options from a terminal:

```sh
magpie plugin options github:Moonofweisheng/magpie-save-xingchi-100 '{"enabled":true,"agents":["codex"],"protocols":["responses"],"models":["group/your-capped-model"],"roundLimit":90,"overflowAt":95,"overflowModel":"other-provider/rescue-model","contextWindowTokens":256000,"compactionRatio":0.95}'
```

Use the installed spec shown by `magpie plugin list` for terminal options and removal commands; local folder installations use their folder path. Use `models` for the exact model identifier sent by the client, commonly a Magpie routing group ID. Do not put provider keys or private account data in options.

For Xingchi, [xingchi.routing-group.json](examples/xingchi.routing-group.json) shows an ordered primary/compaction group and [xingchi.options.json](examples/xingchi.options.json) scopes this plugin to `group/xingchi-gpt` only. Replace the example member IDs with the actual IDs in your Magpie. Select that group in Codex and use the current session's model for compaction so Magpie can apply the group's compact rule. Other providers and groups do not match this configuration. Disable the old round-guard plugin before enabling this replacement on the same model.

`roundLimit` is the request count after whose successful reply the middleware signals native compaction. `overflowAt` optionally sends late or pre-existing windows to `overflowModel` while waiting for compaction; it is clamped to at least `roundLimit`. Leave `overflowModel` empty if no rescue route is needed. The rescue model must be reachable by the same client and able to continue the conversation. The counter reserves one count for each incoming client request, including failed requests and tool continuations, and uses assistant/tool history as a conservative lower bound. Provider retries within a single Magpie request do not reserve separate counts. Parallel tool calls can cause the history correction to reach the threshold early. Keep a margin below the provider's actual cap.

Set `contextWindowTokens` to the effective context size and `compactionRatio` to the client's auto-compaction fraction. The middleware reports `floor(contextWindowTokens * compactionRatio) + 1` on the triggering successful response. Set the effective values for the configured model; incorrect values can trigger too early or fail to trigger before the upstream cap.

Compaction requests are excluded from counting. After native compaction creates a new context window, counting begins again for that window. The optional summary-prompt compatibility logic temporarily masks the known compaction prompt only inside a configured summary preamble while Magpie determines the route, then restores the exact text before the provider receives it. Actual compaction requests are never masked.

## State and privacy

The counter is stored at `$XDG_CONFIG_HOME/magpie/plugins/context-guard/state.json`, or `~/.config/magpie/plugins/context-guard/state.json` when `XDG_CONFIG_HOME` is unset. It contains SHA-256 window keys, request counts and timestamps only. Prompts, replies, account names, API keys and request bodies are not written. The file is capped at 512 windows.

To disable the behavior, set `enabled` to `false` or turn off the plugin. To remove a GitHub installation, use Magpie's Installed plugins page or `magpie plugin rm github:Moonofweisheng/magpie-save-xingchi-100`.

## Compatibility and limits

- Requires a Magpie version with gateway middleware and async request hooks.
- The counter hook and the middleware must both load. If the async hook is unavailable, history-based counting is only a fallback and is not persistent.
- The client must perform native compaction when it receives the configured usage signal.
- A successful model response is needed to deliver the signal. The optional rescue model covers windows that are already near their upstream cap when enabled.
- Test the selected client, model context size, routing group, tool continuations, retries and compaction recovery before relying on a new profile.
- No model provider, API key or organization-specific defaults are enabled by the published package.
- Local state writes are serialized inside one Bun host; multiple gateway processes sharing the same config folder have not been validated.
- This is a version-dependent Codex usage signal, not a general API for every client's compaction. Codex's cumulative token display can include the raised control value.

## Development

```sh
npm test
npm run check
npm run pack:check
```

An optional integration check uses an existing Magpie executable and Bun runtime:

```sh
MAGPIE_VERIFY_BINARY=/absolute/path/to/magpie \
MAGPIE_VERIFY_BUN=/absolute/path/to/bun \
npm run verify:magpie
```

It packs the package, installs the extracted files into an isolated HOME and XDG config, and sends requests to local mock providers. It verifies routing and usage signals; it does not run a real Codex session or real provider. Evidence is written to a sibling `magpie-save-xingchi-100-validation` folder, outside the npm package.

See [verification](docs/VERIFICATION.md) for observed results, [CHANGELOG.md](CHANGELOG.md) for package changes, and [publishing notes](docs/PUBLISHING.zh-CN.md) for release and listing options.
