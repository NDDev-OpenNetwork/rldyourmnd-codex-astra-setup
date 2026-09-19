# Measurements

Everything in this repository is grounded in what this build actually reports,
not in what the documentation says. This file records the instrument, the
reading and the control, so a later reader can re-run a measurement instead of
trusting a number.

Measured on 2026-09-20 against:

| Component | Version |
| --- | --- |
| `@openai/codex` | 0.155.1 |
| ChatGPT desktop (`chatgpt`) | 26.915.31945 |
| Chrome extension `hehggadaopoacecdllhhajmbjkdcmajg` | 1.26.901.11451 |
| `codex-setup-system` | 0.0.72 |

**One thing is unverified throughout.** The account's usage limit was exhausted
for the whole session, so no turn ever completed. Every reading here comes from
config loading, session headers, catalog dumps or prompt rendering — none from a
finished request. Where that matters, it is said so in place.

---

# Part 1 — The instrument

## `--strict-config` is the only validator

```sh
codex exec --strict-config -c '<key>=<value>' --skip-git-repo-check --ephemeral x
```

It rejects fields the config schema does not define, and variants a field does
not accept:

```
$ codex exec --strict-config -c totally_invented_key_xyz=1 …
Error loading config.toml: unknown configuration field `totally_invented_key_xyz`

$ codex exec --strict-config -c model_auto_compact_token_limit_scope='"session"' …
unknown variant `session`, expected `total` or `body_after_prefix`
```

Both the name check and the value check discriminate, which is what makes a
passing result mean something.

## Two instruments that look right and are not

**`codex doctor` does not validate.** A config consisting only of

```toml
totally_invented_key_xyz = 1
```

reports `✓ config loaded · config.toml parse ok`. "It parses" is never evidence
here that a key does anything.

**A `strings` dump cannot tell you which struct a name belongs to.** It confirms
a name exists somewhere in the binary. It does not distinguish a `config.toml`
field from a model-catalog field — and that gap shipped a bug, below.

## The bug this repository shipped

An earlier revision set `auto_compact_token_limit` in every setup and in the
launcher. It parsed. `doctor` said `parse ok`. It did nothing.

```
$ codex exec --strict-config -c auto_compact_token_limit=200000 …
Error loading config.toml: unknown configuration field `auto_compact_token_limit`
```

The name is real, but it belongs to the **model catalog** struct —
`context_window · max_context_window · auto_compact_token_limit · comp_hash · …`
— while the config struct reads `model_provider · model_context_window ·
model_auto_compact_token_limit · model_auto_compact_token_limit_scope ·
approval_policy · …`.

The correct key is **`model_auto_compact_token_limit`**. This is exactly the
failure mode warned about one section earlier, committed by the author of the
warning, and it is recorded rather than quietly deleted.

---

# Part 2 — What this setup sets, measured

The whole configuration:

```toml
model = "gpt-6-astra"
approval_policy = "never"
sandbox_mode    = "danger-full-access"
model_context_window           = 872000
model_auto_compact_token_limit = 700000

[agents]
enabled = false

[features]
multi_agent = false
```

Every key passes `--strict-config`.

Session header under it, and identically through `bin/astra`:

```
model: gpt-6-astra
approval: never
sandbox: danger-full-access
```

Feature count, against an empty config as control:

| `config.toml` | feature flags |
| --- | --- |
| empty (control) | `47 enabled · 0 overridden` |
| this setup | `46 enabled · 1 overridden` |

Exactly one capability off and nothing else touched. Any other number means
something was enabled that nobody decided on.

The counter says nothing about `agents.enabled`, which is not a feature — its
effect is measured on the rendered prompt instead, below. All five
collaboration tool names count `0` under this configuration.

## Autonomy: which flag still exists

Session headers, `codex exec`:

| Invocation | `approval:` | `sandbox:` |
| --- | --- | --- |
| no flags | `never` | `workspace-write [workdir, /tmp, $TMPDIR]` |
| `--yolo` | `never` | `danger-full-access` |
| `--dangerously-bypass-approvals-and-sandbox` | `never` | `danger-full-access` |
| this config, no flags | `never` | `danger-full-access` |

And the ones that do **not** exist at this pin:

```
$ codex exec --full-auto …                     error: unexpected argument '--full-auto' found
$ codex exec --skip-permissions …              error: unexpected argument '--skip-permissions'
$ codex exec --dangerously-skip-permissions …  error: unexpected argument '--dangerously-skip-permissions'
```

`--full-auto` has been removed; the `--skip-permissions` spellings belong to a
different product. Only `--yolo` and its long form survive, and the config pair
reproduces them exactly — which is what lets the posture live in `bin/astra`
rather than depending on a flag.

`--yolo` does not appear in a `strings` dump of the binary yet demonstrably
works. A second reason the string table is only ever a *positive* test.

## Context: how much can actually be given

The question was whether 1M is reachable. It is not, and 872000 is not a
shortfall — it is what Astra's long-context mode *is* in this client.

```
context_window                    272000
max_context_window                872000
effective_context_window_percent      95
supports_experimental_context      false
```

| Number | What it is |
| --- | --- |
| 1050000 | the API model's advertised window |
| 872000 | `max_context_window` in this build's catalog, for every model but `gpt-5.5` |
| 828400 | 872000 × 95% — the usable budget |
| ~258000 | what a default session reports before `model_context_window` is set |
| 272000 | the catalog default, and exactly the long-context surcharge threshold |

`supports_experimental_context` is `false` for `gpt-6-astra`, the
`context_management` and `token_budget` features are both `under development`
and off, and `/extended-context` is reported not to raise Astra's budget
([oh-my-pi#10968](https://github.com/can1357/oh-my-pi/issues/10968)). No switch
found in this build goes above 872000.

`model_context_window = 1000000` passes `--strict-config`, because the schema
accepts any integer. That is not evidence it takes effect, and the catalog
ceiling is what the client plans against. Whether a larger value is clamped
could not be measured: reading the effective budget needs `/status` inside an
interactive session, and the usage limit blocked every turn.

**Settled: window 872000, compaction 700000** — 128400 tokens of headroom under
the 828400 ceiling, which is room for a long turn to finish rather than being
compacted in flight.

Raising the window is not free. The default 272000 is exactly the point above
which input bills at 2× and output at 1.5×, so this posture spends nearly all of
a session in the premium band by choice.

## Subagents: `agents.enabled` is the switch, the feature flag is not

This corrects an earlier reading in this file, which recorded the feature flag
as the control and the tool effect as "unproven". It is now proven, and the flag
is not the control.

Tool names counted in the rendered prompt (`codex debug prompt-input`, counted
with `grep -o | wc -l`):

| Term | nothing | `features.multi_agent=false` | `agents.enabled=false` |
| --- | --- | --- | --- |
| `spawn_agent` | 4 | 4 | 0 |
| `followup_task` | 2 | 2 | 0 |
| `send_message` | 3 | 3 | 0 |
| `wait_agent` | 2 | 2 | 0 |
| `interrupt_agent` | 1 | 1 | 0 |
| prompt bytes | 18221 | 18221 | 14661 |

**`features.multi_agent = false` changes the prompt by zero bytes.** Byte-for-byte
identical at 18221. It does move the feature counter from
`47 enabled · 0 overridden` to `46 · 1`, which is exactly why an earlier revision
believed it worked: the counter moved, so the change looked real.

**`agents.enabled = false` removes 3560 bytes and every collaboration tool.** It
is the documented control — *"Enable/disable multi-agent tools"*, default `true`
— and it does **not** touch the feature counter, because it is not a feature.
Two instruments, each blind to what the other sees.

Also valid at this pin, by `--strict-config`:
`agents.max_concurrent_threads_per_session`, `agents.max_threads` (its legacy
alias), `agents.max_depth`, `agents.job_max_runtime_seconds`. None is in the
setup.

Switch inventory:

```
multi_agent       stable   true     ← feature flag, inert on the prompt
multi_agent_v2    stable   false    ← already off, inert both ways
multi_agent_mode  removed  false
enable_fanout     removed  false
```

`multi_agent_v2` was dropped from the setup once this was measured: it ships off
and changes nothing in either direction. `multi_agent` is kept beside
`agents.enabled` because it does flip a real flag other code paths read —
[openai/codex#31097](https://github.com/openai/codex/issues/31097) reports it
being honoured inconsistently — and costs nothing.

### A residual that was not what it looked like

With `agents.enabled = false` applied in this repository's own checkout,
`spawn_agent` still counted `1`. The occurrence was in **this repository's
`AGENTS.md`**, picked up as project context, not in any tool definition. Against
a clean `CODEX_HOME` the count is `0`.

### `include_collaboration_mode_instructions` is a different thing

Tried and rejected. Setting it to `false` shortens the prompt by about 1044
bytes and drops mentions of "collaboration" from 7 to 2, while every multi-agent
tool name survives — it governs collaboration *modes*, not multi-agent tooling.

(A first attempt at these counts used `grep -c`, which counts matching *lines* —
and the rendered prompt is a single line of JSON, so every term reported `1`
whether it appeared once or thirty times. The numbers here come from
`grep -o | wc -l`.)

---

# Part 3 — Facts about this build

These are readings about Codex 0.155.1 and the account's catalog. They are not
all acted on: several describe capabilities this setup deliberately leaves
unset.

## The model catalog outranks published docs

`codex debug models` returns seven models. `gpt-6-astra` is `priority 1` and
`visibility: list` — not gated here, so the workaround in
[openai/codex#43342](https://github.com/openai/codex/issues/43342) (astra hidden
behind `visibility: hide`) does not apply.

Its `supported_reasoning_levels` carry **six** efforts:

    low · medium · high · xhigh · max · ultra

Published write-ups describe five and omit `ultra`, whose catalog description is
*"Maximum reasoning with automatic task delegation"* — a different thing from
`max`, not a louder version of it. `ultra` and `max` require **0.155.0 or
newer**; on 0.154.0 they cannot be expressed at all.

Also in the entry: `shell_type: unified_exec`, `tool_mode: code_mode_only`,
`multi_agent_version: v2`, `multi_agent_reasoning_effort: xhigh`,
`additional_speed_tiers: ["fast"]`,
`truncation_policy: {mode: tokens, limit: 10000}`.

## Reasoning effort: `none` is a hard failure, not a default

Config parsing accepts every effort string, including `none`, so parsing proves
nothing. The session header shows what would be sent:

```
model: gpt-6-astra
reasoning effort: xhigh
```

Confirmed for `xhigh` (the setting), and previously for `high`, `max` and
`ultra`.

**Leaving the key unset is the one setting that breaks the model.** The header
then reads `reasoning effort: none`, and the request fails outright:

```
Unsupported value: 'none' is not supported with the 'gpt-6-astra' model
```

[openai/codex#44184](https://github.com/openai/codex/issues/44184) reports this
against 0.153.4, with the task failing before the agent answers. The accepted
set there is `low`, `medium`, `high`, `xhigh`, `max` — **five**, with no
`ultra`, while this build's 0.155.1 catalog reports six including it. The issue
predates this pin, and server acceptance of `ultra` remains untested here.

An earlier revision of this repository shipped with the key unset, on the
reasoning that reasoning had not been decided. That left a configuration which
could not complete a single turn.

### Why `xhigh` is the safe choice above `high`

Three independent sources list the accepted efforts, and they disagree:

| Source | Values |
| --- | --- |
| [config reference](https://learn.chatgpt.com/docs/config-file/config-reference) | `minimal` `low` `medium` `high` **`xhigh`** |
| [openai/codex#44184](https://github.com/openai/codex/issues/44184), Astra-specific | `low` `medium` `high` **`xhigh`** `max` |
| this build's model catalog | `low` `medium` `high` **`xhigh`** `max` `ultra` |

`xhigh` is the only level above `high` that appears in **all three**. `max` is
absent from the config reference; `ultra` is absent from two of the three.

The reference qualifies it as *model-dependent, Responses API only*. This
session is on Responses — `codex doctor` reports `wire API: responses` with the
websocket connected — so the qualification is satisfied.

`--strict-config` accepts every effort string, including `minimal`, `ultra` and
`none`. The schema is permissive and validation happens on the wire, which is
the same reason `none` parses cleanly and then fails the request.

Published guidance suggests `high` for typical coding and reserves `xhigh` for
harder work. `xhigh` here is a deliberate choice for how Astra behaves, not a
misreading of that advice.

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

Stable **and** shipped off are **four**, where `full-auto` documents three:

    memories · multi_agent_v2 · recommended_plugins · secret_auth_storage

`secret_auth_storage` is the new one. None of these is enabled by this setup —
they are listed because a maximal posture would have to name them, and because
any setup copying `full-auto`'s "exactly three" sentence is out of date.

Relevant capabilities already stable and on: `browser_use`,
`browser_use_external`, `browser_use_full_cdp_access`, `computer_use`,
`fast_mode`, `hooks`, `goals`, `in_app_browser`, `multi_agent`.

### One reading that does not add up

Each stable-but-off feature set *alone* reports `48 enabled · 1 overridden`, and
the control in the other direction — `browser_use = false`, a feature that ships
on — reports `46 enabled · 1 overridden`. So a single override counts as one.
Four together should therefore report four; they reported **one**. `doctor`
prints the `overrides` and `enabled flags` lists as `<redacted>`, so the
discrepancy could not be resolved from the outside.

Treat `enabled` as the load-bearing number and `overridden` as not understood.
It happens not to matter for the current setup, which turns off exactly one
feature and reports `1`.

## Astra's cost shape

From OpenAI's model page and integration write-ups, not measured here:

- context 1.05M tokens (922K input max, 128K output max), cutoff 2026-04-30
- $10 / 1M input, $50 / 1M output; cached input $1/M
- **above 272K tokens in context: 2× input and 1.5× output** — a cliff, not a ramp
- Fast mode (`service_tier = "priority"`): 2× on top of whatever applies

## The app rewrites `config.toml`, but not `AGENTS.md`

Two measurements, and the difference between them is what the design rests on.

**`config.toml` — rewritten on every launch.** Run twice, identically:

1. append a marker plus posture keys → 1918 bytes, marker present
2. stop the app → marker still present, so this is a *startup* behaviour, not a
   shutdown one
3. start the app and wait for it to come fully up → 1884 bytes, marker gone, and
   `diff` against the app's own generated file is empty

**`AGENTS.md` — untouched.** Same procedure, app confirmed fully up at 15
processes, `config.toml` rewritten to its 1884 bytes in the same pass:

```
AGENTS.md: 25 bytes, marker: 1
config.toml: 1884 bytes
```

So the instructions surface survives and could be owned persistently. Nothing is
placed there yet, because instructions have not been decided.

A first attempt at this measurement was discarded: the app had not actually
started (the launch died with its parent shell), so the file was unchanged for
the wrong reason. A surviving marker only means something once the app is
confirmed up.

## No file layer outranks `config.toml`

All three candidates found in the binary were tested and **none** overrode
`~/.codex/config.toml`:

- `~/.codex/managed_config.toml`
- `/etc/codex/managed-config.toml`
- `/etc/codex/config.toml`

The managed layer appears to require enterprise enrolment. `/etc/codex` was
created for the test and removed afterwards.

That leaves the command line, which nothing rewrites — hence `bin/astra`.

## Browser use depends on the desktop app

Full working in [browser-use.md](browser-use.md). In short: the CLI ships no
`extension-host` binary and no `com.openai.codexextension` strings; with no
`config.toml` it reports *No plugin marketplaces in scope*; the app writes the
`[marketplaces.*]` and `[plugins."chrome@openai-bundled"]` blocks that register
the bridge, using absolute paths this build will not expand from `~` or
`${HOME}`.

Verified live: with Chrome running, `extension-host` appears as a **child
process of `chrome`**. A manifest that parses is not evidence; a child process
is.

---

# Open items

Two earlier items are now closed:

- ~~whether `none` is rejected~~ — it is, with an exact error string, and the
  setup now sets `xhigh`
- ~~whether the subagent switch works~~ — the feature flag does not, and
  `agents.enabled` does; measured on the prompt

What remains:

1. **No turn has completed.** The usage limit held for the entire session, so
   every reading here is taken before a request is answered. `xhigh` is a
   documented-accepted value and reaches the wire — the header says so — but
   server acceptance has not been observed.
2. **Whether `model_context_window` above 872000 is clamped.** Needs `/status`
   inside an interactive session.
3. **Whether `ultra` is accepted server-side.** This build's catalog lists it as
   a supported reasoning level; the accepted set in
   [openai/codex#44184](https://github.com/openai/codex/issues/44184) has five
   and omits it. That issue predates this pin. Not settled either way, and not
   used by this setup regardless — `ultra` delegates to sub-tasks.

All three need the same thing.
