---
title: "End-to-End Arguments in System Design"
source: https://dl.acm.org/doi/10.1145/357401.357402
author: J.H. Saltzer, D.P. Reed, D.D. Clark
company: MIT
date_posted: 1984-11-01
date_digested: 2026-07-31
---

# End-to-End Arguments in System Design

## What's new to learn

- **The end-to-end argument**: A function that requires application-level knowledge to be correct cannot be correctly and completely implemented in a communication subsystem — only incompletely. Correctness always requires endpoint participation.
- **Infrastructure reliability as performance optimization**: Lower-level reliability mechanisms (network checksums, retransmission, deduplication) never provide correctness guarantees; they provide performance benefits by reducing the cost of error recovery at the endpoints.
- **Where to place functions in a layered system**: The e2e principle gives a precise decision criterion — if a function can be validated only by the communicating applications, it belongs there; if placing it lower reduces cost *and* the endpoint check remains in place, moving it lower is a performance optimization, not an architectural shift.

## Prerequisites

- Layered architecture basics: understand that TCP/IP splits responsibilities across network, transport, and application layers
- TCP reliability mechanisms: checksums, sequence numbers, retransmission on loss — what they promise and to whom they promise it
- The difference between a subsystem reporting "done" and an application's goal actually being achieved

## The core idea

Imagine you want to transfer a file `F` from host A to host B, and you need to know it arrived identically. Can the communication system guarantee this?

Walk through every place corruption can occur:

1. **A reads F from disk** — a bug or transient hardware glitch could silently produce wrong bytes.
2. **A's OS copies the data into a send buffer** — a kernel bug, DMA error, or stray write corrupts it in memory.
3. **The network interface card checksums and transmits** — almost all bit errors are caught, but the checksum is only 16 or 32 bits; a 1-in-2³² chance of a missed error is not zero. Also, the CRC is verified and *stripped* at the receiving NIC — if the receiving hardware has a bug, it might pass corrupt data upstream.
4. **The transport (TCP) receives and acknowledges** — the ACK means "the bytes landed in the OS receive buffer." It says nothing about what happened next.
5. **B's OS copies data to the application** — another memory path, another DMA channel, another chance.
6. **B writes to disk** — disk controllers have their own buffering; a power failure between a filesystem "write" returning and the physical medium being updated leaves a corrupted file.

TCP's reliability mechanisms cover step 3 almost completely. They cover nothing in steps 1, 2, 4, 5, or 6.

**The conclusion is stark**: the only way to guarantee the file arrived correctly is for the *application* to compute a checksum over the entire file at A, transmit it end-to-end as a separate entity, and have B verify it *after* writing to disk and reading back — a full round-trip involving the disk itself.

This is the end-to-end argument: the file transfer function can only be implemented correctly by the endpoints. No matter what guarantees the communication system provides, the endpoints must still do the check.

Saltzer, Reed, and Clark state the principle this way:

> "The function in question can completely and correctly be implemented only with the knowledge and help of the application standing at the end points of the communication system. Therefore, providing that questioned function as a feature of the communication system itself is not possible. Sometimes an incomplete version of the function provided by the communication system may be useful as a performance enhancement."

The last sentence is the part engineers most often miss.

## Mechanics

The paper works through six distinct function categories and applies the same logic to each:

### 1. Reliable data delivery

TCP delivers bytes to the OS reliably. But "reliable delivery to the OS" is not the same as "the application successfully processed the data." A message broker can acknowledge "delivered to subscriber" when the subscriber's OS received the bytes — but the subscriber might crash before writing to its database. Application-level acknowledgment (the subscriber's own ACK after committing) is the only correct endpoint check.

### 2. Encryption for secrecy

If A wants to keep data secret from an eavesdropper on the network, network-layer encryption (IPsec, MPLS VPN) works. But if A wants to keep data secret from a *compromised host* on the path (including B's OS!), network-layer encryption is worthless — the ciphertext is decrypted before it reaches the application. End-to-end encryption (TLS, Signal's double-ratchet, PGP) is the only mechanism that covers the right scope. The paper makes this concrete: if the threat model includes the operating system of the destination host, the encryption must live above the OS.

### 3. Duplicate message suppression

A network can deduplicate packets within a session. But what about duplicates that arise from a host restart? If A sends message M, B processes it, and B then crashes and restarts — when A retransmits because it never got the ACK, B has lost its deduplication state. Only the application, using a persistent idempotency key stored in durable storage, can correctly suppress the duplicate. This is exactly the mechanism behind Stripe's idempotency keys and Kafka's producer sequence numbers.

### 4. FIFO message ordering

A queue can guarantee FIFO order of delivery. But if B processes messages concurrently across threads, the application is re-introducing non-FIFO execution order. The network's ordering guarantee is immediately violated by the application's own parallelism. Ordering that matters must be enforced at the point where it matters: in the application's processing logic.

### 5. Transaction management

A communication system can guarantee atomicity for its own operations (e.g., exactly-once delivery of a single message). But atomicity across a debit on A's ledger and a credit on B's ledger requires coordination at the *application* level, because only the application knows that these two operations are semantically part of the same transaction. Two-phase commit lives at the application or middleware level, not in the transport.

### 6. The performance exception

Here is the subtlety the paper is most careful about: lower-level reliability mechanisms are NOT useless — they are valuable **performance optimizations**.

Network-layer checksums catch the vast majority of bit errors before they reach the application. This means the application's end-to-end checksum rarely fails. When it fails, the cost of retransmitting an entire file transfer is far higher than retransmitting a single packet. So the network checksum is reducing the *expected cost* of the endpoint check.

The decision framework the paper offers:

- **Is the function implementable correctly only with application knowledge?** → It must be at the endpoints.
- **Can a partial version at a lower level reduce the frequency or cost of endpoint recovery?** → It may be worth adding, but only if the cost of the lower-level mechanism is less than the expected savings at the endpoint.
- **Does the lower-level mechanism eliminate the need for the endpoint check?** → **Never.**

## Where it breaks

**Misuse 1: Stripping all intermediate-layer protections.** Reading the principle as "endpoints handle everything, so skip network checksums" is wrong. Removing them increases the error rate reaching the endpoint check, which increases retransmission cost. The principle says lower-layer functions don't provide *correctness*, not that they provide no value.

**Misuse 2: Defining the wrong endpoint.** In a microservices system, every service boundary is potentially an endpoint pair. If service A calls service B which calls service C, the "correct endpoint" for a business transaction might be A and C, but intermediate services are doing their own endpoint checks for sub-operations. Getting this wrong leads to double-checking at many levels (expensive) or under-checking at key levels (incorrect). The principle doesn't tell you where to draw the application boundary — that's a design decision. It only says: wherever you draw it, the endpoint check is mandatory.

**Misuse 3: Exactly-once delivery as a correctness substitute.** Kafka, SQS, and most message queues now offer at-least-once with idempotency-key-based deduplication — effectively exactly-once. This is not the same as application-level exactly-once. The message arrived exactly once at the consumer's OS. Whether the consumer's *processing* happened exactly once depends on whether the consumer atomically commits its work and its acknowledgment together — which is the application's responsibility (the transactional outbox pattern, two-phase commit, or idempotent processing logic).

**Where the principle doesn't apply cleanly:** Pure infrastructure-to-infrastructure optimizations, where both "endpoints" are owned by the same team with full observability, sometimes allow the lower-layer guarantee to substitute for the endpoint check — if the lower-layer implementation is provably correct within scope and the scope covers all failure modes. This is rare in practice but exists (e.g., hardware RAID with verified end-to-end integrity at the block device level, where the OS is the application).

## Why it works

The deeper principle is **information asymmetry**: an intermediate layer cannot know the semantic meaning of the data it carries or the success condition for the operation.

A TCP stack knows "I delivered 1,024 bytes to the OS." It does not know:
- Whether those bytes represent a completed financial transaction or half of one
- Whether the application processed them correctly
- Whether the application will survive to commit the result
- What "success" means to the requesting party

This is structurally identical to the problem that makes distributed two-phase commit expensive: the coordinator knows "all participants said ready" but cannot know "all participants actually committed durably" without asking each participant to commit and acknowledge. The intermediate layer always lacks the application-level information needed to provide end-to-end guarantees.

This same asymmetry appears everywhere:

| Domain | What the middle layer guarantees | What the endpoint must verify |
|--------|----------------------------------|-------------------------------|
| TCP | bytes delivered to OS | application processed them correctly |
| HTTPS/TLS | channel confidentiality from wire tapping | data is not malicious or malformed |
| Message queue exactly-once | message arrived exactly once | processing happened exactly once |
| Database replication | replicated bytes are consistent | application-level invariants hold |
| Kubernetes liveness probe | pod responds to HTTP | application is doing correct work |
| AWS SLA 99.9% uptime | service is available | your specific request succeeded |

The generalizable insight: **"X layer says OK" is always a statement about X's correctness condition, not yours.** Every engineer who has debugged a "message delivered but not processed" bug, a "wrote to disk but the file is corrupt" bug, or a "payment acknowledged but not credited" bug has rediscovered this paper empirically.

The paper also clarifies why the Internet Protocol was designed the way it was. IP is deliberately unreliable (no guaranteed delivery, no ordering, no duplication prevention). TCP adds reliability on top for applications that want it. But the designers knew TCP's reliability still would not be sufficient for applications with strong correctness requirements — those applications (file transfer tools, databases, payment processors) must add their own endpoint checks. IP's "dumb pipe" design was not a limitation; it was the correct layering decision.

## Going deeper

1. **The original paper**: Saltzer, Reed, Clark. "End-to-End Arguments in System Design." *ACM Transactions on Computer Systems* 2(4), November 1984. The paper is 10 pages and unusually readable for its era — the file transfer argument occupies only 3 pages and is self-contained.

2. **"Life Beyond Distributed Transactions: an Apostate's Opinion"** — Pat Helland, CIDR 2007. Helland spent 20 years building Microsoft's Distributed Transaction Coordinator, then wrote this paper arguing that at large scale, the "application level" for transactions is necessarily the entity level — and that cross-entity operations must be activities (long-running workflows with compensation), not atomic transactions. This is the end-to-end argument applied to transaction management at scale.

3. **"Designing Data-Intensive Applications"** — Martin Kleppmann, Ch. 8 ("The Trouble with Distributed Systems") and Ch. 12 ("The Future of Data Systems"). Kleppmann covers end-to-end integrity in the context of exactly-once processing, the end-to-end integrity argument for application-level checksums even when the database has its own, and why "the database will handle it" is always an incomplete answer.
