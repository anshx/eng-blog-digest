---
title: "How NAT Traversal Works"
source: https://tailscale.com/blog/how-nat-traversal-works
author: David Anderson
company: Tailscale
date_posted: 2020-08-21
date_digested: 2026-07-17
---

# How NAT Traversal Works

## What's new to learn

1. **NAT mapping types (EIM vs. EDM, per RFC 4787)** — Whether hole punching can work at all depends on a single property of each NAT device: does it assign the *same* external port to all packets from a given internal source (Endpoint-Independent Mapping), or a *different* port per destination (Endpoint-Dependent Mapping)? This taxonomy is the key to predicting which traversal technique to use.

2. **UDP hole punching as a simultaneous race** — By having both peers fire packets at each other at exactly the same time, each peer's outgoing packet causes its own NAT to create a state entry. When the peer's packet arrives a moment later, it matches that entry and passes through. Neither peer "initiated" a connection to the other, but both tricked their own NAT into believing they did.

3. **DERP's "relay first, upgrade later" pattern** — Starting every connection through an encrypted relay (DERP, running over HTTPS) gives millisecond-latency start while hole-punching runs in the background. Once a direct path is established, traffic silently upgrades. This separates *correctness* (always connected) from *performance* (faster when possible).

## Prerequisites

- What a NAT device does in broad strokes: it lets multiple machines share one public IP by rewriting packet source addresses on the way out and maintaining a table to route replies back.
- The difference between UDP and TCP: UDP is connectionless (no handshake), which makes it both easier to hole-punch and easier for NATs to drop.
- Familiarity with firewalls as stateful devices that track "sessions" — an outgoing packet creates state, an unsolicited incoming packet matches nothing and is dropped.

## The core idea

IPv4's 4 billion addresses are far too few for the billions of internet-connected devices. NAT solves this by letting many private-address machines share one public IP. It works beautifully for the client-server model: you initiate a connection outbound, the NAT rewrites your source address, remembers the mapping, and routes the server's replies back to you.

Peer-to-peer breaks this. If both Alice and Bob are behind NATs, neither can receive an unsolicited inbound packet from the other — their NATs will drop it as unmatched. Yet for VPNs, video calls, gaming, and file sync, you need direct peer connections.

The insight at the heart of NAT traversal: **a NAT creates state when *you* send a packet out; that state then permits matching inbound packets**. If Alice sends to Bob's address and Bob simultaneously sends to Alice's address, each sender's NAT creates an entry for that flow. When the other's packet arrives, it matches the entry and is allowed through. Neither side "connected to" the other — but both sides created state for the other's packets at the same moment.

Everything else in NAT traversal — STUN, ICE, DERP, birthday-paradox probing — is bookkeeping around this one central trick.

## Mechanics

### Step 1: Discover your public address via STUN

A peer behind NAT doesn't know its own public IP:port — the NAT assigned it. STUN (Session Traversal Utilities for NAT, RFC 5389) solves this: you send a UDP packet to a well-known STUN server on the internet. The STUN server reads the *source* IP:port in the packet header (which is the NAT's external address, not your LAN address) and sends it back in the reply body. Now you know your public address.

You share this address with peers via an out-of-band signaling channel (Tailscale's control plane, a WebRTC signaling server, etc.).

### Step 2: Classify the NAT

Not all NATs behave the same way. RFC 4787 defines the critical distinction:

**Endpoint-Independent Mapping (EIM):** For all packets from a given internal IP:port, the NAT assigns the *same* external port regardless of destination. The address you discover via STUN is the same address your peers should use. Most home routers implement EIM.

**Endpoint-Dependent Mapping (EDM, "Symmetric NAT"):** The NAT assigns a *different* external port for each destination. The port you discover by talking to the STUN server is only valid for talking to that STUN server — talking to a peer creates a new, unpredictable port. Many corporate gateway firewalls and carrier-grade NATs (CGNAT) use EDM.

This classification is binary but crucial: EIM lets you use STUN-discovered addresses directly; EDM requires a different approach.

### Step 3: Simultaneous hole-punching (for EIM NATs)

Both peers, through the signaling channel, learn each other's STUN-discovered public address. A coordinator (Tailscale's control server) signals both peers to begin at the same time. Each peer sends a UDP packet to the other's discovered address.

The sequence for Alice (behind NAT-A) and Bob (behind NAT-B):

```
Alice sends UDP → 203.0.113.55:4242 (Bob's external address)
  └─ NAT-A creates entry: {int: 10.0.0.2:50001, ext: 198.51.100.1:55123, dst: 203.0.113.55:4242}

Bob sends UDP → 198.51.100.1:55123 (Alice's external address)
  └─ NAT-B creates entry: {int: 10.0.0.3:60002, ext: 203.0.113.55:4242, dst: 198.51.100.1:55123}

Alice's packet arrives at NAT-B → matches Bob's entry → delivered to Bob.
Bob's packet arrives at NAT-A  → matches Alice's entry → delivered to Alice.
```

Both packets look "inbound" to each NAT, but each NAT's entry was created by an *outbound* packet that arrived just before. The race typically completes in under 100ms over the public internet.

### Step 4: Birthday-paradox probing for EDM (symmetric) NATs

If one peer is behind an EDM NAT, STUN gives the wrong port. The peer's actual external port — the one assigned when it talks to *us* — is somewhere in the NAT's ephemeral port pool (typically 1024–65535).

Tailscale's approach: send probes to many candidate ports at once. If the pool is 64511 ports wide and the EDM NAT picks ports uniformly at random, sending 300 simultaneous probes gives a ~56% chance of hitting the target in the first round (birthday paradox: you don't need to cover the whole space, just have a good probability of overlap). Repeat over a few rounds with exponential backoff, and the hit rate exceeds 90% for typical real-world corporate NATs.

### ICE: systematic candidate collection

ICE (Interactive Connectivity Establishment, RFC 8445) generalizes the above into an algorithm:

1. Collect *candidates* on each side: local IP:ports, STUN-derived addresses, and relay (TURN) addresses.
2. Exchange all candidates via the signaling channel.
3. Form pairs (one candidate from each side) and try all pairs in priority order.
4. The highest-priority working pair wins; lower-priority pairs keep being tried in the background.

ICE was standardized for WebRTC (video/audio calls) and covers the same space as Tailscale's implementation — they're parallel solutions to the same problem.

### DERP: relay first, upgrade later

Before hole-punching completes — which can take hundreds of milliseconds — traffic needs somewhere to go. Tailscale uses DERP (Detoured Encrypted Routing Protocol) relay servers as the *initial* path:

- DERP runs over HTTPS (port 443), passing through nearly every corporate firewall that allows web traffic.
- Packets are already encrypted by WireGuard; DERP doesn't decrypt them, just routes by destination public key.
- DERP starts immediately: the connection is usable in ~50ms (DERP server RTT), regardless of NAT complexity.

Once hole-punching succeeds, Tailscale's WireGuard implementation silently switches to the direct path. The application never sees the upgrade. If hole-punching never succeeds (both peers behind EDM NATs with large ports and blocked UDP), DERP remains the permanent path — slower, but correct.

## Where it breaks

**Both peers behind symmetric NAT**: Birthday probing succeeds in practice (>90%) but fails when the NAT's port pool is very large, ports are sequential rather than random, or UDP probes are rate-limited by the firewall.

**UDP blocked entirely**: Some enterprise firewalls block all UDP except DNS. DERP over HTTPS is the fallback, but it carries the latency overhead of a relay hop.

**Carrier-grade NAT (CGNAT)**: Some ISPs run a second layer of NAT at the carrier level (IPv4 exhaustion remedy). Two layers of NAT require *both* to implement EIM. A single EDM layer at the carrier breaks simple hole-punching for all customers behind it.

**NAT entry timeouts**: NAT state entries expire after idle periods (typically 30 seconds to 5 minutes depending on the device). Long-lived connections need keepalive packets to prevent NAT table eviction while idle.

**IPv6**: Eliminates the core problem (every device gets a public address) but stateful firewalls may still drop unsolicited traffic. ICE still applies, but STUN isn't needed for address discovery.

## Why it works

**The mechanism exploits the NAT's own state-tracking logic.** A NAT must track outbound flows to route replies — that's its basic purpose. Hole-punching doesn't bypass this: it triggers it on both sides simultaneously. The NAT is doing exactly what it was designed to do (track an outbound flow); the peer arriving from the same direction as the expected reply just happens to match. There's no exploit, no bug — just two peers convincing their own NATs that they each initiated a connection toward the other.

**The "X is just Y" insight: NAT hole-punching is two-phase commit applied to firewall state.**

In 2PC, a coordinator tells all participants to *prepare* (write durable intent), then tells them to *commit* (make visible). In hole-punching, the control plane tells both peers to *prepare* (fire packets toward each other simultaneously, creating NAT state), and both are now *committed* (the channel is open). The coordinator doesn't participate in the data flow — it just synchronizes the moment of preparation.

This pattern — "create matching state on both sides simultaneously via a third-party coordinator, then communicate directly" — appears throughout distributed systems:

- **TCP three-way handshake**: SYN creates state at the server before data flows.
- **WireGuard's responder handshake**: The Noise_IKpsk2 handshake creates session keys on both sides before tunnel packets flow.
- **Chandy-Lamport snapshots**: A coordinator injects a marker token into each channel so all nodes "prepare" a snapshot at the same logical time.

What makes NAT traversal distinct: the state is created at an *external device* (the NAT) that has no knowledge of the protocol. The peers manipulate NAT state *indirectly*, by triggering the NAT's normal outbound-session behavior. It's a side-channel write.

**DERP's "relay first, upgrade later" pattern** separates two independent concerns: *latency to first byte* (use the relay immediately) and *steady-state throughput* (upgrade to direct when possible). This is the same pattern as HTTP/2 server push (send resources speculatively, reader decides later whether to use the preloaded copy) and speculative decoding (generate draft tokens, verify later, apply the correct prefix). Decouple "can I make progress now?" from "am I on the optimal path?"

**HTTPS-as-universal-transport** for DERP reflects a broader engineering principle: when you don't control the intermediate network, use the most permissively filtered protocol. Port 443 / TLS is the least-likely-blocked communication primitive on the internet, because blocking it would break most web browsing. SSH-over-443, gRPC-over-HTTP/2, and WebSocket all apply the same exploit: disguise application traffic as HTTPS because every firewall allows it.

## Going deeper

1. **RFC 4787 — "NAT Behavioral Requirements for Unicast UDP"** (Audet & Jennings, 2007): The formal taxonomy that defines EIM, EDM, and endpoint filtering variants. Knowing this RFC means you can predict in advance whether a NAT will pass the hole-punching test.

2. **RFC 8445 — "Interactive Connectivity Establishment (ICE)"** (Keranen, Holmberg & Rosenberg, 2018): The complete specification for candidate collection and pair selection used in WebRTC. Reading this alongside the Tailscale post shows how the same ideas are standardized across industries.

3. **Ford, Srisuresh & Kegel — "Peer-to-Peer Communication Across Network Address Translators" (USENIX ATC 2005)**: The foundational paper that characterized NAT behaviors and introduced the techniques (simultaneous open, birthday probing) that all subsequent P2P systems, including Tailscale and WebRTC, built on.
