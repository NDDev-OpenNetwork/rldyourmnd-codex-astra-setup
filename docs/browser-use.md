# Browser use, and why this repository does not touch it

Short version: **the desktop app owns browser use, and this setup deliberately
stays out of its way.** If you remove the app, browser use through the Chrome
extension stops working, and the CLI cannot restore it on its own.

This file records how that was established, because the conclusion is not
obvious and the failure mode is silent.

## What the CLI has, and what it does not

At 0.155.1 the feature registry reports all of these as `stable` and enabled:

    browser_use · browser_use_external · browser_use_full_cdp_access
    computer_use · in_app_browser

So the capability is compiled in. What is missing is the transport.

The CLI package ships **no** `extension-host` binary, and the compiled binary
contains **no** `com.openai.codexextension` or `NativeMessagingHosts` strings:

```sh
find "$CODEX_VENDOR" -iname '*extension-host*' -o -iname '*native*host*'   # empty
strings "$CODEX_VENDOR/bin/codex" | grep -E 'codexextension|NativeMessagingHosts'   # empty
```

With no `config.toml` present, the consequences are visible directly:

```
$ codex plugin marketplace list
No plugin marketplaces in scope.

$ codex plugin add chrome@openai-bundled
Error: plugin `chrome` was not found in marketplace `openai-bundled`
```

## What the app does on launch

It writes `~/.codex/config.toml`, and that file is where the bridge is
registered:

```toml
[marketplaces.openai-bundled]
source_type = "local"
source = "/home/<user>/.codex/.tmp/bundled-marketplaces/openai-bundled"

[plugins."chrome@openai-bundled"]                enabled = true
[plugins."browser@openai-bundled"]               enabled = true
[plugins."unified-computer-use@openai-bundled"]  enabled = true
```

It also installs the native-messaging manifest into **eight** browser profiles
(chrome, chromium, chrome-beta, chrome-unstable, chrome-for-testing, edge, opera,
vivaldi), each pointing at a real executable under
`~/.codex/plugins/cache/openai-bundled/chrome/<app-version>/`, and it
materialises the bundled runtime in `~/.cache/codex-runtimes` (~1.8 GB: node,
python, LibreOffice headless, poppler, git).

**The proof that it is connected**, rather than merely configured, is that
`extension-host` appears as a child process of `chrome` once the browser starts:

```
pid 1543491  ppid 1542989 (chrome)
~/.codex/plugins/cache/openai-bundled/chrome/latest/extension-host/linux/x64/extension-host
```

A manifest that parses is not evidence. A child process is.

## Why the blocks cannot be shipped here

The marketplace `source` must be an absolute path. This build expands neither
`~` nor `${HOME}`:

| `source` | Result |
| --- | --- |
| `/home/<user>/.codex/.tmp/bundled-marketplaces/openai-bundled` | resolves |
| `~/.codex/.tmp/bundled-marketplaces/openai-bundled` | `marketplace root does not contain a supported manifest` |
| `${HOME}/.codex/.tmp/bundled-marketplaces/openai-bundled` | `marketplace root does not contain a supported manifest` |

So these blocks are machine-specific by construction and cannot live in a
repository. The app is not merely the easiest owner of them — it is the only
correct one.

## Why this forces the launcher

The app rewrites `config.toml` **from scratch on every launch**, discarding any
key it did not write. Measured twice, identically:

1. append a marker and posture keys → 1918 bytes, marker present
2. stop the app → marker still present (so it is not a shutdown behaviour)
3. start the app → 1884 bytes, marker gone, `diff` against the app's own
   generated file is empty

There is no higher-authority file layer available to sit above it. All three
candidates found in the binary were tested and **none** overrode
`~/.codex/config.toml`:

- `~/.codex/managed_config.toml`
- `/etc/codex/managed-config.toml`
- `/etc/codex/config.toml`

(The managed layer appears to require enterprise enrolment. `/etc/codex` was
created for the test and removed afterwards.)

That leaves the command line, which nothing rewrites. `bin/astra` carries the
posture there, and the app keeps sole ownership of `config.toml`:

```
$ astra exec "…"
model: gpt-6-astra
approval: never
sandbox: danger-full-access
reasoning effort: max
$ codex plugin list | grep -c 'installed, enabled'
13
```

Both at once: the posture applied, and all thirteen of the app's plugins —
browser use included — still enabled.

## If you remove the desktop app

Browser use through the extension stops, and the extension in Chrome becomes
inert: its native host is gone and the CLI cannot supply one.
[openai/codex#26820](https://github.com/openai/codex/issues/26820) reports the
CLI side seeing `hasNativePipe: false` for exactly this path; it is open and
unresolved.

The remaining route is the Chrome DevTools Protocol —
`browser_use_full_cdp_access` against a Chrome started with
`--remote-debugging-port` — which needs neither the extension nor the app. It is
a different mechanism, not a drop-in replacement, and this repository does not
configure it.
