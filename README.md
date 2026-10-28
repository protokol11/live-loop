# live-loop

One unattended command that runs a desktop app, captures its output, decides pass or fail, and cleans up.

## What you get

- **Spawn, capture, tree-kill in one unattended command** → every run leaves a full timestamped log on disk and no orphan processes.
- **Marker-driven pass/fail** → the loop live-filters log lines for pass, fail, and progress signals, so a bug that only shows in a real window gets a deterministic signal.
- **Fault injection and stress on demand** → kill a chosen process at a chosen second, run a stressor alongside the app, and exercise respawn and recovery without waiting for a rare crash.

## Why

Some bugs only reproduce in a real window: a Tauri host, a Godot renderer, a sidecar. A unit test cannot see them. A manual session can, but it scales poorly and leaves no evidence.

The live loop closes that gap. Before diagnosing a live-only bug, you build a fast, deterministic, agent-runnable signal. A 30-second flaky loop is barely better than no loop; a 2-second deterministic loop changes the diagnosis.

## How it works

```
                 one command, unattended
 ┌─────────────────────────────────────────────────┐
 │  spawn ──► capture (stdout+stderr, timestamped) │
 │              │                                   │
 │              ▼                                   │
 │  live-filter markers (pass / fail / progress)   │
 │              │                                   │
 │      match? ─┴─ timeout? ─┴─ fault at second N   │
 │              │                                   │
 │              ▼                                   │
 │  tree-kill the whole process forest              │
 │              │                                   │
 │              ▼                                   │
 │  exit 0 if completed + marker matched            │
 │  logs survive the kill (per-process, unique)     │
 └─────────────────────────────────────────────────┘
```

The runner contract: `--duration N` kills the app after N seconds, `--until <regex>` stops early on a marker, `--keep-going` continues to the duration after a match. Stress modes and `--kill-<target>-at N` add load and fault injection. Exit code 0 means the run completed and the marker matched.

Before the first run, you lock the scope, list the markers, and map the process forest — wrappers, children, window owners. A `child.kill()` on a wrapper orphans the real process, so the loop always kills from the root.

The package also includes **monster-longrun**, a single-writer pipeline for shipping a batch of specs in one session: one subagent per spec, numbered gates the orchestrator re-runs, eyes-on verification on a real artifact, and a violation pass that expects and fixes new findings.

## Install

Copy the skill folders to `~/.agents/skills/`:

```
cp -r live-loop ~/.agents/skills/
cp -r monster-longrun ~/.agents/skills/
```

Works with any agent that reads `SKILL.md`. In pi, invoke with `/skill:live-loop` or `/skill:monster-longrun`.

## Quick example

Build the loop in this order:

1. Map the launch chain and verify it with a live probe.
2. Write the minimal runner: spawn, tee output, tree-kill on timeout.
3. Add the marker filter and `--until`.
4. Add fault injection and stressors, one lever at a time.
5. Smoke-test the loop itself: marker matched, clean kill, logs complete.

Then attack the loop: does the kill kill the whole forest? Can the marker regex match a line that never comes from the real event? What did the original bug report include that the loop cannot produce?

## Honest status

**Working.** The skill is a protocol plus a reference implementation built 2026-09-15 for a Tauri host, Godot renderer, and TS sidecar (runner script plus two fault-injection stress scripts in the project's `.scratch/`). The loop code is reviewed as code — principles review, violation spec, re-run against the original scenario.

## License

MIT. See [LICENSE](LICENSE).
