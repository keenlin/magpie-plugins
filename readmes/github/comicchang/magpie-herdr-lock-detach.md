<h1 align="center">magpie-herdr-lock-detach</h1>

<p align="center">
  <strong>Cleanly detach local macOS GUI Herdr TUI clients on screen lock or sleep — zero LaunchAgents, zero polling, zero orphans.</strong><br />
  锁屏 / 休眠自动优雅断开本机 GUI 终端中的 Herdr 客户端 · 纯 Foundation + IOKit 事件驱动 · Magpie 原生插件托管
</p>

<p align="center">
  <a href="./LICENSE"><img src="https://img.shields.io/badge/license-MIT-34d399.svg" alt="MIT License" /></a>
  <img src="https://img.shields.io/badge/platform-macOS%20(no--op%20elsewhere)-38bdf8.svg" alt="macOS" />
  <img src="https://img.shields.io/badge/runtime-Node.js%20%E2%89%A520%20%7C%20Bun-6366f1.svg" alt="Node.js >= 20 or Bun" />
  <img src="https://img.shields.io/badge/dependencies-0-10b981.svg" alt="Zero Dependencies" />
  <img src="https://img.shields.io/badge/idle%20CPU-0.0%25-f59e0b.svg" alt="Zero Idle CPU" />
</p>

<p align="center">
  <a href="#why-this-exists">Why</a> ·
  <a href="#architecture--reload-safety">Architecture</a> ·
  <a href="#installation">Installation</a> ·
  <a href="#options">Options</a> ·
  <a href="#safety-invariants">Safety</a> ·
  <a href="#quickstart-3-minute-verification">Quickstart</a>
</p>

---

## Why This Exists

When you attach to a **Herdr** session from a second device (phone, laptop, SSH/Mosh) while your Mac's local GUI terminal still has an interactive `herdr` client attached, Herdr clamps viewport geometry to the smallest attached client — or leaves stale local clients attached when you lock your Mac and walk away.

**`magpie-herdr-lock-detach`** solves this cleanly without modifying Herdr's source code and without managing a separate macOS LaunchAgent:

- Runs as a **headless Magpie plugin** (no sidebar clutter, no open HTTP ports, no `auth` hook, zero network calls).
- Supervises a tiny native Swift watcher built purely on **Foundation + IOKit** (no `AppKit`, no WindowServer connection, ~2–3 MB RSS, **0.0% idle CPU**).
- On `com.apple.screenIsLocked` or `IOKit` `kIOMessageSystemWillSleep`, sends `SIGTERM` **only** to interactive local GUI `herdr` TUI clients. Herdr's client catches `SIGTERM`, sends `ClientMessage::Detach`, restores the terminal, and exits `0` — leaving `herdr server`, all running panes/agents, and any SSH/Mosh clients completely untouched.
- On Linux and Windows, loads as a silent no-op so shared `plugins.json` configs stay cross-platform.

---

## Architecture & Reload Safety

<p align="center">
  <img src="docs/assets/architecture.svg" alt="magpie-herdr-lock-detach architecture and reload-safe handover" width="100%" />
</p>

Magpie's Bun plugin host (`host.js`) can overlap old and new plugin hosts during hot-reload and may hard-kill (`SIGKILL`) a slow-exiting host after `stopWait` (2s). This plugin is explicitly engineered around every reload & shutdown race:

| Race / Failure Mode | Protection Mechanism |
|---|---|
| **Non-blocking Magpie startup** | `server()` returns `{}` immediately (<1ms); first-time `swiftc` compilation runs asynchronously with a `mkdir` build lock. |
| **Overlapping hot-reload hosts** | Single-instance `flock(LOCK_EX \| LOCK_NB)` on `watcher.lock`. A contender exits with code `75` (`EX_TEMPFAIL`); the supervisor treats `75` as a clean handover (retrying every `400ms`, not a crash). |
| **Graceful reload latency** | Hooks `process.stdin` `end`/`close` (fired immediately when Magpie closes host stdin), stopping the old watcher within milliseconds before Magpie's 2s timeout even fires. |
| **Hard `SIGKILL` of Magpie host** | Native watcher runs with `--watch-parent`: monitors **both** stdin pipe EOF (`DispatchSource.makeReadSource`) and parent exit via `kqueue` (`EVFILT_PROC NOTE_EXIT`). Kernel tears down pipe & fires `NOTE_EXIT` in <5ms — **zero orphan processes**. |
| **Concurrent upgrades / builds** | Content-addressed binary filename (`herdr-lock-detach-<arch>-<sha12>`) + atomic temp-file `rename()` + stale binary pruning. |
| **Sleep stall prevention** | Responds to `kIOMessageCanSystemSleep` immediately and calls `IOAllowPowerChange()` right after the ~20ms sweep on `kIOMessageSystemWillSleep`. |

---

## Installation

### Requirements

- **macOS** with Apple Command Line Tools (`swiftc`, installed via `xcode-select --install`)
- **Magpie** (plugin host with Node >= 20 / Bun)
- **Herdr** (optional at install time; watcher simply logs zero matches when no client is attached)

### Add to Magpie (`plugins.json`)

Install from a local path or GitHub repository spec:

```json
{
  "name": "Herdr Lock Detach",
  "spec": "/path/to/magpie-herdr-lock-detach",
  "options": {
    "enabled": true,
    "dryRun": false,
    "watchSleep": true,
    "debounceMs": 1500
  }
}
```

Or via Magpie CLI:

```bash
magpie plugin add github:<owner>/magpie-herdr-lock-detach
```

---

## Options

All options are optional and passed via the plugin entry's `"options"` object in `plugins.json`:

| Option | Type | Default | Description |
|---|---|---|---|
| `enabled` | `boolean` | `true` | Set `false` to disable without uninstalling the plugin. |
| `dryRun` | `boolean` | `false` | Log matched local GUI `herdr` clients on lock/sleep without sending `SIGTERM`. |
| `watchSleep` | `boolean` | `true` | Watch `IOKit` `kIOMessageSystemWillSleep` in addition to `com.apple.screenIsLocked`. |
| `debounceMs` | `number` | `1500` | Minimum milliseconds between sweeps (deduplicates simultaneous lock + sleep bursts). |
| `stateDir` | `string` | `~/.cache/magpie-herdr-lock-detach` | Directory for the content-addressed native binary and `watcher.lock`. |
| `handoverRetryMs` | `number` | `400` | Retry interval when taking over `watcher.lock` during Magpie hot-reload. |

---

## Safety Invariants

The native classifier strictly enforces four safety invariants before sending `SIGTERM`:

1. **Current UID only** — never inspects or signals other users' processes.
2. **Exact executable basename `herdr`** — ignores `herdr-app` (GUI wrapper) or unrelated binaries.
3. **Interactive TUI invocations only** — matches `herdr` (with global flags such as `--session`, `--socket`, `--config`, `--remote`, `--remote-keybindings`, `--machine`, `--host`, `--target`) and `herdr session attach ...`; strictly excludes `herdr server`, `remote-client-bridge`, `agent`, `pane`, `api`, `plugin`, `status`, `--version`, `--help`.
4. **Local GUI TTY only** — requires an attached TTY (not `??`/`?`) and rejects any process descending from `sshd`, `sshd-session`, `sshd-keygen-wrapper`, or `mosh-server`.

---

## Quickstart: 3-Minute Verification

```bash
# 1. Run full verification suite (lint, supervisor unit tests, sanitization guardrails,
#    macOS native compile + zero-AppKit check + flock handover + SIGKILL orphan test, docs check)
make check

# 2. Preview which local GUI Herdr clients would be detached right now (read-only)
make dry-run

# 3. Inspect standalone CLI status
node lib/cli.mjs status
```

---

## Repository Layout

```text
magpie-herdr-lock-detach/
├── index.mjs                    # Headless Magpie plugin entry (`id`, non-blocking `server`)
├── lib/
│   ├── supervisor.mjs           # Content-addressed builder, lock-busy handover, backoff & lifecycle
│   └── cli.mjs                  # Standalone diagnostic CLI (status, build, dry-run, run-once, self-test)
├── native/
│   └── herdr_lock_detach.swift  # Pure Foundation + IOKit watcher (zero AppKit)
├── docs/assets/
│   └── architecture.svg         # Architecture & reload-safety diagram
└── test/
    ├── supervisor.test.mjs      # Cross-platform reload, backoff, atomic build & non-darwin tests
    ├── native.test.mjs          # macOS compile, zero-AppKit, flock handover & SIGKILL orphan tests
    └── sanitization.test.mjs    # Open-source sanitization & binary allowlist guardrails
```

---

## License

MIT — see [LICENSE](./LICENSE). Contributions welcome via [CONTRIBUTING.md](./CONTRIBUTING.md).
