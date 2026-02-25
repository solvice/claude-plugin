---
name: analyze-solution
description: Analyze a VRP solution for quality, constraint violations, and optimization opportunities
argument-hint: "[solve-id]"
---

Analyze VRP solution `$ARGUMENTS`.

## Steps

1. Use `vrp-get-all` with id=`$ARGUMENTS` for the executive summary (score, metrics, top issues)
2. If hard score < 0: use `vrp-get-explanation` to see constraint violations
3. If unserved jobs > 0: use `vrp-get-explanation` to identify blocking constraints
4. Only use `vrp-get-request` if you need to understand input constraints for a fix

## Scoring

- **Hard** must be 0 (feasibility)
- **Medium** = unassigned jobs (0 means all assigned)
- **Soft** = optimization quality (lower magnitude = better)

## Output

Assessment, Score, Top Issues, Recommendations — concise and actionable.
