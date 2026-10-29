# live-loop

Two agent skills: a live pass/fail loop for desktop apps, and a single-writer pipeline for shipping spec batches.

Two skills ship in this pack:

| Skill | What it does |
|---|---|
| `live-loop` | One unattended command: spawn a desktop app, capture timestamped output, decide pass/fail from markers, tree-kill, keep the logs. |
| `monster-longrun` | One session, many specs: single-writer pipeline, numbered gates the orchestrator re-runs, eyes-on verification on a real artifact. |

## Why

Some bugs reproduce only in a real window: a native app host, a game-engine renderer, a sidecar process. A unit test cannot see them. A manual session can, but it scales poorly and leaves no evidence.

An agent has the same problem from the other side. Green tests are a claim, not a fact: an agent that reports "done" needs observable evidence it can re-check, not a summary line. The live loop gives the agent that evidence. Before diagnosing a live-only bug, it builds a fast deterministic signal. A 30-second flaky loop is barely better than no loop. A 2-second deterministic loop changes the diagnosis.

## How the live loop works

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

### Runner contract

| Flag | Effect |
|---|---|
| `--duration N` | Kill the app after N seconds. |
| `--until <regex>` | Stop early when a log line matches the marker. |
| `--keep-going` | Continue to the duration after a match. |
| `--kill-<target>-at N` | Fault injection: kill a chosen process at second N. |
| Stress modes | Run a stressor alongside the app to exercise respawn and recovery. |

Exit code 0 means the run completed and the marker matched. Everything else is a non-zero exit plus surviving logs.

### Principles

- **Scope lock** — before the first run, fix the launch chain, the marker set, and the kill set. No drifting scope mid-loop.
- **Markers** — pass, fail, and progress are regex lines in the live log filter, not vibes.
- **Tree-kill** — map the process forest first (wrappers, children, window owners). A `child.kill()` on a wrapper orphans the real process, so the loop always kills from the root.
- **Forensics** — every run leaves a full timestamped log on disk, per process, with unique names that survive the kill.
- **Fault injection** — kill a chosen process at a chosen second, on demand, instead of waiting for a rare crash.
- **Time-box** — every run has a hard duration. A loop that can hang is not a loop.

### Build order

1. Map the launch chain and verify it with a live probe.
2. Write the minimal runner: spawn, tee output, tree-kill on timeout.
3. Add the marker filter and `--until`.
4. Add fault injection and stressors, one lever at a time.
5. Smoke-test the loop itself: marker matched, clean kill, logs complete.

Then attack the loop: does the kill kill the whole forest? Can the marker regex match a line that never comes from the real event? What did the original bug report include that the loop cannot produce?

## monster-longrun

A pipeline for shipping a batch of specs in one session without losing control:

1. Dispatch one subagent per spec, each with a locked file scope.
2. Run the numbered gates in order. The orchestrator re-runs a gate only after the previous gate passes.
3. Verify on a real artifact with eyes on the screen — a built window, a rendered frame, a served page — not on a status message.
4. Run a violation pass that expects new findings and fixes them.

## Install

Prerequisites: None beyond an agent that reads `SKILL.md`.

```
cp -r live-loop ~/.agents/skills/
cp -r monster-longrun ~/.agents/skills/
```

In pi, invoke with `/skill:live-loop` or `/skill:monster-longrun`.

## Contributing

File issues in the protokol11 repo. Test a change by running the loop against a real app and checking the exit code and the surviving logs.

## License

MIT. See [LICENSE](LICENSE).
