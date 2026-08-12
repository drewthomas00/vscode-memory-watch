# vscode-memory-watch

Find out **which** VS Code process is eating your RAM, before the crash destroys the evidence.

VS Code is a tree of 30–50 processes that mostly share one binary and one name. When the window
balloons to 15–20 GB and you reload it to stay sane, you learn nothing: the process that was
leaking is gone. This samples the tree on an interval and records memory *per tier*, so the tier
that climbs names itself, and it captures a full per-process snapshot at the moment things go bad.

It found a real bug in under four hours. See [What it found](#what-it-found).

## Usage

```console
$ vscode-memory-watch --once      # one reading, right now
VS Code tree: 9589 MB PSS (12622 MB RSS) across 38 processes, 9 Claude session(s)
  renderer=4942MB exthost=497MB claude=2417MB tsserver=346MB langserver=350MB helper=191MB nodeutil=567MB utility=21MB gpu=83MB main=155MB zygote=13MB
  largest: renderer pid=3369519 at 4360 MB
  (4 integrated-terminal process(es) excluded)

  4465437 KB pss    4576384 KB rss  renderer   pid=3369519  window
   439848 KB pss     513072 KB rss  nodeutil   pid=3369526
   336143 KB pss     458388 KB rss  claude     pid=3380222  session=e6e314ca-f038-44a1-8966-6bed829acd46
   326732 KB pss     447744 KB rss  claude     pid=3372538  session=55322f72-2988-4235-a7b2-20b35822b51a
   ...
```

`vscode-memory-watch --report` prints the same tiers as a growth table — start, now, peak and
delta for each — since VS Code last started. That's the view that identifies a leak, and there's a
real one in [What it found](#what-it-found).

Run with no arguments to sample continuously. `--help` lists everything.

## Install

```sh
install -Dm755 vscode-memory-watch ~/.local/bin/vscode-memory-watch
install -Dm644 vscode-memory-watch.service ~/.config/systemd/user/vscode-memory-watch.service
systemctl --user daemon-reload
systemctl --user enable --now vscode-memory-watch.service
```

Needs **bash 4.2+** (for `printf '%()T'`), awk, coreutils and Linux `/proc`. `notify-send` is
optional — without it the snapshot is still written, you just don't get the desktop alert.

## How it finds the tree

It finds every **renderer**, walks *up* to the process that owns it, and then walks back down
through the tree. Renderers are the anchor because a renderer is the one thing only a real editor
instance has. That matters more than it sounds: a `code --wait` used as your git editor, or a
`code --status`, spawns Chromium children of its own and matches every command-line pattern a real
main process does. Identifying roots by pattern instead adopts those as extra instances, and since
a change in the set of main processes is what ends a run, a routine `git commit` would silently
discard hours of accumulated growth curve.

Working from the tree rather than from install paths is also what keeps it working across VS Code,
Code - OSS, VSCodium, Cursor, Windsurf and code-server, none of which agree on where they live or
what they call their user-data directory.

**The walk is bounded.** It descends through VS Code's own process types and stops at anything
else, and it skips any process with a controlling terminal. So an extension's language server is
counted, but the `cargo build` you started in the integrated terminal is not — that is your memory,
not VS Code's, and sweeping it in would both inflate the headline and fire alerts naming the wrong
thing. Excluded terminal processes are counted and reported so you know they were left out.

Processes are then classified into tiers by command line, because every Chromium tier is the same
binary under the same name:

| tier | what it is |
|---|---|
| `renderer` | window renderers and out-of-process webview iframes |
| `exthost` | extension hosts |
| `nodeutil` | other Node utility processes — shared process, pty host, file watcher |
| `claude` | Claude Code session processes (one per open session) |
| `tsserver`, `langserver` | TypeScript server and language servers |
| `helper` | anything else an extension spawned: tool servers, language runtimes |
| `utility`, `gpu`, `main`, `zygote` | the rest of the Electron tree |

Splitting `exthost` from `nodeutil` matters more than it sounds: the extension host, shared
process, pty host and file watcher are *all* `--type=utility --utility-sub-type=node.mojom.NodeService`
with near-identical command lines. Only the extension host is started with a debug port, which is
what separates it here. Without that split, a pty-host leak gets reported as extension-host growth.

## Why PSS

**It reports PSS, not RSS.** RSS counts every shared mapping once per process, so summing it
across a 40-process Electron tree overstates real usage badly — 12.0 GB RSS against 9.0 GB PSS in
the example above. PSS divides each shared page among the processes mapping it, so the tiers
actually add up to the total.

Where `smaps_rollup` can't be read — a process exiting mid-sample, or a kernel built without
`CONFIG_PROC_PAGE_MONITOR` — it falls back to RSS, **marks that figure `(rss estimate)`**, and
counts it in a `pss_fallback` column, rather than quietly reporting RSS under a PSS heading.

## Cost

A sample costs about **0.7 s of CPU on a 40-process tree** — roughly 1% of one core at the 60 s
default. Almost all of it is the kernel: reading `smaps_rollup` makes it walk a process's page
tables, which for a 4 GB renderer is genuinely expensive. That is the price of PSS, and there is no
cheaper way to get an honest number. If it matters on your machine, raise `VSCODE_MEM_INTERVAL`.

What the script itself does is cheap and deliberately kept that way: per-process work is pure bash
reading `/proc`, with no forks or subshells, and the per-process detail table is only sorted when
something is actually going to read it — not on the ~1,400 daily samples that produce no output.
The service ships with `Nice=10` and `IOSchedulingClass=idle` so sampling never competes with the
thing it is measuring.

It walks `/proc` directly rather than using `pgrep -f`, whose pattern would match the watcher's own
command line. The bounded walk keeps that property: the watcher runs from systemd, outside the
tree, and even run by hand from a VS Code terminal it is excluded along with the rest of the
terminal's processes.

## Alerting

When the tree crosses `12000` MB PSS, or any single process crosses `4000` MB, it sends a desktop
notification and writes a full per-process snapshot to `snapshots/`.

The snapshot is rewritten on every **new high**, not just the first crossing — otherwise the
notification cooldown would freeze the evidence at whatever the tree looked like when it first
crossed the line, which is not the peak and not what you want to paste into a bug report.
Notifications themselves stay rate-limited to one per 30 minutes, and that limit persists across
restarts so a crash-looping service can't spam you.

| variable | default | meaning |
|---|---|---|
| `VSCODE_MEM_INTERVAL` | `60` | seconds between samples |
| `VSCODE_MEM_WARN_TOTAL` | `12000` | whole-tree PSS warning line, MB |
| `VSCODE_MEM_WARN_PROC` | `4000` | single-process PSS warning line, MB |
| `VSCODE_MEM_COOLDOWN` | `1800` | seconds between notifications |

## History file

Samples land in `~/.local/state/vscode-memory-watch/history.csv`, one row per sample, with a column
per tier. The header is generated from the tier list, so the columns can't drift from the data, and
`--report` looks columns up by name rather than position.

If the tier list ever changes, the old log is moved aside automatically rather than having new rows
appended under a header that no longer describes them — a mismatched row reads as another tier's
numbers, silently.

`--report` describes the **current run only**. A run ends when the heap under observation was
thrown away, which happens two ways: the editor restarts (the set of main processes changes), or
you reload the window (the renderer tier collapses while the main process carries on — the more
common case by far, and the one a main-PID check alone would miss). Reloads are counted and shown.

It deliberately does *not* split on a gap in the log: a laptop that sleeps overnight leaves a gap,
but the memory survives sleep and the run genuinely continues. Gaps are excluded from the
growth-rate denominator instead, and reported. The sampling cadence used for that is measured from
the data, not read from your environment — otherwise the shell you happen to run `--report` from
could change the headline growth rate by a multiple.

## What it found

I wrote this because VS Code was reaching ~20 GB every couple of days and I was tired of guessing.
Four hours of sampling gave an unambiguous answer:

```console
$ vscode-memory-watch --report
current run: 2026-08-11T18:45:37-04:00 -> 2026-08-11T22:42:01-04:00 (238 samples, ~238 min)

tier              start        now       peak     growth
total_pss        3829 MB     8410 MB     9634 MB    +4581 MB
total_rss        5919 MB    11131 MB    12357 MB    +5212 MB
claude           1030 MB     2163 MB     2234 MB    +1133 MB
exthost          1063 MB     1124 MB     2388 MB      +61 MB
renderer          924 MB     4096 MB     5317 MB    +3172 MB
tsserver          369 MB      490 MB     1784 MB     +121 MB
langserver        176 MB      285 MB      285 MB     +109 MB
gpu                77 MB       80 MB       81 MB       +3 MB
main              148 MB      148 MB      152 MB       +0 MB
other              40 MB       21 MB       40 MB      -19 MB

processes: 30 -> 31   claude sessions: 4 -> 8
tree growth rate: +1155 MB/hour
```

*(Verbatim v1.0 output — the version that was running during the investigation. v1.1 splits
`nodeutil` out of `exthost` and `helper` out of `other`; the raw samples behind it are in
`examples/history-incident-v1.csv`.)*

A **single webview renderer process** grew 924 MB → 5317 MB (~1.1 GB/hour), while every other tier
stayed flat or sawtoothed normally under garbage collection. The extension host netted **+61 MB**
over the same window despite peaking at 2388 MB; tsserver peaked at 1784 MB and came back down to
490 MB. At peak, one renderer held 4.2 GB while the next largest process in the tree held 498 MB.

That's [anthropics/claude-code#84013](https://github.com/anthropics/claude-code/issues/84013): the
Claude Code conversation webview retains all rendered history, and neither `/compact` nor closing
session tabs releases it.

The per-tier split is the whole point. "VS Code is using 20 GB" is not actionable. "One webview
renderer is using 4.2 GB while the next largest process is 498 MB, and the extension host holding
the same conversation data nets +61 MB over four hours" is a bug report.

`examples/` has the raw samples, the snapshot captured at peak, and current-version output.

## License

MIT
