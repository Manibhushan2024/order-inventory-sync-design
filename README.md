# Order & Inventory Sync Service — design submission

Mars Cosmetics engineering assessment · Manibhushan Kumar · October 2026

| File | What it is |
|---|---|
| [DESIGN.md](DESIGN.md) | The design document (architecture, data model, ingestion, inventory consistency, stock push, failure handling, observability, scope, stretch B, assumptions, trade-offs, AI note) |
| [diagram/architecture.png](diagram/architecture.png) | Architecture diagram (rendered from [architecture.mmd](diagram/architecture.mmd), Mermaid) |
| [pseudo/reserve_stock.md](pseudo/reserve_stock.md) | Pseudo-code for the stock reservation step (SQL + TypeScript) and the release path |

The short version: orders from four channels are stored raw and deduplicated at the database, stock is reserved with one conditional `UPDATE` inside the order's transaction (so two channels can never both win the last unit), every change is written to an append-only ledger, and a transactional outbox delivers each confirmed order to the ERP exactly once, even through its Sunday downtime. Channels are updated per SKU with debouncing and per-channel rate limits so the catalogue stays correct within 60 seconds on a sale day.

Stack as required: Node.js / NestJS, PostgreSQL, Redis (BullMQ), AWS. Estimated infra about ₹20k per month.
