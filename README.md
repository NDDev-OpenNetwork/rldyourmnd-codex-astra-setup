# rldyourmnd-codex-astra-setup

A GPT-6-Astra posture for Codex, in one implementation that works for the CLI
and for the ChatGPT desktop app at the same time.

> **Scope so far: the autonomy posture and the context window.** Nothing else is
> configured yet, on purpose. Reasoning effort, capability features, web search
> and subagent policy are all deliberately absent until they are decided.

> **Status: written and locally verified, but no turn has completed.** The
> account's usage limit was exhausted throughout the measurement session, so
> every claim that needs a finished request is marked unverified where it
> appears.

## What is set

```toml
model = "gpt-6-astra"

approval_policy = "never"
sandbox_mode    = "danger-full-access"

model_context_window           = 872000
model_auto_compact_token_limit = 700000
```

That is the whole file. `codex doctor` reports `47 enabled · 0 overridden` for
it — identical to an empty config — which is the check that nothing extra crept
in.

## Full auto

`approval_policy = "never"` plus `sandbox_mode = "danger-full-access"` is exactly
what `--yolo` does. Verified by comparing session headers:

| Invocation | `approval:` | `sandbox:` |
| --- | --- | --- |
| no flags | `never` | `workspace-write [workdir, /tmp, $TMPDIR]` |
| `--yolo` | `never` | `danger-full-access` |
| `--dangerously-bypass-approvals-and-sandbox` | `never` | `danger-full-access` |
| this config, no flags | `never` | `danger-full-access` |

Two flags that guides still recommend do **not** exist at 0.155.1:

```
$ codex exec --full-auto …            error: unexpected argument '--full-auto' found
$ codex exec --skip-permissions …     error: unexpected argument '--skip-permissions'
```

`--full-auto` was removed; `--skip-permissions` is a different product's flag.

## Context: 872000 is the ceiling, and it is the "1M mode"

The 1.05M number quoted everywhere is the **API model's** window. What this
Codex build exposes is smaller, and the catalog says so directly:

```
context_window                    272000   ← the default, and the price cliff
max_context_window                872000   ← the ceiling
effective_context_window_percent      95   → 828400 usable
supports_experimental_context      false
```

872000 is not a compromise on the way to a million — it *is* what Astra's
long-context mode amounts to here. A default session reports roughly 258000
until `model_context_window` is set explicitly, and the `/extended-context`
toggle does not raise it
([oh-my-pi#10968](https://github.com/can1357/oh-my-pi/issues/10968)).

Writing `model_context_window = 1000000` is accepted by the parser, but nothing
indicates it exceeds the catalog ceiling, and the catalog is what the client
plans against. This repository sets the number the build actually offers.

**Raising the window is not free.** The default 272000 is exactly the
long-context surcharge threshold: above it, input bills at 2× and output at
1.5×. At 872000 the session spends nearly all of its life in that band. That is
a trade being made deliberately, not a side effect.

## The compaction key has a `model_` prefix

```toml
model_auto_compact_token_limit = 700000   # correct
auto_compact_token_limit       = 700000   # NOT a config key
```

The unprefixed name is a field of the model *catalog*, not of `config.toml`. An
ordinary run accepts it in silence and does nothing with it; only
`--strict-config` says so:

```
$ codex exec --strict-config -c auto_compact_token_limit=200000 …
Error loading config.toml: unknown configuration field `auto_compact_token_limit`
```

An earlier revision of this repository shipped the wrong one.

## Why a launcher and not just a config file

The desktop app **rewrites `~/.codex/config.toml` from scratch on every launch**,
discarding any key it did not write — measured twice, in
[docs/measurements.md](docs/measurements.md).

The same file is where the app registers browser use, through `[marketplaces.*]`
and `[plugins."chrome@openai-bundled"]` blocks whose paths must be absolute
(this build expands neither `~` nor `${HOME}`), so they cannot live in a
repository at all.

So a posture written into that file is erased at the next app launch, and a
posture that overwrites it erases browser use. The way out is the command line,
which nothing rewrites. Details in [docs/browser-use.md](docs/browser-use.md).

```sh
astra                        # full auto, 872000 window
astra --safe                 # same, but keep the workspace-write sandbox
astra exec "…"               # one-shot
ASTRA_CONTEXT=272000 astra   # stay under the price cliff for one run
```

## Requirements

- `@openai/codex` **0.155.0 or newer**
- for browser use: the ChatGPT desktop app, installed and launched at least once

## Known open item

With no `model_reasoning_effort` set — and it is deliberately not set, because
reasoning has not been decided — the session header reads
`reasoning effort: none`. Published guidance says `none` is the one effort Astra
**rejects**. That could not be confirmed here: the usage limit blocked every
request. If turns fail once quota returns, this is the first thing to look at.

## On trusting this repository

Where a number appears, `docs/measurements.md` names the command that produced
it and a control showing the instrument discriminates. That matters because the
obvious instrument lies: `codex doctor` reports `✓ config loaded · parse ok` for
a file containing nothing but an invented key. `--strict-config` is the one that
tells the truth, and it is what every key here was checked with.

Things measured here that contradict published documentation:

- Astra's usable ceiling is **872000**, not the 1.05M that is quoted everywhere
- `--full-auto` has been **removed**, though guides still recommend it
- the compaction config key is **`model_auto_compact_token_limit`**, not the
  unprefixed name that appears in write-ups
