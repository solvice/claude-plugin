---
name: debug
description: Diagnose constraint violations and unassigned jobs in a VRP solution
argument-hint: "[solve-id]"
---

Debug constraint violations for VRP solution `$ARGUMENTS`.

## Steps

1. Use `vrp-get-explanation` for the constraint breakdown
2. Only use `vrp-get-request` if you need to check specific job/resource settings
3. Trace violations to root causes: time windows, capacity, skills, or relations

## Output

For each violated constraint:
- Constraint name and count
- Root cause analysis
- Suggested fix with exact field paths and before/after values
