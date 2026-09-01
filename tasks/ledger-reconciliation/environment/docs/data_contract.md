# Order / ledger data contract

Two exports of the same business day-to-day activity.

## `data/app_orders.csv` — the operational application (system of record)
| column | meaning |
|---|---|
| `order_id` | primary key |
| `created_at_utc` | instant the order was placed, UTC, ISO-8601 |
| `channel` | how the order was taken (`web`, `mobile`, `partner`, `legacy_pos`) |
| `amount_cents` | order total, integer cents |
| `status` | workflow state (`active`, `cancelled`, `refunded`, `completed`) |
| `deleted_at` | timestamp the record was soft-deleted; empty if it never was |
| `customer_region` | billing region |

## `data/ledger_export.csv` — the downstream finance ledger
| column | meaning |
|---|---|
| `entry_id` | ledger surrogate key |
| `order_id` | foreign key to the application |
| `business_date` | calendar date the order is booked to |
| `amount_usd` | order total in dollars, two decimals |
| `currency` | always `USD` |
| `batch_id` | export batch label, no business meaning |

## Contract the ledger is supposed to satisfy

* Exactly one ledger entry per order that has not been soft-deleted, and no entry
  for an order that has.
* `business_date` is the calendar date of `created_at_utc` in the company's
  operating timezone, **America/Los_Angeles**.
* `amount_usd` equals `amount_cents / 100`.

The company operates on Pacific time year round. `status` has no bearing on
whether an order belongs in the ledger.
