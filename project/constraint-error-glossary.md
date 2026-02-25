# VRP Constraint Error Glossary

Quick reference: constraint name → what broke → how to fix it.

Each entry uses the `SCREAMING_SNAKE_CASE` name that appears in `unservedReasons` and score explanations.

---

## Hard Constraints

---

## TRIP_CAPACITY

**Type:** HARD
**Score level:** hard
**Triggered when:** The total load of all jobs on a single trip (day/shift) exceeds the resource's declared capacity for at least one dimension.
**Typical developer mistake:** Setting `resource.capacity` too small (or omitting it entirely, which defaults to 0) while jobs carry non-zero `load` values, or adding too many jobs to a route without raising the capacity ceiling.
**Fix:** Increase `resource.capacity` to at least the sum of the maximum simultaneous load expected on any one trip, OR reduce the `job.load` values, OR add more resources to spread the load.
**Example fix:**
```json
// Before – resource capacity too small
{
  "resources": [{"name": "truck-1", "capacity": [100]}],
  "jobs": [
    {"name": "J-1", "load": [60]},
    {"name": "J-2", "load": [60]}
  ]
}

// After – capacity raised to cover peak load
{
  "resources": [{"name": "truck-1", "capacity": [150]}],
  "jobs": [
    {"name": "J-1", "load": [60]},
    {"name": "J-2", "load": [60]}
  ]
}
```

---

## RESOURCE_CAPACITY

**Type:** HARD
**Score level:** hard
**Triggered when:** The cumulative load on a resource across its entire planning period (all trips combined) exceeds the declared resource-level capacity.
**Typical developer mistake:** Confusing trip-level capacity (`TRIP_CAPACITY`) with resource-level capacity; setting resource capacity lower than the combined load across all jobs in the period.
**Fix:** Raise `resource.capacity` to accommodate the total load across all trips, or split work across multiple resources.
**Example fix:**
```json
// Before – resource period capacity exceeded
{"name": "van-1", "capacity": [200]}

// After
{"name": "van-1", "capacity": [500]}
```

---

## RESOURCE_CAPACITY2

**Type:** HARD
**Score level:** hard
**Triggered when:** A second capacity dimension on the resource is exceeded (same logic as `RESOURCE_CAPACITY` but for the second index in the `capacity` array).
**Typical developer mistake:** Configuring the first capacity dimension correctly but forgetting the second (e.g., weight is fine but volume is over-limit).
**Fix:** Check all dimensions: `resource.capacity` is an array — ensure index 1 matches the maximum expected sum of `job.load[1]` values on any trip.
**Example fix:**
```json
// Before – second dimension too small
{"name": "van-1", "capacity": [500, 50]}

// After
{"name": "van-1", "capacity": [500, 200]}
```

---

## TAG_HARD

**Type:** HARD
**Score level:** hard
**Triggered when:** A job requires a tag marked `"hard": true` but the assigned resource does not have that tag in its `tags` list.
**Typical developer mistake:** Adding a required tag to a job but forgetting to add the matching tag to any resource — or misspelling the tag name on one side.
**Fix:** Ensure every resource that should serve the job has the exact tag string in `resource.tags`, and that the job's tag entry has `"hard": true`.
**Example fix:**
```json
// Before – resource missing required tag
{
  "job": {"name": "J-1", "tags": [{"name": "plumbing", "hard": true}]},
  "resource": {"name": "R-1", "tags": ["electrical"]}
}

// After
{
  "job": {"name": "J-1", "tags": [{"name": "plumbing", "hard": true}]},
  "resource": {"name": "R-1", "tags": ["electrical", "plumbing"]}
}
```

---

## TYPE_REQUIREMENT

**Type:** HARD
**Score level:** hard
**Triggered when:** A job specifies a type requirement that the assigned resource's shift does not satisfy (shift-level tag gating).
**Typical developer mistake:** Setting `shift.tags` on a resource shift to restrict it to certain job types, but then assigning a job whose type is not in that list.
**Fix:** Either add the job's tag to `shift.tags`, remove the shift-tag restriction, or assign the job to a resource/shift that has the correct tags.
**Example fix:**
```json
// Before – shift only allows "delivery" jobs
{
  "shifts": [{"from": "...", "to": "...", "tags": ["delivery"]}],
  "job": {"name": "J-1", "tags": [{"name": "installation", "hard": true}]}
}

// After – add tag to shift
{
  "shifts": [{"from": "...", "to": "...", "tags": ["delivery", "installation"]}]
}
```

---

## DATE_TIME_WINDOW_CONFLICT

**Type:** HARD
**Score level:** hard
**Triggered when:** A job is scheduled outside all of its declared hard time windows (arrival outside `window.from`/`window.to`), or the job's time window does not overlap with any resource's available shift on any date.
**Typical developer mistake:** Setting tight windows that no resource can reach in time; defining a job window on a date for which no resource has a shift; wrong timezone causing phantom non-overlap; or setting `"hard": true` on a window that was intended as a soft preference.
**Fix:** Widen the window by extending `window.to`, add a resource shift that covers the job's window date/time, change `"hard": false` for soft preference windows, add more resources, or verify that timezone-aware ISO 8601 strings are used consistently.
**Example fix:**
```json
// Before – window closes too early
{
  "windows": [{"from": "2024-03-15T09:00:00Z", "to": "2024-03-15T10:00:00Z", "hard": true}]
}

// After – window extended
{
  "windows": [{"from": "2024-03-15T09:00:00Z", "to": "2024-03-15T12:00:00Z", "hard": true}]
}
```
```json
// Before – resource has no shift on 2024-03-16
{
  "resources": [{"shifts": [{"from": "2024-03-15T08:00:00Z", "to": "2024-03-15T17:00:00Z"}]}],
  "jobs": [{"windows": [{"from": "2024-03-16T09:00:00Z", "to": "2024-03-16T12:00:00Z"}]}]
}

// After – add matching shift
{
  "resources": [{"shifts": [
    {"from": "2024-03-15T08:00:00Z", "to": "2024-03-15T17:00:00Z"},
    {"from": "2024-03-16T08:00:00Z", "to": "2024-03-16T17:00:00Z"}
  ]}]
}
```

---

## SHIFT_END_CONFLICT

**Type:** HARD
**Score level:** hard
**Triggered when:** A resource's trip finishes (all jobs served plus return travel) after the shift's `to` time and no `overtimeEnd` is defined.
**Typical developer mistake:** Underestimating total route duration; not accounting for service durations when setting `shift.to`; or packing too many jobs into a single short shift.
**Fix:** Extend `shift.to`, add an `overtimeEnd` for acceptable overtime, reduce the number of jobs per shift, or add more resource shifts.
**Example fix:**
```json
// Before – shift ends at 17:00 but route finishes at 17:45
{"from": "2024-03-15T08:00:00Z", "to": "2024-03-15T17:00:00Z"}

// After – shift extended or overtimeEnd added
{"from": "2024-03-15T08:00:00Z", "to": "2024-03-15T17:00:00Z",
 "overtimeEnd": "2024-03-15T18:00:00Z"}
```

---

## OVERTIME_END_CONFLICT

**Type:** HARD
**Score level:** hard
**Triggered when:** Work extends past the `overtimeEnd` time on a shift — the resource finishes after even the overtime limit.
**Typical developer mistake:** Setting `overtimeEnd` but still assigning too many jobs; treating `overtimeEnd` as a soft boundary when it is actually a hard stop.
**Fix:** Raise `overtimeEnd` further, remove jobs from that shift, or redistribute jobs to other resources.
**Example fix:**
```json
// Before – route finishes at 19:30 but overtimeEnd is 19:00
{"overtimeEnd": "2024-03-15T19:00:00Z"}

// After
{"overtimeEnd": "2024-03-15T20:00:00Z"}
```

---

## DISALLOWED_RESOURCES

**Type:** HARD
**Score level:** hard
**Triggered when:** A job is assigned to a resource whose name appears in the job's `disallowedResources` list.
**Typical developer mistake:** Blacklisting certain resources on a job but not providing any alternative resources capable of serving the job, leaving the solver with no feasible assignment.
**Fix:** Either remove the resource from `job.disallowedResources`, add other capable resources to the request, or use `partialPlanning: true` to allow the job to go unassigned if truly unserviceable.
**Example fix:**
```json
// Before – all available resources are blacklisted
{
  "job": {"name": "J-1", "disallowedResources": ["vehicle-1", "vehicle-2"]},
  "resources": [{"name": "vehicle-1"}, {"name": "vehicle-2"}]
}

// After – add a third resource not on the blacklist
{
  "job": {"name": "J-1", "disallowedResources": ["vehicle-1", "vehicle-2"]},
  "resources": [{"name": "vehicle-1"}, {"name": "vehicle-2"}, {"name": "vehicle-3"}]
}
```

---

## JOB_PRECEDENCE

**Type:** HARD
**Score level:** hard
**Triggered when:** The day-index difference between two jobs linked by a precedence relation falls outside the `[minimumDaysBetween, maximumDaysBetween]` range — i.e., a job that must precede another is scheduled too late or too early relative to it.
**Typical developer mistake:** Defining a precedence relation without realising the planning horizon is too compressed to honour the minimum gap, or specifying the wrong direction (jobs listed in wrong order).
**Fix:** Ensure the planning horizon is wide enough to fit the gap; check that the leader job appears first in the `jobs` list; adjust `minimumDaysBetween` / `maximumDaysBetween` to match real constraints.
**Example fix:**
```json
// Before – min gap is 2 days but both jobs are on day 0
{
  "type": "SEQUENCE",
  "jobs": ["install-job", "inspection-job"],
  "minTimeInterval": 172800
}

// After – ensure multi-day horizon and correct min interval
// Or reduce minimumDaysBetween if 2 days was wrong
{"minTimeInterval": 86400}
```

---

## LINKED_JOB_CONFLICT

**Type:** HARD
**Score level:** hard
**Triggered when:** Jobs linked via a `SAME_TIME`, `SAME_TRIP`, `SAME_RESOURCE`, or `PICKUP_AND_DELIVERY` relation violate the structural requirement of that relation (e.g., same-time jobs are on different trips, or pickup/delivery order is reversed).
**Typical developer mistake:** Using `SAME_TRIP` when jobs span multiple days; using `PICKUP_AND_DELIVERY` but listing the delivery job before the pickup in the `jobs` array; or combining incompatible constraints.
**Fix:** Correct the `jobs` order in the relation (pickup first, delivery second for `PICKUP_AND_DELIVERY`); use `SAME_RESOURCE` instead of `SAME_TRIP` for multi-day same-resource requirements; verify all jobs in the relation exist.
**Example fix:**
```json
// Before – delivery listed before pickup
{"type": "PICKUP_AND_DELIVERY", "jobs": ["delivery-job", "pickup-job"]}

// After – pickup first
{"type": "PICKUP_AND_DELIVERY", "jobs": ["pickup-job", "delivery-job"]}
```

---

## PLANNED_RESOURCE

**Type:** HARD
**Score level:** hard
**Triggered when:** A job has `plannedResource` set but the solver cannot assign the job to that resource (e.g., the resource lacks required tags, has no available shift, or its capacity is exceeded).
**Typical developer mistake:** Locking a job to a specific resource via `plannedResource` without checking that the resource can actually serve the job within all other constraints.
**Fix:** Remove or correct `job.plannedResource` if the pinning is wrong, OR fix the blocking constraint on the named resource (add tags, extend shifts, increase capacity).
**Example fix:**
```json
// Before – resource-1 has no shift on the job's required date
{"plannedResource": "resource-1"}

// After – either remove the pin or add the shift
{"plannedResource": null}
```

---

## PLANNED_ARRIVAL

**Type:** SOFT
**Score level:** soft
**Triggered when:** The job's actual arrival deviates from `plannedArrival`. The deviation is penalized proportionally.
**Typical developer mistake:** Setting `plannedArrival` to a time that is incompatible with other hard constraints (window, shift, travel time from previous job), causing the job to be unassigned.
**Fix:** If the job is unassigned due to other hard constraints, fix those first. If the arrival accuracy matters, verify that `plannedArrival` falls within `job.windows` and within the resource's shift.
**Example fix:**
```json
// Before – plannedArrival is outside the hard window
{
  "windows": [{"from": "2024-03-15T09:00:00Z", "to": "2024-03-15T11:00:00Z", "hard": true}],
  "plannedArrival": "2024-03-15T14:00:00Z"
}

// After – align plannedArrival with the window
{
  "windows": [{"from": "2024-03-15T09:00:00Z", "to": "2024-03-15T11:00:00Z", "hard": true}],
  "plannedArrival": "2024-03-15T10:00:00Z"
}
```

---

## PLANNED_DATE

**Type:** HARD
**Score level:** hard
**Triggered when:** A job has `plannedDate` set but no resource has a shift on that date.
**Typical developer mistake:** Pinning a job to a specific date (`job.plannedDate`) while omitting a resource shift for that date.
**Fix:** Add a resource shift whose `from`/`to` covers the `plannedDate`, or remove `plannedDate` if the date pin is not required.
**Example fix:**
```json
// Before – no resource shift on 2024-03-18
{"plannedDate": "2024-03-18"}

// After – add a shift for that date
{
  "resource": {
    "shifts": [{"from": "2024-03-18T08:00:00Z", "to": "2024-03-18T17:00:00Z"}]
  }
}
```

---

## HARD_JOBS

**Type:** HARD
**Score level:** hard
**Triggered when:** A job has `"hard": true` (mandatory in partial planning) but `partialPlanning` is enabled AND the job cannot be feasibly assigned — the solver is forced to assign it but cannot satisfy all constraints.
**Typical developer mistake:** Marking too many jobs as `hard: true` in partial planning mode; the hard flag overrides the solver's ability to drop infeasible jobs, potentially leaving the solution infeasible.
**Fix:** Set `"hard": false` (or omit it) for jobs that can be left unserved if necessary. Only mark truly mandatory jobs as `hard: true`. Alternatively, fix the underlying constraint that prevents assignment.
**Example fix:**
```json
// Before – job is hard but no resource can serve it
{"name": "J-1", "hard": true, "tags": [{"name": "rare-skill", "hard": true}]}

// After – allow the solver to drop it if needed
{"name": "J-1", "hard": false, "tags": [{"name": "rare-skill", "hard": true}]}
```

---

## MAX_DRIVE_TIME

**Type:** HARD
**Score level:** hard
**Triggered when:** A resource's total driving time on a trip exceeds `resource.maxDriveTimeInSeconds`.
**Typical developer mistake:** Setting `maxDriveTimeInSeconds` based on legal limits without accounting for the total distance of the assigned route; too many geographically spread jobs for one resource.
**Fix:** Increase `resource.maxDriveTimeInSeconds`, assign geographically distant jobs to separate resources, or add more resources to reduce per-resource driving load.
**Example fix:**
```json
// Before – max drive time set too low for the route
{"name": "van-1", "maxDriveTimeInSeconds": 14400}

// After
{"name": "van-1", "maxDriveTimeInSeconds": 28800}
```

---

## MAX_DRIVE_TIME_JOB

**Type:** HARD
**Score level:** hard
**Triggered when:** The travel time from the previous stop to a specific job exceeds `resource.maxDriveTimeJob` (the per-leg drive limit).
**Typical developer mistake:** Setting `maxDriveTimeJob` as a per-leg limit but placing far-away jobs in the same zone as a resource with a tight per-leg limit.
**Fix:** Remove `maxDriveTimeJob` if per-leg limiting was not intended, increase it to cover the actual maximum leg distance, or ensure jobs are geographically reachable within the limit.
**Example fix:**
```json
// Before – job is 3 hours away but per-job limit is 1 hour
{"maxDriveTimeJob": 3600}

// After
{"maxDriveTimeJob": 10800}
```

---

## Medium Constraints

---

## UNSERVED_JOBS

**Type:** MEDIUM
**Score level:** medium
**Triggered when:** A job cannot be assigned to any resource and ends up unserved (in the `unserved` list of the response). Each unserved job contributes a medium-level penalty.
**Typical developer mistake:** Not enabling `partialPlanning: true` when the problem is over-constrained; not using `priority` to express which jobs are most important to serve.
**Fix:** Enable `"partialPlanning": true` in options to allow the solver to leave some jobs unassigned rather than producing an infeasible solution. Use `job.priority` to rank importance and `options.explanation` to identify why each job is unserved.
**Example fix:**
```json
// Before – all jobs must be served (default), but not all can be
{"options": {}}

// After – allow partial assignment
{"options": {"partialPlanning": true}}

// Also: retrieve unassignment reasons
{"options": {"partialPlanning": true, "explanation": {"enabled": true, "onlyUnassigned": true}}}
```

---

## Soft Constraints

---

## TAG_SOFT

**Type:** SOFT
**Score level:** soft
**Triggered when:** A job has a tag with `"hard": false` and the assigned resource does not have that tag. The solution is still feasible but incurs a soft penalty.
**Typical developer mistake:** Using soft tags for skill preferences but setting the weight so low that the solver routinely ignores them; or accidentally using `"hard": false` on a tag that should be mandatory.
**Fix:** If the tag should be mandatory, change to `"hard": true`. If it should be a strong preference, increase `tag.weight`. If it is truly optional, the current behavior is correct.
**Example fix:**
```json
// Before – soft tag ignored because weight is 1
{"name": "senior-tech", "hard": false, "weight": 1}

// After – stronger preference
{"name": "senior-tech", "hard": false, "weight": 50}
```

---

## RANKING_SOFT

**Type:** SOFT
**Score level:** soft
**Triggered when:** A job is assigned to a resource that is not its most preferred resource per `job.rankings`. Each assignment to a lower-ranked or unranked resource contributes a soft penalty proportional to the rank value.
**Typical developer mistake:** Not specifying rankings at all when customer preference matters; or specifying only one resource in rankings when multiple are acceptable in priority order.
**Fix:** Add a `rankings` list to the job with resource names and rank values (1 = most preferred). Tune `options.weights.rankingWeight` to control how strongly ranking preference is enforced against travel efficiency.
**Example fix:**
```json
// Before – no ranking, any resource assigned
{"name": "J-1"}

// After – express preferred resource
{"name": "J-1", "rankings": [{"name": "tech-alice", "ranking": 1}, {"name": "tech-bob", "ranking": 2}]}
```

---

## TYPE_REQUIREMENT_SOFT

**Type:** SOFT
**Score level:** soft
**Triggered when:** A job's type requirement is not matched by the assigned resource's shift tags, but the constraint is soft rather than hard.
**Typical developer mistake:** Expecting type matching to be enforced strictly when it is configured as soft; the solver may assign a mismatched resource if the soft penalty is outweighed by travel savings.
**Fix:** Change the tag to `"hard": true` for strict enforcement, or increase the tag's weight to make mismatches more costly.
**Example fix:**
```json
// Before – soft type requirement frequently violated
{"name": "certified", "hard": false, "weight": 1}

// After – raise weight or make hard
{"name": "certified", "hard": true}
```

---

## TRAVEL_TIME

**Type:** SOFT
**Score level:** soft
**Triggered when:** The solver includes travel time between jobs in the solution score. This constraint is always active — it penalises total route travel time to encourage efficient routing.
**Typical developer mistake:** Setting `travelTimeWeight` too high relative to other soft constraints, causing the solver to sacrifice customer preferences (time windows, rankings) in pursuit of minimal travel. Or setting it too low, causing unnecessarily long routes.
**Fix:** Tune `options.weights.travelTimeWeight` relative to other weights. Default is `1`. Increase to prioritise shorter routes; decrease if other objectives matter more.
**Example fix:**
```json
// Before – travel time weight same as everything else
{"options": {"weights": {"travelTimeWeight": 1}}}

// After – prioritise short routes over workload balance
{"options": {"weights": {"travelTimeWeight": 10, "workloadSpreadWeight": 1}}}
```

---

## END_LOCATION_TRAVEL_TIME

**Type:** SOFT
**Score level:** soft
**Triggered when:** The last job on a route is far from the shift's end location, causing a large travel-back penalty.
**Typical developer mistake:** Setting a fixed `shift.end` location far from the operational area when the resource does not actually need to return there; this inflates the route cost of peripheral jobs.
**Fix:** Omit `shift.end` if the resource does not need to return to a depot, or set `ignoreTravelTimeFromLastJob: true` on the shift to ignore return travel.
**Example fix:**
```json
// Before – depot return penalty inflating scores for distant jobs
{"from": "...", "to": "...", "end": {"latitude": 51.05, "longitude": 3.71}}

// After – no forced return
{"from": "...", "to": "...", "ignoreTravelTimeFromLastJob": true}
```

---

## RESOURCE_USAGE

**Type:** SOFT
**Score level:** soft
**Triggered when:** A resource is active (has at least one job assigned), contributing a soft penalty per active resource. This encourages the solver to consolidate jobs onto fewer resources.
**Typical developer mistake:** Not realising that having many resources in the request does not mean they will all be used — the solver will try to minimise active resources if `minimizeResourcesWeight` is set.
**Fix:** Increase `options.weights.minimizeResourcesWeight` to push the solver toward using fewer vehicles. Set it to `0` if you want to allow free use of all resources without a consolidation preference.
**Example fix:**
```json
// Before – solver uses many resources because consolidation weight is low
{"options": {"weights": {"minimizeResourcesWeight": 1}}}

// After – strong incentive to use fewer resources
{"options": {"weights": {"minimizeResourcesWeight": 3600}}}
```

---

## RESOURCE_ACTIVATION

**Type:** SOFT
**Score level:** soft
**Triggered when:** A resource/trip is activated (has jobs assigned), triggering its `activationCost` (if set). This is a per-activation fixed cost penalty used in cost-based optimisation.
**Typical developer mistake:** Expecting cost-based routing without setting `activationCost` on resources; or setting it but not tuning `minimizeResourcesWeight` in concert.
**Fix:** Set an `activationCost` on each resource to represent its fixed deployment cost (e.g., driver call-out fee). The solver will weigh this against the savings from using an extra vehicle.
**Example fix:**
```json
// Before – no activation cost, solver uses any resource freely
{"name": "van-1"}

// After – discourage unnecessary vehicle activation
{"name": "van-1", "activationCost": 5000}
```

---

## Diagnostic Quick Reference

| Constraint | Appears in `unservedReasons`? | First thing to check |
|---|---|---|
| `TRIP_CAPACITY` | Yes | `resource.capacity` vs sum of `job.load` |
| `RESOURCE_CAPACITY` | Yes | Total period load vs resource capacity |
| `TAG_HARD` | Yes | Tag name match between `job.tags` and `resource.tags` |
| `DATE_TIME_WINDOW_CONFLICT` | Yes | `window.to` reachable given travel; resource shift exists on window date |
| `SHIFT_END_CONFLICT` | Yes | `shift.to` covers total route duration |
| `OVERTIME_END_CONFLICT` | Yes | `shift.overtimeEnd` covers worst-case finish |
| `DISALLOWED_RESOURCES` | Yes | At least one resource not in blacklist |
| `JOB_PRECEDENCE` | No | Planning horizon fits min/max day gap |
| `LINKED_JOB_CONFLICT` | Yes | Job order in relation is correct |
| `PLANNED_RESOURCE` | Yes | Pinned resource can satisfy all other constraints |
| `PLANNED_DATE` | Yes | Resource shift exists on planned date |
| `HARD_JOBS` | No | Do not over-use `"hard": true` in partial planning |
| `MAX_DRIVE_TIME` | Yes | `maxDriveTimeInSeconds` vs actual route distance |
| `MAX_DRIVE_TIME_JOB` | Yes | `maxDriveTimeJob` vs maximum single-leg distance |
| `UNSERVED_JOBS` | Medium | Enable `partialPlanning: true` |
