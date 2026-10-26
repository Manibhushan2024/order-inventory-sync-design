# Order & Inventory Sync Service — Design

Manibhushan Kumar · October 2026 · Mars Cosmetics engineering assessment
Diagram: [diagram/architecture.png](diagram/architecture.png) · Reservation pseudo-code: [pseudo/reserve_stock.md](pseudo/reserve_stock.md)

## 0. The one idea everything rests on

Stock is reserved by **one conditional row update in PostgreSQL, inside the same transaction that saves the order**:

```sql
UPDATE inventory SET reserved = reserved + :qty
WHERE sku = :sku AND on_hand - reserved - safety_buffer >= :qty;   -- 0 rows updated = no stock
```

There is no read-then-write in the hot path, so two channels can never both win the last unit. Everything else exists to get orders into that statement reliably and to tell the channels and the ERP what happened.

## 1. Architecture

| Component | Where | Job |
|---|---|---|
| **api** (NestJS) | Fargate, 2 × 0.5 vCPU | Receives webhooks (website, Nykaa, Blinkit) and serves the ops screen. A webhook is verified, stored raw in `channel_events` (idempotent insert), enqueued, answered 200 in under 50 ms. No business logic here. |
| **worker** (NestJS + BullMQ) | Fargate, 1 task, auto-scales to 3 on queue depth | Queues: `ingest` (normalise + reserve), `push:website`, `push:nykaa`, `push:flipkart`, `push:blinkit`, `erp`, `poll:flipkart`, `reconcile`. One queue per channel is the isolation boundary. |
| **PostgreSQL 16** | RDS db.t4g.small | System of record: orders, inventory, ledger, outbox, raw events. |
| **Redis 7** | ElastiCache cache.t4g.micro | BullMQ queues, per-channel rate-limit buckets, dirty-SKU set. Never the only copy of anything. |
| **Ops screen** | Next.js page served by api | Failed syncs with Retry; per-SKU ledger view. |

Flow: channel event → `channel_events` → `ingest` → one transaction {order, reservation, ledger row, outbox row} → `erp` queue delivers the order; the SKU is marked dirty and the four `push:*` queues update each channel under its own rate limit.

Cost (ap-south-1, rounded): Fargate ₹9k, RDS ₹4.5k, ElastiCache ₹1.5k, ALB ₹2k, CloudWatch/S3 ₹3k, **about ₹20k/month**, leaving half the budget for sale-day scale-out. No SQS or Kafka: 300 orders/min is 5 per second, which BullMQ handles with a large margin, and the team already knows Node.

## 2. Data model

| Table | Key columns | Constraints |
|---|---|---|
| `channel_events` | channel, external_event_id, external_order_id, event_type, event_time, payload jsonb, processed_at, status | **UNIQUE (channel, external_event_id)**. Raw truth for replay. Flipkart polling makes `external_event_id = order_id + ':' + status + ':' + updated_at`. |
| `orders` | id, channel, external_order_id, status (RECEIVED → RESERVED / STOCK_SHORT → CONFIRMED → CANCELLED / RETURNED), payment_mode, pincode, total_amount, last_event_time, erp_doc_id | **UNIQUE (channel, external_order_id)**: duplicate orders are rejected by the database, not by code. |
| `order_items` | order_id, sku, qty, unit_price | qty > 0 |
| `inventory` | sku PK, on_hand, reserved, safety_buffer, available (generated) | CHECK (reserved BETWEEN 0 AND on_hand). 1,200 rows. |
| `stock_ledger` | id bigserial, sku, delta_on_hand, delta_reserved, reason (RESERVE, RESERVE_SHORT, RELEASE_CANCEL, RELEASE_RETURN, DISPATCH, GRN, ADJUST), order_id, actor, created_at | **Append-only**, written in the same transaction as every `inventory` change. "Why did stock change and when" = `SELECT * FROM stock_ledger WHERE sku = ? ORDER BY id`. |
| `outbox` | kind (ERP_ORDER, STOCK_PUSH, CHANNEL_CANCEL), target, idempotency_key, payload, status (PENDING, SENT, FAILED, DEAD), attempts, next_attempt_at, last_error | **UNIQUE (kind, target, idempotency_key)**. Marked SENT only after the target acknowledged. |
| `channel_stock_view` | sku, channel, qty_shown, pushed_at, status | What each channel shows now; drives deltas and the drift report. |

Indexes: `orders(status, last_event_time)`, `outbox(status, next_attempt_at)`, `stock_ledger(sku, id)`. The ops screen is a view over `outbox` FAILED/DEAD rows plus failed `channel_events`.

## 3. Order ingestion

**Webhooks.** `api` checks the HMAC, does `INSERT … ON CONFLICT (channel, external_event_id) DO NOTHING`, enqueues the event id, returns 200. A duplicate hits the conflict before any logic runs. Processing happens in the worker, so a slow database never makes Nykaa time out and resend.

**Flipkart polling.** Every 2 minutes, `updatedSince = cursor − 5 min` on purpose; each returned order becomes a synthetic event keyed by id + status + updated time, so the overlap is harmless. The cursor advances only after the batch is stored.

**Out of order (Nykaa).** Each order keeps `last_event_time`. A CANCELLED event is applied even if its time is older than the stored CREATED (a cancel is final); a CREATED arriving after a CANCELLED is stored but ignored for state, with a ledger note. The rules are one small state table in code.

**Channel down.** Nothing changes for the others: separate queues, separate concurrency. Inbound is never lost because the raw event is persisted before processing; Flipkart orders placed during an outage are caught by the overlapping cursor.

## 4. Inventory consistency: the last unit, same second

`MRS-LIP-104`: `on_hand 10, reserved 9, buffer 0`, one unit available. Nykaa and Flipkart both sell it at 11:04:52.

1. Both events land in `channel_events` and are enqueued; two worker jobs start, possibly on two machines.
2. Each opens a transaction and inserts its `orders` row (RECEIVED) and items.
3. Each runs the conditional UPDATE. Postgres row-locks the inventory row for the first; the second **blocks** until the first commits. Plain READ COMMITTED, no SERIALIZABLE, no advisory locks.
4. Job A commits: `reserved = 10`, ledger `RESERVE 1 for NYK-…`, outbox `ERP_ORDER`, SKU added to `dirty_skus`. Order A → RESERVED.
5. Job B's UPDATE re-checks its WHERE against the committed row: `10 − 10 − 0 ≥ 1` is false, **0 rows**. B becomes `STOCK_SHORT`, writes ledger `RESERVE_SHORT` (so the audit trail shows the miss), commits, and queues a cancel-back to Flipkart with reason OUT_OF_STOCK. No ERP row for B.
6. Pushers see the dirty SKU and send `available = 0` to all channels within seconds (§5).

Multi-line orders reserve lines in sorted SKU order inside one transaction (no lock-order deadlocks); if any line is short the whole order rolls back and is recorded as STOCK_SHORT.

**Why not reserve in Redis?** A Lua script is faster, but then Redis holds the stock truth and Postgres holds the order; a crash between them leaves a reservation without an order or the reverse. One Postgres transaction removes that class of bug, and at 5 orders/s the row lock is unmeasurable.

**Safety buffer.** Per SKU, default 0, set to 2–3 units on hero SKUs during a sale. It covers the seconds between a reservation and the slowest channel showing the new number, which is where the 0.5 percent cancellation budget is spent. It is a dial tuned from the push-latency metric, not a guess.

**Cancellations and returns (6 percent).** Cancel releases `reserved` (ledger RELEASE_CANCEL). A return adds to `on_hand` only after warehouse QC marks it saleable (RELEASE_RETURN). Both mark the SKU dirty, so stock is back on sale inside the same 60-second budget.

## 5. Stock push within 60 seconds

Pushing every order to every channel would be 1,200 calls a minute at peak and would break Nykaa's limit, so pushes are **per SKU and debounced**:

- Any change adds the SKU to a Redis set `dirty_skus`.
- Every 2 seconds each `push:<channel>` worker takes the dirty SKUs not yet pushed since their last change, reads `available` from Postgres (single truth) and sends it. Ten changes in two seconds become one push.
- Per-channel token buckets match the contract. **Nykaa** 5 req/s, one SKU each: 300 SKUs/min; only changed SKUs are sent, and about 200 distinct SKUs change per minute at peak, so it fits. **Flipkart** 50 SKUs per call at 10 req/s: the whole catalogue is 3 calls; a 429 pauses the bucket for `Retry-After` and the batch is re-queued. **Blinkit** REST for every change (2 percent timeouts retried three times with jitter, then the next tick), daily CSV as the full reset. **Website** written in the reservation transaction itself, exact and immediate.
- SKUs that just reached 0 are pushed first; that is where overselling happens.

Target: reservation-to-last-channel p95 under 15 s normally, under 45 s at peak.

## 6. Failure handling

| Failure | Handling |
|---|---|
| Transient error, timeout, Blinkit's 2 percent | Outbox retry with backoff 5 s → 30 s → 2 min → 10 min, 8 attempts, jitter. Senders are idempotent, so a timeout after the target applied the change is safe. |
| ERP down 1–2 h on Sunday | `ERP_ORDER` rows stay PENDING; the sender backs off; ordering and reservation continue. When the health probe sees the ERP back, the backlog (about 700 orders) drains in minutes at 10/s. Each order carries `idempotency_key = channel:external_order_id` and the connector checks for an existing document with that reference before creating, so a crash mid-send cannot create two sales orders. |
| Flipkart 429 | Bucket pauses for `Retry-After`; not counted as an attempt. |
| Message that keeps failing (bad SKU map, ERP validation 4xx) | DEAD after 8 attempts, or at once for a non-429 4xx; shown on the ops screen with the last error; Retry resets it. DEAD rows older than 30 min alert. Nothing is dropped silently. |
| Worker crash mid-transaction | Postgres rolls back; BullMQ redelivers the job; replay is safe because the order insert is unique and the reservation is conditional. |
| Drift | `reconcile` every 15 min compares `channel_stock_view` with `inventory.available` and re-pushes differences; Blinkit CSV daily. |
| Redis loss | Queues rebuild from `outbox PENDING` and unprocessed `channel_events` at start-up. |

## 7. Observability (day one)

1. **Oversell rate**: STOCK_SHORT orders / orders received per hour. Alert above 0.3 percent on a sale day.
2. **Reservation-to-push latency** p95 per channel. Alert above 60 s.
3. **Outbox age**: oldest PENDING `ERP_ORDER`. Alert above 15 min outside the Sunday window, above 3 h always.
4. **Queue depth and DEAD rows** per queue. Depth growing for 5 min triggers Fargate scale-out; any DEAD row pages the on-call.
5. **Channel API error and 429 rate** per channel over 5 min, to tell a channel outage from our bug.

Logs are structured JSON with `order_id` and `sku` on every line (CloudWatch). The ledger table is the audit log.

## 8. Scope: deliberately out of v1

Multi-warehouse (the `inventory` key is `sku`; adding `warehouse_id` later is a migration, not a rewrite); a returns QC scanning app (v1 uses a one-field form); automatic safety buffers (manual dial in v1); price sync (different pipeline, doubles the surface); fully automated Blinkit CSV upload (generated by the service, uploaded by ops for two weeks while the REST path is proven).

Six weeks, three developers: weeks 1–2 data model, ingestion, reservation (website + Nykaa); 3–4 Flipkart poller, pushers, ERP outbox; 5 ops screen, reconciliation, alerts; 6 load test at 600 orders/min and buffer tuning.

## 9. Stretch B: "Why did SKU X oversell yesterday?"

A thin LLM layer over data we already keep, not a free chatbot. **Reads:** `stock_ledger` and `channel_stock_view` history for that SKU and day, the STOCK_SHORT orders, the push-latency rows. **How:** the model must pick one of five fixed, parameterised SQL templates (ledger timeline, short orders, push latency per channel, buffer changes, reconciliation events); the service runs it and returns rows; the model answers in plain English citing the ledger row ids after each claim; a second cheap call checks every number in the answer exists in the rows, otherwise the answer becomes "not found in the ledger". **Stops hallucination:** no free SQL, templates only, citations required, numeric cross-check, visible "data as of" time. **Cost:** about 3k tokens per question on a small model, roughly ₹0.30 each, under ₹500 a month at 50 questions a day.

## Assumptions

1. Webhooks carry an HMAC signature and a unique event id; otherwise `order_id + event_type + event_time` is the key.
2. Channel stock APIs accept an absolute quantity per SKU.
3. The ERP can be queried by external reference, which enables the idempotency check.
4. Returns are checked at the warehouse and marked saleable by a person before stock goes back.
5. Prepaid and COD reserve stock the same way; COD confirmation calls are out of scope.
6. One SKU code is shared across channels (a mapping table if not).
7. Peak means 300 orders/min for an hour, about 600 line reservations/min.
8. This service may write the website's stock directly.

## Trade-offs

| Decision | Chosen | Rejected | Why |
|---|---|---|---|
| Where stock is reserved | Conditional UPDATE in Postgres, same transaction as the order | Redis Lua script; optimistic version column with retry | One truth, crash-safe, no oversell by construction. Redis splits the truth; optimistic retries thrash on a hot SKU during a sale. |
| Queueing | BullMQ on Redis | SQS, Kafka | Fits budget and team; 5 orders/s is far below its limits. SQS adds a second queue model for little; Kafka is operational weight for no gain. |
| Stock push | Per-SKU debounce + per-channel token buckets | Push on every order; full catalogue sync every minute | Every-order pushes exceed Nykaa's limit at peak; full syncs waste Flipkart calls and are slower to show zero on hot SKUs. |
| ERP delivery | Transactional outbox + idempotency key | Direct API call in the order transaction | A direct call ties order acceptance to ERP uptime and fails every Sunday; the outbox keeps accepting orders and gives exactly-once. |
| Sale-day oversell protection | Per-SKU safety buffer tuned from latency | Fixed per-channel stock allocations | Allocations leave stock unsold on slow channels; a small buffer costs a few units and can be changed live. |

## AI usage

Claude (Anthropic) helped draft and tighten the text, generate the diagram and check the arithmetic; every decision is mine and I can explain and change each one on the call.
