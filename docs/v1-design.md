# V1 Design: Architecture, Use Cases, and Skills

Status: agreed v1 scope. Last updated: 2026-08-31 (added verifier changes
and the verification registry).

## 1. What this project is

This repository is a **source of skills, not a code library**. Nothing here is
imported or executed by research code. Instead, AI agents (Claude Code and
similar) load the skills from this repo while working inside a **target
research repo** (e.g. a training codebase), and the skills tell them how to
plan, run, verify, and document experiments.

The harness therefore ships three kinds of things:

1. **Skills** — instructions for agents (`SKILL.md` files).
2. **Schemas** — machine-readable contracts for experiment documents
   (frontmatter fields, check verdicts).
3. **Templates and references** — file templates the skills instantiate in a
   target repo, plus reference material (e.g. the checker catalog).

**How skills are consumed:** each skill is self-contained; the user hands it
to an agent in the prompt and decides where to install it. Default location is
the target project (e.g. its `.claude/skills/`). No packaging or plugin
mechanism in v1.

Non-goals for v1: no runtime framework, no CLI the training code depends on,
no orchestration daemon, no web UI. If a skill needs executable checks, it
*generates* them inside the target repo, tailored to that repo.

## 2. Core concepts

### Target repo vs harness repo

- **Harness repo** (this one): skills, schemas, templates. Evolves slowly.
- **Target repo**: the actual research codebase. The skills scaffold and
  maintain two directories there: `experiments/` (history) and `knowledge/`
  (current validated truth).

### Experiment — one entity

There is a single kind of experiment. Every experiment has:

- a **nullable parent** (another experiment it derives from), and
- a **possibly-empty generalized diff** against that parent.

The generalized diff is not just code. It covers every axis of the setup:

- **code** — implementation changes (scoped by a file allowlist);
- **config** — model size, batch size, optimizer, any hyperparameters,
  expressed as a diff to the parent's resolved config;
- **data** — dataset version/hash, splits, preprocessing selection;
- **evaluation** — benchmarks, metric definitions.

A root experiment (no parent) pins all of these absolutely instead of as a
diff. A precondition of any child experiment is that its parent has already
been run, verified, and documented.

**"Baseline" is a role, not a type.** A baseline is *the thing you want to
beat*, assigned manually — not necessarily the parent you mutated from. A
paper reimplementation, for example, is a child experiment with a non-empty
diff, and at the same time a baseline for later work. In v1 this role appears
only as `compare_to` targets in specs (defaulting to the parent). A future
**arena** mechanism — comparing experiments on chosen metrics and setups, with
manual baseline marking — is deferred (see section 6).

### Iterations and the agent loop

An experiment is one hypothesis; **iterations** (`i1`, `i2`, …) are attempts
at it within an agent loop. The boundary rule:

- the hypothesis or the declared diff scope changes substantively → **new
  experiment**;
- the implementation is being fixed, or the spec is being enriched with
  details → **new iteration** of the same experiment.

Each iteration goes through the full cycle (implement → verify → run →
analyze → verdict). The verdict `revise` loops back to `planned` with the
iteration counter incremented, passing through the spec-diff approval gate
(see below). Terminal verdicts (`accepted` / `rejected` / `inconclusive`) end
the loop.

The loop is **resumable**: current state is fully derivable from the spec
frontmatter (status + iteration), the registry row, and the last
iteration-tagged commit (see 3.3). No separate state file — it would duplicate
the source of truth.

### Verifier changes — the second change type

Not every change is an experiment. A change that **implements or modifies a
verification** — a metric plus how to read it, a deterministic check script,
an AI-review checker — is a **verifier change** (`VER-NNNN`). It uses the
*same machinery* as an experiment: a living spec with approval gates, the
agent loop with iterations, run records, a report, and an openspec-style
merge into knowledge on acceptance. What differs is the run: instead of
phases 2–4 (training), a verifier change runs **calibration** (the verifier
must pass known-good cases AND fail known-bad ones — a verifier that cannot
fail verifies nothing), **equivalence** (implementing it must not alter
training: unchanged command, few steps, identical loss/metrics/artifacts),
and **reference backfill** (below).

Small refactors and infra remain **maintenance commits**: no id, tree stays
green, result-neutral ones pass the standing `equivalence` verifier.

### Verification registry — a metric is a verification

Verifications are first-class knowledge. The registry
`knowledge/verifications.md` lists every verification available in the repo:
its type (`deterministic-script` / `metric-observation` / `ai-review`), what
it verifies, its phase and gating mode, its params, and — for metric
observations — the metric itself with its kind:

| Metric kind | Purpose | Default verification |
|---|---|---|
| `quality` | measures an aspect of experiment quality or performance (acceptance length, TPF, tokens per second) | assertable in phase 4; advisory for analysis |
| `diagnostic` | training health (grad norm, loss by position, latent std, perf internals) | phase-3 watchdog band or analysis-only |

**The primary metric is chosen per experiment, never fixed globally.** Each
experiment's spec declares `primary_metric` — the metric its success is
judged by (acceptance length for one experiment, tokens per second for
another); the headline assertion and the decision rule are stated in its
terms, and the other `quality` metrics act as that experiment's proxies
(analysis, not accept/reject).

The checker agent derives all checks from registry entries, so "log a new
metric" literally means "implement a new verification". Verifications serve
two purposes: **gating** the loop, and **analysis** — a verification that
reveals a problem whose fix is a different improvement concludes the current
experiment and seeds a **follow-up experiment** that closes it (recorded in
the report's Follow-ups); `revise` is only for fixing the current
hypothesis's own implementation.

**Reference backfill.** A metric added after a reference run is measured by
running the repo's **standard eval command** (a required convention in
`knowledge/conventions.md`) on the frozen reference checkpoint(s) — no code
changes, no baseline retraining; through-training metrics run over all saved
checkpoints. Values live in the registry entry's `references` with
provenance. Run records stay immutable; assertions may ground in records or
registry references.

### Living spec

The spec starts as a change proposal and is **enriched by the loop**. Reasons
a spec changes: important details turn out to be missing or underspecified; a
problem contradicts the spec and it must be corrected; the agent made a
decision the spec did not dictate.

- **Materiality criterion**: any decision that could affect results,
  reproducibility, or interpretation is folded into the spec. Style-level
  choices stay in git history only — the spec must not bloat.
- **Minor decisions**: choices the agent considers obvious and almost
  certainly right still get recorded, in a dedicated `assumptions` list, so
  the human sees them without them cluttering the main sections.
- **Approval gate**: spec changes accumulate during an iteration and are
  presented as one spec diff at the iteration boundary — material changes
  highlighted, `assumptions` additions listed. The human approves the diff and
  the next iteration starts. Immediate escalation happens only when
  continuing the current iteration would contradict the spec.

Every approved spec change is a commit tagged with the iteration (3.3), so
spec history is reconstructible from git.

### Spec lifecycle (openspec-style)

1. A spec starts as a **change proposal** — a claim about what the experiment
   will do, written before any code.
2. While the loop runs, the spec is the contract that verification checks
   against, and it accumulates approved enrichments.
3. On a terminal verdict, the experiment folder is **archived as-is**, and the
   durable conclusions are **merged into `knowledge/`**: validated findings
   and updated conventions.

`knowledge/` is the only place agents read to learn "what is currently true"
about the project; `experiments/` is where they go for provenance.

### Dual-format principle

Every document is a single Markdown file with YAML frontmatter:

- **Frontmatter** is the machine-readable source of truth — ids, hashes,
  metric numbers, statuses — validated against JSON Schemas shipped in this
  repo.
- **Body** is the human narrative — motivation, analysis, conclusions.

No parallel JSON copies of the same data. The only pure-JSON files are check
verdicts, because they are produced and consumed by tools.

## 3. Architecture

### 3.1 Harness repo layout

```
ml_research_harness/
├── README.md
├── docs/
│   └── v1-design.md          # this document
├── skills/
│   ├── experiment-logging/   # skill 1: documentation rules
│   │   └── SKILL.md
│   ├── experiment-planning/  # skill 2: spec rules
│   │   └── SKILL.md
│   └── experiment-process/   # skill 3: how to run an experiment
│       ├── SKILL.md
│       └── references/
│           └── checker-catalog.md   # full phase 0–4 checker list
├── schemas/
│   ├── spec.schema.json           # frontmatter of spec.md (EXP and VER)
│   ├── record.schema.json         # frontmatter of a run record
│   ├── report.schema.json         # frontmatter of report.md
│   ├── verifications.schema.json  # frontmatter of the verification registry
│   └── verdict.schema.json        # check verdict format
└── templates/
    ├── spec.md
    ├── record.md
    ├── report.md
    ├── registry.md
    └── verifications.md
```

### 3.2 Target repo layout (created and maintained by the skills)

```
<target-repo>/
├── experiments/
│   ├── registry.md                 # index: one row per change (EXP and VER)
│   ├── EXP-0042-attn-dropout/
│   │   ├── spec.md                 # living plan, enriched each iteration
│   │   ├── runs/
│   │   │   ├── i01/
│   │   │   │   ├── record.md       # commit, config, artifacts, results
│   │   │   │   └── checks/         # verification verdicts, one JSON per check
│   │   │   └── i02/
│   │   │       └── ...
│   │   └── report.md               # final synthesis: analysis, conclusions, verdict
│   └── VER-0003-acclen-by-position/   # verifier change: identical layout
│       └── ...
├── knowledge/
│   ├── findings.md                 # validated conclusions, linked to EXP/VER ids
│   ├── verifications.md            # verification registry (+ backfilled references)
│   └── conventions.md              # repo-specific facts (env, data, budgets,
│                                   # the standard eval command)
└── .harness/
    └── checkers/                   # generated deterministic checks (see 4.3)
```

Heavy artifacts (checkpoints, raw logs, wandb runs) are **never committed**.
They live in their native stores and are referenced from run records by URI,
plus a content hash where cheap to compute.

### 3.3 Identity, branches, commits, iterations

- **ID**: `EXP-NNNN` — zero-padded, monotonically increasing, allocated by
  appending a row to `experiments/registry.md`. Folder name adds a slug:
  `EXP-0042-attn-dropout`.
- **Lineage**: `parent: EXP-NNNN` in spec frontmatter (`null` for roots).
- **Branches**: experiments are implemented on `exp/EXP-0042-attn-dropout`.
  Root experiments with an empty diff may run directly from a pinned commit
  on `main`.
- **Commit convention**: every commit made inside the loop is prefixed with
  the experiment id and iteration — `[EXP-0042.i3] fix attention mask`. No
  squashing or amending on the `exp/` branch while the loop runs: the commit
  trail is the record of how the agent fixed things, and the anchor for
  resuming an interrupted loop. Squashing is allowed only when merging to
  `main`, and `report.md` records the mapping.
- **Run pinning**: each iteration's `record.md` stores the exact commit hash
  the run used, plus a dirty-tree check result.
- **Merging code**: an accepted experiment's code merges to `main` only if
  the change should become the new default; otherwise the branch is kept (the
  commits are referenced by the records, so history stays reachable). The
  merge decision is recorded in `report.md`.
- Experiment docs (`experiments/`, `knowledge/`) always land on `main` —
  history and knowledge must be visible regardless of what happens to the
  code branch.

## 4. The three v1 skills

### 4.1 Skill: `experiment-logging` — documentation rules

Answers four questions for the agent: **where, what, how, when**.

**Where.** Per-experiment folder under `experiments/` in the target repo,
with per-iteration run records under `runs/iNN/`; index in
`experiments/registry.md`; durable conclusions in `knowledge/`; branch and
commit conventions as in 3.3.

**What.** Field inventory per file (frontmatter unless marked *body*):

| File | Fields |
|---|---|
| `spec.md` | id, slug, parent, compare_to, status, iteration, hypothesis, primary_metric (EXP only: the metric success is judged by), generalized diff (code scope allowlist + config diff + data + evaluation), assertions (expected metric behavior with thresholds vs compare_to), budgets (time, memory, steps/sec), expected artifact manifest, assumptions; *body*: idea, motivation, risks, enrichment log |
| `runs/iNN/record.md` | iteration, commit hash, branch, dirty flag, env snapshot (deps lockfile hash, torch/CUDA), resolved config snapshot (or its path + hash), setup split into model, data, training (hardware + hyperparameters) and evaluation (hardware + benchmarks + sampling params), launch command, artifact links (wandb URL, checkpoint URIs, log paths), final metrics, iteration verdict; *body*: run notes, incidents, per-iteration analysis |
| `report.md` | verdict (accepted / rejected / inconclusive), headline metric deltas vs compare_to, iterations count, knowledge-merge summary, merge-to-main decision; *body*: final analysis, conclusions, follow-up ideas |
| `registry.md` | one row per change (EXP and VER): id, slug, type, parent, status, iteration, headline metric, date |
| `knowledge/verifications.md` | verification registry: per verification — id, type (deterministic-script/metric-observation/ai-review), verifies, phase, gating, params, metric (name + kind quality/diagnostic + definition), implemented_by, backfilled references |

**How.** Dual format per section 2: YAML frontmatter validated against
`schemas/`, narrative in the body. Mutability is explicit: `spec.md` is a
living document (changed only through the approval gate, versioned via
iteration-tagged commits); run records are immutable once their iteration
ends; `report.md` is written once, at the terminal verdict.

**When.** Status drives completeness — each transition requires specific
fields to be filled, so "documentation is done" is checkable. Statuses cycle
per iteration:

| Status | When it is set | Must be complete by then |
|---|---|---|
| `planned` | spec approved — initially, or after the spec-diff approval gate of a `revise` | all of `spec.md` at current iteration |
| `running` | training launched | `runs/iNN/record.md`: commit, env, config, launch command, artifact links |
| `completed` | run finished, artifacts collected | `runs/iNN/record.md`: final metrics, artifact manifest satisfied |
| `analyzed` | iteration analysis written | `runs/iNN/record.md` body + iteration verdict |
| → `revise` | verdict says iterate | spec diff prepared for approval; on approval → `planned`, iteration+1 |
| → `accepted` / `rejected` / `inconclusive` | terminal verdict | `report.md`, knowledge merged, registry row updated |

### 4.2 Skill: `experiment-planning` — spec rules

How to produce and maintain a good `spec.md`.

- **Falsifiable hypothesis**: a claim with a concrete expected effect, not
  "should improve things".
- **Per-experiment primary metric**: `primary_metric` names the metric this
  experiment's success is judged by; the headline assertion and decision
  rule are stated in its terms.
- **Full setup specification**: the generalized diff covers code scope,
  config (model size, batch size, hyperparameters as a diff to the parent's
  resolved config), data (version/hash, splits), and evaluation (benchmarks,
  metric definitions). Root experiments pin all of it absolutely.
- **Grounded assertions**: assertions reference actual numbers from the
  records of `compare_to` experiments (default: the parent) — "val loss ≤
  2.31 − 0.02 by step 10k", never absolute guesses. The skill requires
  reading those records first.
- **Scoped diff**: allowlist of files and config keys the implementation may
  touch. This is what phase-1 scope enforcement checks against. Checker and
  experiment-doc directories are always outside the allowlist.
- **Decision rule declared up front**: what result means accept, reject,
  inconclusive, or revise — including early-kill thresholds used during
  training.
- **Budgets**: wall-clock, memory, throughput non-regression bounds.
- **Knowledge merge plan**: which knowledge files this experiment can update
  on acceptance.
- **Living-spec maintenance** (section 2): fold in every decision that could
  affect results, reproducibility, or interpretation; log obvious-but-real
  decisions in `assumptions`; batch all changes into one spec diff for the
  approval gate at the iteration boundary; escalate immediately only on
  direct contradiction with the spec.

The skill also defines the review step: an isolated-context agent reviews the
initial spec (and each iteration's spec diff) for measurability and minimal
scope before it goes to the human — the cheapest point for feedback.

### 4.3 Skill: `experiment-process` — running an experiment

The verification pipeline, ordered strictly by cost (fail fast). Full checker
catalog ships as `skills/experiment-process/references/checker-catalog.md`;
summary:

| Phase | Deterministic checks | AI review |
|---|---|---|
| **0 — before implementation** | spec schema valid; parent and compare_to records exist with real numbers; clean env recorded | spec review (falsifiable, measurable, minimal scope) |
| **1 — after implementation, before run** | lint/typecheck/tests; scope enforcement (`git diff` ⊆ allowlist); config diff vs parent touches only declared keys; data checks (train/val overlap, aug on train only, shapes); new component connectivity | ML-checklist code review; intent diff (implemented-vs-spec, isolated contexts, severity-calibrated) |
| **2 — first training steps** | smoke run; loss-at-init sanity; loss decreases; NaN/Inf hooks; dead-param and grad-norm checks; single-batch overfit; reproducibility (same seed → same losses); shuffled-labels negative control; **short-horizon ablation sanity** (see below); throughput/memory in budget; checkpoint round-trip; eval-mode check | diagnosis only, on failure — never a gate |
| **3 — during training** | watchdog: NaN, grad-norm band, loss spikes, LR schedule checkpoints, train/val gap, throughput/memory stability, heartbeat; early-kill rules from the spec | triggered on soft anomalies (plateaus, oscillations); advisory, gates only on critical |
| **4 — after training** | artifact manifest satisfied and valid; all spec assertions evaluated vs compare_to; final checkpoint round-trip | results review → iteration verdict proposal; aggregation into a single machine verdict |

**Short-horizon ablation sanity (phase 2).** When the diff adds a toggleable
code component: run K steps with the component disabled — the curve must
match the parent's (we didn't break the setup we started from); run K steps
with it enabled — the curve must differ (the component is actually
connected). Minutes of compute as part of the sanity stage. Not applicable to
config-only diffs (there the config *is* the diff). A full-length ablation
run is never automatic — only on explicit request (e.g. before claiming a
result in a paper).

**Verifier-change run.** Verifier changes go through the same loop, but in
place of phases 2–4: `v/static`, `v/scope`, `v/calibration` (pass every
known-good AND fail every known-bad case from the spec), `v/equivalence`
(the standing verifier: unchanged command, few steps, identical
loss/metrics/artifacts), `v/references` (backfill via the standard eval
command on frozen checkpoints). Verdicts land in the change's
`runs/iNN/checks/` as usual; acceptance merges the registry entry into
`knowledge/verifications.md`.

Cross-cutting rules baked into the skill:

- **Verdict format**: every check emits
  `{id, phase, verdict, severity, evidence}` into
  `experiments/<id>/runs/iNN/checks/`; the loop reacts to verdicts, the AI
  diagnostician reads evidence.
- **Checker isolation**: deterministic checkers are generated per target repo
  into `.harness/checkers/` by a *different* agent than the one implementing
  the experiment; that directory is excluded from every spec's scope
  allowlist, and any checker diff itself goes through intent review. The
  executor never sees verifier prompts.
- **Iteration verdict**: one machine-readable outcome per iteration —
  `accept / revise / reject / escalate-to-human`. `revise` re-enters the loop
  through the spec-diff approval gate (iteration+1). Disagreement between
  executor and reviewers about the spec always escalates to a human.

## 5. Use cases and scenarios

**S1 — Initialize the harness in a target repo.** Agent scaffolds
`experiments/`, `knowledge/`, `.harness/checkers/` from templates, fills
`knowledge/conventions.md` by inspecting the repo (env, entry points, data
paths), and generates the first set of repo-specific deterministic checkers.

**S2 — Document the starting point.** User: "make current main our reference
run". Agent writes a root-experiment spec (no parent; data, benchmarks,
metrics, hyperparameters pinned absolutely; assertions = sanity only), runs
phases 2–4 on unchanged code, fills the run record and report. Later
experiments name it in `compare_to`.

**S3 — Plan an experiment.** User describes an idea. Agent (planning skill)
reads the parent's and compare_to records, drafts `spec.md` with grounded
assertions and a scoped generalized diff, requests spec review, gets user
sign-off, sets `planned`, creates the `exp/` branch.

**S4 — Run the loop.** Agent (process skill) implements within scope, passes
phase 1, launches with phase 2 sanity gates, monitors via phase 3 watchdog,
lands phase 4 checks; `runs/iNN/record.md` fills as the run progresses. On
`revise`: the agent prepares the spec diff (material changes + assumptions),
the human approves it, and iteration N+1 starts. Terminal verdicts exit to
S5.

**S5 — Analyze and merge knowledge.** Agent (logging + planning skills)
writes `report.md`, sets the terminal verdict, merges conclusions into
`knowledge/`, updates the registry, decides on merging code to `main`.

**S6 — Query history.** "What have we tried on attention dropout?", "what is
the current reference?" — answered from `knowledge/` first, then the
registry and per-experiment frontmatter (all machine-readable, so greppable
and parseable).

**S7 — Resume an interrupted loop.** Agent reads the registry row and spec
frontmatter (status + iteration), finds the last `[EXP-NNNN.iN]` commit and
the current iteration's `runs/iNN/` contents, and continues from the first
incomplete step of that iteration's cycle.

**S8 — Implement a verifier.** User: "log acceptance length per block
position" (or "add a leakage check", "add an AI reviewer for masks"). Agent
opens a verifier change (`VER-NNNN`) with a spec declaring what it verifies,
calibration cases, and a registry-entry draft; implements it through the
loop; passes calibration + equivalence + reference backfill; on acceptance
the verification merges into `knowledge/verifications.md` and its check is
generated. The new metric/check is now usable by future specs and analyses.

**S9 — Bootstrap a repo from scratch.** The root experiment's spec is
written first and doubles as the implementation contract; the codebase is
built inside the root experiment's iteration 1 (commits `[EXP-0001.i1]`),
kept green by static checks and smoke as it grows; then the root experiment
pins the finished commit and runs phases 2–4 (S2). The first verifier
changes (the standing `equivalence` verifier, metric observations for
whatever the code logs) follow immediately after.

## 6. V1 scope

**In:**

- The three skills, with schemas and templates — including verifier changes
  (`VER`) sharing the experiment machinery, the verification registry with
  metric-observation entries, and reference backfill via the standard eval
  command.
- Checker catalog as a reference document; skills able to generate phase 0–2
  deterministic checkers for a concrete repo (phases 3–4 may start as
  documented procedures rather than generated code).
- End-to-end acceptance test: in a real repo (e.g. GDFlash), document one
  root experiment and one child experiment using only the skills — including
  at least one `revise` iteration with a spec-diff approval — every status
  transition, schema validation passing, knowledge merged.

**Out (deferred):**

- **Arenas**: comparing experiments on chosen metrics and setups, with manual
  marking of baselines (the runs to beat). Until then, `compare_to` in specs
  covers the need.
- Multi-seed variance / statistical significance (explicitly postponed —
  costly; the schema reserves a `seeds` field for later).
- Unattended orchestration (daemon/CI loop); v1 is agent-driven per session.
- wandb API integration for auto-pulling metrics; v1 records links and copies
  final numbers manually.
- Holdout/rotating verifier checks (anti-Goodhart hardening beyond checker
  isolation).
- Full-length ablation runs (short-horizon ablation sanity in phase 2 covers
  the automatic case).

## 7. Open questions

1. **Registry format**: `registry.md` table (human-first) vs `registry.jsonl`
   (machine-first). Current choice: markdown table, since frontmatter of the
   per-experiment files already covers machine needs.
2. **Where generated checkers live**: `.harness/checkers/` in the target repo
   (current choice) vs inside the experiment folder (per-experiment variants).
3. **Arena design** (deferred past v1): how experiments are grouped for
   comparison, how baseline marking works, and whether arena state lives in
   `knowledge/` or its own directory.
