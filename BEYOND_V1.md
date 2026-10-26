# Beyond v1: real-time controls the ops team gets (EasyEcom-style)

These build on the v1 design in [DESIGN.md](DESIGN.md) without changing its core: the conditional `UPDATE` in Postgres stays the only place stock is reserved. Each item below is a rule layer around it.

## 1. Per-channel reserve and priority (the "keep 5 for the website" dial)

Today `safety_buffer` is one number per SKU. The next step is an **allocation table**:

| Column | Meaning |
|---|---|
| `sku, channel` | one row per SKU per channel |
| `reserve_qty` | units held back from this channel's view (default 0; e.g. keep 5 so Nykaa never sees the last 5) |
| `max_share_pct` | upper cap of available stock a channel may show (e.g. Blinkit 40 percent) |
| `priority` | 1 = highest. When stock drops below the threshold, low-priority channels are set to 0 first |

What each channel is shown becomes `shown(channel) = max(0, min(available - reserve_qty, available * max_share_pct))`, computed by the pusher at push time from the single `inventory.available` value, so the truth never splits. Ops change these numbers in the ops screen; every change is a `stock_ledger` row with reason `ALLOC_RULE` so it is auditable.

## 2. Thresholds that act by themselves

Rules evaluated by the pusher on every dirty SKU (cheap: 1,200 SKUs, in memory):

- **Low-stock threshold** (per SKU, default 5): when `available <= threshold`, channels are updated in priority order within the same 2-second tick, lowest priority set to 0 first, and an alert goes to the ops Slack/WhatsApp with the SKU and the last 5 ledger rows.
- **Zero-first**: SKUs that just reached 0 are pushed before any increase, on all channels, in one burst that ignores the debounce window.
- **Auto-republish**: when stock comes back above `threshold + reserve_qty` (GRN or return), channels are re-enabled in reverse priority order.
- **Velocity brake** (sale day): if a SKU sells more than N units in 60 seconds, its `safety_buffer` is raised automatically by 1 for 10 minutes. This is the dial that keeps the 0.5 percent cancellation budget during the first hour of a sale.

## 3. Real-time sync: from 60 seconds to under 10

v1 already pushes on change, not on a timer. To get every channel inside 10 seconds:

- `dirty_skus` becomes a Redis **stream** per channel, consumed continuously instead of every 2 seconds; the token bucket still enforces each channel's limit.
- Website stock is updated in the reservation transaction itself (already true), so it is instant.
- Nykaa's 5 req/s is the bottleneck: at peak about 200 distinct SKUs change per minute, which is 40 seconds of calls. Priority ordering (zeros first, then hero SKUs) means the SKUs that can oversell are updated within 1 to 2 seconds; the long tail follows.
- Flipkart's 50-SKU batches and the website are well under 5 seconds; Blinkit depends on its API, so its reserve_qty is set higher (it is the least reliable channel) and the daily CSV remains the full reset.

Measured target: p95 reservation-to-channel under 10 s for zero/hero SKUs, under 30 s for everything else, at 300 orders per minute.

## 4. Two orders at the same second: payment-first wins, the other rolls back

The brief asks for reservation at order arrival, so v1 reserves immediately for every channel. The refinement the team asked for is **two-stage reservation with rollback**:

1. **Soft reserve on arrival.** The order gets `reserved` units with `hold_until = now() + 15 min` (prepaid website orders awaiting payment) or no expiry (marketplace orders, which arrive already paid/confirmed).
2. **Hard reserve on payment confirmed.** The payment webhook flips the order to `CONFIRMED` and clears `hold_until`. Only CONFIRMED orders are sent to the ERP.
3. **Rollback.** A sweeper every 30 seconds releases soft reservations whose `hold_until` passed (ledger reason `RELEASE_TIMEOUT`), and the SKU goes back on sale. A failed or abandoned payment releases immediately.
4. **Same second, last unit, both paying.** Two soft reservations cannot both exist for the last unit (the conditional `UPDATE` still runs first). If a channel order arrives while the last unit is soft-held by a website checkout that has not paid yet, the channel order is **queued as `WAITING_STOCK` for up to 15 minutes** instead of being cancelled: if the website payment completes first, the website keeps it and the channel order is cancelled with reason OUT_OF_STOCK; if the website hold expires, the waiting order takes the unit automatically. This is "whoever completes payment first wins", and it is deterministic because both outcomes are decided by the same row lock.
5. **Priority override.** Ops can mark a channel as `priority = 1` for a SKU; a priority-1 WAITING_STOCK order pre-empts a soft hold of a lower-priority channel (the hold is released, the customer sees "payment failed, out of stock" before being charged).

All of this is three columns (`hold_until`, `payment_status`, `waiting_since`) and one sweeper job; the reservation SQL is unchanged.

## 5. Multi-platform parity like EasyEcom

- **Single catalogue mapping** table: our SKU to each channel's listing id, so a push never misses a listing.
- **Channel health scores** from the 5-minute error rate drive the pusher: an unhealthy channel's reserve_qty is bumped automatically so a stale listing cannot oversell.
- **Drift report** every 15 minutes (already in v1) becomes a live tile on the ops screen: SKUs where any channel shows more than `available`.
- **Order status back-sync**: ERP dispatch and courier events are pushed back to each channel (packed, shipped, delivered) through the same outbox, so marketplaces stop penalising late updates.

## Cost of all of the above

No new infrastructure. Redis streams and the sweeper run inside the existing worker; the allocation and threshold tables are a few thousand rows. The only cost is engineering time: about 3 weeks for one developer after v1 ships.
