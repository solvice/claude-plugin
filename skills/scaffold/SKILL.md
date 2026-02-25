---
name: scaffold
description: Build a working VRP request for your routing use case. Asks five targeted questions, generates a minimal OnRouteRequest, solves it live to confirm feasibility, and returns verified JSON ready to paste into your integration.
argument-hint: "[optional: describe your use case]"
---

# Solvice Scaffold

Guide through a five-question interview, generate a realistic `OnRouteRequest` tailored to the use case, solve it with `vrp-solve-sync`, and confirm the result is feasible before presenting it.

If $ARGUMENTS were provided, use them as context for the first question or skip questions whose answers are already clear from the description.

---

## Interview Protocol

Ask each question **one at a time**. Wait for the answer before proceeding. Do not ask multiple questions at once.

---

### Q1 — Job type

> What kind of work are you routing? Choose the option that best fits:
>
> 1. **Deliveries** — drop packages or goods at customer locations
> 2. **Service appointments** — technicians visit customers (plumbing, installation, inspections, etc.)
> 3. **Pickups** — collect items or passengers from locations
> 4. **Pickup + delivery pairs** — pick something up at one location and deliver it to another in the same route

Wait for the answer. Record the job type.

---

### Q2 — Scale

> Roughly how many jobs and how many vehicles (or people) do you expect per day?
> For example: "30 deliveries, 3 vans" or "50 appointments, 10 technicians".

Record the approximate job count (`N_JOBS`) and resource count (`N_RESOURCES`).

---

### Q3 — Time windows (conditional)

> Do your jobs have time windows — specific slots when the customer must be served? (yes / no)

If **yes**, follow up: *What is the typical window width? For example: 2-hour windows, 4-hour windows, all-day.*

If **no**, skip time window fields entirely.

---

### Q4 — Skill-based matching (conditional)

> Does matching require specific skills or equipment — for example, only certain vehicles can handle certain jobs? (yes / no)

If **yes**, follow up: *Give me one or two example tag names (e.g., "refrigerated", "heavy-lift", "certified-electrician").*

If **no**, omit `tags` fields.

---

### Q5 — Capacity constraints (conditional)

> Do vehicles have a capacity limit — weight, volume, number of items, or passengers? (yes / no)

If **yes**, follow up: *What unit and what is a typical vehicle capacity and job load?*

If **no**, omit `capacity` and `load` fields.

---

## Generation Rules

Before generating, read the matching MCP example resource for accurate field names and structure:
- Deliveries → `vrp://examples/basic-routing`
- Service appointments → `vrp://examples/service-appointments`
- Pickups → `vrp://examples/basic-routing`
- Pickup+delivery pairs → `vrp://examples/service-appointments`
- Healthcare → `vrp://examples/home-health-care`

After reading the example, build an `OnRouteRequest` following these rules:

1. **Coordinates**: Use realistic but fictional coordinates for a plausible city. Spread locations naturally across a 5–20 km area.

2. **Distance matrix**: Always set `"routingEngine": "OSM"` inside `options`. This uses real road distances.

3. **Scale**: Generate a representative sample — 2–3 resources, 5–8 jobs.

4. **YAGNI**: Only include fields the developer asked for.

5. **Shifts**: Every resource needs at least one shift. Use a realistic 8–9 hour working day starting at 08:00 tomorrow in ISO-8601. Include `start` and `end` location on the shift.

6. **Duration per job type:**
   - Deliveries: 300–600 s
   - Service appointments: 3600–5400 s
   - Pickups: 180–300 s
   - Pickup+delivery: 300 s each stop

7. **Time windows** (if requested): Set `windows[n].hard: true`. Space windows across the shift so they are reachable.

8. **Tags / skills** (if requested): Add example tags to resources as a flat string array. On each job, add `"tags": [{"name": "<tag>", "hard": true}]`. Assign so at least one resource per job has the required tag.

9. **Capacity** (if requested): Set `capacity` on each resource as `[N]`. Set `load` on each job as `[n]` where total load < total capacity.

10. **Pickup+delivery pairs**: Two jobs per pair. Add `"relations"` array at request level:
    ```json
    {"type": "PICKUP_AND_DELIVERY", "jobs": ["pickup-X", "delivery-X"]}
    ```
    Delivery `load` must be negative (items unloaded).

11. **maxSolveTimeMillis**: Set to `10000` for verification.

---

## Verification

After generating the request, call `vrp-solve-sync` with the generated request.

### If `hardScore == 0` (feasible)

Present:
1. The clean JSON request (no comments, properly formatted)
2. Brief explanation of key choices (coordinates, durations, optional fields)
3. Score interpretation: `hardScore: 0` = feasible; `softScore` = travel penalty (solver minimized it)
4. Next step hint: *"Run `/solvice:debug <id>` if any jobs are unassigned or routes look wrong with real data."*

### If `hardScore < 0` (infeasible) — auto-fix loop

Identify the most likely cause and attempt a fix. Make only the minimal change needed. Re-solve. Repeat up to **3 attempts**.

Check causes in order — **tags first**, then capacity, then time windows, then shift duration:

| Symptom | Fix |
|---|---|
| Many unassigned jobs | Reduce job count to 3–4, or extend shift end by 2 hours |
| Time window violations | Widen windows by 1 hour, or push shift start 30 min earlier |
| Capacity violations | Reduce each job's `load` by 50% |
| Tag violations | Add required tag to at least one more resource |
| Pickup+delivery violations | Extend shift by 2 hours |

After 3 failed attempts: present the last request, explain what was tried, and suggest checking for incompatible constraints.

---

## Output Format

### 1. Verified Request
```json
{ ... }
```
Clean JSON, no comments.

### 2. Key Choices
Short bullets: coordinates, durations, any optional fields, and why.

### 3. Score
- `hardScore: 0` — feasible
- `softScore: <value>` — travel/wait penalty (solver minimized this)

### 4. Next Steps
Copy request into your integration. Replace sample data with real data. Run `/solvice:debug <id>` if jobs are unassigned or routes look unexpected.

---

## Reference

The `integration-patterns` skill contains four complete minimal request shapes (delivery, field service, home healthcare, pickup+delivery) with correct field names and common mistakes. Reference it when the use case matches one of these patterns.
