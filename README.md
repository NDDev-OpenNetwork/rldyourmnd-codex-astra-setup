# rldyourmnd-codex-astra-setup

A GPT-6-Astra posture for Codex, in one implementation that works for the CLI
and for the ChatGPT desktop app at the same time.

> **Status: configuration and launcher are written and locally verified.**
> Server-side behaviour of each reasoning effort is **not** yet confirmed — the
> account's usage limit was exhausted throughout the measurement session, so no
> turn completed. Every claim in this repository that rests on a completed turn
> is marked as unverified where it appears.

## Why a launcher and not just a config file

The desktop app **rewrites `~/.codex/config.toml` from scratch on every launch**,
discarding any key it did not write. That is measured, twice, in
[docs/measurements.md](docs/measurements.md) — not inferred.

The same file is where the app registers browser use: its `[marketplaces.*]` and
`[plugins."chrome@openai-bundled"]` blocks are what put the native-messaging
bridge in place. Those blocks carry absolute paths, and this build expands
neither `~` nor `${HOME}` in a marketplace `source`, so they cannot be shipped in
a repository at all.

That is a genuine conflict, not a preference:

- a posture written into `config.toml` is erased the next time the app starts
- a posture that overwrites `config.toml` erases browser use

So the posture is carried on the command line, where nothing rewrites it, and
the app keeps sole ownership of the file. One entry point, both surfaces,
nothing fighting over the same bytes. The reasoning is in
[docs/browser-use.md](docs/browser-use.md).

## Using it

```sh
astra                      # interactive, the standard posture
astra --safe               # same, but keep the workspace-write sandbox
astra exec "…"             # one-shot, same posture
ASTRA_EFFORT=xhigh astra   # override the effort for a single run
ASTRA_COMPACT=200000 astra # override the compaction limit
```

Everything after the flags is passed through to `codex` untouched, so
`astra resume`, `astra review` and `astra --help` behave as you expect.

## The standard posture

`setups/standard` is the main one. Fully autonomous, unsandboxed, no subagents:

| | |
| --- | --- |
| `approval_policy` | `never` |
| `sandbox_mode` | `danger-full-access` |
| `model_reasoning_effort` | `max` |
| `model_context_window` | 872000 |
| `auto_compact_token_limit` | 800000 |
| `features.multi_agent` | `false` |

Three things here are not what an article would tell you, and each was measured:

**`--full-auto` no longer exists.** At 0.155.1 it is
`error: unexpected argument '--full-auto' found`, and `--skip-permissions` is
another product's flag. Only `--yolo` survives, and the config pair
`approval_policy = "never"` + `sandbox_mode = "danger-full-access"` reproduces it
exactly — same session header.

**`max`, not `ultra`.** The catalog defines `ultra` as *"Maximum reasoning with
automatic task delegation"* — it is the level that spawns sub-work. A
no-subagents posture cannot use it. `max` is the deepest non-delegating level.

**872000, not 1M.** The catalog reports `max_context_window = 872000` with
`effective_context_window_percent = 95`, so the usable ceiling is **828400**
tokens. The 1.05M figure quoted everywhere is the API model's window, not what
this build grants this account. 800000 compaction fits under 828400.

Raising the window is not free: the default `context_window` of 272000 is
exactly the surcharge threshold, above which input bills at 2× and output at
1.5×. This posture spends most of a session in that band deliberately.

`astra --safe` keeps the OS sandbox (`workspace-write`) and changes nothing else.

### Narrower variants

`setups/astra` (effort `high`, compaction 200K) and `setups/astra-ultra`
(effort `ultra`, compaction 160K, subagents left on) predate the standard and
are kept for the cases they describe. `astra-ultra` is the only posture here
that delegates.

## Requirements

- `@openai/codex` **0.155.0 or newer**. `max` and `ultra` cannot be expressed
  before it; on 0.154.0 the value simply does not exist.
- For browser use: the ChatGPT desktop app installed and launched at least once.
  The CLI cannot supply the native messaging host on its own.

## Layout

```
bin/astra                      the launcher — the single entry point
setups/standard/               the main posture: yolo, max, 872K window, no subagents
setups/astra/                  narrower: effort high, compaction 200K
setups/astra-ultra/            narrower: effort ultra — the one posture that delegates
docs/measurements.md           every reading, with the instrument and the date
docs/browser-use.md            why the app owns config.toml here
```

The files under `setups/` are the fallback floor for a machine with no desktop
app, where `config.toml` is stable because nothing else writes it. On a machine
with the app, the launcher is what applies.

## On trusting this repository

Five things here contradict published documentation or widely repeated advice.
In every case the measurement won:

- Astra has **six** reasoning efforts on this account, not five
- its context ceiling here is **872000**, not the 1.05M that is quoted everywhere
- `--full-auto` has been **removed**, though guides still recommend it
- the one subagent switch that matters is `multi_agent`, which ships **on**;
  `multi_agent_v2` is already off and disabling it changes nothing
- the 0.155.1 registry ships **four** stable-but-disabled features, not the three
  that `codex-setup-system`'s `full-auto` documents against an older pin

`codex doctor` reports `✓ config loaded · parse ok` for a config consisting
entirely of an invented key, so "it parses" is never offered here as evidence.
Where a number appears, `docs/measurements.md` names the instrument that
produced it and the control that shows the instrument discriminates.
