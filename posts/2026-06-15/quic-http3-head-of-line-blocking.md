---
title: "Head-of-Line Blocking in QUIC and HTTP/3: The Details"
source: https://calendar.perfplanet.com/2020/head-of-line-blocking-in-quic-and-http-3-the-details/
author: Robin Marx
company: Hasselt University
date_posted: 2020-12-10
date_digested: 2026-06-15
---

# Head-of-Line Blocking in QUIC and HTTP/3: The Details

## What's new to learn

1. **TCP-level head-of-line (HoL) blocking**: when HTTP/2 multiplexes streams over one TCP connection, a single lost packet stalls delivery of *all* streams — because TCP guarantees byte-stream ordering for the entire connection, with no concept of per-stream independence.

2. **QUIC's per-stream loss recovery**: QUIC re-implements reliability and ordering inside the transport layer, but only within each stream. A lost packet causes only the streams whose data was in that packet to wait; every other stream delivers immediately.

3. **Connection IDs instead of 4-tuples**: QUIC identifies connections by a random 64-bit token rather than the (src IP, src port, dst IP, dst port) 4-tuple, enabling seamless migration when a client's IP changes (e.g., WiFi → LTE handoff).

## Prerequisites

- Basic understanding of TCP: 3-way handshake, sequence numbers, in-order delivery guarantee, retransmission on loss.
- HTTP/1.1 and HTTP/2: why HTTP/2 multiplexes requests — to avoid the connection-per-request overhead of HTTP/1.1.
- TLS handshake: that TLS adds at least one extra round trip on top of TCP's handshake.
- Non-obvious: what "head-of-line blocking" means in the context of a shared queue (not just HTTP).

## The core idea

**HTTP/2 made one TCP connection carry many logical streams.** This saved the overhead of opening 20+ TCP connections to load a page, and eliminated the HTTP/1.1 rule that responses had to arrive in the order requests were sent. But it introduced a new bottleneck: TCP sees the entire multiplexed payload as a single ordered byte stream. TCP doesn't know that bytes 1–500 belong to a CSS file while bytes 501–900 belong to a JavaScript file. If packet 2 (carrying bytes 201–400) is lost and retransmitted, TCP holds everything in the receive buffer until that gap is filled — even if all the bytes for the JavaScript file have already arrived. *From the application's view, all streams freeze until the missing bytes of one stream arrive.*

**QUIC is a new transport protocol built on UDP** that moves stream awareness from the application layer down into the transport layer itself. Each QUIC stream has its own offset space, its own delivery queue, and its own acknowledgment tracking. A lost packet only delays the stream(s) that had data in that packet. Streams whose data arrived unharmed are handed to the application immediately.

The punch line: HTTP/2 solved HTTP-level HoL blocking (waiting for a previous *response* before sending the next request), but accidentally introduced transport-level HoL blocking. QUIC solves transport-level HoL blocking by making the transport layer stream-aware.

## Mechanics

### TCP's receive buffer — the real culprit

TCP maintains one receive buffer per connection, indexed by sequence number. When bytes arrive out of order, TCP holds them until all earlier bytes are present. The kernel delivers a contiguous prefix to the application — it never delivers a later byte before an earlier one, regardless of which stream they belong to. The application (HTTP/2 framing layer) can only parse frames once TCP hands bytes over, so a gap in TCP sequence space = silence across all HTTP/2 streams.

### QUIC packet structure

A QUIC packet contains:
- A **packet number** (monotonically increasing, even on retransmit — QUIC never reuses a packet number)
- One or more **QUIC frames**, each tagged with a stream ID and a stream-level **offset**

When packet 17 is lost and retransmitted as packet 42, the stream data inside carries the same stream-level offset (e.g., stream 5, bytes 400–799). The receiver's stream-5 buffer has a gap at offset 400–799; it waits. The receiver's stream-7 buffer may already be complete; it delivers immediately.

The critical invariant: **QUIC acknowledgments are per-packet; QUIC delivery is per-stream-offset.** Loss detection operates on packet numbers (and ACK ranges), but application delivery operates on stream offsets independently per stream.

### QUIC handshake: 1-RTT and 0-RTT

TCP + TLS 1.2 costs 3 RTTs before the first application byte: TCP SYN/SYN-ACK (1 RTT) + TLS Client Hello/Server Hello/Certificate/etc. (2 RTTs). TLS 1.3 over TCP cuts to 2 RTTs (1 TCP + 1 TLS).

QUIC folds the transport handshake and TLS 1.3 handshake into a single 1-RTT exchange. The client's first UDP packet contains both the QUIC "INITIAL" packet (replacing TCP SYN) and the TLS ClientHello. The server's response includes the TLS ServerHello, certificate, and Finished. The client can send application data after this one round trip.

**0-RTT resumption**: On repeat connections, the client stores a *session ticket* from the previous session (a blob containing a pre-shared key). It can send HTTP requests in the very first UDP datagram — before the server responds — by encrypting them with the session key. This costs 0 round trips for latency (though the server still has to process the request). The tradeoff: 0-RTT data is *replay-vulnerable*. An attacker who intercepts the packet can replay it. So only idempotent operations (GETs, essentially) should be sent 0-RTT.

### Connection IDs and migration

TCP connections are bound to a (src IP, src port, dst IP, dst port) 4-tuple. If a mobile device changes IP (roaming from WiFi to LTE), every TCP connection dies. TLS session resumption can restart connections faster, but there's still a handshake.

QUIC assigns each connection a random 64-bit **Connection ID** chosen by the client. Routing infrastructure (load balancers, middleboxes) use this ID to direct packets. When the client's IP changes, it sends packets with the same Connection ID from its new IP. QUIC's path validation (`PATH_CHALLENGE` / `PATH_RESPONSE` frames) confirms reachability on the new path, then the connection continues without renegotiation.

### Remaining HoL blocking in QUIC

Marx's post is careful to note that QUIC doesn't eliminate *all* forms of HoL blocking:

- **Within a single stream**: QUIC still delivers in-order within each stream. If you send 10 resources over a single HTTP/3 stream (you shouldn't, but if you did), you'd have stream-level HoL blocking.
- **QPACK header compression**: HTTP/3 uses QPACK (the successor to HPACK) for header compression. QPACK uses two special unidirectional streams for encoder/decoder instructions. If those streams are delayed, dynamic header table updates are blocked — but critically, QPACK is designed so that this doesn't block HTTP/3 data streams (you can send headers without dynamic table references if needed). HTTP/2's HPACK *did* block: all headers shared one dynamic table accessed in stream-creation order.
- **Application-level prioritization**: Even if transport delivers data, the browser still has to schedule which bytes to use first. Poor HTTP/3 prioritization can re-introduce stalls. This is a separate problem from HoL blocking.

## Where it breaks

- **0-RTT replay attacks**: The 0-RTT session resumption is vulnerable to replay — an on-path attacker can re-send the first UDP datagram to a different server. Servers must protect non-idempotent operations behind the TLS handshake, not 0-RTT. In practice, most browsers only send GET requests 0-RTT.
- **Middlebox ossification**: Many enterprise firewalls, NATs, and load balancers were built assuming TCP's wire format. QUIC over UDP often gets blocked or throttled in corporate networks. Google's original QUIC deployments used port 443 to avoid this, but middlebox friction remains a deployment reality.
- **UDP receive performance at scale**: Linux's TCP receive path is highly optimized (GRO, TSO, sendfile). UDP paths lack many of these offloads. At extremely high throughputs (>100 Gbps), the per-packet cost of QUIC's user-space protocol processing can exceed TCP's kernel-space path. Projects like `quic-go`, Cloudflare's `quiche`, and Facebook's `mvfst` do extensive CPU optimization to close this gap.
- **Loss recovery latency**: With TCP HoL blocking, *all* streams stall on loss, but they all stall together and recover together. With QUIC, each stream has independent loss detection and recovery. Under high loss rates, many streams could be simultaneously in individual recovery cycles, each adding latency to its own data. The aggregate effect can sometimes exceed what you'd see with TCP's bulk retransmit.
- **0-RTT limits don't generalize to stateful APIs**: Because 0-RTT is replay-vulnerable, any API that creates resources, transfers money, or mutates state cannot safely use 0-RTT. This limits the latency win to read-heavy workloads.

## Why it works

The root cause of TCP-level HoL blocking is a **layer boundary mismatch**: HTTP/2 streams are an *application* concept, but they were multiplexed over TCP, which is a *transport* that has no concept of stream boundaries. TCP's job is to deliver a byte stream reliably and in order — and it does exactly that, to the entire connection, without any notion of "this byte matters for a different resource than that byte."

This is a specific instance of a general systems principle: **when you put independent work units into a serialized queue without letting the queue know they're independent, you create spurious coupling**. TCP's byte-stream abstraction is a serialized queue. HTTP/2 poked holes in it with framing, but the transport still treated everything as one ordered sequence.

QUIC's design choice — put streams into the transport layer — is the same insight behind OS thread scheduling vs. cooperative multitasking, or database lock-free MVCC vs. global row locks: by making the lower layer aware of the unit of independence, you can schedule and deliver work without one unit blocking another.

The "X is just an instance of Y" framing: **QUIC's stream multiplexing is TCP redesigned with the correct granularity of ordering guarantees**. TCP orders bytes globally; QUIC orders bytes per-stream. The same way a database ordering writes globally (one WAL writer at a time) vs. per-row (MVCC) gets a throughput gain by exploiting row-level independence, QUIC gets a latency gain by exploiting stream-level independence.

Connection IDs are an instance of the same principle: **name your entities at the right level of abstraction**. TCP named connections by their network path (4-tuple), so path changes broke connections. QUIC names connections by intent (random ID), so path changes are transparent. This is the same lesson as using UUIDs instead of auto-increment keys: stable identity survives location changes.

## Going deeper

1. **IETF RFC 9000 — QUIC: A UDP-Based Multiplexed and Secure Transport** (Iyengar & Thomson, 2021): the normative spec, readable as a design document, with clear rationale for each design decision. Pay special attention to Section 3 (streams) and Section 12 (packet numbers).

2. **"QUIC is not enough" / QUIC performance analyses**: Lychev et al., "How Secure and Quick is QUIC?" (IEEE S&P 2015) and the Brown University production measurement paper "Dissecting Performance of Production QUIC" (WWW 2021) show where the theory meets messy real-world middleboxes and loss rates.

3. **Cloudflare's QUIC & HTTP/3 engineering series** (blog.cloudflare.com): Cloudflare has published detailed posts on their `quiche` implementation (written in Rust), QPACK header compression internals, and how they route QUIC at edge scale — a practical complement to the protocol-level theory here.
