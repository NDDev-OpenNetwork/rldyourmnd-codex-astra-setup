# Agent orientation

This repository carries a GPT-6-Astra posture for Codex. It is one config file
and one shell script — no build, no tests, no runtime.

**Scope is deliberately small.** Decided: the model and its reasoning effort,
reasoning visibility, the autonomy posture, web search, the context window,
memory across sessions, telemetry, and no subagents. Everything else stays at
product defaults, and that is not an oversight. Do not add a key because it
seems obviously good — add one when it has been decided.

## Validate with `--strict-config`, never with `doctor`

`codex doctor` reports `✓ config loaded · parse ok` for a file containing
nothing but an invented key. It is not a validator.

```sh
codex exec --strict-config -c '<key>=<value>' --skip-git-repo-check --ephemeral x
```

That rejects unknown fields and bad variants. It is the only instrument in this
repository that has ever caught a real mistake — an earlier revision shipped
`auto_compact_token_limit`, which is a model-*catalog* field rather than a
config key, in every setup and in the launcher. It parsed. It did nothing.

A `strings` dump of the binary can confirm a key exists somewhere; it cannot
tell you which struct it belongs to. Do not use it as a validator.

## What is set

| Key | Value | Why |
| --- | --- | --- |
| `model_reasoning_effort` | `xhigh` | unset means `none`, which Astra rejects outright |
| `approval_policy` | `never` | with the next row, exactly what `--yolo` sets |
| `sandbox_mode` | `danger-full-access` | |
| `model_context_window` | `872000` | this account's `max_context_window` |
| `model_auto_compact_token_limit` | `700000` | 128400 under the 828400 usable ceiling |
| `model_reasoning_summary` | `detailed` | the only visibility into what `xhigh` bought |
| `web_search` | `live` | cutoff is 2026-04-30 |
| `memories.*` + `features.memories` | on | durable state in CODEX_HOME |
| `analytics` / `feedback` | `false` | both default to on |
| `personality` / `model_verbosity` | `pragmatic` / `medium` | verbosity states the default |
| `tool_output_token_limit` | `32000` | ~125 KB whole; no subagent to absorb a truncation |
| `project_doc_max_bytes` | `65536` | default 32768 truncates instructions silently |
| `shell_environment_policy.inherit` | `all` | the default — and the SSH agent reaches every command |
| `browser_use` / `computer_use` | open | history, access, downloads, uploads, apps |
| `history.persistence` | `save-all` | the default; memory generation depends on it |
| `agents.enabled` | `false` | the switch that actually removes the subagent tools |
| `features.multi_agent` | `false` | flips a real flag; inert on the prompt |

**Never remove `model_reasoning_effort`.** Unset is not a neutral default: it
sends `none`, and the request fails with `Unsupported value: 'none' is not
supported with the 'gpt-6-astra' model` before the agent answers.

**872000, not 1000000.** A larger number is not a bigger window, it is a wrong
one — the catalog ceiling is 872000 and `supports_experimental_context` is
false.

**The `model_` prefix on the compaction key is load-bearing.** See above.

**`agents.enabled`, not the feature flag.** `features.multi_agent = false`
changes the rendered prompt by **zero bytes** while moving the feature counter
to `46 · 1` — which is exactly how an earlier revision convinced itself it had
disabled subagents. `agents.enabled = false` is what removes the tools, and it
does not move the counter at all. Check both instruments, never one.

## `config.toml` is not ours to own

The ChatGPT desktop app rewrites `~/.codex/config.toml` from scratch on every
launch and discards any key it did not write, and that same file is where it
registers browser use with absolute paths this build will not expand from `~` or
`${HOME}`.

So the posture lives in `bin/astra`, on the command line, which nothing
rewrites. `AGENTS.md` in `CODEX_HOME` *is* safe to own — measured, the app
leaves it alone — but nothing is placed there yet, because instructions have not
been decided either.

`docs/browser-use.md` has the evidence.

## Where things live

| Path | What it is |
| --- | --- |
| `bin/astra` | the launcher, and the only entry point that survives the app |
| `setups/standard/home/config.toml` | the same posture, for a machine with no desktop app |
| `docs/measurements.md` | readings, instruments, controls, dates |
| `docs/browser-use.md` | why the app owns `config.toml` |
| `.gds/repository.yaml` | the estate anchor — identity, classification, verification lanes |

## House rules

- Say what is true. If a check did not run, say it did not run.
- A number in the documentation needs the command that produced it and a control
  showing the instrument discriminates.
- Where the model catalog and a published article disagree, the catalog wins and
  the disagreement gets written down.
- Do not quietly resolve a reading that does not add up, and do not quietly
  delete a mistake — `docs/measurements.md` records one of each on purpose.
- English in code, comments, documentation and commits.

## Verifying a change

```sh
gds validate                                     # the estate anchor
sh -n bin/astra
ASTRA_CODEX_BIN=echo ./bin/astra                 # what it would pass through

H=$(mktemp -d); cp setups/standard/home/config.toml "$H/config.toml"
CODEX_HOME="$H" codex exec --strict-config --skip-git-repo-check --ephemeral x
CODEX_HOME="$H" codex doctor --all | grep 'feature flags'
```

The feature count must read `47 enabled · 2 overridden`, against
`47 enabled · 0 overridden` for an empty config. Any other number means
something was turned on that nobody decided on.

The counter is blind to `agents.enabled`. Check that separately, on the prompt:

```sh
CODEX_HOME="$H" codex debug prompt-input \
  | grep -o -e spawn_agent -e followup_task -e wait_agent | wc -l
```

It must be `0`. Run it against a clean `CODEX_HOME`, not this checkout — this
repository's own `AGENTS.md` mentions `spawn_agent` and is picked up as project
context.
