---
# Validate against schemas/report.schema.json. Written once, at the terminal verdict.
experiment: EXP-NNNN
verdict: accepted       # accepted | rejected | inconclusive
iterations: 1
deltas:
  - metric: val/loss
    vs: EXP-NNNN
    reference: 0.0
    value: 0.0
    delta: 0.0
knowledge_merge:
  - file: knowledge/findings.md
    summary: ""
merge_to_main: false
merge_commit: null
---

# EXP-NNNN — report

## Analysis

<!-- Final synthesis across iterations: what the numbers say vs the
     hypothesis and the decision rule. -->

## Conclusions

<!-- Durable takeaways. These are what gets merged into knowledge/. -->

## Follow-ups

<!-- Experiment ideas this result suggests. -->
