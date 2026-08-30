# Checker Catalog

The full list of verification checks, by phase. **[D]** checks are
deterministic: the checker agent generates them as code in
`.harness/checkers/`, tailored to the target repo (thresholds and
expectations come from the spec and from `compare_to` records — adjust the
generated code per experiment where noted). **[AI]** checks run in isolated
contexts with only the inputs listed.

Every check emits a verdict JSON (`schemas/verdict.schema.json`) into
`experiments/EXP-NNNN-slug/runs/iNN/checks/`, named after its id
(`p1-scope-enforcement.json`). Severity defaults to `critical` unless noted.

## Phase 0 — before implementation

| Id | Kind | Check | Pass criterion |
|---|---|---|---|
| `p0/spec-schema` | D | Spec frontmatter validates against `spec.schema.json` | schema-valid; scope excludes `experiments/`, `knowledge/`, `.harness/checkers/` |
| `p0/references-exist` | D | Parent and every `compare_to` experiment is documented | their records exist, terminal or `analyzed`, with the metrics the assertions cite |
| `p0/assertions-grounded` | D | Every assertion's `source` record contains the referenced metric | numbers resolvable, thresholds consistent with them |
| `p0/clean-env` | D | Environment snapshot taken | git status clean (or dirty flag consciously set), lockfile hash recorded, torch/CUDA versions recorded |
| `p0/spec-review` | AI | Isolated review of the spec. Input: spec + cited records only | hypothesis falsifiable; assertions measurable; scope minimal; decision rule covers accept/reject/inconclusive/revise + early-kill; budgets and artifact manifest present |

## Phase 1 — after implementation, before run

| Id | Kind | Check | Pass criterion |
|---|---|---|---|
| `p1/static` | D | Lint, typecheck, unit tests | all pass |
| `p1/scope-enforcement` | D | Changed files vs spec scope | (`git diff --name-only` against the parent's commit, minus `experiments/` and `knowledge/`) ⊆ `diff.code.scope` |
| `p1/config-diff` | D | Resolved run config vs parent's resolved config | changed keys exactly match `diff.config` declarations |
| `p1/data-splits` | D | Train/val/test overlap by content hash | zero overlap |
| `p1/data-pipeline` | D | Augmentations on train only; preprocessing (normalization stats etc.) fitted on train only; first-batch shapes/dtypes as expected | all hold |
| `p1/connectivity` | D | New component is wired in | param-count diff vs parent is nonzero and expected; forward pass through the new path succeeds |
| `p1/code-review` | AI | ML-checklist code review. Input: the diff + checklist | no critical findings. Checklist: tensor shapes and broadcasting; mask off-by-ones; `optimizer.zero_grad`; stray/missing `.detach()`; metric computed on the right split; train/eval mode; device/dtype handling; data-order assumptions |
| `p1/intent-diff` | AI | Agent A (sees only the code diff) describes what was implemented; agent B (sees only that description + the spec) diffs them and rates each divergence | no critical divergences; warnings go to the analysis |

## Phase 2 — first training steps (sanity)

Run on a short budget (smoke config). All thresholds come from the spec or
the `compare_to` records.

| Id | Kind | Check | Pass criterion |
|---|---|---|---|
| `p2/smoke-run` | D | N-step run completes | no exceptions; checkpoint written |
| `p2/loss-at-init` | D | Step-0 loss vs expectation (e.g. log C for classification; declared in spec) | within declared tolerance |
| `p2/loss-decreases` | D | Loss over first K steps | drops by the declared margin |
| `p2/nan-inf` | D | Hooks on loss, gradients, activations | none seen |
| `p2/gradients` | D | Dead-param check; grad norm; update-to-weight ratio | all params receive grads; values inside declared bands |
| `p2/single-batch-overfit` | D | Overfit one batch for M steps | loss below declared near-zero threshold |
| `p2/reproducibility` | D | Two short runs, same seed | loss curves identical for K steps (bitwise or declared tolerance) |
| `p2/negative-control` | D | Shuffled labels, K steps | loss does not go below chance level (below ⇒ leakage) |
| `p2/ablation-sanity` | D | Only when the diff adds a toggleable code component: K steps with it disabled, K steps enabled | disabled ≈ reference curve from the parent (we did not break what we started from); enabled ≠ disabled (the component is actually connected). Skip (`verdict: skip`) for config-only diffs |
| `p2/throughput-memory` | D | Steps/sec and peak memory on the smoke run | within spec budgets relative to `compare_to` |
| `p2/checkpoint-roundtrip` | D | save → load → loss on a fixed batch; resume | identical loss after load; resumed step matches within tolerance |
| `p2/eval-mode` | D | `model.eval()` changes behavior when dropout/BN present; eval is deterministic | both hold |
| `p2/diagnose` | AI | On failure only. Input: failed verdicts + relevant logs | produces a causal hypothesis + suggested fix, returned to the loop as feedback. Never a gate |

## Phase 3 — during training (watchdog)

Streaming checks over the full run; a critical trip kills or pauses the run.
Band parameters (bounds, spike factors, patience) come from the
`verification.params` of `diagnostic` entries in the metric registry
(`knowledge/metrics.md`); the spec may tighten them per experiment.

| Id | Kind | Check | Pass criterion |
|---|---|---|---|
| `p3/nan-inf` | D | NaN/Inf in loss or gradients | none |
| `p3/grad-norm-band` | D | Grad norm inside declared band | holds |
| `p3/loss-spike` | D | Loss vs running median | no spike above declared factor |
| `p3/lr-schedule` | D | LR at declared checkpoints | matches expected values |
| `p3/train-val-gap` | D | Gap inside declared band; val metric not degrading past patience | holds |
| `p3/throughput-memory` | D | Steps/sec stable; memory stable (leak detector) | within bounds |
| `p3/heartbeat` | D | Logging alive | no silence beyond the declared interval |
| `p3/early-kill` | D | Spec early-kill rules (e.g. val metric worse than reference by threshold at checkpoint X) | not triggered; triggered ⇒ kill the run, verdict fail |
| `p3/anomaly-review` | AI | Triggered by soft signals (oscillation, plateau, odd gap dynamics). Input: curves + trigger | advisory verdict with severity; gates only on critical |

## Phase 4 — after training

| Id | Kind | Check | Pass criterion |
|---|---|---|---|
| `p4/artifact-manifest` | D | Everything in `artifacts_expected` exists and is valid | metrics file has required fields; checkpoint loads; config snapshot + commit hash recorded; links resolve |
| `p4/assertions` | D | Every spec assertion evaluated against `compare_to` numbers (from records, or from metric-registry `references` for backfilled metrics) | all pass (or each failure listed with its margin) |
| `p4/checkpoint-roundtrip` | D | Final checkpoint: load + eval | reproduces recorded metrics within tolerance |
| `p4/results-review` | AI | Isolated review. Input: spec + metrics + curves + assertion verdicts (never the executor's conversation) | verdict proposal — hypothesis confirmed / refuted / inconclusive — with anomalies and follow-ups |
| `p4/aggregate` | D | Fold all verdicts of the iteration into one machine-readable iteration verdict per the spec's decision rule | `accept` / `revise` / `reject` / `inconclusive` / `escalate-to-human` |

## Engineering pipeline (ENG changes)

Reduced pipeline for changes with no hypothesis about model quality.
Verdicts land in `experiments/engineering/ENG-NNNN/checks/`.

| Id | Kind | Check | Pass criterion |
|---|---|---|---|
| `e/static` | D | Lint, typecheck, unit tests | all pass |
| `e/scope` | D | Files touched vs the change's declared intent | no unrelated files |
| `e/smoke` | D | Training smoke run | completes without exceptions |
| `e/equivalence` | D | Result-neutral changes only (refactors, pure instrumentation): smoke/eval re-run at a reference checkpoint | every previously registered metric unchanged within declared tolerance |
| `e/metric-known-value` | D | New/changed metrics: compute on inputs with an analytically known value | matches |
| `e/metric-registered` | D | New/changed metrics: `knowledge/metrics.md` entry valid per `metrics.schema.json`; its checker generated; `references` backfilled or waiver recorded | all hold |

## Generation notes for the checker agent

- Generate checkers as small standalone scripts in `.harness/checkers/`,
  named by check id, each printing exactly one verdict JSON to stdout and
  exiting nonzero on `fail`.
- Read thresholds from the spec frontmatter and reference numbers from
  `compare_to` records at runtime — do not hardcode numbers that belong to
  the spec.
- Repo-specific adaptation (entry points, config system, logger) is
  expected; document it in `knowledge/conventions.md`.
- Checkers are code: version them, and route any checker change through
  intent review. The experiment executor must never edit them.
