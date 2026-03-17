# Autoresearch

A Claude Code skill that turns Claude into an autonomous research agent. Give it a metric, point it at your code, and let it run experiments overnight — keeping improvements, reverting failures, and looping forever. All complex shell commands are pre-approved during setup, so the loop runs without permission prompts.

Generalized to work with **any measurable task**: ML training, API latency, bundle size, test coverage, algorithm performance, compression ratios, etc.

### Inspired by

- [karpathy/autoresearch](https://github.com/karpathy/autoresearch) — The original. AI agents running autonomous research on single-GPU nanochat training.
- [trevin-creator/autoresearch-mlx](https://github.com/trevin-creator/autoresearch-mlx) — Apple Silicon (MLX) port, no PyTorch required. Demonstrated hardware-specific optimization strategies.
- [davebcn87/pi-autoresearch](https://github.com/davebcn87/pi-autoresearch) — Extension + skill architecture for pi, separating domain-agnostic infrastructure from domain-specific logic.

## How It Works

```
LOOP forever:
  1. Read context (past wins, dead ends, code)
  2. Form a hypothesis
  3. Make a minimal code change
  4. Commit
  5. Run the experiment
  6. Extract the metric
  7. If improved → keep. If not → revert.
  8. Repeat.
```

All state is tracked in git and two files (`autoresearch.md` for context, `autoresearch.jsonl` for structured logs), so the agent can resume across sessions and context resets.

## Installation

### For all your projects (recommended)

```bash
mkdir -p ~/.claude/skills
ln -s /path/to/autoresearch/skills/autoresearch ~/.claude/skills/autoresearch
```

Replace `/path/to/autoresearch` with wherever you cloned this repo.

### For a specific project only

```bash
mkdir -p /path/to/your-project/.claude/skills
ln -s /path/to/autoresearch/skills/autoresearch /path/to/your-project/.claude/skills/autoresearch
```

### Verify

Start Claude Code and run `/context` — you should see `autoresearch` listed as an available skill.

## Usage

### Quick Start

1. Open Claude Code in your project
2. Type `/autoresearch`
3. Claude will ask you for the path to your `autoresearch.yaml` config file
4. If the file doesn't exist, Claude will walk you through creating one interactively
5. Claude runs a baseline experiment, then starts the autonomous loop

### With a Config File

Create an `autoresearch.yaml` in your project:

```yaml
mutable_files:
  - train.py

read_only_files:
  - prepare.py

run_command: "python train.py"
metric_name: "val_bpb"
metric_direction: "lower"          # "lower" or "higher"
metric_pattern: "^val_bpb:\\s*([\\d.eE+-]+)"
time_budget: 300                   # seconds per experiment
```

Then run `/autoresearch` and point it to the file.

### Config Reference

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `mutable_files` | Yes | — | Files Claude can modify |
| `run_command` | Yes | — | Command to run each experiment |
| `metric_name` | Yes | — | Name of the metric (for display) |
| `metric_direction` | Yes | — | `"lower"` or `"higher"` |
| `metric_pattern` | Yes | — | Regex with one capture group to extract the metric |
| `read_only_files` | No | `[]` | Context files (read but not modified) |
| `time_budget` | No | `300` | Max seconds per experiment |
| `timeout_multiplier` | No | `2` | Kill at `time_budget * this` |
| `working_dir` | No | `"."` | Working directory (relative to config file) |
| `pre_command` | No | `null` | Command to run before each experiment |
| `eval_command` | No | `null` | Separate eval command (metric extracted from its output) |
| `parallel_experiments` | No | `1` | Number of experiments to run concurrently |
| `number_of_experiments` | No | `null` | Max experiments before stopping (`null` = infinite) |
| `orientations` | No | `null` | High-level strategy guidance for the agent |
| `setup_instructions` | No | `null` | One-time bootstrap instructions (create tooling, explore domain) |

See [`skills/autoresearch/templates/config.yaml`](skills/autoresearch/templates/config.yaml) for a fully commented example.

## What Gets Created

When you run `/autoresearch`, these files are created in the same directory as your config:

| File | Git-tracked | Purpose |
|------|-------------|---------|
| `autoresearch.yaml` | Yes | Your configuration |
| `autoresearch.md` | Yes | Living document with wins, dead ends, observations |
| `autoresearch.jsonl` | No | Structured experiment log (one JSON line per run) |
| `.autoresearch_run.log` | No | Output from the last experiment run |
| `autoresearch_run.sh` | No | Generated script: runs experiment with timeout + timing |
| `autoresearch_extract.sh` | No | Generated script: extracts metric from run output |

All experiments happen on a dedicated `autoresearch/<date>` git branch.

## Examples

### Optimize ML Training

```yaml
mutable_files: [train.py]
read_only_files: [prepare.py, model.py]
run_command: "python train.py"
metric_name: "val_loss"
metric_direction: "lower"
metric_pattern: "val_loss:\\s*([\\d.eE+-]+)"
time_budget: 300
```

### Minimize Bundle Size

```yaml
mutable_files: [webpack.config.js, src/index.ts]
run_command: "npm run build 2>&1 && du -sb dist/ | cut -f1"
metric_name: "bundle_bytes"
metric_direction: "lower"
metric_pattern: "^(\\d+)"
time_budget: 60
```

### Maximize Test Coverage

```yaml
mutable_files: [src/utils.ts, src/api.ts]
run_command: "npm test -- --coverage 2>&1"
metric_name: "coverage_pct"
metric_direction: "higher"
metric_pattern: "All files.*?\\|\\s*([\\d.]+)"
time_budget: 120
```

### Optimize Algorithm Performance

```yaml
mutable_files: [solver.py]
read_only_files: [benchmark_data.py]
run_command: "python benchmark.py"
metric_name: "ops_per_second"
metric_direction: "higher"
metric_pattern: "throughput:\\s*([\\d.eE+-]+)\\s*ops/s"
time_budget: 30
```

## Parallel Mode

By default, autoresearch runs one experiment at a time. Set `parallel_experiments` to run multiple experiments simultaneously:

```yaml
parallel_experiments: 3    # Run 3 experiments at once
```

### How It Works

When `parallel_experiments` is greater than 1, the orchestrator:

1. **Plans a batch** of N independent hypotheses
2. **Dispatches N worker agents** in parallel, each in its own git worktree
3. **Collects results** from all workers
4. **Integrates the best winner** by cherry-picking its commit onto the main branch

### Conservative Integration

Since all N experiments in a batch run against the same baseline, only the single best improvement is kept. Other "winners" may conflict with the best one's changes, so they're placed in a **Priority Retry** queue and tried first in the next batch.

This means:
- **Throughput**: N experiments run in the time of 1 (wall-clock)
- **Tradeoff**: At most 1 winner per batch (other winners are retried next batch)
- **Net effect**: Significantly faster exploration, especially when most experiments fail

### Example

With `time_budget: 300` (5 minutes per experiment):

| Mode | Experiments/hour | Winners integrated |
|------|------------------|--------------------|
| Sequential (`parallel_experiments: 1`) | ~12 | Up to 12 |
| Parallel (`parallel_experiments: 3`) | ~36 | Up to 12 (1 per batch) |
| Parallel (`parallel_experiments: 5`) | ~60 | Up to 12 (1 per batch) |

The parallel mode runs more experiments, discovering improvements faster even though only one is integrated per batch.

## Bootstrap Phase

Some workflows need one-time setup before the experiment loop can be effective — querying databases to understand schemas, profiling data files, or creating exploration scripts. The `setup_instructions` field drives an optional **Phase 1.5: Bootstrap** that runs between setup and the loop.

### When to Use It

- **Data science / feature engineering** — explore tables, profile columns, understand distributions before optimizing
- **Complex systems** — create diagnostic scripts to inspect runtime behavior
- **Unfamiliar codebases** — build analysis tools to map dependencies or hotspots

### How It Works

1. **Create Tooling** — the agent builds helper scripts described in your instructions
2. **Suggest Permissions** — prints recommended Claude Code permission rules and waits for your approval (the only time the agent pauses for input)
3. **Initial Exploration** — runs the tools, summarizes findings
4. **Document Findings** — writes an Exploration Findings section in `autoresearch.md`
5. **Commit** — `git commit -m "autoresearch: bootstrap — tooling and exploration"`

After bootstrap, the loop uses Exploration Findings as a source for hypothesis generation. If you resume a session where bootstrap already ran, it won't run again.

### Example: Data Science Feature Engineering

```yaml
mutable_files:
  - feature_pipeline.py

read_only_files:
  - model.py
  - evaluate.py

run_command: "python evaluate.py"
metric_name: "auc_roc"
metric_direction: "higher"
metric_pattern: "AUC-ROC:\\s*([\\d.eE+-]+)"
time_budget: 120

orientations: |
  Focus on feature engineering in feature_pipeline.py.
  Do not change the model architecture or hyperparameters.
  Prioritize features derived from user behavior tables.

setup_instructions: |
  Create an explore_data.py script that connects to our Athena database
  (database: 'prod_analytics', region: 'us-east-1').
  List all available tables, sample 100 rows from each, and summarize:
  - Column names and types
  - Null rates
  - Cardinality for categorical columns
  - Basic distributions for numeric columns
  Focus especially on tables with 'user_' or 'event_' prefixes.
```

## Repo Structure

```
autoresearch/
  CLAUDE.md                           # Project instructions for Claude
  README.md                           # This file
  ai/
    parallel-experiments.md           # Design doc for parallel execution
  skills/autoresearch/
    SKILL.md                          # The core skill definition
    templates/
      config.yaml                     # Reference config with all options
      living-doc.md                   # Template for autoresearch.md
```

## License

MIT
