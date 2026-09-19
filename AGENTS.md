# Agent orientation

This repository carries a GPT-6-Astra posture for Codex. It is configuration and
one shell script — there is no build, no test suite and no runtime.

## The one thing to understand before changing anything

`~/.codex/config.toml` is **owned by the ChatGPT desktop app**, which rewrites it
from scratch on every launch and discards any key it did not write. That file is
also where browser use is registered, through `[marketplaces.*]` and
`[plugins."chrome@openai-bundled"]` blocks carrying absolute paths that this
build will not expand from `~` or `${HOME}`.

So:

- **Do not** move the posture into `~/.codex/config.toml`. It will survive until
  the next app launch and then vanish, silently.
- **Do not** ship the app's marketplace or plugin blocks here. They are
  machine-specific by construction.
- The posture belongs on the command line, in `bin/astra`, where nothing
  rewrites it.
- `AGENTS.md` **is** safe to own: measured, the app leaves it alone.

`docs/browser-use.md` has the evidence. `docs/measurements.md` has every reading
with the instrument that produced it.

## Where things live

| Path | What it is |
| --- | --- |
| `bin/astra` | the launcher, and the only entry point that survives the app |
| `setups/astra/` | everyday posture — `config.toml` fallback plus `AGENTS.md` |
| `setups/astra-ultra/` | escalation posture, same shape |
| `docs/measurements.md` | readings, instruments, controls, dates |
| `docs/browser-use.md` | why the app owns `config.toml` |

The `setups/*/home/config.toml` files are the floor for a machine with **no**
desktop app, where nothing else writes that file. On a machine with the app they
are not what applies — the launcher is.

## House rules for edits here

- Say what is true. If a check did not run, say it did not run.
- A number in the documentation needs the command that produced it and a control
  showing the instrument discriminates. `codex doctor` reports
  `✓ config loaded · parse ok` for a config containing nothing but an invented
  key, so "it parses" is not evidence.
- Where the model catalog and a published article disagree, the catalog wins and
  the disagreement gets written down.
- Do not quietly resolve a reading that does not add up. `docs/measurements.md`
  records one such discrepancy unexplained on purpose.
- English in code, comments, documentation and commits.

## Verifying a change

```sh
sh -n bin/astra                                  # syntax
ASTRA_CODEX_BIN=echo ./bin/astra --ultra         # what it would pass through

H=$(mktemp -d); cp setups/astra/home/config.toml "$H/config.toml"
CODEX_HOME="$H" codex doctor --all | grep 'feature flags'
```

The control for that last one is an empty `config.toml`, which reports
`47 enabled · 0 overridden` at 0.155.1. A setup here should report `51`.
