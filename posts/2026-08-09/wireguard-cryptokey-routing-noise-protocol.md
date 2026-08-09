---
title: "WireGuard: Next Generation Kernel Network Tunnel"
source: https://www.wireguard.com/papers/wireguard.pdf
author: Jason A. Donenfeld
company: Zx2c4 / Edge Security
date_posted: 2017-02-01
date_digested: 2026-08-09
---

# WireGuard: Next Generation Kernel Network Tunnel

## What's new to learn

1. **CryptoKey routing**: A single table mapping each peer's public key to its AllowedIPs doubles as both the routing table (outbound: "which peer do I encrypt this packet to?") and the access control list (inbound: "which source IPs is this peer allowed to claim?"). No separate routing daemon, no ACL engine, no certificate authority.

2. **The Noise Protocol Framework**: A type system for handshake protocols. Noise defines named patterns — IK, XX, KK, NX, etc. — each encoding a specific set of security properties (does the initiator know the responder's static key? does mutual authentication happen in message 1 or message 2?). Choosing a pattern is like choosing a type: the compiler checks it for you. WireGuard uses `Noise_IKpsk2_25519_ChaChaPoly_BLAKE2s`.

3. **Opinionated cryptography as a correctness mechanism**: WireGuard provides *no* cipher agility. One key exchange algorithm, one AEAD, one hash function — no negotiation, no fallback. The result: the entire kernel module fits in ~4,000 lines of C and its security properties are formally verifiable.

## Prerequisites

- **Diffie-Hellman key exchange**: two parties each generate an ephemeral (public, private) pair, exchange publics, and compute a shared secret from `own_private × their_public` without the secret ever traveling over the wire. X25519 is the specific curve used.
- **AEAD (Authenticated Encryption with Associated Data)**: a single primitive that both encrypts and authenticates, producing a ciphertext and a MAC that together can't be forged. ChaCha20-Poly1305 is the specific AEAD used.
- **Layer-3 tunneling basics**: a VPN tunnel encapsulates IP packets (the inner packet) inside UDP datagrams (the outer packet). The kernel's network stack routes outbound inner packets to a virtual interface, which encrypts and encapsulates them; inbound UDP datagrams are decrypted and injected back into the stack.

## The core idea

Every VPN protocol before WireGuard answered three questions separately:

1. *Routing*: which peer does this packet go to?
2. *Key management*: what cryptographic identity does that peer have?
3. *Access control*: which IPs is an authenticated peer allowed to claim?

IPsec uses SPD (Security Policy Database) for routing, IKE for key management, and the SPD again for ACL — three separate systems with complex interactions. OpenVPN uses routing tables, TLS certificates, and iptables rules — three systems from three different security domains.

WireGuard's central observation: **all three questions have the same answer**. A peer IS its public key. Its public key IS its routing destination. Its AllowedIPs IS both what you route to it and what it's allowed to claim.

This collapses to a single table:

```
Peer table:
  peer_public_key  →  (AllowedIPs, endpoint, optional_preshared_key)

Outbound: find the peer whose AllowedIPs contains dest_ip → encrypt to that peer's key
Inbound:  decrypt with session key derived from initiator's key → accept only if source_ip ∈ peer's AllowedIPs
```

There is no separate state. There is no certificate store. There is no CA. The entire security policy is the peer table.

## Mechanics

### Handshake: Noise_IKpsk2

The protocol identifier `Noise_IKpsk2_25519_ChaChaPoly_BLAKE2s` decodes as follows:

- **Noise**: uses the Noise Protocol Framework
- **IK**: the *initiator* knows the *responder's* static public key before the first message (I = initiator static key included in message 1; K = responder's key is Known by initiator)
- **psk2**: an optional pre-shared symmetric key is mixed into the key derivation after message 2
- **25519**: X25519 (Curve25519) for Diffie-Hellman
- **ChaChaPoly**: ChaCha20-Poly1305 for AEAD
- **BLAKE2s**: for hashing, keyed MACs, and HKDF-style key derivation

The handshake is one round trip (1-RTT):

**Message 1 (Initiator → Responder, 148 bytes):**
```
{sender_index, ephemeral_pubkey_e, AEAD(encrypted_static_pubkey_i), AEAD(encrypted_timestamp), MAC1, MAC2}
```
The initiator generates an ephemeral key pair (eᵢ, Eᵢ), then chains together four Diffie-Hellman operations to derive a symmetric key before the first message is complete:
```
DH1 = X25519(eᵢ, Rₛ)          # ephemeral initiator × responder's static key
DH2 = X25519(Iₛ, Rₛ)          # static-static DH (proves initiator's identity)
```
Both operands are fed into a *chaining key* (ck) and a hash state (h) via BLAKE2s MixKey/MixHash operations. The timestamp inside the encrypted payload prevents replay attacks: the responder rejects any initiation older than the last accepted one from this initiator (using a TAI64N timestamp encoded as 12 bytes).

**Message 2 (Responder → Initiator, 92 bytes):**
```
{receiver_index, sender_index, ephemeral_pubkey_e, AEAD(""), MAC1, MAC2}
```
The responder generates its own ephemeral key pair (eᵣ, Eᵣ), performs two more DH operations:
```
DH3 = X25519(eᵣ, Eᵢ)          # responder ephemeral × initiator ephemeral
DH4 = X25519(eᵣ, Iₛ)          # responder ephemeral × initiator static
```
After all four DH operations, both parties independently derive identical transport key material via HKDF. Two separate session keys are derived: one for sending and one for receiving (asymmetric).

**Transport phase**: each IP packet is encrypted as:
```
UDP payload = {type=4, receiver_index, counter, AEAD(inner_ip_packet)}
```
The counter serves as the nonce for ChaCha20-Poly1305. A 64-bit counter with window-based anti-replay detection prevents replay at the cost of bounded reordering.

### Silent by default: DoS resistance

The responder sends *no response* to a handshake initiation unless the AEAD decryption of the encrypted_static_pubkey field succeeds. That decryption requires knowing the responder's static private key, which means only parties in possession of the responder's *public* key can trigger a cryptographic response.

From a port scanner's perspective, WireGuard's UDP port is indistinguishable from "filtered" (no response). This eliminates:
- Port scanning disclosure
- DDoS reflection amplification (a UDP flood gets zero response)
- OS fingerprinting via banner

Under high load, the responder can additionally refuse to perform the X25519 DH and instead send a **cookie reply** — a MAC of the initiator's IP address, keyed on a rotating server secret. The initiator includes this cookie as MAC2 in its retry. This is a proof-of-IP mechanism: the server only performs expensive crypto for clients who can receive a UDP packet at their claimed IP.

### Timer state machine

WireGuard rotates session keys before they can be exhausted:

- **REKEY_AFTER_TIME** (180 s): if a session key is 3 minutes old, a new handshake is initiated for the *next* packet (the old key continues serving in-flight packets)
- **REJECT_AFTER_TIME** (180 s + 90 s = 270 s): after this, the old key is rejected even for decryption
- **REJECT_AFTER_MESSAGES** (2⁶⁰): nonce counter limits — beyond this many messages on one key, nonces would repeat, so a rekey is forced

After `KEEPALIVE_TIMEOUT + REKEY_TIMEOUT` (about 5 minutes) of silence, the session is torn down and memory zeroed. The next packet triggers a new handshake. There is no persistent session state across inactivity periods — just static configuration.

### Code size vs alternatives

```
WireGuard Linux kernel module:  ~4,000 lines of C
IPsec XFRM stack (kernel):      ~20,000 lines
OpenVPN:                        ~600,000 lines
strongSwan (IKEv2 daemon):      ~400,000 lines
```

The kernel module includes the crypto, the virtual network interface driver, the peer table management, the handshake state machine, the timer machinery, and the routing logic. In 4,000 lines.

## Where it breaks

**No anonymity**: WireGuard operates at layer 3. The outer UDP packet carries the real source IP of the sending machine. This is by design — WireGuard is a point-to-point tunnel, not an anonymizing proxy.

**No overlapping AllowedIPs**: Each IP address can belong to exactly one peer's AllowedIPs. You cannot route the same subnet to multiple peers for load balancing at the WireGuard layer. Multi-path routing requires a higher-level mechanism.

**Manual key revocation**: There is no certificate authority, no CRL, no OCSP. If a peer's private key is compromised, every other peer's configuration must be updated by hand. In large deployments (hundreds of peers), this is a real operational burden — commercial solutions like Tailscale layer a control plane on top.

**PSK distribution problem**: The optional pre-shared key protects against a future break of Curve25519 (e.g., by a quantum computer), but distributing a symmetric secret out-of-band to every peer pair is `O(peers²)`.

**1-RTT minimum**: Unlike TLS 1.3's 0-RTT resumption, WireGuard requires a full handshake for every session (though sessions persist for minutes). Post-handshake latency is excellent, but the first packet after a long idle period pays a full round trip.

**IP address identity coupling**: The AllowedIPs table equates routing addresses with cryptographic identities. Changing a peer's VPN IP requires reconfiguring every endpoint that routes traffic to it.

## Why it works

The deeper principle is **reducing the state space until formal verification becomes tractable**.

Security vulnerabilities come from unexpected states: a protocol in an unanticipated combination of cipher + key exchange + MAC algorithm, or a code path only reachable in one of thousands of possible configurations. The traditional response is "more testing." The WireGuard response is "fewer states."

By fixing the algorithm suite to a single point — X25519 for DH, ChaCha20-Poly1305 for AEAD, BLAKE2s for hash/KDF, SipHash24 for lookup tables — WireGuard eliminates the entire class of "negotiation bugs." There is no downgrade path, no legacy cipher suite, no algorithm handshake to exploit. A cryptanalyst auditing WireGuard knows exactly which mathematical primitives to study.

This connects directly to three other archive entries:

- **eBPF verifier** (2026-07-08): instead of running arbitrary C in the kernel, eBPF programs must pass bounded abstract interpretation. Fixed instruction set → tractable verification.
- **Firecracker microVM** (2026-07-21): instead of emulating 1,500 device types (QEMU), Firecracker emulates a handful of minimal virtio devices. Minimal device model → small Trusted Computing Base.
- **FoundationDB simulation** (2026-05-19): instead of relying on general-purpose testing, FoundationDB replaces all non-determinism with a seeded PRNG. Fixed execution model → reproducible bugs.

The common structure: **choose one thing to do and do it completely, rather than choosing many things and leaving verification gaps in every combination**.

The second insight is the **Noise Protocol Framework** as a type system for handshake protocols. Noise defines a finite vocabulary of *patterns* (IK, XX, KK, NX, NK, XK, …), where each pattern name encodes:
- Which static keys are transmitted in which messages
- Which DH operations are performed in which order
- Which security properties are guaranteed at each point in the handshake

A Noise IK handshake is not something you design from scratch each time — it is an *instance* of the pattern grammar, the way an HTTP/2 HEADERS frame is an instance of the HPACK specification. Security researchers have proven security properties for each named pattern; using a named pattern lets you inherit those proofs.

This is the same insight as typed functional programming: push invariants into the type system, and "if it compiles, it's safe" becomes meaningful.

The **CryptoKey routing** model also surfaces in a surprising cousin: modern service meshes. Envoy/Istio implement *identity-based routing* at layer 7 using SPIFFE certificates — traffic is routed to a workload based on its cryptographic identity (a SVID/SPIFFE URI), not just its IP address. WireGuard does the same thing at layer 3. In both cases, the routing table IS the trust model. WireGuard just uses a Curve25519 public key where Envoy uses an X.509 certificate.

## Going deeper

1. **The Noise Protocol Framework specification** (noiseprotocol.org) — the formal grammar WireGuard's handshake is an instance of. Reading it teaches you how to design secure handshakes compositionally, not from scratch.

2. **"A Cryptographic Analysis of the WireGuard Protocol"** (Dowling & Paterson, 2018) — a formal Bellare-Rogaway security model proof that WireGuard achieves mutual authentication and forward secrecy. Available at `wireguard.com/papers/dowling-paterson-computational-2018.pdf`. This is what a formal security argument for an opinionated protocol looks like in practice.

3. **Tailscale's engineering blog on their overlay network** (tailscale.com/blog) — specifically posts on the control plane they built on top of WireGuard, showing what the "manual key revocation" problem looks like at scale and how a distributed key management system resolves it.
