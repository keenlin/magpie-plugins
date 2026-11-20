# magpie-jev-utc

A [magpie](https://github.com/yetone/magpie) plugin that adds two decision
models which behave exactly like the built-in Jev, except that every System
One decision carries the current UTC date and time:

- `jev/jev-latest-date-utc`
- `jev/jev-preview-date-utc`

Pick one as a routing group's classifier (or call `/v1/systemone` with it) and
the decision is made by magpie's own built-in Jev, with the UTC time injected.
No extra API key, no sign-in step, no settings page.

## What it injects

For each decision request it adds two state fields and one line to every
question's instructions, so the model both carries the time and is told to
read it:

```json
{
  "state": { "message": "...", "utc_now": "2026-11-20T08:00:00.000Z", "utc_date": "2026-11-20" },
  "questions": { "intent": { "instructions": "...?\nCurrent UTC time: 2026-11-20T08:00:00.000Z." } }
}
```

## How it works

- The plugin registers a decide provider `jev` with the two models above.
  `typesafe` is magpie's built-in Jev preset id; a plugin cannot own it or add
  models to it, so the wrapped models live under `jev`.
- On load it signs itself in with a local placeholder key, because magpie only
  materialises a plugin provider once it has an account. The placeholder is
  never sent anywhere.
- Its `/systemone` fetch injects the UTC fields, then forwards the request to
  magpie's own gateway on loopback (`MAGPIE_ADDR`) with the model rewritten to
  `typesafe/jev-latest` or `typesafe/jev-preview`. Loopback requests need no
  key, so the gateway uses the built-in Jev provider's stored key. Your real
  Jev key stays where it was.

Because the forward names the built-in model explicitly, the wrapper never
routes back into itself.

## Install

From GitHub (recommended):

```sh
magpie plugin add github:4fuu/magpie-jev-utc
```

The full repository URL works too:

```sh
magpie plugin add https://github.com/4fuu/magpie-jev-utc
```

For a local checkout (development):

```sh
magpie plugin add /path/to/magpie-jev-utc
```

Installing is the whole setup: the plugin signs itself in with a local
placeholder, so there is no key to paste and no sign-in step. Then choose
`jev/jev-latest-date-utc` (or the preview one) as the decision model of a
routing group. Nothing else to configure.

To update later, run `magpie plugin update`; to remove it,
`magpie plugin rm magpie-jev-utc`.

## Development

```sh
bun test
```

## Limits

- Only `/v1/systemone` is served; the models are decision-only, not chat.
- The time is UTC, read from the machine running magpie at decision time.
- If magpie's gateway is not on loopback or `MAGPIE_ADDR` points elsewhere,
  the forward follows that address.
