# RFC-0000: On-demand Coretime as Core Scheduling

|                 |                                                                                             |
| --------------- | ------------------------------------------------------------------------------------------- |
| **Start Date**  | 13 August 2026                                                                              |
| **Description** | Move on-demand order placement, pricing, and payment to the Coretime chain; serve paid orders as one-shot core assignments through the existing scheduling channel. |
| **Authors**     | Sergej Sakač                                                                                |

## Summary

On-demand ordering moves to the Coretime chain, and the Relay Chain stops knowing that on-demand exists. The broker sells individual future blocks on pool cores and streams them to the Relay Chain as one-shot core assignments, through the same `assign_core` channel that bulk coretime already uses. Users pay DOT on the Coretime chain at a spot price computed from purely local state. The Relay Chain's `on_demand` pallet is removed, along with its credit and revenue accounting.

## Motivation

RFC-1 split coretime into two products: Bulk Coretime, sold on the Coretime chain, and Instantaneous (on-demand) Coretime, sold on the Relay Chain. Bulk allocation is fully owned by the Coretime chain today: the broker pallet computes each core's schedule and commits it to the Relay Chain via the `assign_core` call specified in RFC-5. On-demand is the exception. To sell a single block, the Relay Chain maintains:

- an order queue (`on_demand` pallet) holding up to `on_demand_queue_max_size` orders, which is scanned in full on every scheduling round;
- an adaptive spot price (`traffic * on_demand_base_fee`), updated every block and on every order;
- three user-facing extrinsics (`place_order_allow_death`, `place_order_keep_alive`, `place_order_with_credits`);
- a payment pot and revenue tracking, so that income can later be teleported to the Coretime chain;
- a `Credits` ledger, tracking credits purchased on the Coretime chain and spent on relay-side orders.

This is the last piece of coretime still living on the Relay Chain; RFC-32 (Minimal Relay) set the direction of migrating such subsystems into system chains.

### Requirements

1. A collator MUST be able to obtain a block for its parachain with a single DOT-paying transaction on the Coretime chain.
2. A paid order MUST NOT be overwritten by a later schedule, nor silently dropped.
3. The pricing input MUST be derived purely from Coretime chain state; no Relay Chain state may be required.
4. The Relay Chain SHOULD end up with zero on-demand-specific logic.
5. The UMP traffic sent per Coretime chain block MUST remain bounded, regardless of how many orders are placed.

## Stakeholders

- Parachain teams producing blocks on demand, and their collators.
- Bulk coretime holders who contribute regions to the instantaneous pool and receive its revenue.
- Runtime developers of the Relay Chain and the Coretime chain.
- Wallets, SDK tooling, and collator-node automation currently built against the Relay Chain `place_order_*` extrinsics.

## Explanation

### Overview

Pool cores are no longer committed to the Relay Chain as `CoreAssignment::Pool`; the broker sells their individual blocks instead. An order proceeds as follows:

1. A user calls `place_order(max_amount, para_id)` on the Coretime chain.
2. The broker computes the spot price from its local count of outstanding orders. The order is rejected, with nothing charged, if the price exceeds `max_amount`, every pool core is at its pending-order cap, or the block's order limit has been reached.
3. Otherwise the spot price is charged and the order is buffered against the pool core with the fewest pending orders.
4. At the end of the Coretime chain block, the whole buffer is flushed as a single UMP message: one `Transact` per core, each carrying one `assign_core_once(core, tasks)` call with every order assigned to that core in this block.
5. The Relay Chain appends each call's tasks to the target core's schedule queue as a *one-shot* schedule: each task is served for exactly one block, in order, retrying while the core is blocked. Once the schedule is exhausted, the next one in the queue takes over.

The `on_demand` pallet and all remaining on-demand logic is deleted from the Relay Chain.

### Ordering on the Coretime chain

`place_order(max_amount: Balance, para_id: TaskId)` is a new dispatchable on the Coretime chain. As on the Relay Chain today, `max_amount` is only a price cap; the caller is charged the current spot price.

`place_order` does not send an XCM message by itself; it only buffers the order. All orders buffered during a block are sent to the Relay Chain together, at the end of the block, so UMP traffic stays bounded no matter how many orders are placed.

The broker assigns each order to the pool core with the fewest pending orders. It refuses an order only when:

- the spot price exceeds the caller's `max_amount`;
- every pool core has reached `MAX_PENDING_ORDERS_PER_CORE` pending orders; or
- `MAX_ORDERS_PER_BLOCK` orders have already been accepted in this block.

Orders MUST NOT be accepted onto a core that leaves the pool before the order's expected service block (plus a safety margin). Bulk scheduling has priority on the Relay Chain: a windowed schedule takes over the core at its `begin` even if one-shot tasks are still pending. The safety margin is therefore what guarantees that every paid order is served before the core changes hands.

Because whole blocks of a core are sold, pooling now requires a complete `CoreMask`. Attempts to pool a region whose mask is not complete MUST be rejected.

### Batching

At the end of each Coretime chain block, the pallet drains the order buffer and constructs, per pool core with new orders, a single call:

```rust
assign_core_once(core, vec![p1, p2, ...])
```

The tasks are listed in the order in which the orders were placed. All per-core calls are packed into one UMP message containing one `Transact` per core, prefixed with `UnpaidExecution`, the same way `assign_core` messages are constructed today.

Two limits apply to this message. The Relay Chain caps the number of instructions per XCM message (100 on Polkadot today), which bounds the number of `Transact`s, and it has only limited weight for executing incoming messages. `MAX_ORDERS_PER_BLOCK` MUST be chosen to respect both; if the number of pool cores ever approaches the per-message `Transact` limit, the flush can spill into multiple messages.

### Pool cores

With the relay-side order queue gone, `CoreAssignment::Pool` has nothing to point at. When a core enters the pool, the broker instead commits `assign_core(core, begin, [(Idle, 57600)], None)`. This is needed because schedules have no expiry. Without it, the previous tenant would keep producing blocks on the core. Sold blocks arrive via `assign_core_once`, and between them the core stays idle.

### The one relay-side change: one-shot assignments

The Relay Chain's coretime-assignment scheduler gains a second schedule kind:

```rust
pub enum ScheduleKind {
    /// The assignments share the core, each getting blocks in proportion to its ratio.
    /// Replaced as soon as the next schedule's `begin` is reached. This is the existing
    /// behaviour of every `assign_core` created schedule.
    Windowed,
    /// Each assignment is served for exactly one block, in order. The schedule never
    /// expires on its own, but a windowed schedule takes priority once its `begin` is
    /// reached.
    OneShot,
}
```

and the relay-side coretime pallet gains a second broker-facing call next to `assign_core`:

```rust
pub fn assign_core_once(
    origin: OriginFor<T>,
    core: BrokerCoreIndex,
    tasks: Vec<TaskId>,
) -> DispatchResult
```

with the same origin rule as `assign_core` (root, or the broker parachain), and the following semantics:

- Each task occupies the full core for exactly one block; tasks are served in the order they arrived. There are no ratios and no `end_hint`.
- A task that cannot be served in a given block retries the next block, keeping its place. This replaces the `on_demand` pallet's `push_back_order`.
- The call SHOULD never fail. The order is already paid for on the Coretime chain; if the Relay Chain rejected the call, the user would have paid for a block that never gets scheduled.

Because one-shots only ever append behind existing entries, they never replace assignments already visible in the claim queue.

A draft implementation of the relay-side changes: <https://github.com/paritytech/polkadot-sdk/compare/master...szegoo-oneshot-assignments>

### Pricing

Pricing uses no Relay Chain data: the Coretime chain has everything it needs in local state.

The price retains the exact form used by the Relay Chain today:

```
spot_price = traffic * on_demand_base_fee
```

with `traffic` updated by the same adaptive formula the Relay Chain uses now, but fed from local state:

- *queue size* is the number of orders accepted but not yet served. The broker can track this locally, since the Relay Chain serves one task per pool core per block;
- *queue capacity* is `number_of_pool_cores * MAX_PENDING_ORDERS_PER_CORE`.

TODO...

## Drawbacks

TODO

## Testing, Security, and Privacy

TODO

## Prior Art and References

TODO

## Future Directions and Related Material

TODO
