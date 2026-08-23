# Hivemind

**Claude Code plans and reviews. Free opencode workers do the grunt work.**

Your expensive model's context is the scarce resource — not its intelligence. Hivemind moves
mechanical work (multi-lens review, research sweeps, bulk migrations, partitioned test runs) onto
disposable [opencode](https://opencode.ai) workers running on free models, and returns **exactly one
compact JSON line per worker**. Raw agent streams never touch the orchestrator's context.

The orchestrator stays the only planner, the only reviewer, and the only merger. Workers are
cattle, not colleagues.

```
/hive migrate every fetch() call in src/ to the shared http client
   |
   +-- worktree-1  coder   -> { ok: true, tokens: 8214, cost_usd: 0, duration_ms: 41203 }
   +-- worktree-2  coder   -> { ok: true, tokens: 6980, cost_usd: 0, duration_ms: 38771 }
   +-- worktree-3  tester  -> { ok: true, result: "PASS 214/214", ... }
   |
   +-- you review every diff, you merge
```

## Install

As a Claude Code plugin:

```
/plugin marketplace add Hanishchow/hivemind
/plugin install hivemind@hivemind
```

Or by hand:

```bash
git clone https://github.com/Hanishchow/hivemind
cp -r hivemind/skills/hivemind ~/.claude/skills/
cp hivemind/commands/*.md ~/.claude/commands/
cp hivemind/skills/hivemind/assets/agents/*.md ~/.config/opencode/agent/
export HIVEMIND_HOME="$HOME/.claude/skills/hivemind"
```

## Prerequisites

Hivemind is an orchestration layer, not a model. It needs:

| Requirement | Notes |
|---|---|
| Node.js >= 18 | The scripts use `fetch` and `node:timers/promises`. |
| `opencode` CLI on `PATH` | Install and authenticate it yourself. |
| Default model `opencode/mimo-v2.5-free` | A free tier offered by **opencode**, not by Anthropic. Availability, rate limits, and pricing are theirs to change — override with `--model`. |
| Windows: `OPENCODE_GIT_BASH_PATH` | Point at `C:\Program Files\Git\bin\bash.exe`, set persistently. |

Worker traffic goes to opencode's endpoints. **Don't delegate secrets or private code you wouldn't
send there.**

## Commands

| Command | Shape |
|---|---|
| `/hive <task>` | Auto-router — classifies the task, then picks one of the below. Start here. |
| `/oc <task>` | One worker, one task. |
| `/swarm <task>` | 2-5 independent subtasks in parallel, each in its own git worktree. |
| `/review-panel <diff>` | Four reviewers (correctness / security / performance / style), consensus-filtered. |
| `/research-sweep <question>` | 3-5 scouts on different angles, synthesized, every claim cited. |
| `/migration <task>` | Batched per-worktree migration workers with a sequenced merge. |
| `/test-fleet <target>` | A test suite partitioned across parallel tester workers. |

Three worker personas ship with it: **scout** (read-only, cannot write or run shell), **coder**
(writes, confined to one worktree), **tester** (runs tests, never edits source).

## The contract

Every worker is invoked through one hardened entry point, and returns one line:

```bash
node "$HIVEMIND_HOME/scripts/oc-worker.mjs" --agent scout --run r1 --label auth "Where is the session token validated?"
```

```json
{"ok":true,"result":"...","tokens":{"total":8214},"cost_usd":0,"duration_ms":41203,"label":"auth","agent":"scout"}
```

Failures return the same shape: `{ ok: false, stage: "args"|"exec"|"api"|"parse"|"empty", error }`
with stderr capped at 300 chars. Nothing else reaches the orchestrator — no NDJSON, no stack
traces, no partial streams. `--run` / `--label` append lifecycle events to `.runs/<id>.jsonl`, so
`oc-status.mjs` can recover fleet progress even after the orchestrator loses its context.

## Fallback ladder

1. Worker returns `ok:false` -> re-invoke once against the same directory.
2. Still failing -> the orchestrator does that subtask itself, marked `[orchestrator-sourced]`.
3. opencode down entirely -> announce it, abandon the workers, do the task directly.

A swarm should never fail a task the orchestrator could have done alone.

## Benchmarking

`scripts/bench/` ships an A/B/C harness — A: Claude solo, B: opencode solo, C: Claude orchestrating
two workers — appending `(ts, config, tokens, cost, duration)` records to `bench-results.jsonl`.
Grade the artifacts blind with `scripts/bench/grader-prompt.md`, which hides everything but the task
spec and the output. Run it on your own workload; results depend far more on your task mix than on
anything published here.

## Known limits

- Free-tier rate limits will 429 under heavy swarms. Space out retries.
- Worker quality varies. Review every diff — that's why the orchestrator is the only merger.
- Config C still spends real orchestrator tokens (~1-2k/task) on coordination.
- `opencode-go/*` models require workspace billing. Avoid them if you want the $0 path.

## Status

Also submitted upstream, both open at the time of writing:
[anthropics/skills#1628](https://github.com/anthropics/skills/pull/1628) and
[alirezarezvani/claude-skills#979](https://github.com/alirezarezvani/claude-skills/pull/979).

## License

Apache 2.0 — see [LICENSE](LICENSE).
