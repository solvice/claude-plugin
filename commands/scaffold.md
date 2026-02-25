---
name: scaffold
description: Build a working VRP request through a guided interview
---

Build a VRP request by asking the user five questions, one at a time:

1. **Job type** — What kind of work? (deliveries, service appointments, pickups, home visits)
2. **Scale** — How many jobs and resources?
3. **Time windows** — Do jobs have appointment slots or flexible timing?
4. **Skills/tags** — Do specific jobs require specific resources? (e.g., certifications, vehicle types)
5. **Capacity** — Weight, volume, or item count limits?

After gathering answers, build a valid `OnRouteRequest` JSON with realistic coordinates, then:

1. Solve it with `vrp-solve-sync`
2. If hard score < 0 or unassigned jobs > 0, diagnose and fix (up to 3 attempts)
3. Return the verified request JSON the user can paste into their integration
