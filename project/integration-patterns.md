# Solvice VRP API — Integration Patterns

Eight patterns for the Solvice VRP API. Each shows the minimal request shape with real field names.

---

## Pattern: Basic Delivery

**Use case:** Drop off packages from a depot to customer locations — no time windows, no skill matching.
**Typical scale:** 20-200 jobs, 2-10 resources
**Key constraints:**
- Vehicle capacity (total load must not exceed vehicle limit)
- Shift hours (driver working hours)
- Return to depot at end of shift (optional)

**Distance:** always use `"routingEngine": "OSM"` for real road distances

### Minimal request shape

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

### Common mistakes

- Omitting `load` on jobs when `capacity` is set on the resource → either set both or omit both; mismatched dimensions cause infeasible solutions
- Setting `end` in the shift to a different location than `start` without intending a one-way route → omit `end` if the vehicle should finish wherever the last delivery is

---

## Pattern: Field Service Appointments

**Use case:** Schedule skilled technicians to visit customer sites within agreed time windows.
**Typical scale:** 20-100 jobs per day, 3-15 technicians
**Key constraints:**
- Time windows must be respected (hard windows = no lateness allowed)
- Skill matching: each job requires a specific skill tag the technician must possess
- Technician lunch/break windows during the day
- Snap arrivals to 15-minute intervals for customer communication

**Distance:** always use `"routingEngine": "OSM"` for real road distances

### Minimal request shape

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
    },
    {
      "name": "tech-bob",
      "shifts": [
        {
          "from": "2024-03-15T08:00:00Z",
          "to": "2024-03-15T17:00:00Z",
          "start": { "latitude": 40.7128, "longitude": -74.0060 },
          "end":   { "latitude": 40.7128, "longitude": -74.0060 }
        }
      ],
      "tags": ["hvac", "appliance"]
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
    },
    {
      "name": "appt-002",
      "location": { "latitude": 40.7489, "longitude": -73.9680 },
      "duration": 3600,
      "windows": [
        { "from": "2024-03-15T13:00:00Z", "to": "2024-03-15T16:00:00Z", "hard": true }
      ],
      "tags": [{ "name": "hvac", "hard": true }]
    }
  ],
  "options": {
    "routingEngine": "OSM",
    "snapUnit": 900
  }
}
```

### Common mistakes

- Writing `"tags": ["plumbing"]` as a plain string array on a job → job tags must be objects with `"name"` and `"hard"` fields: `[{ "name": "plumbing", "hard": true }]`
- Omitting `"hard": true` on the job tag → becomes a soft preference, not a requirement; the solver may assign a technician without the skill
- Forgetting `"hard": true` on `windows` → time windows become soft (lateness is penalised but not forbidden)
- Not setting `priority` on urgent appointments → solver may leave high-value jobs unassigned to minimise travel

---

## Pattern: Home Healthcare

**Use case:** Assign caregivers or therapists to patient visits over multiple days, ensuring the same caregiver handles all visits for a patient (continuity of care).
**Typical scale:** 30-150 visits per week, 5-20 caregivers, 2-7 days
**Key constraints:**
- `SAME_RESOURCE` relation ensures one caregiver handles all visits for the same patient
- Skill/certification matching (e.g., physical therapy, wound care)
- Hard time windows per visit agreed with patients
- Multi-day shifts on each resource (one shift object per working day)

**Distance:** always use `"routingEngine": "OSM"` for real road distances

### Minimal request shape

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
    },
    {
      "name": "caregiver-mike",
      "shifts": [
        { "from": "2024-03-18T09:00:00Z", "to": "2024-03-18T18:00:00Z" },
        { "from": "2024-03-19T09:00:00Z", "to": "2024-03-19T18:00:00Z" }
      ],
      "start": { "latitude": 40.7614, "longitude": -73.9776 },
      "tags": ["occupational-therapy"]
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

### Common mistakes

- Putting all days into a single shift object with a very long `from`/`to` span → the solver treats it as one continuous shift; use one shift entry per working day so breaks between days are respected
- Forgetting the `SAME_RESOURCE` relation → the solver assigns visits to different caregivers, violating continuity of care
- Including visits from different patients in the same `SAME_RESOURCE` relation → one relation should cover only the visits of a single patient; create a separate relation per patient
- Using `"start"` only inside shifts when it should also be set at the resource level for multi-day resources → set `"start"` on the resource so all shifts inherit the same home location; `start`/`end` inside a shift override it per day

---

## Pattern: Pickup and Delivery

**Use case:** Collect items from one location and deliver them to a different location in a single trip, keeping each pair together on the same vehicle in the correct order.
**Typical scale:** 10-80 pickup/delivery pairs, 2-10 vehicles
**Key constraints:**
- Each pickup must be visited before its corresponding delivery (`PICKUP_AND_DELIVERY` relation enforces this)
- Vehicle must visit both locations — pickup and delivery cannot be split across vehicles
- Vehicle capacity must accommodate the load from the pickup until it is dropped at the delivery
- Optional time windows on either pickup or delivery stop

**Distance:** always use `"routingEngine": "OSM"` for real road distances

### Minimal request shape

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
    },
    {
      "name": "pickup-B",
      "location": { "latitude": 51.5200, "longitude": -0.1020 },
      "duration": 600,
      "load": [7],
      "windows": [
        { "from": "2024-03-15T09:00:00Z", "to": "2024-03-15T11:00:00Z", "hard": true }
      ]
    },
    {
      "name": "delivery-B",
      "location": { "latitude": 51.4750, "longitude": -0.0950 },
      "duration": 300,
      "load": [-7]
    }
  ],
  "relations": [
    {
      "type": "PICKUP_AND_DELIVERY",
      "jobs": ["pickup-A", "delivery-A"]
    },
    {
      "type": "PICKUP_AND_DELIVERY",
      "jobs": ["pickup-B", "delivery-B"]
    }
  ],
  "options": {
    "routingEngine": "OSM"
  }
}
```

### Common mistakes

- Using positive `load` on both pickup and delivery → delivery `load` must be negative to reflect items being unloaded from the vehicle (reduces current load)
- Listing jobs in wrong order inside the relation → the first job in `"jobs"` is always the pickup; the second is the delivery; the solver enforces this sequence
- Creating one relation with all pickups and deliveries mixed together → each `PICKUP_AND_DELIVERY` relation must contain exactly two jobs: one pickup and its paired delivery
- Omitting `capacity` on the resource when using `load` → without capacity the load constraint is inactive and over-loading is not detected

---

## Pattern: Real-Time Re-Optimization

**Use case:** Routes are already in progress. A new job arrives or conditions change — re-optimize from the current state without starting from scratch.
**Typical scale:** 20-100 jobs, 3-15 resources, mid-day re-plan
**Key fields:**
- `initialResource` and `initialArrival` on each already-assigned job (warm start)
- `plannedWeight` in weights to control how much the solver deviates from the initial plan

### Minimal request shape

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

### Common mistakes

- Omitting `initialArrival` when `initialResource` is set → both are required for a warm start
- Setting `plannedWeight` to 0 → solver ignores the initial plan entirely and may reshuffle everything
- Not setting `initialResource`/`initialArrival` on completed jobs → solver may reassign them

---

## Pattern: Cost Optimization

**Use case:** Minimize total operational cost. Each vehicle has a fixed hourly rate — use fewer expensive vehicles when cheaper ones can handle the load.
**Typical scale:** 20-200 jobs, 5-20 resources with varying costs
**Key fields:**
- `hourlyCost` on resources (per-hour rate; only active time counts)
- `minimizeResourcesWeight` in weights (penalizes activating additional vehicles)

### Minimal request shape

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

### Common mistakes

- Setting `minimizeResourcesWeight` too low → solver uses all available vehicles even when consolidation would be cheaper
- Not setting `hourlyCost` → all resources treated as equal cost, no cost-based preference
- Setting `travelTimeWeight` to 0 → solver ignores route efficiency entirely

---

## Pattern: Fair Workload Distribution

**Use case:** Distribute work evenly across vehicles/technicians — no one gets overloaded while others sit idle.
**Typical scale:** 20-100 jobs, 5-15 resources
**Key fields:**
- `fairWorkloadPerTrip` in options (balance across all resources and days)
- `fairWorkloadPerResource` in options (balance across days for each individual resource)
- `workloadSensitivity` in options (±% variance before penalizing, e.g. 10 = ±10%)
- `workloadSpreadWeight` in weights

### Minimal request shape

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

### Common mistakes

- Setting `workloadSpreadWeight` too high relative to `travelTimeWeight` → routes become inefficient just to balance workload
- Using `fairWorkloadPerTrip` for multi-day setups when `fairWorkloadPerResource` is more appropriate
- Forgetting that workload is measured in service time (duration), not job count

---

## Pattern: Geographic Clustering

**Use case:** Keep routes in distinct geographic zones — avoid vehicles criss-crossing each other's areas.
**Typical scale:** 50-200 jobs, 5-20 resources spread across a metro area
**Key fields:**
- `enableClustering` in options (activates overlap penalty)
- `clusteringThresholdMeters` in options (buffer zone in meters around route bounding boxes)
- `clusteringWeight` in weights (range 0-10000)

### Minimal request shape

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

### Common mistakes

- Setting `clusteringThresholdMeters` too small → clustering has no effect because routes don't overlap
- Setting `clusteringWeight` too high → solver sacrifices route efficiency to avoid any geographic overlap
- Using clustering with very few jobs → not enough density for meaningful geographic zones
