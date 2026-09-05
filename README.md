# TransitMind

**An AI-Agent-Powered, IoT-Driven, Digital-Twin-Based Public Transportation Operating System for the Colombo Metropolitan Region.**

> IEEE Computer Society R10 Summer School 2026 — Mini Ideathon
> Theme: *Empowering Sustainable Innovation* · Track: **SDG 11 — Sustainable Cities and Communities**

---

## The one-line pitch

Colombo's buses do not fail because nobody is watching. They fail because everybody is watching *too late*. TransitMind is a multi-agent AI operating system that sits on top of the bus network's telemetry, predicts overcrowding, delay, and mechanical failure **15–60 minutes before they happen**, simulates the response inside a digital twin, and executes only the actions a human has pre-authorised it to take.

---

## Why this is not a bus tracking app

A bus tracking app answers *"where is my bus?"*
TransitMind answers *"what is about to go wrong on this corridor, and what is the least-cost intervention?"*

| | Tracking app | TransitMind |
|---|---|---|
| Time orientation | Past / present | **Next 15–60 minutes** |
| Output | A dot on a map | A ranked, simulated, risk-scored **action** |
| Intelligence | One model, one answer | **Five specialised agents that negotiate** |
| Testing a decision | Not possible | **Digital twin runs the counterfactual first** |
| Authority | N/A | **Least-privilege, human-in-the-loop, fully audited** |

---

## Repository map

```
TransitMind/
├── CLAUDE.md ................ System scope + agent roles (primary spec)
├── spec.md .................. Inputs, outputs, constraints, interfaces
├── README.md ................ You are here
│
├── docs/
│   ├── 01-problem-and-sdg.md ......... Problem statement + SDG mapping
│   ├── 02-solution-overview.md ....... What TransitMind is
│   ├── 03-system-architecture.md ..... Seven-layer architecture
│   ├── 04-agentic-ai-os.md ........... The Agentic AI OS blueprint
│   ├── 05-iot-pipeline.md ............ Virtual sensors → agents
│   ├── 06-digital-twin.md ............ Counterfactual simulation engine
│   ├── 07-cybersecurity-raid.md ...... Security architecture + RAID register
│   ├── 08-data-model.md .............. Database design
│   ├── 09-api-spec.md ................ REST + WebSocket + MQTT contracts
│   ├── 10-dashboard-and-app.md ....... Operator console + passenger app
│   ├── 11-simulation-scenarios.md .... Four demo scenarios
│   ├── 12-evaluation-metrics.md ...... How we measure success
│   ├── 13-tech-stack.md .............. Stack with justification
│   ├── 14-roadmap-mvp.md ............. MVP vs future production
│   ├── 15-risks-and-limitations.md ... Honest limitations
│   ├── 16-sri-lanka-context.md ....... Grounding data + sources
│   └── 17-judge-qa.md ................ Anticipated questions
│
├── architecture/diagrams/ ... Mermaid source for every diagram
├── agents/<agent>/AGENT.md .. One contract per agent
├── iot/ ..................... Telemetry schema, MQTT topic tree, samples
├── digital_twin/ ............ Twin state model + simulation contract
├── security/ ................ Permission matrix, command allowlist, audit schema
├── simulation/ .............. Scenario definitions + Colombo route dataset
├── demo/ .................... Timed live-demo runbook
└── pitch/ ................... 8-minute script + slide outline
```

## Reading order for a judge with five minutes

1. `docs/01-problem-and-sdg.md` — why this problem, in Colombo, now
2. `docs/04-agentic-ai-os.md` — the agent blueprint and decision loop
3. `docs/07-cybersecurity-raid.md` — guardrails and the RAID register
4. `docs/11-simulation-scenarios.md` — Scenario 2, the flagship demo

## Reading order for an engineer who wants to build it

`spec.md` → `docs/03-system-architecture.md` → `docs/08-data-model.md` → `docs/09-api-spec.md` → `agents/*/AGENT.md` → `docs/14-roadmap-mvp.md`

---

## Status

This repository is an **ideathon specification and blueprint**, not a production system. Every number labelled *Simulation Result* or *Projected Impact* comes from a modelled scenario, not from field deployment. Real-world figures used to frame the problem are cited with their sources in `docs/16-sri-lanka-context.md`.

## Team

IEEE Computer Society R10 Summer School 2026 — Mini Ideathon team.
Add member names, universities, and role assignments here before submission.

## Licence

MIT — see `LICENSE`.
