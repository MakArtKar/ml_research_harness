---
# Validate against schemas/spec.schema.json.
id: EXP-NNNN            # EXP-NNNN (experiment) or VER-NNNN (verifier change)
slug: short-slug
parent: null            # EXP-NNNN of the parent, or null for a root experiment
compare_to: []          # runs to beat; assertions reference their numbers (default: [parent])
status: planned         # planned | running | completed | analyzed | accepted | rejected | inconclusive
iteration: 1
hypothesis: >
  One falsifiable claim with a concrete expected effect, not "should improve
  things".
primary_metric: ""      # EXP only: the metric success is judged by — its delta
                        # vs compare_to is the headline result (chosen per
                        # experiment; must exist in knowledge/verifications.md)
diff:
  code:
    scope: []           # file allowlist the implementation may touch
    summary: ""
  config: {}            # keys changed vs parent's resolved config: {key: {from: X, to: Y}}
  data: {}              # changes vs parent; roots pin dataset/version/hash/splits here
  evaluation: {}        # benchmark/metric + eval-setup changes (hardware, batch,
                        # sampling params) vs parent; roots pin the full list here
assertions:
  - id: example-assertion
    metric: val/loss
    at: "step 10000"
    condition: "<= 0.00"   # grounded in a real number from a compare_to record
    source: EXP-NNNN
budgets:
  wall_clock_hours: 0
  gpu_memory_gb: 0
  min_steps_per_sec: 0
artifacts_expected: []
decision_rule: >
  What result means accept / reject / inconclusive / revise, including
  early-kill thresholds used during training.
assumptions: []         # obvious-but-real off-spec decisions, logged during iterations
# verifier:             # VER changes only — what verification is implemented
#   type: metric-observation   # deterministic-script | metric-observation | ai-review
#   verifies: ""
#   calibration:               # >=1 known-good (pass) and >=1 known-bad (fail) case
#     - {case: "", expect: pass}
#     - {case: "", expect: fail}
#   registry_entry: {}         # draft knowledge/verifications.md entry to merge on acceptance
---

# EXP-NNNN — Title

## Idea

<!-- What the experiment is about, in a paragraph. -->

## Motivation

<!-- Why this is worth compute; what prior experiments or findings suggest it. -->

## Risks

<!-- What could invalidate the result or waste the run. -->

## Enrichment log

<!-- One entry per approved spec diff:
- i02 (2026-08-31): declared warmup schedule explicitly — was undocumented and
  the implementation had to pick one. Approved by <user>.
-->
