---
name: experiment-logging
description: >-
  Rules for documenting ML experiments in a research repo: where records live
  (folders, branches, commit conventions), what is recorded (fields per file),
  how (Markdown with schema-validated YAML frontmatter plus narrative body),
  and when (required fields per status transition). Use when scaffolding
  experiments/ and knowledge/, creating or updating specs, run records,
  reports, or the registry, or auditing documentation completeness.
---

# Experiment Logging

Rules for keeping experiment documentation complete, machine-readable, and
human-readable at the same time. Follow them for every experiment.

## Model

- An **experiment** is one hypothesis, identified as `EXP-NNNN` plus a slug.
  It has a nullable `parent` (the experiment it derives from) and a
  possibly-empty **generalized diff** vs that parent covering four axes:
  code, config (hyperparameters, model size, batch size, …), data, and
  evaluation (benchmarks, metrics). A root experiment (no parent) pins all
  axes absolutely instead of as a diff.
- **"Baseline" is a role, not a type**: the run you are trying to beat, named
  in the spec's `compare_to` list (default: the parent). Any experiment can
  serve as a baseline for later work.
- **Iterations** `i01, i02, …` are attempts at the hypothesis inside the
  agent loop. Each iteration gets its own run record and check verdicts.
  Iteration statuses cycle: `planned → running → completed → analyzed`, then
  either back to `planned` (verdict `revise`, iteration+1) or to a terminal
  status (`accepted` / `rejected` / `inconclusive`).
- **Verifier changes** (`VER-NNNN`) implement or modify a **verification**:
  a metric plus how to read it, a deterministic check script, or an
  AI-review checker. They use the *same machinery* as experiments — living
  spec, agent loop with iterations, run records, report, approval gates —
  but their run is verifier calibration instead of model training
  (experiment-process skill). On acceptance the verification merges into the
  registry `knowledge/verifications.md`.
- **Maintenance commits** (small refactors, infra) carry no id. They must
  keep the tree green and, when result-neutral, pass the standing
  `equivalence` verifier (unchanged command, few steps → identical loss,
  metrics, artifacts).

## Where

Per-repo layout (create missing pieces from the harness `templates/`, or from
the field definitions below if templates are unavailable):

```
experiments/
  registry.md                     # index: one row per change (EXP and VER)
  EXP-0042-attn-dropout/
    spec.md                       # living plan (see experiment-planning skill)
    runs/
      i01/
        record.md                 # run record for iteration 1
        checks/                   # check verdicts, one JSON per check
      i02/
        ...
    report.md                     # final synthesis, written at terminal verdict
  VER-0003-acclen-by-position/    # verifier change: identical layout
    spec.md
    runs/i01/ ...
    report.md
knowledge/
  findings.md                     # validated conclusions, linked to EXP/VER ids
  verifications.md                # verification registry: every verification + params + references
  conventions.md                  # repo-specific facts (env, data paths, budgets,
                                  # the standard eval command)
.harness/
  checkers/                       # generated checks — OFF-LIMITS to the implementer
```

Rules:

- Heavy artifacts (checkpoints, raw logs, wandb data) are **never committed**.
  Link them by URI from the run record, with a content hash when cheap to
  compute.
- `.harness/checkers/` is maintained by a separate checker agent (see the
  experiment-process skill), never by the agent implementing the experiment.
- `knowledge/` holds the current validated truth; `experiments/` holds
  immutable provenance. Read `knowledge/` first when you need "what is true
  now", and `experiments/` when you need "how we learned it".

## Branches and commits

- Implement on branch `exp/EXP-NNNN-slug`. A root experiment with an empty
  diff may run directly from a pinned commit on `main`.
- Prefix **every** commit made inside the loop with the experiment id and
  iteration: `[EXP-0042.i3] fix attention mask`. Never squash or amend on the
  `exp/` branch while the loop runs — the commit trail is the record of how
  errors were fixed and the anchor for resuming an interrupted loop.
- Verifier changes follow the same conventions with their own prefix:
  branch `ver/VER-NNNN-slug`, commits `[VER-0003.i1] ...`.
- Maintenance commits carry no id prefix; each must leave the tree green.
- Docs (`experiments/…`, `knowledge/…`) are committed on the experiment
  branch during the loop. At a terminal verdict, docs always land on `main`
  (merge or cherry-pick) regardless of the code-merge decision, so history
  and knowledge stay visible even if the code branch is abandoned.
- The code itself merges to `main` only if the change should become the new
  default. Record the decision in `report.md`; if the merge squashes, record
  the commit mapping there too.

## How: format

Every document is one Markdown file:

- **YAML frontmatter** — the machine-readable source of truth (ids, hashes,
  numbers, statuses). Validate against the harness `schemas/*.schema.json`
  when available; otherwise follow the templates exactly.
- **Body** — the human narrative (motivation, notes, analysis, conclusions).

Never keep a parallel JSON copy of frontmatter data. The only pure-JSON files
are check verdicts under `runs/iNN/checks/`.

Mutability:

| File | Mutability |
|---|---|
| `spec.md` | Living — changes only via the approval gate (experiment-planning skill), versioned by iteration-tagged commits |
| `runs/iNN/record.md` | Immutable once iteration N ends |
| `report.md` | Written once, at the terminal verdict |
| `registry.md` | Row updated at every status change |

## What: files and fields

**`spec.md`** frontmatter: `id`, `slug`, `parent`, `compare_to`, `status`,
`iteration`, `hypothesis`, `diff` (`code.scope` + `code.summary`, `config`,
`data`, `evaluation`), `assertions` (each: `id`, `metric`, `at`, `condition`,
`source`), `budgets`, `artifacts_expected`, `decision_rule`, `assumptions`.
Body: Idea, Motivation, Risks, Enrichment log.

**`runs/iNN/record.md`** frontmatter: `experiment`, `iteration`, `commit`,
`branch`, `dirty`, `env` (lockfile hash, python/torch/cuda, hardware),
`config` (path + resolved hash), `setup` (materialized, not a diff: `model`,
`data` with version/hash, `training` — hardware + hyperparameters,
`evaluation` — hardware + benchmarks + sampling params), `launch_command`,
`artifacts` (wandb URL, checkpoint URIs, log paths), `metrics` (final
numbers), `verdict` (iteration verdict). Body: Run notes, Incidents,
Analysis (per-iteration; the cross-iteration synthesis belongs in the
report).

**`report.md`** frontmatter: `experiment`, `verdict`
(`accepted`/`rejected`/`inconclusive`), `iterations`, `deltas` (per metric:
`vs`, `reference`, `value`, `delta`), `knowledge_merge` (file + summary),
`merge_to_main`, `merge_commit`. Body: Analysis, Conclusions, Follow-ups.

**`registry.md`**: one table row per change — ID, slug, type (exp/ver),
parent, status, iteration, headline metric, date.

**Verifier-change specs** additionally fill the `verifier` frontmatter block:
`type` (`deterministic-script` / `metric-observation` / `ai-review`),
`verifies`, `calibration` cases, and a `registry_entry` draft.

**`knowledge/verifications.md`**: the verification registry (schema:
`verifications.schema.json`) — per verification: `id`, `type`, `verifies`,
`phase`, `gating` (`gate`/`advisory`/`analysis-only`), `params`, for
metric-observation entries the `metric` (name, kind
`primary`/`proxy`/`diagnostic`, definition, logged_by), `implemented_by`
(the VER/EXP that introduced it), `references` (backfilled values with
provenance). Maintained per the experiment-planning skill.

## When: status transitions

Set a status only after its required documentation is complete. Incomplete
docs at a transition are a process failure — stop and fill them.

| Status | Set when | Must be complete by then |
|---|---|---|
| `planned` | spec approved — initially, or after a `revise` spec-diff approval | all of `spec.md` at the current iteration; registry row exists |
| `running` | training launched | `runs/iNN/record.md`: commit, branch, dirty flag, env, config, setup, launch command, artifact links |
| `completed` | run finished, artifacts collected | `runs/iNN/record.md`: final metrics; `artifacts_expected` manifest satisfied |
| `analyzed` | iteration analysis written | `runs/iNN/record.md`: Analysis section + iteration `verdict` |
| `accepted` / `rejected` / `inconclusive` | terminal verdict | `report.md` complete; knowledge merged; registry row updated; docs on `main` |

On `revise`: prepare the spec diff for approval (experiment-planning skill);
on approval, bump `iteration` in the spec, set `planned`, create
`runs/iNN+1/`.

## Verifier changes

A verifier change is documented exactly like an experiment: registry row
(type `ver`), folder `experiments/VER-NNNN-slug/` with living `spec.md`,
per-iteration `runs/iNN/` records and checks, and a final `report.md`. The
same status table applies. What differs:

- the spec fills the `verifier` block (type, what it verifies, calibration
  cases, registry-entry draft);
- the iteration run is verifier calibration + equivalence + reference
  backfill instead of model training (experiment-process skill);
- the knowledge merge on acceptance targets `knowledge/verifications.md`.

A logged metric without a `metric-observation` entry in the registry is an
audit failure — logging a metric *is* implementing a verification.

## Scaffolding (first use in a repo)

1. Create `experiments/registry.md` from the template.
2. Create `knowledge/findings.md` (empty list), `knowledge/verifications.md`
   (register every metric the repo already logs as a `metric-observation`
   verification, plus the standing verifiers from the checker catalog), and
   `knowledge/conventions.md`, filling conventions by inspecting the repo:
   environment and how it is built, entry points, data locations, typical
   budgets, logging setup, and **the standard eval command** — how to run
   validation on a frozen checkpoint with no code changes (required for
   reference backfill).
3. Leave `.harness/checkers/` to the checker agent (experiment-process
   skill) — do not populate it while scaffolding.

## Auditing documentation

To audit an experiment: validate all frontmatter against the schemas; check
the required-fields table for its current status; verify the registry row
matches the spec frontmatter (status, iteration); verify artifact links in
run records resolve; verify every `[EXP-NNNN.iN]` commit's iteration has a
`runs/iNN/` directory.

To audit instrumentation: every metric the code logs has a
`metric-observation` entry in `knowledge/verifications.md`; every registry
entry's `implemented_by` change exists and is terminal; the standard eval
command in `knowledge/conventions.md` still works.
