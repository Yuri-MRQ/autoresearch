# Autoresearch

A Claude Code skill that turns Claude into an autonomous research agent. Give it a metric, point it at your code, and let it run experiments overnight — keeping improvements, reverting failures, and looping forever.

Inspired by [Karpathy's autoresearch](https://github.com/karpathy/autoresearch), generalized to work with **any measurable task**: ML training, API latency, bundle size, test coverage, algorithm performance, compression ratios, etc.

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

See [`skills/autoresearch/templates/config.yaml`](skills/autoresearch/templates/config.yaml) for a fully commented example.

## What Gets Created

When you run `/autoresearch`, these files are created in the same directory as your config:

| File | Git-tracked | Purpose |
|------|-------------|---------|
| `autoresearch.yaml` | Yes | Your configuration |
| `autoresearch.md` | Yes | Living document with wins, dead ends, observations |
| `autoresearch.jsonl` | No | Structured experiment log (one JSON line per run) |
| `.autoresearch_run.log` | No | Output from the last experiment run |

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

## Repo Structure

```
autoresearch/
  CLAUDE.md                           # Project instructions for Claude
  README.md                           # This file
  skills/autoresearch/
    SKILL.md                          # The core skill definition
    templates/
      config.yaml                     # Reference config with all options
      living-doc.md                   # Template for autoresearch.md
```

## License

MIT
