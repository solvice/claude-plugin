---
name: constraint-glossary
description: VRP constraint reference for the Solvice API. Load automatically when diagnosing constraint violations, unservedReasons codes, or infeasible scores. Covers all SCREAMING_SNAKE_CASE constraint names with root causes, developer mistakes, and JSON fix examples.
user-invocable: false
---

This skill provides the full constraint glossary. It is loaded automatically by Claude when diagnosing VRP constraint violations — you do not need to invoke it directly.

See the full glossary below.

---

# VRP Constraint Error Glossary

Quick reference: constraint name → what broke → how to fix it.

Each entry uses the `SCREAMING_SNAKE_CASE` name that appears in `unservedReasons` and score explanations.

---

## Hard Constraints

---

## TRIP_CAPACITY

**Type:** HARD
**Triggered when:** The total load of all jobs on a single trip exceeds the resource's declared capacity.
**Typical mistake:** Setting `resource.capacity` too small while jobs carry non-zero `load` values.
**Fix:** Increase `resource.capacity` to at least the peak trip load, or reduce `job.load`, or add more resources.
**Example fix:**
```json
// Before
{"resources": [{"name": "truck-1", "capacity": [100]}],
 "jobs": [{"name": "J-1", "load": [60]}, {"name": "J-2", "load": [60]}]}

// After
{"resources": [{"name": "truck-1", "capacity": [150]}]}
```

---

## RESOURCE_CAPACITY

**Type:** HARD
**Triggered when:** Cumulative load across all trips on a resource exceeds the resource-level capacity.
**Typical mistake:** Confusing trip-level (`TRIP_CAPACITY`) with resource-level capacity.
**Fix:** Raise `resource.capacity` to accommodate total load across all trips.

---

## RESOURCE_CAPACITY2

**Type:** HARD
**Triggered when:** A second capacity dimension is exceeded.
**Typical mistake:** Configuring the first dimension correctly but forgetting the second (e.g., weight fine but volume exceeded).
**Fix:** Check all dimensions — `resource.capacity` is an array; ensure index 1 matches maximum expected sum of `job.load[1]`.

---

## TAG_HARD

**Type:** HARD
**Triggered when:** A job requires a tag marked `"hard": true` but the assigned resource does not have it.
**Typical mistake:** Adding a required tag to a job but forgetting to add the matching tag to any resource — or misspelling it.
**Fix:** Ensure every resource that should serve the job has the exact tag string in `resource.tags`.
**Example fix:**
```json
// Before
{"job": {"tags": [{"name": "plumbing", "hard": true}]},
 "resource": {"tags": ["electrical"]}}

// After
{"resource": {"tags": ["electrical", "plumbing"]}}
```

---

## TYPE_REQUIREMENT

**Type:** HARD
**Triggered when:** A job's type requirement is not satisfied by the assigned resource's shift tags.
**Fix:** Add the job's tag to `shift.tags`, or assign to a resource/shift that has the correct tags.

---

## DATE_TIME_WINDOW_CONFLICT

**Type:** HARD
**Triggered when:** Job arrival falls outside all declared hard time windows, or the job's time window does not overlap with any resource's available shift on any date.
**Typical mistake:** Setting tight windows that no resource can reach in time; defining a job window on a date for which no resource has a shift; wrong timezone causing phantom non-overlap.
**Fix:** Widen the window, add a resource shift that covers the job's window date/time, add more resources, or verify ISO 8601 timestamps are timezone-consistent.
**Example fix:**
```json
// Before (window too narrow)
{"windows": [{"from": "2024-03-15T09:00:00Z", "to": "2024-03-15T10:00:00Z", "hard": true}]}

// After
{"windows": [{"from": "2024-03-15T09:00:00Z", "to": "2024-03-15T12:00:00Z", "hard": true}]}
```
```json
// Before — no shift on 2024-03-16
{"resources": [{"shifts": [{"from": "2024-03-15T08:00:00Z", "to": "2024-03-15T17:00:00Z"}]}],
 "jobs": [{"windows": [{"from": "2024-03-16T09:00:00Z", "to": "2024-03-16T12:00:00Z"}]}]}

// After — add matching shift
{"shifts": [{"from": "2024-03-15T08:00:00Z", "to": "2024-03-15T17:00:00Z"},
            {"from": "2024-03-16T08:00:00Z", "to": "2024-03-16T17:00:00Z"}]}
```

---

## SHIFT_END_CONFLICT

**Type:** HARD
**Triggered when:** A resource's trip finishes after `shift.to` with no `overtimeEnd` defined.
**Typical mistake:** Underestimating total route duration; packing too many jobs into a short shift.
**Fix:** Extend `shift.to`, add `overtimeEnd`, reduce jobs per shift, or add more resources.
**Example fix:**
```json
// After — add overtime buffer
{"from": "2024-03-15T08:00:00Z", "to": "2024-03-15T17:00:00Z",
 "overtimeEnd": "2024-03-15T18:00:00Z"}
```

---

## OVERTIME_END_CONFLICT

**Type:** HARD
**Triggered when:** Work extends past `shift.overtimeEnd`.
**Fix:** Raise `overtimeEnd` further, remove jobs, or redistribute to other resources.

---

## DISALLOWED_RESOURCES

**Type:** HARD
**Triggered when:** Job is assigned to a resource in its `disallowedResources` list.
**Typical mistake:** Blacklisting resources without providing any alternative capable of serving the job.
**Fix:** Remove the resource from `job.disallowedResources`, add other capable resources, or use `partialPlanning: true`.

---

## JOB_PRECEDENCE

**Type:** HARD
**Triggered when:** Day-index difference between linked jobs falls outside `[minimumDaysBetween, maximumDaysBetween]`.
**Typical mistake:** Planning horizon too compressed to honor the minimum gap; jobs listed in wrong order.
**Fix:** Widen the planning horizon; check that the leader job appears first in the `jobs` array.

---

## LINKED_JOB_CONFLICT

**Type:** HARD
**Triggered when:** Jobs in a `SAME_TIME`, `SAME_TRIP`, `SAME_RESOURCE`, or `PICKUP_AND_DELIVERY` relation violate the structural requirement.
**Typical mistake:** For `PICKUP_AND_DELIVERY`, listing delivery before pickup in the `jobs` array.
**Fix:** Correct the job order (pickup first, delivery second); use `SAME_RESOURCE` instead of `SAME_TRIP` for multi-day requirements.
**Example fix:**
```json
// Before
{"type": "PICKUP_AND_DELIVERY", "jobs": ["delivery-job", "pickup-job"]}

// After
{"type": "PICKUP_AND_DELIVERY", "jobs": ["pickup-job", "delivery-job"]}
```

---

## PLANNED_RESOURCE

**Type:** HARD
**Triggered when:** A job has `plannedResource` set but that resource cannot serve it.
**Fix:** Remove or correct `job.plannedResource`, or fix the blocking constraint on the named resource.

---

## PLANNED_ARRIVAL

**Type:** SOFT
**Triggered when:** The job's actual arrival deviates from `plannedArrival`. The deviation is penalized proportionally.
**Fix:** If the arrival accuracy matters, verify that `plannedArrival` falls within `job.windows` and within the resource's shift. If the job is unassigned, fix the blocking hard constraints first.

---

## PLANNED_DATE

**Type:** HARD
**Triggered when:** `plannedDate` is set but no resource has a shift on that date.
**Fix:** Add a resource shift for the planned date, or remove `plannedDate`.

---

## HARD_JOBS

**Type:** HARD
**Triggered when:** `"hard": true` is set on a job in partial planning mode but the solver cannot feasibly assign it.
**Typical mistake:** Marking too many jobs as hard; the flag overrides the solver's ability to drop infeasible jobs.
**Fix:** Set `"hard": false` for jobs that can be left unserved if necessary.

---

## MAX_DRIVE_TIME

**Type:** HARD
**Triggered when:** Total driving time on a trip exceeds `resource.maxDriveTimeInSeconds`.
**Fix:** Increase `maxDriveTimeInSeconds`, assign distant jobs to separate resources, or add more resources.

---

## MAX_DRIVE_TIME_JOB

**Type:** HARD
**Triggered when:** Travel time from the previous stop to a job exceeds `resource.maxDriveTimeJob` (per-leg limit).
**Fix:** Remove `maxDriveTimeJob` if per-leg limiting was not intended, or increase it to cover actual maximum leg distance.

---

## Medium Constraints

---

## UNSERVED_JOBS

**Type:** MEDIUM
**Triggered when:** A job cannot be assigned and ends up unserved.
**Fix:** Enable `"partialPlanning": true` to allow unserved jobs rather than an infeasible solution. Use `job.priority` to rank importance. Use `options.explanation` to identify why each job is unserved.
**Example fix:**
```json
{"options": {"partialPlanning": true,
             "explanation": {"enabled": true, "onlyUnassigned": true}}}
```

---

## Soft Constraints

---

## TAG_SOFT

**Triggered when:** Job has a tag with `"hard": false` and the assigned resource doesn't have it.
**Fix:** Change to `"hard": true` if mandatory, or increase `tag.weight` for stronger preference.

---

## TRAVEL_TIME

**Always active.** Penalizes total route travel time.
**Tune:** `options.weights.travelTimeWeight` (default 1). Increase to prioritize shorter routes.

---

## END_LOCATION_TRAVEL_TIME

**Triggered when:** Last job is far from shift end location, causing a large return penalty.
**Fix:** Omit `shift.end` if the resource doesn't return to depot, or set `ignoreTravelTimeFromLastJob: true`.

---

## RESOURCE_USAGE / RESOURCE_ACTIVATION

**Triggered when:** A resource is active (has jobs assigned), contributing a per-resource penalty to encourage consolidation.
**Tune:** `options.weights.minimizeResourcesWeight`. Increase to consolidate onto fewer vehicles. Set to `0` to disable consolidation.

---

## Diagnostic Quick Reference

| Constraint | In `unservedReasons`? | First thing to check |
|---|---|---|
| `TRIP_CAPACITY` | Yes | `resource.capacity` vs sum of `job.load` |
| `TAG_HARD` | Yes | Tag name match on both job and resource |
| `DATE_TIME_WINDOW_CONFLICT` | Yes | `window.to` reachable given travel; resource shift exists on window date |
| `SHIFT_END_CONFLICT` | Yes | `shift.to` covers total route duration |
| `DISALLOWED_RESOURCES` | Yes | At least one resource not blacklisted |
| `LINKED_JOB_CONFLICT` | Yes | Job order in relation is correct |
| `PLANNED_RESOURCE` | Yes | Pinned resource satisfies all other constraints |
| `MAX_DRIVE_TIME` | Yes | `maxDriveTimeInSeconds` vs actual route distance |
| `UNSERVED_JOBS` | Medium | Enable `partialPlanning: true` |
