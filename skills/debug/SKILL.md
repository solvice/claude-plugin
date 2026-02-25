---
name: debug
description: Diagnose a VRP solve result. Given a solve run ID, fetches explanation and request data in parallel, triages hard constraint violations and unassigned jobs, and returns ranked root causes with exact field paths and before/after fix values.
argument-hint: "[solve-id]"
---

# Solvice Debug

Diagnose infeasibility and unassigned jobs for a VRP solve run. Produces a structured diagnosis with exact field paths, root causes, and concrete fixes.

## Argument Handling

If a solve run ID was provided as `$ARGUMENTS`, use it directly.

If no argument was given, ask:
> Which solve run ID should I diagnose? (Find it in the API response as the `solveId` field.)

---

## Step 1: Fetch Data

Call `vrp-get-all` with the solve run ID. This returns the original request, solution, and explanation in a single call.

---

## Step 2: Score Triage

```
SCORE SUMMARY
─────────────────────────────────────────────
hardScore:    <value>   (<interpretation>)
mediumScore:  <value>   (<interpretation>)
softScore:    <value>   (<interpretation>)
feasible:     <true/false>
─────────────────────────────────────────────
```

Interpretations:
- `hardScore: 0` → "feasible — all hard constraints satisfied"
- `hardScore: -N` → "INFEASIBLE — N hard violations must be fixed"
- `mediumScore: 0` → "all jobs assigned"
- `mediumScore: -N` → "N jobs unassigned"
- `softScore: -N` → "optimization penalty (travel/wait cost)"

If `hardScore == 0` and `mediumScore == 0`: print "No issues found" and stop.

---

## Step 3: Hard Constraint Violations

For each violation, output:

```
HARD VIOLATION #<n>: <constraintName>
  Affected job:      request.jobs[<index>] ("<name>")
  Affected resource: request.resources[<index>] ("<name>")
  Root cause:        <one sentence>

  Field to fix:      <exact path>
  Current value:     <value>
  Suggested fix:     <before → after>

  Why:               <one to two sentences>
```

### Constraint → Root Cause Mapping

| Constraint | Root cause | What to check |
|---|---|---|
| `DATE_TIME_WINDOW_CONFLICT` | Job arrival outside hard window, or window doesn't overlap any resource shift | `jobs[i].windows[0].from/to` vs travel time and `resources[j].shifts[k].from/to` |
| `TRIP_CAPACITY` | Route load exceeds vehicle capacity | Sum of `jobs[i].load` on route vs `resources[j].capacity` |
| `TAG_HARD` | Job requires a tag the resource doesn't have | `jobs[i].tags` vs `resources[j].tags` |
| `SHIFT_END_CONFLICT` | Route finishes after shift end | `resources[j].shifts[k].to` vs total route duration |
| `JOB_PRECEDENCE` | Relation order or day-gap constraint violated | `relations` array — check job order and day gap settings |
| `MAX_DRIVE_TIME` | Total drive time exceeds resource limit | `resources[j].maxDriveTimeInSeconds` vs actual route distance |
| `DISALLOWED_RESOURCES` | Job assigned to a blacklisted resource | `jobs[i].disallowedResources` |

If more than 10 violations of the same type, group them:

```
HARD VIOLATION: <constraintName> — <N> occurrences
  Affected jobs: request.jobs[i], request.jobs[j], ...
  Common root cause: <explanation>
  Fix pattern: <field and how to change it>
```

---

## Step 4: Unassigned Jobs

If `mediumScore < 0`, for each unassigned job:

```
UNASSIGNED JOB #<n>: "<name>" (request.jobs[<index>])
  Reasons: <comma-separated reason codes>

  — <reason>: <explanation>
    Field:    <exact path>
    Current:  <value>
    Fix:      <concrete change>
```

### Reason Mapping

| Reason | Meaning |
|---|---|
| `DATE_TIME_WINDOW_CONFLICT` | Window doesn't overlap any resource shift, or unreachable given travel time |
| `SHIFT_TIME_CONFLICT` | No shift long enough for job duration |
| `TRIP_CAPACITY` | Job load exceeds all vehicle capacity |
| `TAGS` | No resource has all required tags |

If `unservedReasons` is absent from the response, note:
> Re-run with `options.explanation.enabled: true` and `options.explanation.onlyUnassigned: true` to get per-job reasons.

---

## Step 5: Problem Classification

```
CLASSIFICATION: <Over-constrained | Data error | Under-resourced>

  Reasoning: <2–4 sentences — name the jobs and resources involved>

  Recommended action: <one concrete next step>
```

- **Over-constrained** — Multiple requirements cannot all be satisfied simultaneously. Needs more resources or relaxed constraints.
- **Data error** — A single field has an impossible value (window ends before it starts, load > all capacity). Fix the bad value.
- **Under-resourced** — Total demand exceeds total supply. Add resources or extend shifts.

---

## Next Steps

```
NEXT STEPS
  1. Apply the field fixes above.
  2. Re-solve with vrp-solve-sync to verify.
  3. If unassigned jobs remain, re-run with
     options.explanation.enabled: true, onlyUnassigned: true.
  4. If Over-constrained: add resources, extend shifts,
     remove conflicting tags, or relax time windows.
```

---

## Reference

The `constraint-glossary` skill contains full documentation for every constraint type with root causes, developer mistakes, and JSON fix examples. Reference it when diagnosing a constraint not covered in the table above.
