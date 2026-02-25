---
name: quickstart
description: Hands-on quickstart guide for the Solvice VRP API. This skill should be used when the user asks to "get started with Solvice", "learn the VRP API", "run the quickstart", "how does Solvice work", "walk me through the API", "give me a tour", or wants to understand the request/response cycle, scoring, and debugging workflow through live examples.
---

# Solvice VRP Quickstart

Hands-on guide that teaches the VRP API by doing — solve a real problem, understand the response, modify it, break it, and learn to debug it. Each step builds on the previous one and teaches a core concept a developer needs for integration.

Assumes API connection is already configured. If `vrp-demo` fails with an authentication error, tell the user to run `/solvice:setup` first.

---

## Step 1: Solve a Problem

Call `vrp-demo`. This solves a sample routing problem and renders the interactive route map.

After the map loads, explain the request/response cycle:
- **Request**: jobs (locations to visit) + resources (vehicles/people) + constraints (time windows, capacity, skills) → sent to the solver
- **Response**: optimized trips (which resource visits which jobs, in what order) + score + unassigned jobs
- **Map**: each color is a route, numbered stops show visit order, click any marker for arrival time and service details

Then present the exploration menu.

---

## Step 2: Choose What to Learn Next

Present via `AskUserQuestion`:

1. **How does scoring work?** — Understand hard, medium, and soft scores
2. **Can I modify a solution?** — Move jobs between routes and see the impact
3. **What happens when constraints conflict?** — Break a request on purpose and learn to debug it
4. **What types of problems can I solve?** — See the request patterns for different use cases

After the user picks a topic, execute it (see below), then return to the menu with remaining topics plus **I'm ready to build my own**.

When the user picks "I'm ready to build my own", proceed to the wrap-up.

---

## Topic: How Does Scoring Work?

Use the solve ID from the demo.

1. Call `vrp-get-explanation` — the score dashboard renders
2. Walk through the three score components:
   - **Hard score** = constraint satisfaction. `0` means feasible (all hard constraints met). Negative means the solver couldn't find a valid solution — something is physically impossible or rule-breaking.
   - **Medium score** = assignment completeness. `0` means every job is assigned. `-N` means N jobs couldn't fit.
   - **Soft score** = solution quality. The solver minimizes this — it represents travel time, fairness penalties, and preferences. Lower magnitude = better.
3. Point to specific constraints in the dashboard and explain what each one penalizes (e.g., `TRAVEL_TIME` drives route compactness, `SHIFT_END_CONFLICT` would appear if routes ran too long).
4. Key takeaway: "In your integration, check `score.feasible` first. If `true`, the solution is valid. Then look at `mediumScore` for unassigned jobs and `softScore` to compare solution quality across runs."

---

## Topic: Can I Modify a Solution?

Use the solve ID from the demo.

1. Call `vrp-what-if` — the scenario builder renders
2. Explain how it works:
   - The left panel shows all routes with their jobs. Each job card is draggable.
   - Drag a job from one route to another. The right panel shows the score before and after — hard, medium, and soft.
   - This calls `vrp-change` under the hood with `operation: "evaluate"` — it scores the change without committing it.
3. Prompt the user: "Try moving a job — drag any card from one route section to another and watch the scores update."
4. After they interact, explain the programmatic equivalent:
   - `vrp-change` with `operation: "evaluate"` — test a change without committing
   - `vrp-change` with `operation: "solve"` — apply changes and re-optimize around them
   - This is how to implement mid-day replanning: lock completed stops, move or add new jobs, re-solve

---

## Topic: What Happens When Constraints Conflict?

Build a broken request inline to demonstrate debugging. Do not ask the user for input.

### Create the broken request

Build a minimal `OnRouteRequest` with a deliberate time window violation:
- 1 resource with a shift from 08:00 to 17:00
- 3 jobs with normal time windows, but one job has a hard window from 05:00 to 06:00 (before the shift starts — impossible to reach)
- Use `"routingEngine": "OSM"`, realistic coordinates, `maxSolveTimeMillis: 5000`

### Solve and diagnose

1. Call `vrp-solve-sync` — expect `hardScore < 0` or unassigned jobs
2. Call `vrp-get-explanation` — the dashboard shows the violations in red

### Walk through the diagnosis

Explain it as a developer would read it:
- "Hard score is negative — the solution violates at least one constraint. Look at the constraint table."
- Point to the specific violation (e.g., `DATE_TIME_WINDOW_CONFLICT` or `UNSERVED_JOBS`)
- Trace back to the cause: "Job X has `windows[0].to: 06:00` but the resource's shift doesn't start until 08:00. The job's window closes before any vehicle can reach it."
- Show the fix: "Either widen the window (`to: 10:00`) or start the shift earlier (`from: 05:00`)."
- Explain the debugging pattern: "In your integration, when `score.feasible` is `false`, fetch the explanation. Each violated constraint names the exact rule that broke. The `/solvice:debug <id>` command automates this — it fetches the data, maps violations to root causes, and suggests fixes with exact field paths."

---

## Topic: What Types of Problems Can I Solve?

Conversational — explain the request patterns a developer can build:

| Use case | What makes it different | Key fields |
|---|---|---|
| **Basic delivery** | Drop packages, no appointments | `routingEngine: "OSM"`, `load`/`capacity` |
| **Field service** | Technician appointments with time slots and skills | `windows` (hard), `tags` |
| **Home healthcare** | Same caregiver visits same patient across days | `SAME_RESOURCE` relation |
| **Pickup & delivery** | Pick at A, deliver at B, same trip | `PICKUP_AND_DELIVERY` relation, negative `load` on delivery |
| **Real-time replanning** | Mid-day re-optimization around completed stops | `initialResource`, `initialArrival`, `plannedWeight` |
| **Cost optimization** | Minimize fleet size and travel cost | `hourlyCost`, `minimizeResourcesWeight` |
| **Fair workload** | Even distribution across drivers | `fairWorkloadPerTrip`, `workloadSensitivity` |
| **Geographic clustering** | Keep routes in distinct zones | `enableClustering`, `clusteringThresholdMeters` |

After the table, ask: "Which of these is closest to what you're building?" If the user answers, suggest running `/solvice:scaffold` with their use case to generate a working request they can build on.

The `integration-patterns` skill contains complete minimal request shapes with correct field names and common mistakes for all eight patterns.

---

## Wrap-Up

When the user picks "I'm ready to build my own" or has explored all topics:

Summarize what was covered, then point to the logical next steps:
- `/solvice:scaffold` — describe a use case and get a working, verified request to start from
- `/solvice:debug <id>` — paste any solve ID to get automated diagnosis with field-level fixes
- API reference — https://docs.solvice.io

---

## Narration Style

- Teach, don't pitch. Explain *why* things work the way they do.
- Use exact field names and constraint names — the developer needs to map these to their code.
- 2-3 sentences between tool calls. The interactive apps carry the experience.
- When explaining a concept, ground it in what the developer will do in their integration (e.g., "check `score.feasible` first", "fetch the explanation when hard score is negative").
