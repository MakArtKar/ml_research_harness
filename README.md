# ML Research Harness

A **source of skills** for running ML research with AI agents.

Nothing here is imported or executed by research code. Instead, agents
(Claude Code and similar) load the skills from this repo while working inside
a **target research repo** — a training codebase — and the skills tell them
how to plan, run, verify, and document experiments: the full loop from
hypothesis to validated knowledge.

## Core ideas

- **One experiment entity.** Every experiment has a nullable parent and a
  possibly-empty *generalized diff* against it — code, config
  (hyperparameters, model size, batch size), data, and evaluation
  (benchmarks, metrics). A root experiment pins everything absolutely.
  "Baseline" is a role, not a type: the run you want to beat, named in the
  spec's `compare_to` (a paper reimplementation is a child experiment *and*
  a baseline for later work).
- **Agent loop with iterations.** An experiment is one hypothesis;
  iterations are attempts at it. Each iteration cycles
  `planned → running → completed → analyzed`; the verdict `revise` loops
  back with iteration+1, terminal verdicts
  (`accepted`/`rejected`/`inconclusive`) end it. Commits are prefixed
  `[EXP-0042.i3]` and never squashed mid-loop, so the fixing history is
  preserved and an interrupted loop is resumable from git + frontmatter
  alone.
- **Living spec.** The spec is the contract. Any decision it did not dictate
  that could affect results, reproducibility, or interpretation gets folded
  back into it; obvious-but-real decisions land in an `assumptions` list.
  Changes are batched into one spec diff per iteration and approved by a
  human at the iteration boundary.
- **Verification pyramid.** Checks are ordered strictly by cost across five
  phases (before implementation → after implementation → first training
  steps → during training → after training). Everything that can be a
  deterministic check is one; AI review is reserved for what hard checks
  cannot express (spec review, intent diff, results interpretation). Every
  check emits a machine-readable verdict
  `{id, phase, verdict, severity, evidence}`.
- **Verifications are first-class.** The registry
  (`knowledge/verifications.md`) lists every verification: deterministic
  scripts, AI-review checkers, and metric observations — a metric plus how
  to read it, as `primary` (headline quality → assertions), `proxy`
  (localizes what to improve → advisory), or `diagnostic` (training health
  → watchdog bands). Checkers are derived from the registry. Implementing a
  verifier (a metric, a script, an AI checker) is a **verifier change**
  (`VER-NNNN`) that uses the same spec-and-loop machinery as an experiment,
  with calibration (must fail known-bad cases), equivalence (must not alter
  training), and reference backfill on frozen checkpoints in place of a
  training run. Verifications gate the loop *and* drive analysis: a revealed
  problem seeds a follow-up experiment.
- **Checker isolation.** Deterministic checkers are generated per target
  repo by a *different* agent than the one implementing the experiment, live
  outside the implementer's allowed scope, and verifier prompts are never
  shown to the executor — otherwise the loop learns to pass checks instead
  of being correct.
- **Dual format everywhere.** Every document is one Markdown file:
  schema-validated YAML frontmatter as the machine-readable source of truth,
  narrative body for humans. No parallel JSON copies.

## Repository layout

```
skills/
  experiment-logging/     # where/what/how/when of experiment documentation
  experiment-planning/    # specs: hypotheses, generalized diffs, grounded
                          # assertions, living-spec maintenance
  experiment-process/     # the loop: phases 0-4, verdicts, resumption
    references/
      checker-catalog.md  # full checker list with pass criteria
schemas/                  # JSON Schemas for document frontmatter and verdicts
templates/                # file templates the skills instantiate
docs/
  v1-design.md            # design document: architecture, use cases, scope
```

## The skills

| Skill | What it governs |
|---|---|
| [experiment-logging](skills/experiment-logging/SKILL.md) | Where records live (folders, branches, commit conventions), which fields go in which file, the dual format, and which fields must be complete at each status transition |
| [experiment-planning](skills/experiment-planning/SKILL.md) | Writing specs: falsifiable hypotheses, diffs across all four axes, assertions grounded in recorded numbers, decision rules with early-kill thresholds, spec enrichment with human approval gates, knowledge merges |
| [experiment-process](skills/experiment-process/SKILL.md) | Running the loop: verification phases 0–4, deterministic-first gating, check verdicts, iteration verdicts and the revise loop, checker isolation, resuming interrupted loops |

## Using the harness

Skills are self-contained: hand a `SKILL.md` to your agent in the prompt, or
install it where your agent discovers skills — by default in the target
project (e.g. its `.claude/skills/`). Then:

1. **Initialize** — ask the agent to scaffold the target repo per the
   logging skill: `experiments/` with a registry, `knowledge/` with
   conventions filled by inspecting the repo, and (via a separate checker
   agent) `.harness/checkers/`.
2. **Document the starting point** — a root experiment: setup pinned
   absolutely, sanity assertions, phases 2–4 on unchanged code. Later
   experiments cite it in `compare_to`.
3. **Plan** — draft a spec for your idea (planning skill), pass isolated
   spec review, sign off.
4. **Run the loop** — implement, verify, launch, monitor, judge (process
   skill). On `revise`, approve the spec diff and the next iteration starts.
5. **Conclude** — report, knowledge merge, registry update, code-merge
   decision.

A target repo ends up looking like:

```
experiments/
  registry.md             # one row per change (EXP and VER)
  EXP-0042-attn-dropout/
    spec.md               # living plan
    runs/i01/ i02/ ...    # per-iteration record.md + checks/
    report.md             # final synthesis
  VER-0003-.../           # verifier change: identical layout
knowledge/
  findings.md             # validated conclusions, linked to EXP/VER ids
  verifications.md        # verification registry (+ backfilled references)
  conventions.md          # repo-specific facts, incl. the standard eval command
.harness/
  checkers/               # generated checks, off-limits to the implementer
```

## Status

V1 — see [docs/v1-design.md](docs/v1-design.md) for the architecture, use
cases, and scope. Deliberately deferred: arenas (cross-experiment comparison
with manual baseline marking), multi-seed statistics, unattended
orchestration, wandb API integration, full-length ablation runs.
