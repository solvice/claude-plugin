---
name: move-job
description: Move a job to a different resource and evaluate the impact
argument-hint: "[solve-id] [job-name] [target-resource]"
---

Move job `$1` to resource `$2` in solution `$0`.

## Steps

1. Use `vrp-get-solution` to see current assignments
2. Find the current position of the job and the target resource
3. Use `vrp-change` with: `{changes: [{job: "$1", resource: "$2", after: null}], operation: "evaluate"}`
4. Review the response: new score, updated routes, any new violations
5. If violations appear, suggest alternatives

## Tips

- `evaluate` = quick scoring (~200ms), does not commit
- `solve` = re-optimize (~5s), commits the change
- Set `after` to position the job after a specific other job
