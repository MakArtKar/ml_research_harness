---
# Validate against schemas/metrics.schema.json.
metrics:
  - name: val/example_metric
    kind: primary          # primary | proxy | diagnostic
    definition: ""
    logged_by: ""
    added_by: ENG-NNNN
    verification:
      type: assertion      # assertion | band | advisory | analysis-only
      phase: 4
      params: {}
    references: []         # backfilled reference values with provenance
---

# Metric registry

Every logged metric has an entry above; every entry maps the metric to its
verification — logging a new metric is preparing a new verification. The
checker agent derives phase-3 bands and phase-4 assertion inputs from this
file.

<!-- Narrative notes per metric go here: rationale, known caveats,
     interpretation guidance. -->
