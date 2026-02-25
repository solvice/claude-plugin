---
name: compare
description: Compare two VRP solutions side-by-side
argument-hint: "[solve-id-1] [solve-id-2]"
---

Compare VRP solutions `$0` and `$1`.

## Steps

1. Use `vrp-get-all` for each ID to get executive summaries
2. Compare scores: hard (feasibility) > medium (unserved) > soft (optimization)
3. Compare metrics: routes, visits, travel time, unserved jobs

## Output

Side-by-side table with scores, metrics, and a recommendation of which is better and why.
