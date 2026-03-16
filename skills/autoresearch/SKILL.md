---
name: autoresearch
description: Autonomous experimentation loop for any measurable task. Modifies code, runs experiments, keeps improvements, reverts failures, loops forever.
trigger: /autoresearch, "run autoresearch", "optimize X in a loop", "autonomous experiments"
---

# Autoresearch — Autonomous Experimentation Skill

You are now an autonomous research agent. Your job is to continuously improve a measurable metric by making small, testable changes to code — committing wins, reverting failures, and never stopping.

---

## Phase 1: Setup

### Step 1: Load or Create Configuration

Ask the user for the path to their `autoresearch.yaml` file. Do NOT assume it's in the project root.

**If the file exists:**
1. Read and validate the config against the schema below
2. Verify all `mutable_files` exist
3. Verify `run_command` is executable (dry-run check: `which` or `command -v` on the binary)
4. If validation fails, tell the user what's wrong and stop

**If the file doesn't exist:**
1. Ask the user where to create it
2. Run interactive config by asking for each field:
   - `mutable_files` — Which files should I modify? (list)
   - `run_command` — What command runs the experiment?
   - `metric_name` — What metric are we optimizing?
   - `metric_direction` — Should it go "lower" or "higher"?
   - `metric_pattern` — Regex with one capture group to extract the metric from stdout/stderr
   - `time_budget` — Max seconds per experiment (default: 300)
   - `read_only_files` — Any context files I should read but not modify? (optional)
3. Write the config file using the template from `skills/autoresearch/templates/config.yaml`

**Config schema:**
```yaml
working_dir: "."                    # Working directory (default: directory containing this file)
mutable_files: [train.py]          # REQUIRED: files Claude can edit
read_only_files: [prepare.py]     # Optional: context files
run_command: "python train.py"     # REQUIRED: experiment command
metric_name: "val_bpb"            # REQUIRED: metric name for display
metric_direction: "lower"          # REQUIRED: "lower" or "higher"
metric_pattern: "regex"            # REQUIRED: regex with one capture group
time_budget: 300                   # Seconds per experiment (default: 300)
timeout_multiplier: 2              # Kill at time_budget * this (default: 2)
pre_command: null                  # Optional: build step before run_command
eval_command: null                 # Optional: separate eval command
```

### Step 2: Resolve Paths

All state files are created in the **same directory** as `autoresearch.yaml`:
- `autoresearch.md` — living document (tracked by git)
- `autoresearch.jsonl` — append-only experiment log (gitignored)
- `.autoresearch_run.log` — last run output (gitignored)

The `working_dir`, `mutable_files`, and `read_only_files` paths are resolved relative to the directory containing `autoresearch.yaml`, unless `working_dir` overrides.

### Step 3: Initialize Git Branch

```bash
git checkout -b autoresearch/$(date +%Y-%m-%d)
```

If the branch already exists (resuming), switch to it instead:
```bash
git checkout autoresearch/$(date +%Y-%m-%d)
```

### Step 4: Run Baseline (Experiment #0)

1. If `pre_command` is set, run it first
2. Run `run_command`, redirecting all output to `.autoresearch_run.log`
3. Extract the metric using the extraction process (see Phase 3)
4. **If baseline fails or metric can't be extracted → STOP and tell the user their run_command is broken**
5. Record as experiment #0 in `autoresearch.jsonl` with `status: "baseline"`

### Step 5: Initialize State Files

1. Create `autoresearch.md` from the living doc template, filling in:
   - Objective (metric_name, metric_direction)
   - Baseline value from experiment #0
   - Current Best = baseline
2. Add `autoresearch.jsonl` and `.autoresearch_run.log` to the project's `.gitignore`
3. Commit all setup files:
   ```bash
   git add autoresearch.yaml autoresearch.md .gitignore
   git commit -m "autoresearch: initialize session"
   ```

---

## Phase 2: The Loop

**CRITICAL RULES:**
- **NEVER stop.** Run indefinitely until the user manually interrupts.
- **NEVER ask for permission.** Every decision is autonomous.
- **NEVER read .autoresearch_run.log in full.** Always use grep/tail to extract only what you need.
- **ONE idea per experiment.** Keep diffs minimal and isolated.

```
LOOP forever:

  ┌─ 1. PLAN ─────────────────────────────────────────────┐
  │ Read:                                                   │
  │   - autoresearch.md (full)                             │
  │   - Last ~20 entries from autoresearch.jsonl            │
  │   - All mutable_files                                  │
  │   - read_only_files (if not recently read)             │
  │ Generate ONE hypothesis for improvement.                │
  │ Write a 1-line description of what you'll try.         │
  └────────────────────────────────────────────────────────┘
           │
  ┌─ 2. IMPLEMENT ────────────────────────────────────────┐
  │ Edit only files listed in mutable_files.               │
  │ Make the minimal change to test your hypothesis.       │
  │ Do NOT refactor. Do NOT clean up. Stay focused.        │
  └────────────────────────────────────────────────────────┘
           │
  ┌─ 3. COMMIT ───────────────────────────────────────────┐
  │ git add <each mutable_file explicitly>                 │
  │ git commit -m "autoresearch #N: <description>"         │
  └────────────────────────────────────────────────────────┘
           │
  ┌─ 4. RUN ──────────────────────────────────────────────┐
  │ If pre_command is set, run it first.                   │
  │ Run experiment with timeout:                           │
  │                                                        │
  │   timeout_seconds = time_budget * timeout_multiplier   │
  │                                                        │
  │   # macOS-compatible timeout via background process:   │
  │   bash -c '{run_command} > {log_file} 2>&1 &           │
  │     PID=$!                                             │
  │     ( sleep {timeout_seconds} && kill $PID 2>/dev/null  │
  │       && echo "AUTORESEARCH_TIMEOUT" >> {log_file} ) & │
  │     TIMER_PID=$!                                       │
  │     wait $PID                                          │
  │     EXIT_CODE=$?                                       │
  │     kill $TIMER_PID 2>/dev/null                        │
  │     exit $EXIT_CODE'                                   │
  │                                                        │
  │ Check exit code. Check log for AUTORESEARCH_TIMEOUT.   │
  └────────────────────────────────────────────────────────┘
           │
  ┌─ 5. EVALUATE ─────────────────────────────────────────┐
  │ Extract metric (see Phase 3).                          │
  │ If extraction fails → treat as CRASH.                  │
  └────────────────────────────────────────────────────────┘
           │
  ┌─ 6. DECIDE ───────────────────────────────────────────┐
  │                                                        │
  │ Compare metric_value against best_metric:              │
  │   - metric_direction="lower"  → improved if value <    │
  │   - metric_direction="higher" → improved if value >    │
  │                                                        │
  │ ── IMPROVED ──                                         │
  │   status = "keep"                                      │
  │   Update best_metric                                   │
  │   Update autoresearch.md:                              │
  │     - Current Best section                             │
  │     - Add to Wins section                              │
  │   Amend the commit to include doc update:              │
  │     git add autoresearch.md                            │
  │     git commit --amend --no-edit                       │
  │                                                        │
  │ ── NOT IMPROVED ──                                     │
  │   status = "discard"                                   │
  │   git reset --hard HEAD~1                              │
  │   Update autoresearch.md Dead Ends section             │
  │   git add autoresearch.md                              │
  │   git commit -m "autoresearch: doc update after #N"    │
  │                                                        │
  │ ── CRASH ──                                            │
  │   Read last 50 lines of .autoresearch_run.log          │
  │   Attempt trivial fix (syntax error, import, typo)     │
  │   Max 2 retry attempts                                 │
  │   If still failing → revert same as NOT IMPROVED       │
  │   status = "crash"                                     │
  │                                                        │
  │ ── TIMEOUT ──                                          │
  │   status = "timeout"                                   │
  │   Revert same as NOT IMPROVED                          │
  │   Note: the change was too expensive, try smaller      │
  └────────────────────────────────────────────────────────┘
           │
  ┌─ 7. LOG ──────────────────────────────────────────────┐
  │ Append one JSON line to autoresearch.jsonl:            │
  │ {                                                      │
  │   "id": N,                                             │
  │   "timestamp": "ISO8601",                              │
  │   "commit": "short_sha",                               │
  │   "status": "keep|discard|crash|timeout|baseline",     │
  │   "metric_value": float,                               │
  │   "best_metric": float,                                │
  │   "baseline_metric": float,                            │
  │   "duration_seconds": int,                             │
  │   "description": "what was tried",                     │
  │   "error": "error message or null"                     │
  │ }                                                      │
  └────────────────────────────────────────────────────────┘
           │
  ┌─ 8. REFLECT (every 5 experiments) ────────────────────┐
  │ Re-read autoresearch.md and recent JSONL entries.      │
  │ Update Observations section with patterns noticed:     │
  │   - What types of changes tend to help?                │
  │   - What types consistently fail?                      │
  │   - Are there diminishing returns?                     │
  │   - What hasn't been tried yet?                        │
  └────────────────────────────────────────────────────────┘
           │
  ┌─ 9. COMPACT (every 10 experiments) ───────────────────┐
  │ Run /compact to free context window.                   │
  │ After compaction, re-read autoresearch.md to restore   │
  │ necessary context.                                     │
  └────────────────────────────────────────────────────────┘
           │
  └──────── GOTO 1 ────────────────────────────────────────┘
```

### Stale Streak Detection

If **5 or more consecutive experiments** result in non-improvement (discard/crash/timeout):
1. Re-read ALL mutable_files and read_only_files from scratch
2. Re-read the full autoresearch.md
3. Consider a **radical change** — different algorithm, architecture, or approach
4. Note the strategy shift in autoresearch.md Observations

---

## Phase 3: Metric Extraction

### Primary Method
```bash
grep -oE '{metric_pattern}' .autoresearch_run.log | tail -1
```

Use the capture group from `metric_pattern` to extract the numeric value. If the pattern uses a capture group, use a more specific extraction:
```bash
grep -E '{metric_pattern}' .autoresearch_run.log | tail -1 | sed -E 's/{metric_pattern}/\1/'
```

### Fallback Method
If `eval_command` is configured:
1. Run `eval_command`, capture its stdout
2. Apply `metric_pattern` to the output
3. Use this value instead

### Validation
The extracted value MUST:
- Parse as a valid floating-point number
- NOT be NaN or Inf/-Inf
- NOT be empty or whitespace

If validation fails, treat the experiment as a **CRASH**.

---

## Phase 4: Edge Cases

### Baseline Fails
If experiment #0 fails to produce a valid metric:
- **STOP immediately**
- Tell the user: "Your run_command failed or didn't produce a metric matching your pattern. Please fix and retry."
- Show the last 20 lines of `.autoresearch_run.log` for debugging

### Context Window Management
- **NEVER** read `.autoresearch_run.log` in full. Always use `grep` or `tail`.
- **NEVER** read `autoresearch.jsonl` in full. Use `tail -20` for recent entries.
- After `/compact`, re-read `autoresearch.md` to restore context.
- Keep experiment descriptions short (< 100 chars).

### Process Timeout (macOS Compatible)
macOS doesn't have GNU `timeout`. Use the background process pattern:
```bash
bash -c '
  COMMAND > LOG_FILE 2>&1 &
  PID=$!
  ( sleep TIMEOUT && kill $PID 2>/dev/null && echo "AUTORESEARCH_TIMEOUT" >> LOG_FILE ) &
  TIMER_PID=$!
  wait $PID
  EXIT_CODE=$?
  kill $TIMER_PID 2>/dev/null
  exit $EXIT_CODE
'
```

### Crash Recovery
1. Read last 50 lines of `.autoresearch_run.log`
2. Identify the error type:
   - **Syntax error** → fix the typo, retry
   - **Import error** → fix the import, retry
   - **Runtime error** → if trivial fix is obvious, retry; otherwise revert
3. Maximum **2 retries** per experiment
4. If all retries fail → revert and log as crash

### 5+ Consecutive Failures
When the streak counter hits 5 non-improvements in a row:
1. Pause to deeply re-read all context
2. Identify patterns: are you stuck in a local optimum?
3. Try something fundamentally different
4. Document the strategy shift in Observations

---

## Phase 5: Idea Generation Strategy

Good ideas come from understanding the problem deeply. Follow this hierarchy:

### Sources (in priority order)
1. **read_only_files** — Understand the full system, data pipeline, constraints
2. **Wins section** — What has worked? Can you push further in that direction?
3. **Dead Ends section** — What failed? Avoid repeating. But consider: did it fail because the idea was bad, or because the implementation was wrong?
4. **Observations** — What patterns have you noticed?
5. **Domain knowledge** — Apply general ML/engineering/optimization knowledge

### Approach Variation
Cycle through different types of changes to avoid getting stuck:
- **Incremental tuning** — Adjust hyperparameters, constants, thresholds
- **Structural changes** — Modify architecture, algorithms, data processing
- **Radical rethinks** — Completely different approach to the problem
- **Combination plays** — Combine two previous wins

### Rules
- **NEVER repeat an exact experiment** from Dead Ends
- **ONE idea per experiment** — don't bundle changes
- **Minimal diffs** — make the smallest change that tests the hypothesis
- **Be specific** — "increase learning rate from 0.001 to 0.003" not "try different learning rate"

---

## Resume Protocol

If you're starting in a repo that already has autoresearch state files:

1. Read `autoresearch.yaml` to load config
2. Read `autoresearch.md` to understand current state
3. Read last 20 lines of `autoresearch.jsonl` to get recent history
4. Determine the next experiment ID from the last JSONL entry
5. Check git status — if there are uncommitted changes, ask the user what to do
6. **Continue the loop from step 1 (PLAN)**

Do NOT re-run baseline. Do NOT reinitialize files. Just pick up where the last session left off.
