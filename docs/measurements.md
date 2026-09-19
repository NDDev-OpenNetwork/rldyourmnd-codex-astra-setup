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

## Autonomy: which flag still exists at 0.155.1

Articles about "Codex YOLO mode" disagree with this build, so the flags were run
rather than read about. Session headers, `codex exec`:

| Invocation | `approval:` | `sandbox:` |
| --- | --- | --- |
| no flags | `never` | `workspace-write [workdir, /tmp, $TMPDIR]` |
| `--yolo` | `never` | `danger-full-access` |
| `--dangerously-bypass-approvals-and-sandbox` | `never` | `danger-full-access` |
| `approval_policy`/`sandbox_mode` in config, no flags | `never` | `danger-full-access` |

And the ones that do **not** exist:

```
$ codex exec --full-auto …
error: unexpected argument '--full-auto' found

$ codex exec --skip-permissions …
error: unexpected argument '--skip-permissions'

$ codex exec --dangerously-skip-permissions …
error: unexpected argument '--dangerously-skip-permissions'
```

`--full-auto` has been removed at this pin; the `--skip-permissions` spellings
belong to a different product entirely. Only `--yolo` and its long form survive,
and the config pair reproduces them exactly — which is what lets the posture live
in `bin/astra` instead of depending on a flag.

(`--yolo` does not appear in a `strings` dump of the binary, yet demonstrably
works. Another reason the string table is only ever used here as a *positive*
test for a key's existence, never as proof of absence.)

## Astra's real context window on this account

The catalog contradicts the widely quoted figure:

```
context_window                    272000
max_context_window                872000
effective_context_window_percent      95
supports_experimental_context      false
```

So the usable ceiling is 872000 × 95% = **828400 tokens**. The 1.05M number that
appears in write-ups is the API model's window, not what this Codex build grants
this account — asking for a 1M window here cannot be satisfied.

Note also that the *default* `context_window` of 272000 is exactly the
long-context surcharge threshold. Raising the window past it is a decision to
spend at 2× input and 1.5× output, not a free upgrade.

## Subagents: `multi_agent_v2` is not the one that matters

```
multi_agent       stable  true      ← ships ON
multi_agent_v2    stable  false     ← already off
multi_agent_mode  removed false
enable_fanout     removed false
```

Turning off `multi_agent_v2` alone changes nothing, because it is off already.
`multi_agent` is the one that ships enabled, and the model entry carries
`multi_agent_version = "v2"` and `multi_agent_reasoning_effort = "xhigh"`, so
the capability is live and would be used.

Measured: `[features] multi_agent = false` moves the count from
`47 enabled · 0 overridden` to `46 enabled · 1 overridden`.

**`ultra` also delegates.** The catalog describes it as "Maximum reasoning with
automatic task delegation". A no-subagents posture therefore cannot use `ultra`;
`max` is the deepest level that does not delegate.

## The `standard` setup, measured

| `config.toml` | feature flags |
| --- | --- |
| empty (control) | `47 enabled · 0 overridden` |
| `setups/standard` | `49 enabled · 1 overridden` |

47 + 3 enabled (`memories`, `recommended_plugins`, `secret_auth_storage`) − 1
disabled (`multi_agent`) = 49, and `multi_agent_v2` accounts for no change
because it was already off.

Session header under that config, and identically through `bin/astra`:

```
model: gpt-6-astra
approval: never
sandbox: danger-full-access
reasoning effort: max
reasoning summaries: detailed
```

With `bin/astra --safe`, only the sandbox changes:
`approval: never · sandbox: workspace-write`. In both cases
`codex plugin list` still reports 13 plugins `installed, enabled`, so the app's
browser-use registration is untouched.

## `--strict-config` is the instrument. Everything else lies.

This supersedes the `strings` method described above, which was good enough to
confirm a key exists and **not** good enough to catch a key that exists in the
*wrong struct*. It let a broken key ship.

`codex exec --strict-config -c <key>=<value>` rejects any field the config
schema does not define:

```
$ codex exec --strict-config -c totally_invented_key_xyz=1 …
Error loading config.toml: unknown configuration field `totally_invented_key_xyz`

$ codex exec --strict-config -c auto_compact_token_limit=200000 …
Error loading config.toml: unknown configuration field `auto_compact_token_limit`
```

**The second one is the correction.** `auto_compact_token_limit` is a field of
the model *catalog* struct (`context_window · max_context_window ·
auto_compact_token_limit · comp_hash · …`), not of `config.toml`, whose struct
reads `model_provider · model_context_window · model_auto_compact_token_limit ·
model_auto_compact_token_limit_scope · approval_policy · …`.

An earlier revision of this repository set the unprefixed name in all three
setups and in the launcher. It parsed, `codex doctor` said `parse ok`, and it
did nothing at all — exactly the failure mode this file warns about two sections
earlier, reproduced by the author of the warning.

Every key now in the repository, re-checked this way:

| Key | `--strict-config` |
| --- | --- |
| `model` | OK |
| `approval_policy` | OK |
| `sandbox_mode` | OK |
| `model_context_window` | OK |
| `model_auto_compact_token_limit` | OK |
| `auto_compact_token_limit` | **unknown configuration field** |

`model_auto_compact_token_limit_scope` also exists, and takes `total` or
`body_after_prefix` — `"session"` is rejected with
`unknown variant \`session\``, which is a second demonstration that the
instrument discriminates on values and not only on names.

## How much context can actually be given

The question was whether 1M is reachable. It is not, and 872000 is not a
shortfall — it is what Astra's long-context mode *is* in this client.

| Number | What it is |
| --- | --- |
| 1050000 | the API model's advertised window |
| 872000 | `max_context_window` in this build's catalog, for every model except `gpt-5.5` |
| 828400 | 872000 × 95% `effective_context_window_percent` — the usable budget |
| ~258000 | what a default session reports before `model_context_window` is set |
| 272000 | the catalog default, and exactly the long-context surcharge threshold |

`supports_experimental_context` is `false` for `gpt-6-astra`, the
`context_management` and `token_budget` features are both `under development`
and off, and `/extended-context` is reported not to raise Astra's budget
([oh-my-pi#10968](https://github.com/can1357/oh-my-pi/issues/10968)). There is
no switch found in this build that goes above 872000.

`model_context_window = 1000000` passes `--strict-config`, because the schema
accepts any integer. That is not evidence it takes effect, and the catalog
ceiling is what the client plans against. Whether a larger value is clamped
could not be measured: reading the effective budget needs `/status` inside an
interactive session, and the usage limit blocked every turn.

**The settled values:** window 872000, compaction 700000. That leaves 128400
tokens of headroom under the 828400 ceiling, which is room for a long turn to
finish rather than being compacted in flight.

## Subagents: the flag applies, the tool effect is unproven

Switches that exist at 0.155.1:

```
multi_agent       stable   true     ← ships ON, the one that matters
multi_agent_v2    stable   false    ← already off
multi_agent_mode  removed  false
enable_fanout     removed  false
```

Measured, with an empty config as the control:

| `config.toml` | feature flags |
| --- | --- |
| empty | `47 enabled · 0 overridden` |
| `multi_agent = false` | `46 enabled · 1 overridden` |
| `multi_agent = false` + `multi_agent_v2 = false` | `46 enabled · 1 overridden` |

The second and third rows being identical is the point: `multi_agent_v2` is
already off, so setting it changes nothing. Turning off only `v2` — the more
modern-looking name — would have looked like a no-subagents posture and done
nothing at all.

**The part that is not established.** Whether the flag actually withholds the
`spawn_agent` tool from the request could not be shown from here.
`codex debug prompt-input` returns only the message list — there is no `tools`,
`functions` or `tool_choice` key anywhere in its output — and its text still
describes `spawn_agent`, `followup_task`, `send_message`, `wait_agent` and
`interrupt_agent` with the feature off. Diffing the rendered prompt with the
feature on and off gives identical text apart from ids, paths and timestamps.

So the prompt is evidence for neither conclusion, and the question needs a
completed turn. The usage limit blocked every one.

The collaboration text does not come from the catalog either:
`base_instructions` for `gpt-6-astra` is 21420 characters and contains no
`spawn_agent`.

### `include_collaboration_mode_instructions` is a different thing

It was tried and rejected for this purpose. Setting it to `false` shortens the
prompt by about 1044 bytes and drops mentions of "collaboration" from 7 to 2,
but every multi-agent tool name survives unchanged:

| Term | `multi_agent=false` | + `include_collaboration_mode_instructions=false` |
| --- | --- | --- |
| `spawn_agent` | 3 | 3 |
| `followup_task` | 2 | 2 |
| `send_message` | 3 | 3 |
| `wait_agent` | 2 | 2 |
| `interrupt_agent` | 1 | 1 |
| `collaboration` | 7 | 2 |

It governs collaboration *modes*, not multi-agent tooling, so it is not in the
setup.

(A first attempt at this count used `grep -c`, which counts matching *lines* —
and the rendered prompt is a single line of JSON, so every term reported `1`
whether it appeared once or thirty times. The numbers above come from
`grep -o | wc -l`.)
