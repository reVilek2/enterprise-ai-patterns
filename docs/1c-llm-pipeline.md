# 1C-PostgreSQL-LLM Gate Pattern

Production architectural pattern for integrating **1C:Enterprise (1С:Предприятие / ERP / КА / УТ)** with external LLM reasoning engines and operational workflows (Telegram, CRM).

---

## 1. High-Level Architecture

```mermaid
flowchart TD
    subgraph Accounting_Core["1C:Enterprise (OLTP Core)"]
        A1[1C Database] -->|Scheduled JSON Export| A2[Staging Sync Service]
    end

    subgraph Data_Layer["Decoupled Data Hub"]
        A2 -->|Upsert Snapshot| B1[(PostgreSQL Staging DB)]
        MP[Marketplace APIs<br/>WB / Ozon] -->|Webhooks / Polling| B1
    end

    subgraph Analytics_Engine["Deterministic Compute Engine"]
        B1 --> C1[Python FastAPI Worker]
        C1 -->|Calculate Unit-Economics & Runout Dates| C2{Variance / Margin Drop?}
    end

    subgraph AI_Reasoning["LLM Orchestration Layer"]
        C2 -->|Threshold Breached<br/>Structured JSON Payload| D1[Prompt & Context Assembler]
        D1 --> D2[LLM / Qwen 2.5 / OpenAI]
        D2 -->|Root-Cause Summary + Draft Action| D3[Structured Action Proposal]
    end

    subgraph Human_Gate["Human-in-the-Loop (HITL) Gate"]
        D3 --> E1{Severity Level}
        E1 -->|P1 / Financial Impact| E2[Telegram Executive Bot<br/>One-Click Approval]
        E1 -->|Operational Task| E3[CRM / Bitrix24 Task Queue]
    end

    subgraph Execution_Loop["Execution & Closed Loop"]
        E2 -->|Approved| F1[Marketplace Repricing API / 1C Staging Order]
        F1 --> F2[Verification Worker<br/>48h Closed-Loop Check]
        F2 -.->|Confirmed Margin Recovery| E2
    end
```

---

## 2. Core Architectural Principles

### 1. Isolated Staging (Protecting 1C Core)
- **Problem:** Running analytical joins, vector searches, or frequent API polling on a live 1C Microsoft SQL/Postgres cluster risks table locks, schema cache invalidations, and license violations.
- **Solution:** 1C publishes lightweight daily or hourly incremental JSON snapshots (SKU balances, cost-of-goods, reserved quantities) via HTTP/REST into an external **PostgreSQL Staging DB**.
- The analytics and LLM layer **never connects directly to the 1C primary cluster**.

### 2. Deterministic Math Guardrail (No LLM Hallucinations on Numbers)
- **Rule:** Large Language Models are strictly prohibited from performing arithmetic calculations (e.g., unit margin, commission tiers, runout velocity).
- **Implementation:**
  - Python / SQL code computes precise numbers (e.g. `Margin = 4.1%`, `Loss = 142,000 RUB`, `Days_to_OOS = 6.2`).
  - The LLM receives pre-calculated metrics and serves solely as a **Root-Cause Reasoner and Executive Synthesizer**: explaining *why* the variance occurred and drafting actionable next steps.

### 3. Human-in-the-Loop (HITL) Guardrail
- Automated price changes and purchase orders are routed through an approval gate:
  - **C-Level Telegram Bot:** One-Click `[Approve]` / `[Reject]` for critical financial threshold changes.
  - **CRM Tasks (Bitrix24):** Structured task cards with checklists for operational personnel (e.g. warehouse audits, logistics rerouting).

### 4. Closed-Loop Verification (48h Check)
- Any executed action schedules an automated verification job.
- After 48 hours, the worker verifies that the market price is updated, stock velocity is stabilized, and margin has returned above the target threshold.
