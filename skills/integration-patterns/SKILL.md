---
name: integration-patterns
description: Canonical Solvice VRP request shapes for common routing use cases. Load automatically when building or reviewing VRP requests for delivery, field service, home healthcare, pickup-and-delivery, real-time re-optimization, cost optimization, fair workload, and geographic clustering. Contains minimal request shapes with correct field names and common mistakes.
user-invocable: false
---

This skill provides reference request shapes. It is loaded automatically by Claude when scaffolding VRP requests or answering questions about request structure — you do not need to invoke it directly.

---

# Solvice VRP API — Integration Patterns

Eight patterns for the Solvice VRP API. Each shows the minimal request shape with real field names.

---

## Pattern: Basic Delivery

**Use case:** Drop off packages from a depot to customer locations — no time windows, no skill matching.
**Typical scale:** 20-200 jobs, 2-10 resources

**Key field:** `"routingEngine": "OSM"` inside `options` — always use this for real road distances.

```json
{
  "resources": [
    {
      "name": "van-1",
      "shifts": [
        {
          "from": "2024-03-15T08:00:00Z",
          "to": "2024-03-15T17:00:00Z",
          "start": { "latitude": 52.5200, "longitude": 13.4050 },
          "end":   { "latitude": 52.5200, "longitude": 13.4050 }
        }
      ],
      "capacity": [100]
    }
  ],
  "jobs": [
    {
      "name": "parcel-001",
      "location": { "latitude": 52.5219, "longitude": 13.4132 },
      "duration": 180,
      "load": [3]
    },
    {
      "name": "parcel-002",
      "location": { "latitude": 52.5096, "longitude": 13.3761 },
      "duration": 180,
      "load": [5]
    }
  ],
  "options": {
    "routingEngine": "OSM"
  }
}
```

**Common mistakes:**
- Omitting `load` on jobs when `capacity` is set — either set both or omit both
- Setting `end` to a different location without intending a one-way route

---

## Pattern: Field Service Appointments

**Use case:** Schedule skilled technicians to visit customers within time windows.
**Typical scale:** 20-100 jobs/day, 3-15 technicians

```json
{
  "resources": [
    {
      "name": "tech-alice",
      "shifts": [
        {
          "from": "2024-03-15T08:00:00Z",
          "to": "2024-03-15T17:00:00Z",
          "start": { "latitude": 40.7128, "longitude": -74.0060 },
          "end":   { "latitude": 40.7128, "longitude": -74.0060 },
          "breaks": [
            {
              "type": "WINDOWED",
              "from": "2024-03-15T12:00:00Z",
              "to":   "2024-03-15T13:00:00Z",
              "duration": 2700
            }
          ]
        }
      ],
      "tags": ["plumbing", "electrical"]
    }
  ],
  "jobs": [
    {
      "name": "appt-001",
      "location": { "latitude": 40.7260, "longitude": -73.9897 },
      "duration": 5400,
      "windows": [
        { "from": "2024-03-15T09:00:00Z", "to": "2024-03-15T11:00:00Z", "hard": true }
      ],
      "tags": [{ "name": "plumbing", "hard": true }],
      "priority": 100
    }
  ],
  "options": {
    "routingEngine": "OSM",
    "snapUnit": 900
  }
}
```

**Common mistakes:**
- Writing job tags as `["plumbing"]` (plain string array) — job tags must be objects: `[{ "name": "plumbing", "hard": true }]`
- Omitting `"hard": true` on job tag → becomes a soft preference, technician without the skill may be assigned
- Forgetting `"hard": true` on `windows` → time windows become soft

---

## Pattern: Home Healthcare

**Use case:** Assign caregivers to patient visits over multiple days with continuity of care.
**Typical scale:** 30-150 visits/week, 5-20 caregivers, 2-7 days

**Key pattern:** `SAME_RESOURCE` relation ensures one caregiver handles all visits for a patient.

```json
{
  "resources": [
    {
      "name": "caregiver-sarah",
      "shifts": [
        { "from": "2024-03-18T08:00:00Z", "to": "2024-03-18T17:00:00Z" },
        { "from": "2024-03-19T08:00:00Z", "to": "2024-03-19T17:00:00Z" },
        { "from": "2024-03-20T08:00:00Z", "to": "2024-03-20T17:00:00Z" }
      ],
      "start": { "latitude": 40.7614, "longitude": -73.9776 },
      "tags": ["physical-therapy", "geriatric"]
    }
  ],
  "jobs": [
    {
      "name": "patient-001-visit1",
      "location": { "latitude": 40.7580, "longitude": -73.9855 },
      "duration": 3600,
      "windows": [
        { "from": "2024-03-18T09:00:00Z", "to": "2024-03-18T11:00:00Z", "hard": true }
      ],
      "tags": [{ "name": "physical-therapy", "hard": true }]
    },
    {
      "name": "patient-001-visit2",
      "location": { "latitude": 40.7580, "longitude": -73.9855 },
      "duration": 3600,
      "windows": [
        { "from": "2024-03-20T09:00:00Z", "to": "2024-03-20T11:00:00Z", "hard": true }
      ],
      "tags": [{ "name": "physical-therapy", "hard": true }]
    }
  ],
  "relations": [
    {
      "type": "SAME_RESOURCE",
      "jobs": ["patient-001-visit1", "patient-001-visit2"]
    }
  ],
  "options": {
    "routingEngine": "OSM"
  }
}
```

**Common mistakes:**
- Putting all days in a single long shift — use one shift entry per working day
- Forgetting `SAME_RESOURCE` relation → visits assigned to different caregivers
- Including visits for different patients in the same relation — one relation per patient
- Using `"start"` only inside shifts when it should also be set at the resource level for multi-day setups — set `"start"` on the resource so all shifts inherit the same home location

---

## Pattern: Pickup and Delivery

**Use case:** Collect items from one location and deliver to a different location in the same trip.
**Typical scale:** 10-80 pairs, 2-10 vehicles

**Key pattern:** `PICKUP_AND_DELIVERY` relation keeps pairs on the same vehicle in order. Delivery `load` must be **negative**.

```json
{
  "resources": [
    {
      "name": "truck-1",
      "shifts": [
        {
          "from": "2024-03-15T07:00:00Z",
          "to": "2024-03-15T18:00:00Z",
          "start": { "latitude": 51.5074, "longitude": -0.1278 },
          "end":   { "latitude": 51.5074, "longitude": -0.1278 }
        }
      ],
      "capacity": [20]
    }
  ],
  "jobs": [
    {
      "name": "pickup-A",
      "location": { "latitude": 51.5155, "longitude": -0.0922 },
      "duration": 600,
      "load": [4]
    },
    {
      "name": "delivery-A",
      "location": { "latitude": 51.4890, "longitude": -0.1445 },
      "duration": 300,
      "load": [-4]
    }
  ],
  "relations": [
    {
      "type": "PICKUP_AND_DELIVERY",
      "jobs": ["pickup-A", "delivery-A"]
    }
  ],
  "options": {
    "routingEngine": "OSM"
  }
}
```

**Common mistakes:**
- Using positive `load` on delivery — delivery `load` must be negative (items unloaded)
- Listing delivery before pickup in the relation — pickup must be first
- One relation with all pickups and deliveries mixed — each relation must have exactly two jobs (one pair)
- Omitting `capacity` when using `load`

---

## Pattern: Real-Time Re-Optimization

**Use case:** Routes are already in progress. A new job arrives or conditions change — re-optimize from the current state without starting from scratch.
**Typical scale:** 20-100 jobs, 3-15 resources, mid-day re-plan

**Key fields:** `initialResource` and `initialArrival` on each job that is already assigned or completed. These tell the solver where things stand so it only changes what needs to change.

```json
{
  "resources": [
    {
      "name": "van-1",
      "shifts": [
        {
          "from": "2024-03-15T08:00:00Z",
          "to": "2024-03-15T17:00:00Z",
          "start": { "latitude": 52.5200, "longitude": 13.4050 },
          "end":   { "latitude": 52.5200, "longitude": 13.4050 }
        }
      ]
    }
  ],
  "jobs": [
    {
      "name": "job-001",
      "location": { "latitude": 52.5219, "longitude": 13.4132 },
      "duration": 1800,
      "initialResource": "van-1",
      "initialArrival": "2024-03-15T09:15:00Z"
    },
    {
      "name": "job-002",
      "location": { "latitude": 52.5096, "longitude": 13.3761 },
      "duration": 1800,
      "initialResource": "van-1",
      "initialArrival": "2024-03-15T10:30:00Z"
    },
    {
      "name": "new-urgent-job",
      "location": { "latitude": 52.5150, "longitude": 13.3900 },
      "duration": 1800,
      "priority": 200
    }
  ],
  "options": {
    "routingEngine": "OSM"
  },
  "weights": {
    "plannedWeight": 100
  }
}
```

**How it works:**
- Jobs with `initialResource` + `initialArrival` → solver starts from this assignment (warm start)
- Jobs without these fields → solver assigns them optimally
- `plannedWeight` penalizes deviations from the initial plan — increase to minimize disruption, decrease to allow more reshuffling

**Common mistakes:**
- Omitting `initialArrival` when `initialResource` is set — both are required for a warm start
- Setting `plannedWeight` to 0 → solver ignores the initial plan entirely and may reshuffle everything
- Not setting `initialResource`/`initialArrival` on completed jobs → solver may reassign them

---

## Pattern: Cost Optimization

**Use case:** Minimize total operational cost instead of just travel time. Each vehicle has a fixed hourly rate — use fewer expensive vehicles when cheaper ones can handle the load.
**Typical scale:** 20-200 jobs, 5-20 resources with varying costs

**Key fields:** `hourlyCost` on resources and `minimizeResourcesWeight` in weights.

```json
{
  "resources": [
    {
      "name": "full-time-driver",
      "shifts": [
        {
          "from": "2024-03-15T08:00:00Z",
          "to": "2024-03-15T17:00:00Z",
          "start": { "latitude": 48.8566, "longitude": 2.3522 },
          "end":   { "latitude": 48.8566, "longitude": 2.3522 }
        }
      ],
      "hourlyCost": 25,
      "capacity": [100]
    },
    {
      "name": "contractor",
      "shifts": [
        {
          "from": "2024-03-15T08:00:00Z",
          "to": "2024-03-15T17:00:00Z",
          "start": { "latitude": 48.8600, "longitude": 2.3400 },
          "end":   { "latitude": 48.8600, "longitude": 2.3400 }
        }
      ],
      "hourlyCost": 45,
      "capacity": [100]
    }
  ],
  "jobs": [
    {
      "name": "delivery-001",
      "location": { "latitude": 48.8700, "longitude": 2.3500 },
      "duration": 600,
      "load": [5]
    }
  ],
  "options": {
    "routingEngine": "OSM"
  },
  "weights": {
    "minimizeResourcesWeight": 3600,
    "travelTimeWeight": 1
  }
}
```

**How it works:**
- `hourlyCost` sets a per-hour rate per resource — the solver multiplies active time by this rate
- `minimizeResourcesWeight` penalizes activating additional vehicles — set high (e.g. 3600) to strongly consolidate onto fewer vehicles
- The solver balances cost vs travel time based on relative weights

**Common mistakes:**
- Setting `minimizeResourcesWeight` too low → solver uses all available vehicles even when consolidation would be cheaper
- Not setting `hourlyCost` → all resources treated as equal cost, no cost-based preference
- Setting `travelTimeWeight` to 0 → solver ignores route efficiency entirely

---

## Pattern: Fair Workload Distribution

**Use case:** Distribute work evenly across vehicles/technicians — no one gets overloaded while others sit idle.
**Typical scale:** 20-100 jobs, 5-15 resources

**Key options:** `fairWorkloadPerTrip` and `workloadSensitivity` in options, plus `workloadSpreadWeight` in weights.

```json
{
  "resources": [
    {
      "name": "tech-1",
      "shifts": [
        {
          "from": "2024-03-15T08:00:00Z",
          "to": "2024-03-15T17:00:00Z",
          "start": { "latitude": 40.7128, "longitude": -74.0060 },
          "end":   { "latitude": 40.7128, "longitude": -74.0060 }
        }
      ]
    },
    {
      "name": "tech-2",
      "shifts": [
        {
          "from": "2024-03-15T08:00:00Z",
          "to": "2024-03-15T17:00:00Z",
          "start": { "latitude": 40.7200, "longitude": -73.9900 },
          "end":   { "latitude": 40.7200, "longitude": -73.9900 }
        }
      ]
    }
  ],
  "jobs": [
    {
      "name": "appt-001",
      "location": { "latitude": 40.7260, "longitude": -73.9897 },
      "duration": 3600
    },
    {
      "name": "appt-002",
      "location": { "latitude": 40.7489, "longitude": -73.9680 },
      "duration": 3600
    },
    {
      "name": "appt-003",
      "location": { "latitude": 40.7350, "longitude": -73.9950 },
      "duration": 3600
    },
    {
      "name": "appt-004",
      "location": { "latitude": 40.7400, "longitude": -73.9800 },
      "duration": 3600
    }
  ],
  "options": {
    "routingEngine": "OSM",
    "fairWorkloadPerTrip": true,
    "workloadSensitivity": 10
  },
  "weights": {
    "workloadSpreadWeight": 100,
    "travelTimeWeight": 1
  }
}
```

**How it works:**
- `fairWorkloadPerTrip: true` enables balancing across all resources on all days
- `fairWorkloadPerResource: true` balances across different days for each individual resource (use for multi-day)
- `workloadSensitivity: 10` means ±10% workload variance is acceptable before penalizing
- `workloadSpreadWeight` controls how strongly the solver prefers even distribution vs shorter routes

**Common mistakes:**
- Setting `workloadSpreadWeight` too high relative to `travelTimeWeight` → routes become inefficient just to balance workload
- Using `fairWorkloadPerTrip` for multi-day setups when `fairWorkloadPerResource` is more appropriate
- Forgetting that workload is measured in service time (duration), not job count

---

## Pattern: Geographic Clustering

**Use case:** Keep routes in distinct geographic zones — avoid vehicles criss-crossing each other's areas.
**Typical scale:** 50-200 jobs, 5-20 resources spread across a metro area

**Key options:** `enableClustering` and `clusteringThresholdMeters` in options.

```json
{
  "resources": [
    {
      "name": "north-driver",
      "shifts": [
        {
          "from": "2024-03-15T08:00:00Z",
          "to": "2024-03-15T17:00:00Z",
          "start": { "latitude": 51.5200, "longitude": -0.1000 },
          "end":   { "latitude": 51.5200, "longitude": -0.1000 }
        }
      ]
    },
    {
      "name": "south-driver",
      "shifts": [
        {
          "from": "2024-03-15T08:00:00Z",
          "to": "2024-03-15T17:00:00Z",
          "start": { "latitude": 51.4800, "longitude": -0.1200 },
          "end":   { "latitude": 51.4800, "longitude": -0.1200 }
        }
      ]
    }
  ],
  "jobs": [
    {
      "name": "north-job-1",
      "location": { "latitude": 51.5300, "longitude": -0.0900 },
      "duration": 1800
    },
    {
      "name": "south-job-1",
      "location": { "latitude": 51.4700, "longitude": -0.1300 },
      "duration": 1800
    }
  ],
  "options": {
    "routingEngine": "OSM",
    "enableClustering": true,
    "clusteringThresholdMeters": 5000
  },
  "weights": {
    "clusteringWeight": 5000,
    "travelTimeWeight": 1
  }
}
```

**How it works:**
- `enableClustering: true` activates the overlap penalty between route bounding boxes
- `clusteringThresholdMeters` sets the buffer zone (in meters) around each route's bounding box — routes that overlap within this buffer are penalized
- `clusteringWeight` controls penalty strength (range 0-10000)
- Optionally add `jobProximityRadius` and `jobProximityWeight` to additionally reward grouping nearby jobs on the same route

**Common mistakes:**
- Setting `clusteringThresholdMeters` too small → clustering has no effect because routes don't overlap
- Setting `clusteringWeight` too high → solver sacrifices route efficiency to avoid any geographic overlap
- Using clustering with very few jobs → not enough density for meaningful geographic zones
