# Checker Catalog

The full list of verification checks, by stage. Stages are named — **spec
review** (before implementation), **pre-run** (after implementation, before
launch), **smoke** (first training steps), **watchdog** (during training),
**post-run** (after training) — and map to the numeric `phase` field 0–4 in
machine-readable verdicts. **[D]** checks are deterministic: the checker
agent generates them as code in `.harness/checkers/`, tailored to the target
repo (thresholds and expectations come from the spec and from `compare_to`
records — adjust the generated code per experiment where noted). **[AI]**
checks run in isolated contexts with only the inputs listed.

Every check emits a verdict JSON (`schemas/verdict.schema.json`) into
`experiments/EXP-NNNN-slug/runs/iNN/checks/`, named after its id
(`pre-run-scope-enforcement.json`). Severity defaults to `critical` unless
noted.

## Spec review (phase 0) — before implementation

| Id | Kind | Check | Pass criterion |
|---|---|---|---|
| `spec-review/spec-schema` | D | Spec frontmatter validates against `spec.schema.json` | schema-valid; scope excludes `experiments/`, `knowledge/`, `.harness/checkers/` |
| `spec-review/references-exist` | D | Parent and every `compare_to` experiment is documented | their records exist, terminal or `analyzed`, with the metrics the assertions cite |
| `spec-review/assertions-grounded` | D | Every assertion's `source` record contains the referenced metric | numbers resolvable, thresholds consistent with them |
| `spec-review/clean-env` | D | Environment snapshot taken | git status clean (or dirty flag consciously set), lockfile hash recorded, torch/CUDA versions recorded |
| `spec-review/spec-review` | AI | Isolated review of the spec. Input: spec + cited records only | hypothesis falsifiable; assertions measurable; scope minimal; decision rule covers accept/reject/inconclusive/revise + early-kill; budgets and artifact manifest present |

## Pre-run (phase 1) — after implementation, before launch

| Id | Kind | Check | Pass criterion |
|---|---|---|---|
| `pre-run/static` | D | Lint, typecheck, unit tests | all pass |
| `pre-run/scope-enforcement` | D | Changed files vs spec scope | (`git diff --name-only` against the parent's commit, minus `experiments/` and `knowledge/`) ⊆ `diff.code.scope` |
| `pre-run/config-diff` | D | Resolved run config vs parent's resolved config | changed keys exactly match `diff.config` declarations |
| `pre-run/data-splits` | D | Train/val/test overlap by content hash | zero overlap |
| `pre-run/data-pipeline` | D | Augmentations on train only; preprocessing (normalization stats etc.) fitted on train only; first-batch shapes/dtypes as expected | all hold |
| `pre-run/connectivity` | D | New component is wired in | param-count diff vs parent is nonzero and expected; forward pass through the new path succeeds |
| `pre-run/code-review` | AI | ML-checklist code review. Input: the diff + checklist | no critical findings. Checklist: tensor shapes and broadcasting; mask off-by-ones; `optimizer.zero_grad`; stray/missing `.detach()`; metric computed on the right split; train/eval mode; device/dtype handling; data-order assumptions |
| `pre-run/intent-diff` | AI | Agent A (sees only the code diff) describes what was implemented; agent B (sees only that description + the spec) diffs them and rates each divergence | no critical divergences; warnings go to the analysis |

## Smoke (phase 2) — first training steps

Run on a short budget (smoke config). All thresholds come from the spec or
the `compare_to` records.

| Id | Kind | Check | Pass criterion |
|---|---|---|---|
| `smoke/smoke-run` | D | N-step run completes | no exceptions; checkpoint written |
| `smoke/loss-at-init` | D | Step-0 loss vs expectation (e.g. log C for classification; declared in spec) | within declared tolerance |
| `smoke/loss-decreases` | D | Loss over first K steps | drops by the declared margin |
| `smoke/nan-inf` | D | Hooks on loss, gradients, activations | none seen |
| `smoke/gradients` | D | Dead-param check; grad norm; update-to-weight ratio | all params receive grads; values inside declared bands |
| `smoke/single-batch-overfit` | D | Overfit one batch for M steps | loss below declared near-zero threshold |
| `smoke/reproducibility` | D | Two short runs, same seed | loss curves identical for K steps (bitwise or declared tolerance) |
| `smoke/negative-control` | D | Shuffled labels, K steps | loss does not go below chance level (below ⇒ leakage) |
| `smoke/ablation-sanity` | D | Only when the diff adds a toggleable code component: K steps with it disabled, K steps enabled | disabled ≈ reference curve from the parent (we did not break what we started from); enabled ≠ disabled (the component is actually connected). Skip (`verdict: skip`) for config-only diffs |
| `smoke/throughput-memory` | D | Steps/sec and peak memory on the smoke run | within spec budgets relative to `compare_to` |
| `smoke/checkpoint-roundtrip` | D | save → load → loss on a fixed batch; resume | identical loss after load; resumed step matches within tolerance |
| `smoke/eval-mode` | D | `model.eval()` changes behavior when dropout/BN present; eval is deterministic | both hold |
| `smoke/diagnose` | AI | On failure only. Input: failed verdicts + relevant logs | produces a causal hypothesis + suggested fix, returned to the loop as feedback. Never a gate |

## Watchdog (phase 3) — during training

Streaming checks over the full run; a critical trip kills or pauses the run.
Band parameters (bounds, spike factors, patience) come from the `params` of
`diagnostic` metric-observation entries in the verification registry
(`knowledge/verifications.md`); the spec may tighten them per experiment.

| Id | Kind | Check | Pass criterion |
|---|---|---|---|
| `watchdog/nan-inf` | D | NaN/Inf in loss or gradients | none |
| `watchdog/grad-norm-band` | D | Grad norm inside declared band | holds |
| `watchdog/loss-spike` | D | Loss vs running median | no spike above declared factor |
| `watchdog/lr-schedule` | D | LR at declared checkpoints | matches expected values |
| `watchdog/train-val-gap` | D | Gap inside declared band; val metric not degrading past patience | holds |
| `watchdog/throughput-memory` | D | Steps/sec stable; memory stable (leak detector) | within bounds |
| `watchdog/heartbeat` | D | Logging alive | no silence beyond the declared interval |
| `watchdog/early-kill` | D | Spec early-kill rules (e.g. val metric worse than reference by threshold at checkpoint X) | not triggered; triggered ⇒ kill the run, verdict fail |
| `watchdog/anomaly-review` | AI | Triggered by soft signals (oscillation, plateau, odd gap dynamics). Input: curves + trigger | advisory verdict with severity; gates only on critical |

## Post-run (phase 4) — after training

| Id | Kind | Check | Pass criterion |
|---|---|---|---|
| `post-run/artifact-manifest` | D | Everything in `artifacts_expected` exists and is valid | metrics file has required fields; checkpoint loads; config snapshot + commit hash recorded; links resolve |
| `post-run/assertions` | D | Every spec assertion evaluated against `compare_to` numbers (from records, or from verification-registry `references` for backfilled metrics) | all pass (or each failure listed with its margin) |
| `post-run/checkpoint-roundtrip` | D | Final checkpoint: load + eval | reproduces recorded metrics within tolerance |
| `post-run/results-review` | AI | Isolated review. Input: spec + metrics + curves + assertion verdicts (never the executor's conversation) | verdict proposal — hypothesis confirmed / refuted / inconclusive — with anomalies and follow-ups |
| `post-run/aggregate` | D | Fold all verdicts of the iteration into one machine-readable iteration verdict per the spec's decision rule | `accept` / `revise` / `reject` / `inconclusive` / `escalate-to-human` |

## Verifier-change run (VER changes)

Replaces the smoke, watchdog, and post-run stages inside a verifier
change's iteration. Verdicts land in the change's `runs/iNN/checks/` as
usual.

| Id | Kind | Check | Pass criterion |
|---|---|---|---|
| `v/static` | D | Lint, typecheck, unit tests | all pass |
| `v/scope` | D | `git diff` vs spec scope | ⊆ allowlist |
| `v/calibration` | D/AI | Run the verifier on the spec's calibration cases (known metric values, planted bugs, shuffled labels, curated diffs for AI checkers) | passes every known-good AND fails every known-bad case |
| `v/equivalence` | D | The standing `equivalence` verifier: unchanged training command, few steps | loss, previously registered metrics, artifacts identical within tolerance |
| `v/references` | D | Metric observations: backfill via the standard eval command on frozen reference checkpoint(s); through-training metrics over all saved checkpoints | values + provenance in the registry-entry draft, or waiver recorded |

## Standing verifiers

Instantiated at scaffold time from this catalog and registered in
`knowledge/verifications.md` like everything else:

- `equivalence` — re-run an unchanged command for a few steps: identical
  loss, registered metrics, and artifacts. The gate for every result-neutral
  change (refactors, instrumentation, verifier implementations).

## Generation notes for the checker agent

- Generate checkers as small standalone scripts in `.harness/checkers/`,
  named by check id, each printing exactly one verdict JSON to stdout and
  exiting nonzero on `fail`.
- Read thresholds from the spec frontmatter, band/calibration params from
  `knowledge/verifications.md`, and reference numbers from `compare_to`
  records or registry `references` at runtime — do not hardcode numbers that
  belong to the spec or the registry.
- Repo-specific adaptation (entry points, config system, logger) is
  expected; document it in `knowledge/conventions.md`.
- Checkers are code: version them, and route any checker change through
  intent review. The experiment executor must never edit them.
