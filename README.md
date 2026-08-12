# vscode-memory-watch

Find out **which** VS Code process is eating your RAM, before the crash destroys the evidence.

VS Code is a tree of 30+ identically-named `electron` processes. When the window balloons to
15–20 GB and you reload it to stay sane, you learn nothing — the process that was leaking is
gone. This samples the tree every 60 seconds and records memory *per tier*, so the climbing
tier names itself, and it dumps a full per-process snapshot at the moment things get bad.

It found a real bug in under four hours. See [What it found](#what-it-found).

## Usage

```console
$ vscode-memory-watch --once      # one reading, right now
VS Code tree: 8410 MB PSS (11131 MB RSS) across 31 processes, 8 Claude session(s)
  claude=2163MB exthost=1124MB renderer=4096MB tsserver=490MB langserver=285MB gpu=80MB main=148MB
  largest: renderer pid=3369519 at 4096 MB
```

```console
$ vscode-memory-watch --report    # growth since the last VS Code restart
current run: 2026-08-11T18:45:37 -> 2026-08-11T22:42:01 (238 samples, ~238 min)

tier              start        now       peak     growth
total_pss        3829 MB     8410 MB     9634 MB    +4581 MB
claude           1030 MB     2163 MB     2234 MB    +1133 MB
exthost          1063 MB     1124 MB     2388 MB      +61 MB
renderer          924 MB     4096 MB     5317 MB    +3172 MB
tsserver          369 MB      490 MB     1784 MB     +121 MB

processes: 30 -> 31   claude sessions: 4 -> 8
tree growth rate: +1155 MB/hour
```

Run with no arguments to sample continuously, logging to
`~/.local/state/vscode-memory-watch/history.csv`.

## Install

```sh
install -Dm755 vscode-memory-watch ~/.local/bin/vscode-memory-watch
install -Dm644 vscode-memory-watch.service ~/.config/systemd/user/vscode-memory-watch.service
systemctl --user daemon-reload
systemctl --user enable --now vscode-memory-watch.service
```

Requires bash, awk and `/proc` — so Linux. No other dependencies.

## What it measures

Processes are classified from their command line, not their name, because every tier is the
same `electron` binary:

| tier | what it is |
|---|---|
| `renderer` | window renderers and out-of-process webview iframes |
| `exthost` | extension hosts |
| `claude` / `tsserver` / `langserver` | language servers and extension-spawned helpers |
| `gpu`, `main`, `utility`, `zygote` | the rest of the Electron tree |

**It reports PSS, not RSS.** RSS counts every shared mapping once per process, so summing RSS
across a 30-process Electron tree overstates real usage badly — in the example above, 11.1 GB
RSS vs 8.4 GB PSS. PSS divides shared pages among the processes mapping them, so the tiers
actually add up.

It walks `/proc` directly rather than using `pgrep -f`, whose pattern would match the watcher's
own command line.

## Alerting

When the tree crosses `12000` MB PSS, or any single process crosses `4000` MB, it sends a
desktop notification and writes a full per-process snapshot to
`~/.local/state/vscode-memory-watch/snapshots/` — so you get the peak captured instead of
losing it to a reload. Notifications are rate-limited to one per 30 minutes.

Thresholds are environment variables on the service:

| variable | default | meaning |
|---|---|---|
| `VSCODE_MEM_INTERVAL` | `60` | seconds between samples |
| `VSCODE_MEM_WARN_TOTAL` | `12000` | whole-tree PSS warning line, MB |
| `VSCODE_MEM_WARN_PROC` | `4000` | single-process PSS warning line, MB |
| `VSCODE_MEM_COOLDOWN` | `1800` | seconds between notifications |

## What it found

I wrote this because VS Code was reaching ~20 GB every couple of days and I was tired of
guessing. Four hours of sampling gave an unambiguous answer: a **single webview renderer
process** grew 924 MB → 5317 MB (~1.1 GB/hour), while every other tier stayed flat or
sawtoothed normally under garbage collection — the extension host netted **+61 MB** over the
same window despite peaking at 2388 MB, and tsserver peaked at 1784 MB and returned to 490 MB.

That's [anthropics/claude-code#84013](https://github.com/anthropics/claude-code/issues/84013):
the conversation webview retains all rendered history, and neither `/compact` nor closing
session tabs releases it.

The per-tier split is the whole point. "VS Code is using 20 GB" is not actionable. "One webview
renderer is using 4.2 GB while the next largest process is 498 MB, and the extension host holding
the same conversation data nets +61 MB" is a bug report.

`examples/` has a real report and a real snapshot captured at peak.

## License

MIT
