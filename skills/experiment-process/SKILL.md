---
name: experiment-process
description: >-
  The process for implementing, verifying, running, and analyzing an ML
  experiment in an agent loop: verification phases 0-4 ordered by cost
  (deterministic checks first, AI review last), machine-readable check
  verdicts, iteration verdicts with a revise loop and spec-diff approval,
  checker isolation, and resuming interrupted loops. Use when executing a
  planned experiment, generating or running verification checks, monitoring
  training, judging results, or continuing an interrupted experiment.
---

# Experiment Process

How to take an experiment from `status: planned` to a terminal verdict.
Companion skills: experiment-logging (where/what/how/when of documentation)
and experiment-planning (the spec this process verifies against).

## The loop

One iteration:

```
planned ──implement──▶ phase 1 gate ──launch──▶ phase 2 gate (sanity)
   ▲                                                  │
   │                                        phase 3 (watchdog, full run)
   │                                                  │
   │                                          phase 4 gate + analysis
   │                                                  │
   └── spec-diff approval ◀── verdict: revise ◀── iteration verdict
                                                      │
                                  accept / reject / inconclusive / escalate
                                                      ▼
                                        report, knowledge merge, registry
```

Phase 0 runs once before the first iteration (and re-runs on any spec
change). A failed gate returns the loop to implementation within the same
iteration if the fix is mechanical, or produces a `revise` verdict if the
spec must change. Order is strictly by cost: fail in seconds before failing
in minutes before burning a full run.

## Roles and isolation

- **Executor** — implements the experiment and drives the loop. May not
  read or modify `.harness/checkers/`, and never sees verifier prompts
  (otherwise it optimizes for passing checks, not for correctness).
- **Checker agent** — a separate agent (fresh context) that generates and
  maintains the deterministic checkers in `.harness/checkers/`, tailored to
  this repo, guided by `references/checker-catalog.md`. Metric-backed checks
  (phase-3 bands, phase-4 assertion inputs) are derived from the metric
  registry `knowledge/metrics.md`, never invented. Checkers are versioned as
  code; any checker diff itself goes through intent review.
- **Reviewers / diagnosticians** — isolated-context agents for spec review,
  code review, intent diff, failure diagnosis, and results review. Each gets
  only the inputs named for its check in the catalog, never the executor's
  conversation.
- `.harness/checkers/`, `experiments/`, and `knowledge/` are always outside
  the spec's code scope allowlist.

## Verification phases

Full checker list with pass criteria: `references/checker-catalog.md`.
Summary — **[D]** deterministic, **[AI]** isolated-context review:

| Phase | Gate | Checks |
|---|---|---|
| **0 — before implementation** | spec is executable | [D] spec schema valid; parent and compare_to records exist with real numbers; clean env recorded. [AI] spec review (falsifiable, measurable, minimal scope) |
| **1 — after implementation, before run** | code matches spec | [D] lint/typecheck/tests; scope enforcement; config diff matches declaration; data checks; component connectivity. [AI] ML-checklist code review; intent diff |
| **2 — first training steps** | training is sane | [D] smoke run, loss-at-init, loss decreases, NaN/Inf hooks, dead-param and grad-norm checks, single-batch overfit, seed reproducibility, shuffled-labels negative control, short-horizon ablation sanity, throughput/memory budget, checkpoint round-trip, eval-mode check. [AI] diagnosis on failure only — never a gate |
| **3 — during training** | run stays healthy | [D] watchdog: NaN, grad-norm band, loss spikes, LR schedule, train/val gap, throughput/memory stability, heartbeat, early-kill rules from the spec. [AI] triggered on soft anomalies; advisory, gates only on critical |
| **4 — after training** | result is judged | [D] artifact manifest satisfied; all spec assertions evaluated vs compare_to; final checkpoint round-trip. [AI] results review → iteration verdict proposal |

## Check verdicts

Every check — deterministic or AI — writes one JSON file to
`experiments/EXP-NNNN-slug/runs/iNN/checks/`, validating against
`schemas/verdict.schema.json`:

```json
{
  "id": "p2/single-batch-overfit",
  "phase": 2,
  "verdict": "pass",
  "severity": "critical",
  "evidence": "loss 2.31 -> 0.004 in 180 steps (threshold: <0.01 in 300)",
  "checker": "deterministic",
  "timestamp": "2026-08-31T12:00:00Z"
}
```

The loop reacts to `verdict` + `severity`: a critical fail blocks the phase
gate; warnings accumulate into the analysis; the AI diagnostician reads
`evidence` when invoked on a failure.

## Iteration verdict

After phase 4, set exactly one verdict in the iteration's record:

- **accept** — decision rule satisfied → terminal `accepted`.
- **revise** — fixable within the same hypothesis → prepare the spec diff
  (material changes + assumptions, per the experiment-planning skill), get
  human approval, bump iteration, return to `planned`.
- **reject** — decision rule says the hypothesis failed → terminal
  `rejected`.
- **inconclusive** — decision rule says stop without a clean answer →
  terminal `inconclusive`.
- **escalate-to-human** — executor and reviewers disagree about the spec, or
  the decision rule does not cover the outcome. The human's decision maps to
  one of the above.

On any terminal verdict: write `report.md`, merge conclusions into
`knowledge/` (experiment-planning skill), update the registry row, land docs
on `main`, decide the code merge.

## Engineering pipeline (no loop)

Engineering changes (`ENG-NNNN` — no hypothesis about model quality; see the
experiment-planning boundary rule) skip the experiment loop and run a
reduced pipeline. Verdicts use the same JSON format and land in
`experiments/engineering/ENG-NNNN/checks/`:

| Id | Check | Applies to |
|---|---|---|
| `e/static` | lint, typecheck, unit tests | all |
| `e/scope` | files touched match the change's declared intent | all |
| `e/smoke` | training smoke run completes | all |
| `e/equivalence` | re-run smoke/eval at a reference checkpoint: previously registered metrics unchanged within tolerance | result-neutral changes (refactors, pure instrumentation) |
| `e/metric-known-value` | unit check of the metric on inputs with a known value | new/changed metrics |
| `e/metric-registered` | registry entry exists in `knowledge/metrics.md`, its checker is generated, references backfilled or waived | new/changed metrics |

Mark the engineering-log row verified only when all applicable checks pass.

## Resuming an interrupted loop

1. Read the registry row and spec frontmatter → experiment, `status`,
   `iteration`.
2. Find the last `[EXP-NNNN.iN]` commit on the `exp/` branch and the current
   iteration's `runs/iNN/` contents (record fields filled, check verdicts
   present).
3. Continue from the first incomplete step of that iteration's cycle: the
   status says which gate comes next; the check verdicts say which checks in
   the current phase already passed. Re-run a phase's checks if the commit
   hash has changed since they were written.

## Cost discipline

- Never start a full run while a cheaper phase is failing.
- Respect the spec's early-kill thresholds — hopeless runs must die early.
- Full-length ablation runs are never automatic; the short-horizon ablation
  sanity in phase 2 covers the automatic case (component off ≈ reference
  curve, on ≠ off). Run a full ablation only on explicit human request.
- AI review is the expensive tail, not the default: deterministic checks
  gate; AI interprets, diffs intents, and judges results.
