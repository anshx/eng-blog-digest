---
title: "Apache Kafka, Purgatory, and Hierarchical Timing Wheels"
source: https://www.confluent.io/blog/apache-kafka-purgatory-hierarchical-timing-wheels/
author: Yasuhiro Matsuda
company: Confluent / Apache Kafka
date_posted: 2015-10-28
date_digested: 2026-06-22
---

# Apache Kafka, Purgatory, and Hierarchical Timing Wheels

## What's new to learn

1. **Timing wheels as an index into time**: A circular array can replace a priority queue for timer management by using the expiry time directly as an array index — converting O(log n) heap operations into O(1) array indexing at the cost of bounded time range.

2. **Hierarchical decomposition for arbitrary time ranges**: Stacking multiple wheels at different granularities (like hour/minute/second clock hands) extends a single-level O(1) wheel to cover arbitrary time ranges while keeping amortized O(1) insert, delete, and expiry — the same divide-and-conquer-by-range idea behind radix sort and multi-level page tables.

3. **Lazy cascade as amortized cost hiding**: Timers in coarse-grained wheels are only demoted into finer-grained wheels at cascade time, not at insert time — the cascade cost is bounded by the number of timers expiring in that bucket, so the amortized cost per timer remains O(1).

## Prerequisites

- **Min-heap / priority queue**: the data structure timing wheels replace — why heap operations are O(log n) and why that matters under high timer throughput.
- **Circular buffer (ring buffer)**: the modular-arithmetic indexing trick that makes the wheel "circular."
- **Linked list splice and detach**: each wheel bucket is a doubly-linked list; O(1) insert and delete require holding a direct pointer to the list node.
- **OS tick / timer interrupt**: what a "tick" means — periodic hardware interrupt advancing the software clock, the basic unit of timer granularity.

## The core idea

The standard tool for managing N active timers is a min-heap: insert in O(log N), find-minimum (next expiry) in O(1), delete in O(log N). For a database handling a million connections, each with a keepalive timer, that's 20 comparisons per timer event.

Timing wheels trade this log-factor away by answering: _if I know the tick at which a timer expires, I can store it in an array indexed by that tick instead of comparing it against other timers_.

Concretely: maintain a circular array of W slots. Slot _i_ holds a linked list of timers that expire at virtual-time _i_. The current time is tracked as an integer `now`. When inserting a timer expiring at time T:

```
slot = T % W
append timer to array[slot]
```

On each tick:
```
now++
fire all timers in array[now % W]
```

Insert and fire are both O(1). No sorting, no comparison, no heap bubbling.

The limitation: a wheel of size W can only span `W` ticks into the future. A timer expiring `W+1` ticks from now would land in the same slot as one expiring 1 tick from now.

Hierarchical timing wheels solve this by stacking multiple wheels at increasing granularities — exactly like a clock has hour, minute, and second hands. A three-level wheel with sizes [60, 60, 24] and resolutions [1 ms, 60 ms, 3600 ms] can track timers from 1 ms to 86,400,000 ms (one day) using just 144 array slots.

## Mechanics

### Simple timing wheel (Varghese & Lauck, Scheme 4)

```
wheel: array[W] of linked-list   # W slots
now:   integer                   # current virtual time (in ticks)

insert(timer, delay_ticks):
    slot = (now + delay_ticks) % W
    append timer to wheel[slot]

per_tick():
    now++
    fire_and_clear(wheel[now % W])
```

**Constraint**: delay_ticks must be < W. Timers expiring beyond the wheel's range must be queued elsewhere (e.g., an overflow list).

**Cost**: O(1) insert, O(1) + (number of expiring timers) per tick.

### Hashed timing wheel (Scheme 6)

To handle overflows without a separate structure, store the full absolute expiry time in each timer node and check it at fire time:

```
insert(timer, delay_ticks):
    timer.expiry = now + delay_ticks
    slot = timer.expiry % W
    append timer to wheel[slot]   # may land in the same slot as a far-future timer

per_tick():
    now++
    for timer in wheel[now % W]:
        if timer.expiry == now:
            fire(timer)
        else:
            reinsert(timer)       # spurious wake, put back
```

This handles arbitrary delays but has O(k) per-tick work where k is the number of timers sharing a slot, which degrades if timers cluster on popular modular residues.

### Hierarchical timing wheel (Scheme 7 — the recommended approach)

Three (or more) wheels at increasing granularity:

```
# Three levels: ms, sec, min
wheel_ms:  array[1000]   resolution = 1 ms   range = 0–999 ms
wheel_sec: array[60]     resolution = 1 sec  range = 1–59 sec
wheel_min: array[60]     resolution = 1 min  range = 1–59 min
```

**Insert** routes a timer to the coarsest wheel that contains its expiry:

```
insert(timer, delay_ms):
    abs_expiry = now_ms + delay_ms

    if delay_ms < 1000:
        slot = abs_expiry % 1000
        append to wheel_ms[slot]
    elif delay_ms < 60_000:
        slot = (abs_expiry / 1000) % 60
        append to wheel_sec[slot]
    else:
        slot = (abs_expiry / 60_000) % 60
        append to wheel_min[slot]
```

**Cascade** happens when a coarse wheel ticks:

```
per_ms_tick():
    now_ms++
    fire_all(wheel_ms[now_ms % 1000])

    if now_ms % 1000 == 0:          # seconds wheel ticks
        cascade_sec(now_ms / 1000)

    if now_ms % 60_000 == 0:        # minutes wheel ticks
        cascade_min(now_ms / 60_000)

cascade_sec(sec_tick):
    slot = sec_tick % 60
    for timer in wheel_sec[slot]:   # reinsert into ms wheel
        reinsert(timer)
    clear(wheel_sec[slot])
```

At cascade time, every timer in the coarse bucket is re-examined and inserted into the finer wheel based on its remaining time. The cascade cost is bounded by the number of timers in that coarse bucket — which, amortized over the W_fine ms ticks before the coarse tick, is O(1) per timer.

### Kafka's purgatory implementation

Kafka's broker must track thousands of pending requests that can only be answered later. The two canonical cases:

- **Produce (acks=all)**: complete when all in-sync replicas have acknowledged. Timeout if no ack by `request.timeout.ms`.
- **Fetch (min.bytes=N)**: complete when N bytes of new data arrive. Timeout if no data by `fetch.max.wait.ms`.

The old implementation used Java's `DelayQueue` (a min-heap). Under load, every produce or fetch inserted and removed from the heap: O(log n) per operation, with lock contention under high concurrency.

The redesigned purgatory uses a hierarchical timing wheel for the timeout timer, combined with a `DelayQueue` of _bucket timestamps_ (not individual timers) to advance the clock hand only when the next bucket actually needs firing. This avoids busy-polling the clock on every millisecond.

```
RequestPurgatory:
    timer: HierarchicalTimingWheel
    watchers: HashMap<Key, WatcherList>  # event-driven completion

put(request, timeoutMs):
    timer.add(request, timeoutMs)        # O(1)
    watchers.add(request.key, request)  # O(1)

on_event(key, data):                    # e.g., replication ack arrives
    for request in watchers[key]:
        if request.can_complete(data):
            request.complete()
            timer.remove(request)        # O(1) with direct node pointer

on_timer_fire(request):
    if not request.completed:
        request.complete_with_timeout()
```

**Key property**: requests that complete early (by event) are removed from the timer in O(1) by holding a direct pointer to their linked-list node. No heap scan needed. The timer fires only for requests that genuinely time out.

## Where it breaks

**No ordering within a bucket**: all timers in the same bucket fire at the same tick, regardless of their sub-tick expiry. For a wheel at 1-ms resolution, you cannot distinguish a 1001 ms timer from a 1000 ms timer within the same slot.

**Memory for sparse long delays**: a wheel of size W takes O(W) memory regardless of how many timers are active. If you need microsecond resolution for timeouts up to one hour, you need 3.6 billion slots — impractical. In practice, hierarchical wheels trade memory for range, and the finest level must fit in cache.

**Clock advancement cost**: in Kafka's hybrid design, a `DelayQueue` of bucket-timestamps drives clock advancement — meaning Kafka still has one O(log n) operation per _bucket expiry_, not per timer. The trade-off is acceptable because the number of distinct bucket timestamps is bounded by the wheel size (a small constant), not by the number of active requests.

**Not suitable for absolute wall-clock timers**: timing wheels track ticks from insertion. If the process sleeps or pauses (GC stop-the-world), the wheel "loses" ticks and must be re-synced. For precise wall-clock scheduling, you still need epoch-based absolute timers.

**Overflow handling varies by design**: Scheme 4 (simple wheel) cannot handle delays beyond W ticks at all. Scheme 6 (hashed wheel) handles them but degrades to O(k) on crowded slots. Scheme 7 (hierarchical) amortizes the cost but cascades must be carefully bounded.

## Why it works

The deep principle: **convert a comparison-based ordering problem into a direct-address lookup by exploiting the structure of time**.

A min-heap works for arbitrary comparable keys because it makes no assumptions about the key space. Timer expiry times, however, _do_ have structure: they are non-negative integers bounded by `now + max_delay`. This lets us use the expiry value directly as an array index — the same insight that makes counting sort O(n) instead of O(n log n), and that makes hash tables O(1) instead of O(log n) for integer keys.

This is actually a specific instance of the **index-versus-comparison trade-off** that appears throughout CS:

| Problem | Comparison-based | Direct-address |
|---------|-----------------|----------------|
| Sorting integers | O(n log n) comparison sort | O(n) counting/radix sort |
| Set membership | O(log n) BST | O(1) hash table / bit array |
| Timer management | O(log n) min-heap | O(1) timing wheel |
| Page table lookup | O(depth) trie | O(1) direct-mapped TLB |

In each case, the speedup comes from knowing something specific about the key space that a general comparison-based structure cannot exploit.

The hierarchical trick then answers: _what if the range is too large for a single array?_ The answer is multi-level decomposition — exactly the same idea as:

- **Radix sort**: sort by least-significant digit first, then next, avoiding O(n log n) altogether
- **Multi-level page tables**: a 4-level PT covers 48-bit virtual addresses with 4×512 entries instead of 2^48
- **DNS hierarchy**: root → TLD → domain → host, each level reducing the fanout

All three are instances of the same principle: represent a large sparse domain as a hierarchy of smaller dense arrays, paying cascade cost at boundary crossings.

The Kafka purgatory use case reveals a subtler point: **for most practical workloads, request timeouts are not the common case**. Most requests complete via the event path (replication ack, data arrival) long before their timeout. The timing wheel's O(1) insert/remove makes early removal cheap, while the heap's O(log n) would penalize even the successful path. The data structure is chosen not just for its asymptotic behavior at expiry but for the cost of _cancellation_ — a detail that heap-based designs often overlook.

## Going deeper

1. **Original paper**: Varghese & Lauck, "Hashed and Hierarchical Timing Wheels: Data Structures for the Efficient Implementation of a Timer Facility," SOSP 1987. The seven schemes they enumerate remain the canonical taxonomy of timer implementations. PDF at cs.columbia.edu/~nahum/w6998/papers/sosp87-timing-wheels.pdf.

2. **Linux kernel timer wheel**: `kernel/time/timer.c` in the Linux source. Since 4.8, Linux uses a five-level hierarchical wheel (`TVR`/`TVN` arrays) that covers 2^49 ns (~6 days) with 1 ns resolution. Reading the `add_timer` and `__run_timers` functions side by side with the paper makes the cascade logic concrete.

3. **Netty's `HashedWheelTimer`**: `io.netty.util.HashedWheelTimer` is a production-grade Java implementation used in Cassandra, Finagle, and RxJava. The source is well-commented and shows how to handle GC pauses (clock drift recovery), worker thread lifecycle, and the hybrid approach of tracking only bucket-level timestamps in an auxiliary queue.
