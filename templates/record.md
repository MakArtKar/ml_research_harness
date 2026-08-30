---
# Validate against schemas/record.schema.json. Immutable once the iteration ends.
experiment: EXP-NNNN
iteration: 1
commit: ""              # exact commit hash the run used
branch: exp/EXP-NNNN-short-slug
dirty: false
env:
  lockfile_hash: ""
  python: ""
  torch: ""
  cuda: ""
  hardware: ""
config:
  path: ""              # config file in the repo
  resolved_hash: ""     # hash of the fully resolved config
setup:                  # materialized for this run — not a diff
  model: ""
  data: ""              # dataset with version/hash and splits
  training:
    hardware: ""
    hyperparameters: {} # batch_size, lr, schedule, steps/epochs, seed, ...
  evaluation:
    hardware: ""
    benchmarks: []
    params: {}          # batch_size, temperature, max_new_tokens, ...
launch_command: ""
artifacts:              # URIs only — heavy artifacts are never committed
  wandb: ""
  checkpoints: ""
  logs: ""
metrics: {}             # final numbers; required by status `completed`
verdict: null           # accept | revise | reject | inconclusive | escalate-to-human
---

# EXP-NNNN / iNN — run record

## Run notes

<!-- What happened during the run: restarts, observations, watchdog events. -->

## Incidents

<!-- Failures and how they were handled. -->

## Analysis

<!-- Per-iteration analysis backing the iteration verdict. The final synthesis
     across iterations goes to report.md. -->
