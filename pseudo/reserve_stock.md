# Stock reservation — pseudo-code (question 4)

The whole guarantee lives in one SQL statement: a conditional `UPDATE` that Postgres evaluates on the locked, committed row. No read-then-write, no application-side lock.

## SQL

```sql
-- inventory(sku PK, on_hand int, reserved int, safety_buffer int,
--           available int GENERATED ALWAYS AS (on_hand - reserved - safety_buffer) STORED)
-- CHECK (reserved >= 0 AND reserved <= on_hand)

BEGIN;

INSERT INTO orders (id, channel, external_order_id, status, payment_mode, pincode, total_amount, last_event_time)
VALUES (:order_id, :channel, :external_order_id, 'RECEIVED', :payment_mode, :pincode, :total, :event_time)
ON CONFLICT (channel, external_order_id) DO NOTHING;      -- duplicate event: stop here, nothing else runs
-- if 0 rows inserted → ROLLBACK and mark the channel_event processed (duplicate)

-- lines sorted by sku so two orders sharing SKUs always lock rows in the same order (no deadlock)
-- for each line in ORDER BY sku:
UPDATE inventory
   SET reserved = reserved + :qty, updated_at = now()
 WHERE sku = :sku
   AND on_hand - reserved - safety_buffer >= :qty;        -- the condition is the whole point
-- rowcount 1 → reserved. rowcount 0 → short; remember it and keep going to collect all shortages

INSERT INTO stock_ledger (sku, delta_reserved, reason, order_id, actor)
VALUES (:sku, :qty, 'RESERVE', :order_id, 'ingest');     -- audit row, same transaction

-- after all lines:
-- all reserved  → UPDATE orders SET status = 'RESERVED'; INSERT INTO outbox (kind, target, idempotency_key, payload)
--                 VALUES ('ERP_ORDER', 'BC', :channel || ':' || :external_order_id, :order_json);
-- any short     → ROLLBACK the reservations of this order (they are inside the same transaction, so simply ROLLBACK),
--                 then in a NEW short transaction: INSERT the order with status 'STOCK_SHORT',
--                 INSERT stock_ledger (sku, delta_reserved = 0, reason 'RESERVE_SHORT'), and an outbox row to cancel on the channel.

COMMIT;
-- after commit (not before): SADD dirty_skus <sku...>  → the pushers update the four channels
```

## TypeScript shape (NestJS worker, `ingest` queue)

```ts
async function reserveOrder(evt: NormalisedOrder): Promise<'RESERVED' | 'STOCK_SHORT' | 'DUPLICATE'> {
  const lines = [...evt.items].sort((a, b) => a.sku.localeCompare(b.sku));   // fixed lock order

  const result = await db.transaction(async (tx) => {
    const ins = await tx.insertOrder(evt);                 // ON CONFLICT DO NOTHING
    if (ins.rowCount === 0) return 'DUPLICATE';

    const short: string[] = [];
    for (const l of lines) {
      const upd = await tx.query(
        `UPDATE inventory SET reserved = reserved + $2, updated_at = now()
          WHERE sku = $1 AND on_hand - reserved - safety_buffer >= $2`, [l.sku, l.qty]);
      if (upd.rowCount === 0) { short.push(l.sku); continue; }
      await tx.ledger({ sku: l.sku, deltaReserved: l.qty, reason: 'RESERVE', orderId: evt.orderId });
    }

    if (short.length) throw new StockShort(short);          // rolls back every reservation of this order

    await tx.setOrderStatus(evt.orderId, 'RESERVED');
    await tx.outbox({ kind: 'ERP_ORDER', target: 'BC',
                      idempotencyKey: `${evt.channel}:${evt.externalOrderId}`, payload: evt });
    return 'RESERVED';
  }).catch(async (e) => {
    if (!(e instanceof StockShort)) throw e;                // real errors → BullMQ retry
    await db.transaction(async (tx) => {                    // separate, tiny transaction
      await tx.insertOrder({ ...evt, status: 'STOCK_SHORT' });
      for (const sku of e.skus) await tx.ledger({ sku, deltaReserved: 0, reason: 'RESERVE_SHORT', orderId: evt.orderId });
      await tx.outbox({ kind: 'CHANNEL_CANCEL', target: evt.channel,
                        idempotencyKey: `cancel:${evt.channel}:${evt.externalOrderId}`, payload: { reason: 'OUT_OF_STOCK' } });
    });
    return 'STOCK_SHORT';
  });

  if (result !== 'DUPLICATE') await redis.sadd('dirty_skus', ...lines.map(l => l.sku));  // after commit
  return result;
}
```

## Release (cancel / return)

```sql
BEGIN;
UPDATE orders SET status = 'CANCELLED', last_event_time = :event_time
 WHERE id = :order_id AND status IN ('RESERVED','CONFIRMED');     -- 0 rows → already cancelled, idempotent
UPDATE inventory SET reserved = reserved - oi.qty
  FROM order_items oi WHERE oi.order_id = :order_id AND inventory.sku = oi.sku;
INSERT INTO stock_ledger (...) SELECT sku, -qty, 'RELEASE_CANCEL', :order_id, 'ingest' FROM order_items WHERE order_id = :order_id;
COMMIT;
-- returns after dispatch: on_hand += qty (reason RELEASE_RETURN) only when QC marks the unit saleable
```

## Why this holds under "two channels, same second"

Postgres row-level locking under READ COMMITTED: the second `UPDATE` waits for the first transaction, then re-checks `WHERE` against the committed row. One sees `available >= qty`, the other sees `0 >= 1` and updates nothing. The invariant `reserved <= on_hand` is also a table CHECK, so even a bug elsewhere cannot push it negative.
