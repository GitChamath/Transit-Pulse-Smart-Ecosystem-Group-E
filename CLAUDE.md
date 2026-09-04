
# spec.md — Transit-pulse 

Inputs, outputs and constraints for the RideFlow Agentic AI OS.

---

## 1. Inputs

### 1.1 Live telemetry (simulated)

Published every 30 seconds per bus to the MQTT topic `rideflow/138/<device_id>`.

```json
{
  "device_id": "bus-138-07",
  "operator": "private",
  "route": "138",
  "direction": "inbound",
  "ts": "2026-08-01T10:15:30Z",
  "lat": 6.8721,
  "lon": 79.8886,
  "speed_kmh": 18.4,
  "next_halt": "nugegoda",
  "occupancy": 47,
  "capacity": 54,
  "headway_s": 40,
  "sig": "ed25519:9f2c8ab1..."
}
```

### 1.2 Halt telemetry (simulated)

```json
{
  "device_id": "halt-nugegoda",
  "ts": "2026-08-01T10:15:30Z",
  "waiting_est": 22,
  "last_departure_ts": "2026-08-01T10:07:10Z",
  "assist_request": false,
  "sig": "ed25519:41ba07d9..."
}
```

### 1.3 Static reference data

- Route 138 halt list with sequence, coordinates and expected run times.
- Published timetable and planned headway per time band.
- Fleet register: bus id, operator, capacity, depot.
- Driver duty-hour records (aggregate, no personal identifiers).

### 1.4 Virtual sensor set

| Sensor | Metric | Frequency | Purpose |
|---|---|---|---|
| GPS tracker | `lat`, `lon`, `speed_kmh` | 30 s | position and headway |
| Passenger counter | `occupancy` | on door close | load and crowding |
| Halt crowd sensor | `waiting_est` | 30 s | demand ahead of the bus |
| Fuel / emission meter | `fuel_lph` | 60 s | SDG 13 reporting only |

---

## 2. Outputs

### 2.1 Proposal (Planner → Validator)

```json
{
  "proposal_id": "p-20260801-1015-0042",
  "agent": "planner",
  "target": "bus-138-07",
  "command": "HOLD",
  "value_s": 75,
  "at_halt": "nugegoda",
  "reason": "headway 40 s against a 300 s target; bus-138-09 is 40 s behind",
  "expected_effect": "headway restored to ~210 s within 2 halts",
  "confidence": 0.78
}
```

### 2.2 Advisory (Action → driver / commuter / dashboard)

```json
{
  "advisory_id": "a-20260801-1015-0042",
  "proposal_id": "p-20260801-1015-0042",
  "target": "bus-138-07",
  "display": "HOLD 75s",
  "expires_ts": "2026-08-01T10:17:00Z",
  "approved_by": "validator",
  "override_url": "depot://cancel/a-20260801-1015-0042"
}
```

### 2.3 Audit record (every cycle, append-only)

```json
{
  "ts": "2026-08-01T10:15:32Z",
  "proposal_id": "p-20260801-1015-0042",
  "verdict": "approved",
  "rules_checked": ["hold_cap", "halt_starvation", "assist_flag", "duty_hours", "fairness"],
  "validator_note": "within all limits",
  "outcome_score": null
}
```

### 2.4 Commuter output

A corrected ETA per halt, plus a crowding indicator (`green` / `amber` / `red`).
No bus identifier, no driver identifier, no passenger data.

---

## 3. Constraints

### 3.1 Safety constraints

| ID | Constraint |
|---|---|
| S1 | Maximum hold duration: 120 seconds |
| S2 | No halt unserved for more than 20 minutes |
| S3 | Skip-stop forbidden when an assistance request is registered |
| S4 | No advisory to a driver over duty hours |
| S5 | No advisory that would require exceeding the speed limit |

### 3.2 Operational constraints

| ID | Constraint |
|---|---|
| O1 | Holds distributed fairly across operators over a rolling 60 minutes |
| O2 | Maximum 1 short-turn per bus per shift |
| O3 | Advisories expire after 120 seconds if not acknowledged |
| O4 | On sensor loss, the OS degrades to schedule-only mode and issues no holds |

### 3.3 Security constraints

| ID | Constraint |
|---|---|
| C1 | Every telemetry record must carry a valid Ed25519 signature |
| C2 | Records older than 60 seconds are discarded (replay protection) |
| C3 | Device authentication by mTLS client certificate, one topic per device |
| C4 | Agents may issue only the four approved commands |
| C5 | Telemetry fields are treated as data and never parsed as instructions |
| C6 | Audit log is append-only and readable by the regulator |

### 3.4 Privacy constraints

| ID | Constraint |
|---|---|
| P1 | Counts only — no faces, no CCTV, no fare-card identifiers |
| P2 | Halt demand aggregated at halt level, never per person |
| P3 | Location data retained for 7 days, then aggregated and purged |
| P4 | Driver performance reported per depot, not per named individual |

---

## 4. Success metrics (pilot)

| Metric | Baseline | Target |
|---|---|---|
| Standard deviation of headway | measured week 1 | −30% |
| Denial-of-boarding events per day | measured week 1 | −40% |
| Advisory compliance rate | — | ≥ 60% by week 8 |
| Validator veto rate | — | tracked, investigated if > 15% |
| New vehicles required | — | 0 |
| New per-bus hardware cost | — | Rs. 0 on day one |

---

## 5. Out of scope

- Fare collection and ticketing.
- Route planning or timetable redesign.
- Any direct control of a vehicle.
- Inter-provincial long-distance services.
