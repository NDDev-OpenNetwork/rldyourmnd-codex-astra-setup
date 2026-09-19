# Measurements

Everything in this repository is grounded in what this build actually reports,
not in what the documentation says. This file records the instrument, the
reading and the date, so a later reader can re-run the measurement instead of
trusting the number.

Measured on 2026-09-20 against:

| Component | Version |
| --- | --- |
| `@openai/codex` | 0.155.1 |
| ChatGPT desktop (`chatgpt`) | 26.915.31945 |
| Chrome extension `hehggadaopoacecdllhhajmbjkdcmajg` | 1.26.901.11451 |
| `codex-setup-system` | 0.0.72 |

## The model catalog is the source of truth, not the published docs

`codex debug models` on this account returns seven models. `gpt-6-astra` is
`priority 1` and `visibility: list` — it is not gated here, so the workaround in
[openai/codex#43342](https://github.com/openai/codex/issues/43342) (astra hidden
behind `visibility: hide`) does not apply.

```json
{ "slug": "gpt-6-astra", "display_name": "GPT-6-Astra",
  "default_reasoning_level": "low", "visibility": "list", "priority": 1,
  "shell_type": "unified_exec", "supported_in_api": true,
  "additional_speed_tiers": ["fast"] }
```

Its `supported_reasoning_levels` carry **six** efforts:

    low · medium · high · xhigh · max · ultra

Published write-ups describe five and omit `ultra`, whose catalog description is
*"Maximum reasoning with automatic task delegation"* — a different thing from
`max`, not a louder version of it. Where this repository and an article
disagree, the catalog wins.

`ultra` and `max` require **0.155.0 or newer**; on 0.154.0 they cannot be
expressed at all.

## `codex doctor` does not validate unknown config keys

This is the trap already documented in `codex-setup-system`'s `full-auto` setup,
and it still holds at 0.155.1. A config consisting only of

```toml
totally_invented_key_xyz = 1
```

reports `✓ config loaded · config.toml parse ok`. So "parse ok" is not evidence
that a key does anything.

**The instrument that does discriminate** is the key's presence in the compiled
binary's config field table:

```sh
strings .../vendor/x86_64-unknown-linux-musl/bin/codex > strings.txt
grep -c '<key>' strings.txt
```

Readings taken this way:

| Key | Occurrences | Real |
| --- | --- | --- |
| `model_provider` | 81 | yes |
| `service_tier` | 59 | yes |
| `auto_compact_token_limit` | 37 | yes |
| `model_reasoning_effort` | 27 | yes |
| `model_reasoning_summary` | 23 | yes |
| `model_verbosity` | 22 | yes |
| `plan_mode_reasoning_effort` | 19 | yes |
| `totally_invented_key_xyz` | 0 | **no** |

The control is the last row: the instrument returns zero for a key nobody
defined, so a non-zero reading means something.

## The effort value survives to the request

Config parsing accepts every effort string, including `none`, so parsing proves
nothing. What does prove something is the session header `codex exec` prints
before it calls the API:

```
model: gpt-6-astra
reasoning effort: ultra
```

Confirmed for `ultra`, `max` and `high`. With no `model_reasoning_effort` set at
all, the same header reads `reasoning effort: none` — and `none` is documented
as **rejected by Astra**. That is why every setup here sets the key explicitly
rather than leaving it to the default.

Server-side acceptance of each value is *not* yet verified: the account's usage
limit was exhausted at the time of measurement, so every request returned
`You've hit your usage limit` before a turn completed. This is the one open
reading in this file.

## Feature registry at 0.155.1

`codex features list` counts 142 specs, against 126 at the 0.151.0 pin that
`codex-setup-system`'s `full-auto` was measured on:

| Stage | Count |
| --- | --- |
| under development | 56 |
| stable | 42 |
| removed | 37 |
| experimental | 4 |
| deprecated | 3 |

Stable **and** shipped off — the ones a maximal posture has to name — are now
**four**, where `full-auto` documents three:

    memories · multi_agent_v2 · recommended_plugins · secret_auth_storage

`secret_auth_storage` is the new one. Any setup copying `full-auto`'s "exactly
three" sentence is out of date at this pin.

Relevant capabilities already stable and on: `browser_use`,
`browser_use_external`, `browser_use_full_cdp_access`, `computer_use`,
`fast_mode`, `hooks`, `goals`, `in_app_browser`.

## Astra's cost shape

From OpenAI's model page and the integration write-ups, not measured here:

- context 1.05M tokens (922K input max, 128K output max), cutoff 2026-04-30
- $10 / 1M input, $50 / 1M output; cached input $1/M
- **above 272K tokens in context: 2× input and 1.5× output** — a cliff, not a ramp
- Fast mode (`service_tier = "priority"`): 2× on top of whatever applies

`auto_compact_token_limit` is set below that 272K cliff in every setup here, so
a long session compacts rather than silently crossing into the premium band.

## Browser-use depends on the desktop app, not the CLI

Established by elimination:

- the CLI package ships no `extension-host` binary and no
  `com.openai.codexextension` / `NativeMessagingHosts` strings
- with no `config.toml`, `codex plugin marketplace list` reports
  *No plugin marketplaces in scope*, and `codex plugin add chrome@openai-bundled`
  fails with *plugin `chrome` was not found in marketplace `openai-bundled`*
- the desktop app, on launch, writes `~/.codex/config.toml` containing
  `[marketplaces.openai-bundled]` (a **local** source under
  `~/.codex/.tmp/bundled-marketplaces/`) and enables
  `chrome@openai-bundled`, `browser@openai-bundled` and
  `unified-computer-use@openai-bundled`
- it also registers the native-messaging manifest in eight browser profiles,
  pointing at a real executable under
  `~/.codex/plugins/cache/openai-bundled/chrome/<app-version>/`

Verified live: with Chrome running, `extension-host` appears as a **child
process of `chrome`**, which is the only proof that the bridge is actually
connected rather than merely configured.

[openai/codex#26820](https://github.com/openai/codex/issues/26820) reports the
CLI seeing `hasNativePipe: false` for this path; it is open and unresolved.

**The consequence for this repository:** `config.toml` is one of the five paths
`codex-setup-system` owns, so installing a setup over it removes the marketplace
and plugin blocks and breaks browser-use. See `docs/browser-use.md`.

## The app rewrites `config.toml`, but not `AGENTS.md`

Two separate measurements, and the difference between them is what the whole
design rests on.

**`config.toml` — rewritten on every launch.** Run twice, identically:

1. append a marker plus posture keys → 1918 bytes, marker present
2. stop the app → marker still present, so this is a *startup* behaviour, not a
   shutdown one
3. start the app and wait for it to come fully up → 1884 bytes, marker gone,
   and `diff` against the app's own generated file is empty

**`AGENTS.md` — untouched.** Same procedure, app confirmed fully up at 15
processes, `config.toml` rewritten to its 1884 bytes in the same pass:

```
AGENTS.md: 25 bytes, marker: 1
config.toml: 1884 bytes
```

The instructions surface survives. That is why this repository owns `AGENTS.md`
persistently and refuses to own `config.toml` at all: one file the app leaves
alone, one file it claims, and both are read by the CLI and the app alike.

A first attempt at this measurement was discarded: the app had not actually
started (the launch died with its parent shell), so the file was unchanged for
the wrong reason. A surviving marker only means something once the app is
confirmed up.

## The setups in this repository, measured

`codex doctor --all` against a throwaway `CODEX_HOME` holding only the setup's
`config.toml`:

| `config.toml` | feature flags |
| --- | --- |
| empty (control) | `47 enabled · 0 overridden` |
| `setups/astra` | `51 enabled · 1 overridden` |
| `setups/astra-ultra` | `51 enabled · 1 overridden` |

The enabled count moves by exactly four, which is the four features each setup
turns on, and the control shows the instrument responds to the file at all.

**One reading here does not add up, and is recorded rather than explained
away.** Each of the four features set *alone* reports `48 enabled · 1
overridden`, and the control in the other direction — `browser_use = false`,
a feature that ships on — reports `46 enabled · 1 overridden`. So a single
override counts as one. All four together should therefore report four, and it
reports one. `doctor` prints the `overrides` and `enabled flags` lists as
`<redacted>`, so the discrepancy could not be resolved from the outside.

Treat `enabled` as the load-bearing number and `overridden` as not understood.

(`doctor` also reports `✗ auth` in these probes: the throwaway home carries no
credentials. That is a property of the measurement, not of the setup.)
