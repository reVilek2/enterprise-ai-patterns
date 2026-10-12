# Enterprise AI Architecture Patterns

A curated collection of production-grade architectural patterns for integrating Large Language Models (LLMs) with enterprise ERP systems (1C:Enterprise, SAP), relational databases, and multi-channel e-commerce operations.

## Architecture Patterns

- [1C-PostgreSQL-LLM Gate (Inventory & Margin Ops Pipeline)](docs/1c-llm-pipeline.md) — Reliable decoupled architecture for connecting legacy 1C ERP accounting databases with external LLM reasoning engines without transactional locks or hallucinations.

## Key Principles

1. **Decoupled Staging:** Never run analytical or vector queries against the primary OLTP accounting database.
2. **Deterministic Math Guardrails:** LLMs never calculate financial metrics, margins, or inventory thresholds. Math is strictly executed by Python/SQL engines; LLMs interpret variances and formulate business actions.
3. **Human-in-the-Loop (HITL):** Automated actions require explicit approval for P1/financial changes via Telegram or CRM webhooks.
