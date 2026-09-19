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
astra                      # interactive, everyday posture
astra --ultra              # escalation posture
astra exec "…"             # one-shot, same posture
ASTRA_EFFORT=max astra     # override the effort for a single run
```

Everything after the flags is passed through to `codex` untouched, so
`astra resume`, `astra review` and `astra --help` behave as you expect.

## The two postures

| | `astra` | `astra --ultra` |
| --- | --- | --- |
| `model_reasoning_effort` | `high` | `ultra` |
| `plan_mode_reasoning_effort` | `xhigh` | `ultra` |
| `auto_compact_token_limit` | 200000 | 160000 |

`ultra` is the top of Astra's six-level scale and is **not** a louder `max` —
the catalog describes it as *"Maximum reasoning with automatic task
delegation"*. Published write-ups list five levels and omit it; this build's own
catalog is what says it exists.

The compaction limit goes **down** at the higher effort, not up: above 272K
tokens input bills at 2× and output at 1.5×, and `ultra` spends more tokens per
turn, so it reaches that cliff sooner.

`model_reasoning_effort` is set explicitly in both, and that is not stylistic.
Unset, the session header reads `reasoning effort: none` — and `none` is the one
value Astra rejects.

## Requirements

- `@openai/codex` **0.155.0 or newer**. `max` and `ultra` cannot be expressed
  before it; on 0.154.0 the value simply does not exist.
- For browser use: the ChatGPT desktop app installed and launched at least once.
  The CLI cannot supply the native messaging host on its own.

## Layout

```
bin/astra                      the launcher — the single entry point
setups/astra/home/             config.toml for the CLI-only case (no app installed)
setups/astra-ultra/home/       the escalation variant of the same
docs/measurements.md           every reading, with the instrument and the date
docs/browser-use.md            why the app owns config.toml here
```

The files under `setups/` are the fallback floor for a machine with no desktop
app, where `config.toml` is stable because nothing else writes it. On a machine
with the app, the launcher is what applies.

## On trusting this repository

Two things here contradict published documentation, and in both cases the
measurement won:

- Astra has **six** reasoning efforts on this account, not five
- the 0.155.1 registry ships **four** stable-but-disabled features, not the three
  that `codex-setup-system`'s `full-auto` documents against an older pin

`codex doctor` reports `✓ config loaded · parse ok` for a config consisting
entirely of an invented key, so "it parses" is never offered here as evidence.
Where a number appears, `docs/measurements.md` names the instrument that
produced it and the control that shows the instrument discriminates.
