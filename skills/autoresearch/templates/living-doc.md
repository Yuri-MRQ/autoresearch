# Autoresearch — Living Document

## Objective

Optimize **{{metric_name}}** ({{metric_direction}} is better).

**Mutable files:** {{mutable_files}}
**Run command:** `{{run_command}}`
**Time budget:** {{time_budget}}s per experiment

## Current Best

| Metric | Value | Experiment | Description |
|--------|-------|------------|-------------|
| {{metric_name}} | {{baseline_value}} | #0 (baseline) | Initial baseline |

{{#if setup_instructions}}
## Exploration Findings

_Domain context discovered during bootstrap. Used by the loop for hypothesis generation._

_(to be filled during bootstrap)_

{{/if}}
## Ideas Backlog

_Promising ideas to try next. Add, reorder, or remove as you learn._

1. _(to be filled by the agent)_

## Priority Retry

_Ideas that showed improvement in parallel but couldn't be integrated (measured against stale baseline). Try these first in the next batch._

_(none yet)_

## Experiment Log

| # | Status | {{metric_name}} | Best | Description |
|---|--------|------|------|-------------|
| 0 | baseline | {{baseline_value}} | {{baseline_value}} | Initial baseline |

## Wins

_Changes that improved the metric. These are committed and kept._

_(none yet)_

## Dead Ends

_Changes that did not improve the metric. Reverted. Do not repeat these._

_(none yet)_

## Observations

_Patterns, insights, and strategy notes. Updated every 5 experiments._

_(none yet)_
