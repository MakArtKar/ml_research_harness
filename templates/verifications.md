---
# Validate against schemas/verifications.schema.json.
verifications:
  - id: equivalence
    type: deterministic-script
    verifies: >
      A result-neutral change (refactor, instrumentation) did not alter
      training: re-running the unchanged command for a few steps yields
      identical loss, metrics, and artifacts.
    gating: gate
    params:
      steps: 0            # few-step horizon
      tolerance: 0.0
    implemented_by: VER-NNNN
  - id: metric/example
    type: metric-observation
    verifies: ""
    phase: 4
    gating: advisory
    params: {}
    metric:
      name: val/example_metric
      kind: primary        # primary | proxy | diagnostic
      definition: ""
      logged_by: ""
    implemented_by: VER-NNNN
    references: []         # backfilled via the standard eval command on frozen checkpoints
---

# Verification registry

Every verification available in this repo: deterministic scripts, AI-review
checkers, and metric observations (a metric plus how to read it). Logging a
new metric is implementing a new verification. The checker agent derives
phase-3 bands and phase-4 assertion inputs from this file.

Verifications serve two purposes: **gating** the experiment loop, and
**analysis** — a verification that reveals a problem in the current
experiment usually seeds a follow-up experiment that closes it (see the
experiment-planning skill), not a revise of the current one.

<!-- Narrative notes per verification: rationale, caveats, interpretation
     guidance. -->
