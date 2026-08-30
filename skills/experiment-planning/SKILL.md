---
name: experiment-planning
description: >-
  Rules for writing and maintaining ML experiment specs: falsifiable
  hypotheses, generalized diffs (code, config, data, evaluation), assertions
  grounded in recorded numbers, declared decision rules and budgets, and
  living-spec maintenance with human approval gates. Use when planning a new
  experiment, drafting or reviewing a spec, deciding between a new experiment
  and a new iteration, folding off-spec decisions into a spec mid-loop, or
  merging conclusions into knowledge.
---

# Experiment Planning

How to produce and maintain a `spec.md` that an agent loop can execute and
verify against. The spec is the contract: verification (experiment-process
skill) checks the implementation and the run against it, and any decision the
spec does not dictate must eventually be folded back into it.

## New experiment, new iteration, or engineering change?

Before planning, decide which you are doing:

- The change carries **no hypothesis about model quality** —
  instrumentation/metrics, refactoring, infrastructure, bootstrap
  implementation → an **engineering change** (`ENG-NNNN`), not an experiment:
  engineering log + reduced pipeline (experiment-logging and
  experiment-process skills), no spec, no loop.
- The hypothesis or the declared diff scope **changes substantively** → a
  **new experiment** (new `EXP-NNNN`, new spec, parent set appropriately).
- The implementation is being **fixed**, or the spec is being **enriched
  with details** → a **new iteration** of the same experiment (spec diff
  through the approval gate, iteration+1).

When in doubt between experiment and iteration: if the new work would
invalidate comparisons against the experiment's own earlier iterations, it
is a new experiment.

## Writing the initial spec

Prerequisite: the parent and every `compare_to` experiment must already be
run, verified, and documented. **Read their run records first** — assertions
must reference their actual numbers.

Fill the spec (template: harness `templates/spec.md`; fields defined in the
experiment-logging skill) following these rules:

1. **Falsifiable hypothesis.** One claim with a concrete expected effect:
   "attention dropout 0.1 reduces val loss by ≥0.02 at step 10k vs
   EXP-0007" — not "dropout should help".
2. **Generalized diff — specify every axis**, not just code:
   - `code`: file allowlist (`scope`) + a one-paragraph summary of the
     change. The allowlist must never include `experiments/`, `knowledge/`,
     or `.harness/checkers/`.
   - `config`: changed keys vs the parent's *resolved* config, each as
     `{from, to}` — model size, batch size, optimizer, schedules, any
     hyperparameter.
   - `data`: dataset version/hash, splits, preprocessing changes vs parent.
   - `evaluation`: benchmark and metric-definition changes vs parent.
   - Root experiments pin `data` and `evaluation` absolutely and record the
     full training config; their diff may otherwise be empty.
3. **Grounded assertions.** Every threshold traces to a real number — in a
   `compare_to` record, or in the metric registry's `references`
   (`knowledge/metrics.md`) for metrics backfilled after the reference run —
   cited via `source`. Never invent absolute targets. Include non-regression
   assertions (throughput, memory, secondary metrics) alongside the headline
   claim. Prefer asserting on `primary` metrics; use `proxy` metrics for
   analysis, not for accept/reject decisions.
4. **Decision rule covering every outcome**: what result means `accept`,
   `reject`, `inconclusive`, and what failure modes mean `revise` — plus
   **early-kill thresholds** the training watchdog enforces (e.g. "kill if
   val loss worse than reference by 0.05 at checkpoint 5k").
5. **Budgets**: wall-clock, GPU memory, minimum throughput.
6. **Expected artifacts manifest**: everything the run must produce
   (metrics file, checkpoints, wandb run, plots).
7. **Knowledge merge plan** (in the body): which `knowledge/` files this
   experiment may update on acceptance, so the merge is decided before the
   result exists.

## Spec review

Before human sign-off, have an **isolated-context agent** (one that has not
seen the planning conversation) review the spec:

- Is the hypothesis falsifiable and the effect size stated?
- Is every assertion measurable, with thresholds traceable to cited records?
- Is the scope minimal for the hypothesis — no opportunistic extras?
- Does the decision rule cover accept, reject, inconclusive, revise, and
  early-kill?
- Are budgets and the artifact manifest present and realistic?

Fix findings, then get human sign-off. This is the cheapest point for human
feedback — spend it here, not after a run. On sign-off: set
`status: planned`, `iteration: 1`, create the `exp/EXP-NNNN-slug` branch and
the registry row.

## Living-spec maintenance (during the loop)

The spec is enriched by the loop — an ongoing explore that makes it more
precise. A spec change is warranted when:

- something important turns out underspecified or missing;
- reality contradicts the spec and the spec must be corrected;
- the agent made **any decision the spec did not dictate**.

Rules:

- **Materiality criterion.** Fold in every decision that could affect
  results, reproducibility, or interpretation. Style-level choices stay in
  git history — the spec must not bloat.
- **Assumptions field.** Decisions the agent considers obvious and almost
  certainly right still get recorded — as one-liners in the spec's
  `assumptions` list, so the human sees them without them cluttering the
  main sections.
- **Approval gate.** Accumulate changes during the iteration; present them
  as **one spec diff at the iteration boundary** (with the `revise`
  verdict): material changes highlighted, `assumptions` additions listed.
  The human approves the diff, then the next iteration starts.
- **Immediate escalation** only when continuing the current iteration would
  directly contradict the spec — then stop and ask instead of accumulating.
- Log every approved change in the spec body's **Enrichment log** (iteration,
  date, what, why), and commit spec changes with the `[EXP-NNNN.iN]` prefix.

If the executor disagrees with a verifier about what the spec should say,
that is always a human escalation — never silently resolved by either agent.

## Registering a new metric

Logging a new metric is **preparing a new verification** — a metric is
logged either to measure experiment quality, to localize what to improve, or
to watch training health, and each of those purposes implies a check.
Adding or changing a metric is an engineering change (`ENG-NNNN`) with these
extra obligations:

1. **Registry entry** in `knowledge/metrics.md`: `name`, `kind`, precise
   `definition`, `logged_by`, `added_by`, and a `verification` mapping.
   Defaults by kind:
   - `primary` (measures experiment quality) → phase-4 **assertion** vs
     `compare_to`;
   - `proxy` (correlates with primary, localizes improvements) →
     **advisory** check feeding analysis and anomaly triggers;
   - `diagnostic` (training health: grad norm, loss by position, latent
     std, perf) → phase-3 **band** or `analysis-only`.
2. **Checker generation**: the checker agent derives the metric's check from
   the registry entry (never from the executor's description).
3. **Backfill**: measure the metric on the current reference checkpoint(s)
   with an eval-only run and write the values into the entry's `references`
   with provenance (experiment, checkpoint, measuring commit, date). Run
   records are immutable — backfilled numbers live in the registry, never
   patched into old records. If backfill is infeasible (metric requires
   training-time state), record an explicit waiver in the entry instead.

## Merging into knowledge (terminal verdict)

On `accepted` / `rejected` / `inconclusive`:

1. Extract the durable conclusions from `report.md` into
   `knowledge/findings.md` — short entries, each linking its `EXP-NNNN`.
   Negative and inconclusive results are findings too.
2. Update `knowledge/conventions.md` if the experiment changed repo-level
   facts (new default config, new data version, revised budgets).
3. The experiment folder stays archived as-is under `experiments/` —
   knowledge holds conclusions, experiments hold provenance.
