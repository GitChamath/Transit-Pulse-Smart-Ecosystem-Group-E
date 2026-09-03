# TransitPulse — Agentic AI for Urban Transit Fleet Reallocation

> IEEE Computer Society R10 Summer School 2026 — Mini Ideathon Challenge
> **Track:** SDG 11: Sustainable Cities & Communities | **Group E**

## Overview
Buses typically run on fixed timetables while commuter demand fluctuates unpredictably. During peak hours, high-demand routes experience severe overcrowding (>90% capacity) while buses on quieter routes run nearly empty at the exact same moment.

**TransitPulse** is a multi-agent AI ecosystem designed to solve this imbalance in real time. By ingesting simulated IoT bus telemetry, decoupled AI agents deliberate trade-offs between passenger relief and operating costs, governed by an in-line Zero-Trust cybersecurity verification layer.

---

## Core Differentiators
1. **Agentic Debate Architecture:** Rather than following rigid thresholds, separate AI agents advocate for opposing priorities. One agent maximizes passenger throughput, another protects municipal budgets and emissions, and a deterministic Arbiter resolves disputes using codified rules.
2. **Zero-Trust In-Line Telemetry Validation:** Incoming sensor data is treated as untrusted by default. The Security Agent validates physics and signatures before telemetry is fed into reasoning models.

---

## The 6 AI Agents
* **Security Agent:** Gatekeeper that authenticates device signatures, runs physical feasibility checks (speed, route boundaries, door counts), and enforces command whitelists.
* **Monitor Agent:** Ingests live telemetry every 10 seconds, tracks route performance metrics, and triggers anomaly/overcrowding watch events.
* **Prediction Agent:** Evaluates rolling trends against historical baselines to forecast crowding 10–30 minutes ahead with a calibrated confidence score.
* **Planning Agent:** Proposes operational mitigations, such as dispatching idle relief buses or merging under-utilized trips.
* **Efficiency Agent:** Analyzes fuel expenditure, driver duty hours, and cross-route service disruption to counter unviable plans.
* **Arbiter Agent:** Applies a strict 6-rule policy hierarchy to approve, modify, reject, or escalate dispatch interventions.

---

## System Workflow
```text
Simulated IoT Bus Sensors (Every 10s)
               │
               ▼
       [ Security Agent ]   ──► Verifies physical plausibility & assigns trust score (0.0 - 1.0)
               │
               ▼
       [ Monitor Agent ]    ──► Detects threshold breaches & issues watch alerts
               │
               ▼
      [ Prediction Agent ]  ──► Forecasts conditions 10–30 min ahead + confidence score
               │
               ▼
      [ Planning Agent ]    ◄──► [ Efficiency Agent ] (Debate: Passenger Relief vs. Fuel/Cost)
               │
               ▼
       [ Arbiter Agent ]    ──► Resolves debate using strict 6-rule policy table
               │
               ▼
       [ Security Agent ]   ──► Validates outgoing dispatch instructions against whitelist
               │
               ▼
     Actuation / Alerts     ──► Driver instruction sent & passenger notifications updated
               │
               ▼
        Feedback Loop       ──► Compares actual vs. forecast outcomes to calibrate weights
