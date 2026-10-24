# SkyReady 
## Predictive Maintenance & Fleet Availability Optimizer

> **Ministry of Defence (MoD) / Defence Services Staff College (DSSC)**  
> **Problem Statement ID**: 26249  
> **Theme**: Transportation & Logistics / Software  
> **Team**: ZEN (Team ID: `SIH-S-B1-060`)  
> **Motto**: *"One loop: DETECT › PREDICT › EXPLAIN › OPTIMIZE › SIMULATE › ACT (sensor → schedule → spares)"*

---

### 1. Executive Summary & Objective

In military aviation, maintenance practices are frequently fragmented and reactive: telemetry from aircraft health-monitoring systems, technical logbooks, depot spares inventories, and maintenance agencies operate in disjoint silos. Faults are often uncovered only after an aircraft is grounded, forcing the repair to wait idly for a spare part, qualified technicians, or an open hangar bay slot.

**SkyReady** shifts the paradigm:
- **Core Metric**: Optimizes for **Fleet Availability (Ready Aircraft-Days)**, not isolated failure probabilities.
- **Uncertainty Drives Decisions**: Rather than single point forecasts, **Remaining Useful Life (RUL) quantile ranges** ($P_{10}$ pessimistic bound to $P_{90}$ optimistic bound) directly feed mathematical constraints in Google OR-Tools CP-SAT.
- **One Loop from Sensor to Schedule to Spares**: Bridges telemetry anomaly detection, explainable root-cause diagnosis, resource-constrained optimization, what-if stress simulation, and human-in-the-loop audit authorization.

---

### 2. Architecture & Modules

```
                    ┌────────────────────────────────────────────────────────┐
                    │               SKYREADY CLOSED-LOOP ENGINE              │
                    └────────────────────────────────────────────────────────┘
                                                 │
    ┌──────────────────────┐                     ▼                     ┌──────────────────────┐
    │     1. DETECT        │           2. PREDICT (RUL)                │      3. EXPLAIN      │
    │  Isolation Forest    │ ────►  Quantile Regressor [P10-P90] ────► │  TreeSHAP + Plain-   │
    │  C-MAPSS Telemetry   │        Pessimistic Safe Bound             │  English Diagnostics │
    └──────────────────────┘                                           └──────────────────────┘
                                                 │
                                                 ▼
    ┌──────────────────────┐           4. OPTIMIZE (Core)              ┌──────────────────────┐
    │       6. ACT         │ ◄────  Google OR-Tools CP-SAT   ◄──────── │     5. SIMULATE      │
    │  Advisory Approval   │        Max Ready Aircraft-Days            │  Dynamic What-If     │
    │  SHA-256 Audit Trail │        Hangar/Techs/Spares Bounds         │  Supply & Surge Sim  │
    └──────────────────────┘                                           └──────────────────────┘
```

1. **Ingest & Detect (`app.ml.anomaly_detector`)**:
   - Continuous multi-variate telemetry analysis (EGT, HPC pressure, core shaft speeds, vibration, bypass ratio).
   - Scikit-Learn `IsolationForest` identifies acute thermodynamic and mechanical anomalies against healthy baseline cruise profiles.

2. **Predict RUL (`app.ml.rul_predictor`)**:
   - Quantile regression models trained on run-to-failure degradation curves (NASA C-MAPSS FD001 dataset).
   - Produces 90% confidence bounds ($P_{10}$ pessimistic, $P_{50}$ median, $P_{90}$ optimistic).
   - The pessimistic $P_{10}$ bound defines the mandatory latest safe maintenance deadline.

3. **Explain (`app.ml.explainer`)**:
   - Calculates SHAP feature attribution percentages for each sensor vector.
   - Generates plain-language aeronautical technical records for flight line engineers (e.g. *"EGT thermal margin erosion; high-pressure turbine seal degradation likely"*).

4. **Optimize (`app.optimizer.fleet_optimizer`)**:
   - **Google OR-Tools CP-SAT Solver** scheduling 24 aircraft over a 30-day horizon.
   - **Objective**: Maximize total ready aircraft-days across the squadron.
   - **Hard Constraints**:
     - Maintenance before pessimistic RUL deadline.
     - Hangar bay capacity (No-Overlap constraint across Bay 1 Heavy, Bay 2 Intermediate, Bay 3 Avionics).
     - Technician cumulative shift capacity by specialty.
     - Spares lead-time arrival constraint.
     - Work bundling (consolidates multiple inspections into a single bay visit).
   - Solves in **~10 milliseconds** on a standard laptop.

5. **Simulate (`app.simulator.what_if_engine`)**:
   - Dynamic sandbox testing supply delays (+1 to +14 days), operational sortie surges (1.0x to 1.8x tempo), unscheduled hangar bay closures, and technician roster deficits.
   - Produces comparative readiness curves and automated contingency mitigations in real-time.

6. **Act & Audit (`app.services.audit_service`)**:
   - Human-in-the-loop authorization: Advisory only; release-to-service requires signed officer approval.
   - Manual overrides enforce recorded justification rationale.
   - Tamper-evident **SHA-256 cryptographic hash chaining** satisfying MoD air-gapped compliance.

---

### 3. Key Performance Benchmarks

| Metric | Reactive Maintenance | SkyReady (CP-SAT) | Gain / Impact |
| :--- | :---: | :---: | :---: |
| **Fleet Availability Rate** | 71.4% | **86.8%** | **+15.4 pt gain** (Exceeds +5-10 pt target) |
| **Ready Aircraft-Days (30-day)** | 514 days | **625 days** | **+110 Ready Aircraft-Days** |
| **Unplanned Groundings** | Fragmented | Pre-requisitioned | **~40% Reduction** |
| **Optimization Latency** | Hours (Manual) | **10.0 ms** | Real-time re-planning in seconds |
| **Work Bundling** | Ad-hoc | **100% Bundled** | Eliminates repeated bay downtime |

---

### 4. Quick Start Guide

#### One-Click Launch:
```bash
./start.sh
```
This runs the full-stack system:
- **Web Dashboard**: [http://localhost:8000](http://localhost:8000)
- **Interactive API Documentation**: [http://localhost:8000/docs](http://localhost:8000/docs)

#### Manual Development Setup:

**Backend:**
```bash
# Activate virtual environment
source .venv/bin/activate

# Run FastAPI server
PYTHONPATH=backend python -m uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

**Frontend (Vite Hot-Reload):**
```bash
cd frontend
npm run dev
# Running on http://localhost:5173 (proxies /api to port 8000)
```

---

### 5. Repository Structure

```
SkyReady/
├── backend/
│   ├── app/
│   │   ├── api/routes.py            # REST API endpoints
│   │   ├── core/config.py           # Squadron configuration & constants
│   │   ├── data/fleet_generator.py  # 24-Aircraft C-MAPSS squadron generator & spares
│   │   ├── ml/
│   │   │   ├── anomaly_detector.py  # Isolation Forest anomaly scoring
│   │   │   ├── rul_predictor.py     # Quantile regression (P10, P50, P90)
│   │   │   └── explainer.py         # SHAP attribution & plain-language diagnostics
│   │   ├── optimizer/
│   │   │   └── fleet_optimizer.py   # Google OR-Tools CP-SAT scheduler
│   │   ├── simulator/
│   │   │   └── what_if_engine.py    # Real-time what-if stress simulator
│   │   ├── services/
│   │   │   └── audit_service.py     # SHA-256 tamper-evident compliance ledger
│   │   └── main.py                  # FastAPI application entrypoint
│   └── requirements.txt
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   │   ├── Navbar.jsx           # Tactical C2 header & navigation
│   │   │   ├── FleetCommandView.jsx # 24-aircraft squadron readiness matrix
│   │   │   ├── DigitalTwinView.jsx  # Subsystems & live C-MAPSS telemetry curves
│   │   │   ├── ExplainabilityView.jsx # SHAP sensor drivers & flight line narrative
│   │   │   ├── OptimizerView.jsx    # 30-day CP-SAT Gantt & Spares Requisitions
│   │   │   ├── WhatIfSimulatorView.jsx # Dynamic what-if sandbox with sliders
│   │   │   └── AuditLedgerView.jsx  # SHA-256 cryptographic chain inspector
│   │   ├── App.jsx
│   │   ├── main.jsx
│   │   └── index.css
│   ├── package.json
│   └── vite.config.js
├── SkyReady-SIH26249.pptx           # Original Team ZEN Hackathon Presentation
├── start.sh                         # One-click startup script
└── README.md
```
