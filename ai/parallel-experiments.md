# Parallel Experiments Architecture

Design doc for the parallel execution mode in autoresearch.

## Why Parallel?

Sequential execution runs ~12 experiments/hour at 5min each. With `parallel_experiments: N`, we dispatch N agents simultaneously, each in its own git worktree, multiplying throughput by N.

## Architecture

```
ORCHESTRATOR (main agent, runs the skill)
  │
  ├─ Setup (same as sequential: config, baseline, git branch)
  │
  └─ LOOP forever:
       │
       ├─ 1. PLAN BATCH — Generate N independent hypotheses
       │
       ├─ 2. DISPATCH — Launch N agents in parallel (isolation: "worktree")
       │     ├─ Agent 1: hypothesis A → implement → commit → run → evaluate → report
       │     ├─ Agent 2: hypothesis B → implement → commit → run → evaluate → report
       │     └─ Agent N: hypothesis N → implement → commit → run → evaluate → report
       │
       ├─ 3. COLLECT — Gather results from all N agents
       │
       ├─ 4. INTEGRATE — Best-first processing:
       │     ├─ Best improvement → cherry-pick onto main branch
       │     └─ Other "winners" → Priority Retry queue (stale baseline)
       │
       ├─ 5. LOG — Append all N results to autoresearch.jsonl
       │
       ├─ 6. REFLECT/COMPACT — Same cadence as sequential
       │
       └─ GOTO 1
```

## Why Worktrees?

Each agent needs to edit the same `mutable_files` independently. Git worktrees give each agent a full copy of the repo on its own branch, so there are no conflicts during parallel execution. The orchestrator cherry-picks winning commits back to the main autoresearch branch.

## Conservative Integration Strategy

When multiple experiments in a batch improve the metric, we can only trust the first one. All were measured against the *same* baseline, so subsequent "wins" may conflict with the best winner's changes.

Strategy:
1. Sort batch results by improvement magnitude (best first)
2. Cherry-pick the best winner onto the main branch
3. **Discard remaining winners** — they were measured against a stale baseline
4. Those ideas go into a "Priority Retry" queue to be tried in the next batch

This is conservative but correct. No false wins.

## Worker Agent Contract

### Input (what each worker receives)

- Full config: `run_command`, `metric_pattern`, `time_budget`, `timeout_multiplier`, etc.
- Its specific hypothesis and 1-line description
- Current `mutable_files` content (so it knows what to edit)
- Current `best_metric` to compare against
- Experiment ID for the commit message

### Output (what each worker returns)

JSON object:
```json
{
  "metric_value": 0.9910,
  "status": "keep|discard|crash|timeout",
  "description": "increase hidden dim to 512",
  "duration_seconds": 287,
  "error": null
}
```

The worker's worktree branch is automatically available for cherry-picking if status is "keep".

### Worker Behavior

1. Enters worktree (automatic via `isolation: "worktree"`)
2. Reads the mutable files
3. Implements the hypothesis (minimal change)
4. Commits: `git add <files> && git commit -m "autoresearch #N: description"`
5. Runs the experiment with the macOS-compatible timeout pattern
6. Extracts metric via grep/sed
7. Compares against best_metric to determine status
8. Returns structured JSON result

## JSONL Batch Semantics

Parallel experiments share a `batch_id`:

```json
{"id": 4, "batch_id": 2, "status": "keep", "metric_value": 0.9910, ...}
{"id": 5, "batch_id": 2, "status": "discard", "metric_value": 0.9950, ...}
{"id": 6, "batch_id": 2, "status": "priority_retry", "metric_value": 0.9920, ...}
```

- `batch_id` groups experiments that ran simultaneously
- Sequential mode (`parallel_experiments: 1`) omits `batch_id` or sets it equal to `id`

## Edge Cases

### All experiments fail
- All get logged as crash/timeout/discard
- Stale streak counter increments by N
- Triggers radical rethink at threshold

### Multiple winners in a batch
- Only the best is cherry-picked
- Others are logged with `status: "priority_retry"` and added to the Priority Retry section in `autoresearch.md`
- Priority Retry ideas are tried first in the next batch

### Crash in a worktree
- Worker returns `status: "crash"` with error details
- Worktree is automatically cleaned up (no commits to cherry-pick)
- Logged to Dead Ends

### Worktree cleanup
- Worktrees with no changes are auto-cleaned by the Agent tool
- Winning worktrees: after cherry-pick, the branch/worktree can be discarded
- Failed worktrees: automatically cleaned up

## Fallback to Sequential

When `parallel_experiments: 1` (default), the loop behaves identically to the original sequential mode — no worktrees, no agents, direct execution. This preserves full backward compatibility.
