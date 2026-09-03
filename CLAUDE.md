# TransitPulse — Agentic AI Ecosystem for Predictive Bus Capacity & Dynamic Route Allocation

**Track:** SDG 11 – Sustainable Cities & Communities
**Team deliverable:** System scope + agent roles (GitHub repo root spec)

---

## 1. One-Sentence Description
TransitPulse is a multi-agent AI ecosystem that ingests real-time (simulated) bus telemetry to predict overcrowding and delays, and autonomously recommends verified, secure bus-reallocation actions before problems reach passengers.

## 2. Exact Problem Statement
Public bus fleets are dispatched on fixed timetables regardless of real demand. As a result, some buses run overcrowded on high-demand routes while others run nearly empty on low-demand routes at the same time. This causes long passenger wait times, unsafe crowding, wasted fuel/emissions from underused trips, and no mechanism to detect or correct the imbalance in real time.

## 3. Target Users
| User | Need |
| :--- | :--- |
| **Passengers** | Reliable capacity, shorter waits, real-time alerts |
| **Bus drivers** | Clear, validated instructions (no conflicting orders) |
| **Transport operators / fleet managers** | Visibility + AI-assisted, explainable dispatch decisions |
| **City traffic authorities** | Reduced congestion & emissions from smarter allocation |

## 4. Scope
* **In scope:** One urban bus network (multiple routes, shared fleet pool), simulated IoT telemetry, agentic reasoning over that telemetry, and a security layer that protects the decisions the agents make.
* **Out of scope:** Ticketing/payments, physical hardware procurement, city-wide traffic-signal control (traffic level is *consumed* as an input, not controlled).

## 5. System Actors
* **Passengers** — generate implicit demand signals (boarding/alighting) and receive alerts.
* **Buses** — carry sensors; execute dispatch/reallocation instructions.
* **Bus drivers** — human-in-the-loop executors of agent recommendations.
* **Transport operators** — approve high-impact actions; can override any agent.
* **Traffic systems** — external data source (congestion level) consumed by the ecosystem.
* **IoT devices/sensors** — GPS module, passenger-counting sensor, capacity/door sensor, telemetry transmitter (simulated).
* **AI agents** — Monitor, Prediction, Planning, Efficiency, Arbiter, Security.
* **Security systems** — device authentication, anomaly detection, audit logging.

## 6. Agentic AI OS – Roles

### 6.1 Monitor Agent
* **Input:** Live telemetry stream (JSON events, see `spec.md`).
* **Processing:** Compares each reading against thresholds (capacity %, speed near-zero duration, GPS gaps); tags anomalies and "watch" states.
* **Output:** Structured events — `normal`, `overcrowding_watch`, `delay_watch`, `data_anomaly`.
* **Talks to:** Prediction Agent (forwards watch events), Security Agent (forwards `data_anomaly`).

### 6.2 Prediction Agent
* **Input:** Telemetry history (sliding 30-minute window) + current watch event + static route baseline.
* **Processing:** Short-horizon forecast: "will this route exceed 90% capacity or 15-minute delay in the next 10–30 minutes?" Computes a calibrated confidence score.
* **Output:** `prediction_report` with `predicted_overcrowding_prob`, `predicted_delay_min`, `confidence_score` (0.0 to 1.0).
* **Talks to:** Planning Agent (if confidence >= 0.70); logs otherwise.

### 6.3 Planning Agent
* **Input:** `prediction_report` + current fleet state (idle buses, buses on low-demand routes, driver duty limits).
* **Processing:** Identifies possible remediations: dispatch idle relief bus, reallocate a bus from an under-utilized route, or issue passenger advisory.
* **Output:** `proposed_action` with action type, source bus ID, target route ID, estimated relief impact.
* **Talks to:** Efficiency Agent.

### 6.4 Efficiency Agent
* **Input:** `proposed_action` from Planning Agent.
* **Processing:** Computes cost, fuel use, driver shift limits, and evaluates cross-route impact (does moving this bus cause a shortage on its current route?).
* **Output:** `efficiency_assessment` (supports proposal or raises objection).
* **Talks to:** Arbiter Agent.

### 6.5 Arbiter Agent
* **Input:** Proposals from Planning Agent + Objections from Efficiency Agent.
* **Processing:** Applies strict 6-rule priority table (safety/security first, capacity >85% favors passengers, human override for cross-route harm).
* **Output:** `final_decision` (`approve`, `modify`, `reject`, or `escalate_to_human`).
* **Talks to:** Security Agent for final command validation.

### 6.6 Security Agent
* **Input:** Raw telemetry packets, system command queue.
* **Processing:** Validates device signatures, verifies physical telemetry bounds (speed, odometer vs GPS), checks command whitelists.
* **Output:** Device trust score (0.0–1.0) and command clearance/veto.
* **Talks to:** Monitor Agent, Arbiter Agent, and System Dispatcher.
