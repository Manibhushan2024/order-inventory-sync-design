# Order & Inventory Sync Service — design submission

Mars Cosmetics engineering assessment · Manibhushan Kumar · October 2026

| File | What it is |
|---|---|
| [DESIGN.md](DESIGN.md) | The design document (architecture, data model, ingestion, inventory consistency, stock push, failure handling, observability, scope, stretch B, assumptions, trade-offs, AI note) |
| [diagram/architecture.png](diagram/architecture.png) | Architecture diagram (rendered from [architecture.mmd](diagram/architecture.mmd), Mermaid) |
| [pseudo/reserve_stock.md](pseudo/reserve_stock.md) | Pseudo-code for the stock reservation step (SQL + TypeScript) and the release path |
| [BEYOND_V1.md](BEYOND_V1.md) | Next-level controls: per-channel reserve and priority, automatic thresholds, sub-10 s real-time sync, payment-first reservation with rollback, channel parity |

The short version: orders from four channels are stored raw and deduplicated at the database, stock is reserved with one conditional `UPDATE` inside the order's transaction (so two channels can never both win the last unit), every change is written to an append-only ledger, and a transactional outbox delivers each confirmed order to the ERP exactly once, even through its Sunday downtime. Channels are updated per SKU with debouncing and per-channel rate limits so the catalogue stays correct within 60 seconds on a sale day.

Stack as required: Node.js / NestJS, PostgreSQL, Redis (BullMQ), AWS. Estimated infra about ₹20k per month.

## Setup / how to review

This is a design submission (the brief asks for a design doc and a diagram; code is not required), so there is nothing to install or run.

1. Clone or browse the repo: `git clone https://github.com/Manibhushan2024/order-inventory-sync-design.git`
2. Read `DESIGN.md` first (about 4 pages). Section 4 is the concurrency walkthrough: two channels selling the last unit in the same second.
3. Open `diagram/architecture.png` for the architecture. To edit or re-render it, paste `diagram/architecture.mmd` into https://mermaid.live.
4. `pseudo/reserve_stock.md` has the reservation SQL and a NestJS-style TypeScript sketch you can drop into a worker.
5. `BEYOND_V1.md` covers what comes after v1: per-channel reserves and priority, automatic thresholds, sub-10-second sync, payment-first reservation with rollback.

## Approach

I started from the oversell incident: the root cause is that stock lives in a spreadsheet and is copied to channels twice a day, so any design has to make one place the truth and update channels from it continuously. I put that truth in PostgreSQL and made reservation a single conditional `UPDATE` in the same transaction as the order, so correctness does not depend on timing, locks in application code or Redis. Everything else follows from protecting that statement: raw events are stored first and deduplicated by unique keys, each channel gets its own queue and rate limiter so one failure cannot block the others, and a transactional outbox makes ERP delivery exactly-once even through Sunday downtime. I kept the stack to what the brief allows (NestJS, Postgres, Redis via BullMQ, AWS) and sized it at about ₹20k a month.
