# CLAUDE.md — TransitPulse System Specification

> This file is the contract. Every agent module loads its section at boot and a startup
> test fails the build if a module declares behaviour this document does not authorise.
> If the code and this file disagree, the code is wrong.

**Version:** 1.0 · **SDG track:** 11 — Sustainable Cities & Communities
**Classification:** decision-support system. Not autonomy.

---

## 1. Purpose and scope

TransitPulse turns live bus telemetry into safe, explainable operational recommendations
so that existing fleet capacity is allocated where demand actually is, before overcrowding
happens. It does not add vehicles.

**The system DOES:**

- Monitor validated telemetry from a fleet in real time
- Predict occupancy 10–30 minutes ahead with an explicit confidence score
- Propose operational responses from a fixed, whitelisted action set
- Validate every proposal against safety, efficiency and security constraints
- Log every decision — including every rejection — to an append-only audit store

**The system DOES NOT:**

- Drive buses or issue any command to vehicle control systems
- Control traffic signals or any external infrastructure
- Collect fares, fare identity, or any passenger-identifying data
- Execute any action without a human operator confirming it

### 1.1 Non-negotiable invariants

These hold in every code path. A change that breaks one of these is a spec change, not a
bug fix, and requires this file to be updated first.

| # | Invariant |
|---|-----------|
| I1 | Only the `arbiter` agent may emit an action. No other module has an emit path. |
| I2 | Every emitted action is a member of the command whitelist in §7.1. |
| I3 | No decision is made on telemetry with `trust_score < 0.40`. |
| I4 | The audit record is written **before** the action is dispatched, never after. |
| I5 | No action executes without an explicit human confirmation event. |
| I6 | No passenger-identifying data enters the pipeline. Occupancy is a count. |
| I7 | Every rejection is audited with the same rigour as an approval. |

---

## 2. SDG 11 alignment

| Target | Indicator | TransitPulse lever |
|---|---|---|
| 11.2 | 11.2.1 — access to public transport | Reallocating capacity raises effective service level on under-served runs without new vehicles |
| 11.6 | 11.6.2 — environmental impact | Fewer empty-seat kilometres; higher load factor per kilometre already driven |
| 11.3 / 11.a | planning capacity | The decision audit log becomes a demand dataset for route and timetable revision |

### 2.1 Reported KPIs

Measured inside the simulation harness. These are decision-quality metrics, not verified
city-scale outcomes.

| Metric | Definition | Baseline | Target |
|---|---|---|---|
| Overcrowding minutes / route-hour | minutes above 90% of capacity | 18 | ≤ 7 |
| Load-factor gap | max − min occupancy across paired routes | 34 pts | ≤ 15 pts |
| Warning lead time | prediction timestamp vs. observed breach | 0 (reactive) | 10–30 min |
| Empty-seat kilometres / shift | seats × km below 30% occupancy | — | −12% |

---

## 3. Repository structure

```
CLAUDE.md              this file — agent contracts and guardrails
README.md              setup, run instructions, architecture overview
ARCHITECTURE.md        layer diagrams and data-flow notes
agents/
  monitor.py           telemetry watch, anomaly flagging
  prediction.py        occupancy forecast with confidence
  planning.py          option generation over the action set
  efficiency.py        hard-constraint checking
  security.py          trust, RBAC and whitelist policy
  arbiter.py           arbitration and the single emit path
  contracts.py         loads §6 of this file; enforced at boot
iot-simulator/
  fleet.py             bus kinematics and sensor emitters
  demand.py            per-stop arrival and boarding model
  edge_gateway.py      timestamp, sequence, HMAC signature, buffer
  scenarios.py         fault injection catalogue (§8.2)
ingest/
  schema.py            JSON Schema validation
  sanity.py            physical plausibility and cross-sensor checks
  trust.py             trust scoring
security/
  rbac.py              roles and permission matrix
  whitelist.py         the five permitted actions
  audit.py             append-only, hash-chained writer
dashboard/             live telemetry, reasoning trace, demo controls
tests/
  test_schema.py       telemetry contract tests
  test_guardrails.py   one test per invariant in §1.1
  test_contracts.py    startup test — module actions ⊆ whitelist
```

### 3.1 Stack

Python 3.11 · FastAPI (orchestration) · MQTT / Mosquitto (transport) ·
SQLite in development, PostgreSQL in pilot · React (dashboard) ·
pytest + JSON Schema (validation).

---

## 4. Layers

Data flows upward. Security is cross-cutting and applies at every layer.

| Layer | Responsibility |
|---|---|
| IoT simulation | Virtual fleet, demand model, edge gateway, MQTT transport |
| Ingest & validation | Schema check → physical sanity → trust score → release to agent bus |
| Agentic AI OS | Six agents, closed reasoning loop, arbitration |
| Orchestration API | Routes telemetry and agent calls, sequences the cycle, dispatches |
| Presentation | Dashboard, operator console, confirm/reject, demo controls |
| Data & audit store | Telemetry log, decision records, hash-chained audit, outcome scores |

---

## 5. Telemetry contract

### 5.1 Event schema (`schema_version: "1.0"`)

```json
{
  "bus_id": "BUS_101",
  "route_id": "R138",
  "trip_id": "TRIP_5521",
  "timestamp": "2026-09-04T15:30:10Z",
  "seq": 1045,
  "location": { "latitude": 6.9271, "longitude": 79.8612 },
  "speed_kmh": 42,
  "passenger": { "count": 46, "capacity": 50, "boarded": 4, "alighted": 1 },
  "fuel_pct": 72,
  "sensors": { "gps": "healthy", "door": "healthy",
               "imu": "healthy", "odometer": "healthy" },
  "device_signature": "SIG_A8F4C92",
  "schema_version": "1.0"
}
```

### 5.2 Field purposes

| Field | Why it exists |
|---|---|
| `seq` | Detects missing, duplicate or replayed messages |
| `timestamp` | Confirms freshness; data older than 60 s is stale |
| `device_signature` | HMAC proving the event came from a trusted device |
| `sensors{}` | Per-sensor health, so failing sensors are excluded early |
| `schema_version` | Allows the ingest layer to reject unknown contracts |

### 5.3 Sensors

| Sensor | Detects | Used for |
|---|---|---|
| GPS | latitude, longitude, time | tracking, delay detection, spoofing checks |
| Door | open / closed state | boarding context, dwell-time anomalies |
| IMU | harsh braking, abnormal motion | incident detection, driver-safety signals |
| Odometer | wheel rotation, distance, speed | cross-check against GPS speed |

### 5.4 Validation pipeline

1. **Schema** — reject anything failing JSON Schema or an unknown `schema_version`
2. **Signature** — verify HMAC against the per-device key; failure drops trust to 0
3. **Sequence** — reject duplicates; flag gaps; reject replays outside the timestamp window
4. **Freshness** — `timestamp` older than 60 s marks the event stale
5. **Physical sanity** — speed, acceleration, position delta and occupancy within bounds
6. **Cross-sensor** — GPS speed vs. odometer speed; discrepancy above 15% flags the feed
7. **Trust score** — computed per §7.2; below 0.40 the event never reaches the agents

> Cross-check example: GPS reporting 180 km/h while the odometer reports 38 km/h is
> flagged as suspicious before it ever reaches an agent.

---

## 6. Agent contracts

Each agent declares `role`, `reads`, `emits`, `may_not`, `guardrail` and `escalates`.
`may_not` clauses are enforced, not advisory.

### 6.1 `monitor`

```
role:        Watch validated telemetry across the fleet in real time
reads:       telemetry.validated (trust >= 0.40)
emits:       {bus_id, route_id, occupancy, anomalies[], observed_at}
method:      threshold rules + rolling-window anomaly detection
may_not:     issue commands · forecast · propose actions
guardrail:   sensor health != healthy -> exclude that channel, raise flag
             fleet-wide feed loss     -> emit DEGRADED, halt the cycle
escalates:   prediction (always) | human (fleet-wide feed loss)
```

### 6.2 `prediction`

```
role:        Forecast occupancy 10-30 min ahead
reads:       telemetry.validated (trust >= 0.40)
             history.occupancy[route, dow, tod]
emits:       {route_id, window_min, p_overcrowd, confidence, evidence[]}
method:      occupancy trend blended with historical profile
may_not:     issue commands · write telemetry · call external services
guardrail:   confidence < 0.55     -> emit ADVISORY only, never a proposal
             data older than 60s   -> abstain and raise a staleness flag
             horizon > 30 min      -> refuse; accuracy is not established
escalates:   arbiter (always) | human (p_overcrowd > 0.90 and fleet-wide)
```

### 6.3 `planning`

```
role:        Propose an operational response to a forecast condition
reads:       prediction.emits · monitor.emits · fleet.state
emits:       {proposed_action, target_route, source_route, expected_relief,
              rationale, evidence[]}
method:      option generation over the whitelist, ranked by expected relief
may_not:     propose anything outside the command whitelist (§7.1)
             execute · dispatch · contact any external system
guardrail:   no option above relief threshold -> propose hold_and_notify
             more than one viable option      -> rank, do not choose
escalates:   efficiency and security (parallel), then arbiter
```

### 6.4 `efficiency`

```
role:        Check operational feasibility before any action is authorised
reads:       planning.emits · fleet.state · driver.hours · fuel.levels
emits:       {verdict: PASS | SOFT_VETO | HARD_VETO, reason, constraint}
method:      hard-constraint evaluation, no optimisation, no learning
may_not:     approve an action · modify a proposal · override security
guardrail:   driver hours at or over statutory limit -> HARD_VETO
             fuel reserve below policy minimum       -> HARD_VETO
             coverage loss on the source route       -> SOFT_VETO + escalate
escalates:   arbiter (always) | human (any coverage-loss SOFT_VETO)
```

### 6.5 `security`

```
role:        Enforce data trust, identity and permission on every decision
reads:       telemetry.trust · agent.identity · rbac.policy · whitelist
emits:       {verdict: PASS | VETO, trust_score, role, reason}
method:      policy evaluation; deterministic, no model inference
may_not:     propose actions · alter proposals · be overridden by any agent
guardrail:   trust_score < 0.40        -> VETO, absolute
             agent identity unverified -> VETO, absolute
             action not in whitelist   -> VETO, absolute
escalates:   arbiter (always) | human (any VETO caused by suspected attack)
```

### 6.6 `arbiter`

```
role:        Weigh every input and issue the single authorised action
reads:       all agent emits for the current cycle
emits:       {decision_id, action, rationale, confidence, evidence_chain[],
              vetoing_agent | null}
method:      weighted score with hard vetoes; deterministic tie-breaking
may_not:     originate a proposal · bypass a HARD_VETO or security VETO
             emit more than one action per cycle
guardrail:   any security VETO      -> REJECT, no exceptions
             any efficiency HARD_VETO -> REJECT, fall back to §7.3
             tie or low confidence  -> least-intervention action
guarantee:   writes the audit record BEFORE dispatch (invariant I4)
escalates:   human (always — every action requires confirmation)
```

---

## 7. Decision policy

### 7.1 Command whitelist

The complete set of permitted actions. Anything else is dropped at validation, never at
execution.

| Action | Meaning |
|---|---|
| `dispatch_relief` | Send a relief vehicle to a route with predicted overcrowding |
| `hold_and_notify` | Take no vehicle action; notify the dispatcher with the forecast |
| `adjust_headway` | Recommend a spacing change between vehicles on one route |
| `reroute_short_turn` | Recommend a short-turn to restore frequency on a segment |
| `escalate_to_human` | Hand the decision to an operator with the full evidence chain |

### 7.2 Trust scoring

Score in `[0.00, 1.00]`, computed per feed per cycle.

| Signal | Effect |
|---|---|
| Valid HMAC signature | required; failure sets the score to 0.00 |
| Sequence continuity | gaps and duplicates reduce the score |
| Timestamp freshness | staleness beyond 60 s reduces the score sharply |
| Cross-sensor agreement | GPS vs. odometer divergence reduces the score |
| Sensor health flags | any non-healthy channel reduces the score |
| Recent anomaly history | sustained anomalies decay the score across cycles |

**Threshold:** `< 0.40` excludes the feed from all reasoning (invariant I3).

### 7.3 Arbitration rules, in precedence order

1. **Security veto is absolute.** Nothing proceeds on data below trust 0.40, or from an
   agent that fails authentication.
2. **Efficiency vetoes hard on safety limits** — driver hours, fuel reserve — and soft on
   cost. A soft veto lowers the score; it does not block.
3. **Planning may only propose from the whitelist.** Anything else is dropped at
   validation, never at execution.
4. **Ties and low-confidence cases resolve to the least-intervention action.**
5. **Any action that would strand coverage on another route escalates to a human.**
   The system recommends; the operator confirms.
6. **Every path writes an audit record** — including, and especially, the rejections.

### 7.4 Worked rejection case

```
prediction  p_overcrowd 0.81 · 20 min window · Route A
planning    proposes dispatch_relief, sourced from Route B
efficiency  relief driver at 9h42m -> statutory limit -> HARD_VETO
security    data trust 0.93 · RBAC check passed
arbiter     dispatch_relief REJECTED -> hold_and_notify + escalate_to_human
audit       decision 4471 stored with rationale and the vetoing agent
```

### 7.5 The control loop

```
OBSERVE  -> ORIENT -> DECIDE -> ACT -> VERIFY
   ^                                      |
   +--- outcome score feeds prediction ---+
        priors and agent trust weights
```

| Stage | What happens |
|---|---|
| Observe | Monitor ingests validated telemetry on the 10 s tick |
| Orient | Prediction forecasts; Efficiency loads the constraint picture |
| Decide | Planning proposes, Security validates, Arbiter authorises |
| Act | Command reaches the operator console; audit written first |
| Verify | Monitor re-checks the predicted window and scores the outcome |

**Loop budget:** one full cycle completes in ≈180 ms, well inside the 10 s telemetry tick.
A cycle that exceeds 2 s is abandoned and logged rather than allowed to act on stale state.

---

## 8. IoT simulation

### 8.1 Generation model

The virtual fleet is a model, not a random number generator.

| Stage | Behaviour |
|---|---|
| Demand model | Per-stop arrival rates by route, day and hour; Poisson boarding with event and weather multipliers |
| Bus kinematics | Timetable, dwell time and a congestion factor produce position, speed and odometer distance |
| Sensor emitters | GPS, door, IMU, odometer sampled every 10 s, each with its own health state |
| Edge gateway | Stamps time, adds sequence, signs with HMAC, buffers locally on link loss |
| MQTT broker | `fleet/{route_id}/{bus_id}/telemetry`, QoS 1, retained last-known state |
| Ingest API | Schema validation → trust scoring → release onto the agent bus |

**Fleet:** 5 buses × 2 routes · 10 s cadence · ≈30 events/min · ≈12,450 events per demo run.

**Determinism:** every scenario runs from a fixed random seed. The same scenario replayed
produces the same telemetry, the same reasoning and the same decision.

### 8.2 Fault injection catalogue

| Injected fault | Expected system response |
|---|---|
| Sensor stuck, drifting or dead | health flag set, channel excluded from Monitor |
| GPS jump to an impossible position | odometer cross-check fails, trust drops below 0.40 |
| Network loss (45 s) | gateway buffers, replays in order, sequence gaps flagged |
| Duplicate or replayed frames | rejected by sequence and timestamp window |
| Clock skew on a device | data marked stale, Prediction abstains |
| Passenger surge | full loop runs; a recommendation is produced |
| Ingest flood | per-device rate limit engages, degraded mode logged |

Each entry above has a corresponding test in `tests/test_guardrails.py`.

---

## 9. Security

### 9.1 Threat model

| Attack surface | Threat class | Abuse case | Control | Detection signal |
|---|---|---|---|---|
| Sensor / device | Spoofing | Forged bus reports fake position or occupancy | HMAC device signature, per-device key | Signature mismatch, trust drop |
| Telemetry channel | Tampering, replay | Frames altered or re-sent to fabricate a surge | TLS, sequence numbers, timestamp window | Seq gap or duplicate, stale stamp |
| Agent-to-agent bus | Elevation of privilege | Compromised agent issues a command outside its role | Per-agent identity, command whitelist | Non-whitelisted call, RBAC denial |
| Operator console | Spoofing, elevation | Stolen credentials used to dispatch buses | MFA, RBAC, short-lived API tokens | Unusual role and action pairing |
| Decision log | Repudiation | Someone edits or deletes an inconvenient decision | Append-only store, hash-chained records | Hash chain verification failure |
| Ingest availability | Denial of service | Flooding the ingest API to blind the fleet | Per-device rate limits, gateway buffering | Ingest rate anomaly, degraded mode |

**Defence in depth:** spoofed GPS must defeat the device signature, then the odometer
cross-check, then the trust score, before it can influence one decision.

### 9.2 RBAC matrix

| Role | Read telemetry | View decisions | Confirm action | Configure agents | Manage keys |
|---|---|---|---|---|---|
| Admin | ✓ | ✓ | ✓ | ✓ | ✓ |
| Operator | ✓ | ✓ | ✓ | — | — |
| AI Agent | ✓ (scoped) | ✓ (own cycle) | — | — | — |
| Technician | ✓ (device health) | — | — | — | ✓ (device keys) |
| Viewer | ✓ (aggregate) | ✓ | — | — | — |

Least privilege applies per role **and** per agent. Authentication is MFA for humans and
short-lived API tokens for services; authorisation is checked on every action, never
cached across a cycle.

### 9.3 Audit record

```json
{
  "decision_id": 4471,
  "cycle_id": "C_20260904T153010Z",
  "timestamp": "2026-09-04T15:30:11Z",
  "outcome": "REJECTED",
  "action": "hold_and_notify",
  "proposed_action": "dispatch_relief",
  "vetoing_agent": "efficiency",
  "reason": "driver_hours_statutory_limit",
  "confidence": 0.81,
  "evidence_chain": ["monitor:obs_88213", "prediction:fc_5521",
                     "planning:opt_3", "efficiency:veto_hard",
                     "security:pass"],
  "trust_score": 0.93,
  "actor": "arbiter",
  "confirmed_by": null,
  "prev_hash": "9f2c…",
  "hash": "a41e…"
}
```

Records are append-only and hash-chained. `prev_hash` links each record to its predecessor;
a broken chain is a detected integrity failure, not a recoverable state.

### 9.4 Privacy

Occupancy is a count, never a person. No camera feeds, no fare identity, no passenger
records anywhere in the pipeline. Driver-hours data is operational, access-restricted to
Admin and Operator roles, and never leaves the deployment.

---

## 10. RAID register

**Risks**

| Risk | Mitigation | Owner |
|---|---|---|
| Falsified sensor data | Trust scoring plus device signature | `security` |
| Prediction wrong at peak | Advisory-only below 0.55 confidence | `prediction` |
| Operator over-trusts the AI | Confidence and rationale shown on every recommendation | dashboard |
| Demand patterns drift | Outcome scoring in the verify stage | `prediction` |

**Assumptions**

- Buses can expose GPS, door, IMU and odometer data
- The operator has at least one relief vehicle available to reallocate
- A human dispatcher confirms every dispatch decision
- Occupancy can be estimated within roughly ±10%

**Issues (live, being managed)**

- Occupancy is simulated; real APC sensors drift and need calibration
- No real ridership history yet — profiles are synthetic
- Accuracy falls off past 30 minutes, so the horizon is capped by design
- Single operator only; no multi-operator federation yet

**Dependencies**

- Operator willingness to share a live AVL telemetry feed
- GTFS / GTFS-RT route and timetable data availability
- Network coverage on the corridor (mitigated by gateway buffering)
- Privacy sign-off on vehicle and driver-hours data

---

## 11. Deployment path

| Phase | Timeline | Scope |
|---|---|---|
| 1 | Now | Simulated fleet, full agent OS, security layer, live demo |
| 2 | 8 weeks | Shadow mode on one operator's existing AVL feed; system recommends, nobody acts |
| 3 | 3 months | Assisted dispatch on two corridors; human confirms every action |
| 4 | 6–12 months | Multi-route rollout plus planner analytics from the audit log |

**What an operator must provide:** an AVL/GPS feed, door or APC counts (or a door-event
proxy), GTFS timetable data, and one dispatcher's attention. No new hardware, timetable or
workflow change is required for Phase 2.

**What is not claimed:** this is not a validated ridership forecast, not autonomous, and
not a hardware product.

---

## 12. Engineering conventions

- **Contract enforcement.** `agents/contracts.py` parses §6 at boot. A module declaring an
  action outside §7.1, or reading a source its contract does not list, fails the startup
  test and the build.
- **Testing.** One test per invariant in §1.1. One test per fault in §8.2. Schema tests
  run against `schema_version` fixtures. Guardrail tests must include the rejection paths,
  not only the approval paths.
- **CI.** Every push runs schema validation and the guardrail suite. A red guardrail test
  blocks merge.
- **Commits.** Conventional commits. A change to agent behaviour must update this file in
  the same commit as the code.
- **Determinism.** Simulation seeds are committed with scenario definitions so any result
  can be reproduced from the repository alone.
- **Logging.** Every agent logs its inputs, its verdict and its evidence, at every cycle,
  whether or not an action results.

---

## 13. Changing this document

1. Open a PR that edits `CLAUDE.md` first, describing the behaviour change in the contract.
2. Update the affected agent module and its tests in the same PR.
3. If the change touches §1.1, §7.1 or §9, it requires review from a second contributor.
4. Startup tests must pass against the amended contract before merge.

A change that weakens an invariant in §1.1 should be assumed wrong until argued otherwise
in the PR description.
