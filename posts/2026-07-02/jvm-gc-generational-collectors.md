---
title: "A deep dive into Java garbage collectors"
source: https://www.datadoghq.com/blog/understanding-java-gc/
author: Jean-Philippe Bempel, Scott Gerring, Nicholas Thomson, Will Roper
company: Datadog
date_posted: 2025-10-17
date_digested: 2026-07-02
---

# A deep dive into Java garbage collectors

## What's new to learn

- **Generational hypothesis as an architectural driver** — "most objects die young" isn't just a trivia fact; it's the load-bearing assumption that lets young-gen collections be frequent and cheap because a copying collector only touches *live* objects, and there are almost none of them in Eden.
- **Write barriers vs read barriers** — These are two different "interception points" in time. Write barriers fire when a pointer is *stored* and maintain cross-generation bookkeeping (remembered sets). Read barriers fire when a pointer is *loaded* and can transparently fix up references to objects that moved while the application was running.
- **Concurrent compaction via colored pointers** — ZGC embeds GC metadata directly in unused high bits of 64-bit pointers, turning every pointer load into an atomic color-check that can detect and forward stale references without any STW pause — making compaction, the last remaining stop-the-world operation in prior collectors, fully concurrent.

## Prerequisites

- How the JVM heap is divided into regions and how objects are allocated (bump-pointer)
- What a "stop-the-world" (STW) pause means: the JVM pauses *all* application threads so the GC can safely inspect and modify the heap
- Basic understanding of what "garbage" means: objects with no live references from the root set (stack frames, static fields)
- Familiarity with the concept of a "pointer" or "reference" in memory

## The core idea

Every garbage collector is a trade-off between three competing forces: **pause time**, **throughput**, and **implementation complexity**. The history of JVM GC is a progression from "stop everything, scan and compact the whole heap, resume" (simple but multi-second pauses) toward "never stop, but intercept certain memory operations to stay correct" (sub-millisecond pauses at the cost of every pointer load or store carrying extra logic).

The key architectural insight is that each generation of collector adds a **barrier** — a small piece of code the JVM inserts at pointer reads or writes — at a *different point in the object lifecycle*:

1. **Write barriers** (used by generational collectors including G1GC): fire when the application *stores* a pointer. This maintains a remembered set so the collector knows which old-generation regions might hold references into the young generation, avoiding a full-heap scan on every minor collection.

2. **Read barriers** (used by Shenandoah and ZGC): fire when the application *loads* a pointer. This allows the collector to concurrently *move* objects to new locations while the application runs, because any thread that loads a stale reference immediately gets the forwarded address.

Moving the barrier from write-time to read-time is what finally breaks the last stop-the-world phase: compaction. G1GC still needs to stop all threads to evacuate objects from a region (because it has no way to safely update all in-flight stale pointers). ZGC inserts the forwarding check at every load, so compaction can proceed concurrently — the application just transparently follows forwarding pointers until the collector finishes updating them.

## Mechanics

### Young generation: copying collection with TLABs

Each application thread has a **Thread-Local Allocation Buffer (TLAB)**: a private chunk of Eden that it bump-allocates into with a simple pointer increment and no lock. When a TLAB fills, the thread requests a new one from the GC. This makes allocation itself nearly free.

A **minor GC** copies all *live* objects out of Eden and the current Survivor space into the other Survivor space (or promotes them to old-gen if they're old enough). Because most objects are dead immediately, the work done is proportional to the small number of survivors, not the large amount of garbage. The live set is found by starting from GC roots plus the **remembered set** (see below) and tracing references.

### Write barriers and remembered sets

Old-generation objects can reference young-generation objects. If a minor GC only traces from GC roots, it would miss objects kept alive only by an old-gen reference and incorrectly collect them. Scanning the entire old generation on every minor GC would defeat its purpose.

The solution: a **write barrier** inserted at every pointer store. When code does `obj.field = ref`, the barrier checks whether `obj` is in the old generation and `ref` is in the young generation. If so, it records `obj`'s region in the **remembered set** — a metadata structure that maps old-gen regions to the young-gen objects they reference. Minor GC then scans only the remembered set regions rather than all of old-gen, finding all cross-generational references in O(updated pointers) time rather than O(heap size).

### G1GC: region-based heap with concurrent marking

G1 ("Garbage-First") divides the heap into ~2048 equal-sized regions (typically 1–32 MB each). Each region is independently tagged as Eden, Survivor, Old, or Humongous (for objects larger than half a region). This removes the hard boundary between young and old gen: G1 can tune the collection set dynamically.

G1 prioritizes regions by **garbage density** — the ratio of dead objects to total live objects. By collecting the highest-density regions first (hence the name), it achieves the most bytes reclaimed per unit of STW pause. The old-gen marking phase is concurrent: G1 scans live references in the background while the application runs. However, the **evacuation** phase — actually moving live objects out of the chosen regions and compacting them — is still stop-the-world. This is the fundamental limitation: G1 cannot safely move an object while another thread might have a stale pointer to the old location.

Practical implication: G1 targets 250 ms default pause and can be tuned to ~50 ms reliably. Below that, it becomes hard to avoid evacuation pauses.

### Shenandoah: concurrent evacuation with indirection pointers

Shenandoah takes a different approach to concurrent compaction: every object gets a **Brooks indirection pointer** — a header word that normally points to the object itself but, during evacuation, is updated to point to the new copy. Any barrier-instrumented load dereferences the header first, transparently following the forwarding chain if evacuation is in progress. GC threads and application threads can both safely update this indirection pointer using a CAS (compare-and-swap).

Because every object access goes through the indirection pointer, Shenandoah achieves concurrent evacuation at the cost of one extra memory dereference per object access. Pause times under 10 ms are realistic.

### ZGC: colored pointers and load barriers

ZGC takes the most radical approach. Modern 64-bit CPUs use only 48 bits of the virtual address space, leaving the upper 16 bits unused. ZGC uses **four of these bits** as color metadata embedded directly in each pointer:

- **Marked0 / Marked1**: alternating bits that indicate which GC cycle marked this object as live (the bit alternates each cycle to avoid a clearing pass)
- **Remapped**: set once the pointer has been updated to the object's current location
- **Finalizable**: set when the object is only reachable via a finalizer

Every pointer load goes through a **load barrier** that checks these bits and takes fast-path or slow-path action:

```
ref = load(addr)
if (ref.color != expected_color):
    ref = slow_path(ref)   // fix up: relocate, remap, or mark
store_back(addr, ref)       // heal the pointer in place
```

The slow path relocates the object if needed and updates the pointer in the heap ("self-healing"). Once the GC finishes a relocation phase, all pointers in the heap are gradually healed as the application accesses them — no single STW phase is needed to update them all.

This design enables ZGC to perform **all GC work concurrently** — marking, relocation selection, object movement, and pointer remapping — with STW pauses under 1 ms even on multi-terabyte heaps. The cost is a ~1–5% throughput overhead from the load barrier on every pointer dereference.

JDK 21 introduced **Generational ZGC** (default in JDK 23): ZGC now splits the heap into young and old generations, combining colored-pointer concurrent compaction with the generational hypothesis. This reduces the write/read barrier frequency by collecting the high-turnover young generation more often at lower cost.

## Where it breaks

**Generational hypothesis violations**: If an application has a large long-lived working set (ML model weights, caches, analytics buffers), most objects survive every minor GC. Young-gen collections become expensive because the survivor set is large, and old-gen fills quickly, triggering expensive major collections. Batch and ML workloads often need explicit tuning or non-generational collectors.

**Floating garbage**: Because concurrent marking traces the heap while the application mutates it, some objects marked live at the start of a cycle may become garbage before the cycle finishes. These "floating" objects survive until the next collection. Under high allocation pressure, floating garbage can accumulate faster than collections complete, triggering a fallback stop-the-world full GC.

**Load barrier overhead**: ZGC's read barrier fires on every pointer dereference. Pointer-heavy workloads (deep object graphs, many small objects) pay a larger throughput penalty than array-heavy workloads.

**ZGC requires 64-bit**: Colored pointers steal bits from the virtual address. ZGC does not run on 32-bit JVMs.

**Humongous allocations**: In G1, objects larger than half a region bypass the young generation entirely and go directly to old-gen, skipping the generational filtering that makes minor GCs cheap. Applications with large short-lived arrays (e.g., per-request serialization buffers) should tune region size.

## Why it works

The STW → concurrent-marking → concurrent-compaction progression is a specific instance of a general systems pattern: **replace global coordination with per-access optimistic checks**.

- **Serial/Parallel GC** uses global coordination (STW) to get a consistent heap view. Simple, but O(heap) pause.
- **G1GC** replaces old-gen scanning with write-barrier bookkeeping (coordination at write time), concurrent marking (optimistic background scan), but falls back to STW for compaction.
- **ZGC** replaces compaction STW with read-barrier forwarding (coordination at load time), making the pause truly bounded.

This is exactly the same trade-off as MVCC in databases: instead of acquiring a lock to serialize access to a row (global coordination), a reader follows a version chain at read time to find the snapshot it should see (per-access check). The "load barrier follows a forwarding pointer" and the "MVCC reader follows a version chain" are structurally identical: both intercept pointer/reference resolution at access time to transparently redirect to the current authoritative location.

Similarly, Linux RCU (Read-Copy-Update) uses the same pattern: readers take no locks but load through a mechanism that guarantees they see a consistent version of a data structure, while writers atomically install a new copy. RCU's "grace period" is ZGC's "remapping phase" — both are waiting for all in-flight reads that could see the old copy to complete before reclaiming it.

The deeper principle: **the later in the access chain you place the interception, the more you can do concurrently, but the more overhead every access carries**. Write barriers are cheaper per-access but can't handle compaction. Read barriers are more expensive per-access but make the full operation concurrent. Choosing where to intercept is choosing where to pay.

## Going deeper

1. **"The Garbage Collection Handbook" — Jones, Hosking, Moss (2011, updated 2023)**: The definitive reference for GC algorithms. Chapters 6–9 cover copying, generational, incremental, and concurrent GC in full algorithmic detail.

2. **Azul Systems — C4: The Continuously Concurrent Compacting Collector (2011)**: The academic paper behind Azul's Zing JVM that first demonstrated fully-concurrent compaction at scale. ZGC builds on many of the same ideas. Available at azul.com/resources.

3. **OpenJDK ZGC source and wiki** (`openjdk.org/projects/zgc`): The implementation notes explain exactly how colored bits map to virtual addresses, why the self-healing barrier design was chosen over a global remapping pass, and how the generational extension changed the barrier design in JDK 21.
