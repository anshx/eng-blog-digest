---
title: "The Double Ratchet Algorithm"
source: https://signal.org/docs/specifications/doubleratchet/
author: Trevor Perrin, Moxie Marlinspike
company: Signal Foundation
date_posted: 2016-11-20
date_digested: 2026-10-04
---

# The Double Ratchet Algorithm

## What's new to learn

1. **KDF chain (symmetric-key ratchet)**: A one-way key derivation chain where each step advances the chain key and peels off a message key; deleting the old chain key makes past message keys irrecoverable — forward secrecy for free, at zero per-message negotiation cost.

2. **DH ratchet (Diffie-Hellman ratchet)**: Each party embeds a fresh ephemeral DH public key in every message header; when the receiver sees a new key it derives a new root key from DH(own\_private, their\_public), then deletes the old private key — injecting entropy the attacker never saw, healing the session after a device compromise.

3. **Post-compromise security (PCS, or "break-in recovery")**: A security property orthogonal to forward secrecy — after an attacker captures your full session state, future messages (once the next DH ratchet step occurs) become opaque to them again, because the new DH secrets are unpredictable.

## Prerequisites

- **Diffie-Hellman key exchange** (ECDH): two parties derive a shared secret from their keypairs without transmitting the secret.
- **HKDF / HMAC-based KDF**: a keyed pseudorandom function that derives output material indistinguishable from random.
- **Authenticated encryption (AEAD)**: e.g. AES-256-GCM — encryption + integrity in one construction.
- **X3DH (Extended Triple Diffie-Hellman)**: the initial key-agreement protocol used to bootstrap a Double Ratchet session; covers the first shared secret, out-of-band public key distribution, and pre-keys. (The Double Ratchet spec treats X3DH output — a shared secret `SK` — as its input.)

## The core idea

Imagine you want a messaging protocol where:

1. If your phone is seized *today*, the attacker cannot read *yesterday's* messages. (Forward secrecy.)
2. If your phone is seized *today*, the attacker cannot read *tomorrow's* messages, once the other party sends the next message. (Post-compromise security.)

These properties are in tension. Forward secrecy requires destroying old key material. But if an attacker copies your state at time T, they can compute every future key in a deterministic chain — unless fresh entropy enters the system after T.

The Double Ratchet solves both with two interlocking ratchets:

- The **symmetric-key ratchet** (a KDF chain) provides per-message forward secrecy cheaply: derive message key and next chain key, delete the current one. Moving forward is fast; moving backward is cryptographically impossible.
- The **DH ratchet** provides break-in recovery: generate a fresh ephemeral DH keypair, publish the public half in the message header, delete the private half after use. The next time the other party responds they will include *their* new DH public key; your DH exchange of (new\_yours, new\_theirs) produces a root key that neither party had before and no attacker who captured your state at T can derive.

The two ratchets are coupled through a **root chain**: DH outputs feed into the root KDF to produce new symmetric chain keys. Per-message key isolation is the job of the symmetric ratchet; root-key renewal is the job of the DH ratchet.

## Mechanics

### Session state (per party)

| Variable | Type | Role |
|---|---|---|
| `RK` | 32-byte key | root key; mixed with each DH output to advance |
| `CKs` | 32-byte key | current sending chain key |
| `CKr` | 32-byte key | current receiving chain key |
| `DHs` | ECDH keypair | own current DH ratchet keypair |
| `DHr` | ECDH public key | remote party's last-seen DH ratchet public key |
| `Ns`, `Nr` | integers | sending and receiving message counters (within current chain) |
| `PN` | integer | number of messages in the previous sending chain (for header) |
| `MKSKIPPED` | dict `{(DHr, N) → mk}` | stored message keys for out-of-order messages |

### KDF primitives

Two keyed functions cover all derivations:

```
KDF_RK(rk, dh_out) → (new_rk, chain_key)   # HKDF with rk as salt, dh_out as IKM
KDF_CK(ck)         → (new_ck, message_key)  # HMAC: mk = HMAC(ck, 0x01), new_ck = HMAC(ck, 0x02)
```

The separation matters: `KDF_RK` mixes in external entropy (`dh_out`); `KDF_CK` is purely one-way — no external input, no going back.

### Sending a message

```python
def RatchetEncrypt(state, plaintext, AD):
    # Advance the symmetric sending chain
    state.CKs, mk = KDF_CK(state.CKs)
    header = Header(state.DHs.public, state.PN, state.Ns)
    state.Ns += 1
    return header, ENCRYPT(mk, plaintext, AD=Encode(header) + AD)
```

No DH operation happens on every send — only on the first send after receiving a new DH key. Within a run of consecutive sends, only the fast HMAC ratchet fires.

### Receiving a message (simplified)

```python
def RatchetDecrypt(state, header, ciphertext, AD):
    if header.dh != state.DHr:              # NEW DH public key from remote
        SkipMessageKeys(state, header.PN)   # store keys for any skipped messages on old chain
        DHRatchetStep(state, header.dh)     # advance DH ratchet
    SkipMessageKeys(state, header.N)        # store keys for any skipped messages on new chain
    state.CKr, mk = KDF_CK(state.CKr)
    state.Nr += 1
    return DECRYPT(mk, ciphertext, AD=Encode(header) + AD)
```

### The DH ratchet step (the healing operation)

```python
def DHRatchetStep(state, dh_new):
    state.PN = state.Ns          # remember how many messages were in the old chain
    state.Ns = state.Nr = 0
    state.DHr = dh_new
    # Receiving chain: DH with old own key and new remote key
    state.RK, state.CKr = KDF_RK(state.RK, DH(state.DHs.private, state.DHr))
    # Sending chain: DH with FRESH own key and new remote key
    state.DHs = GENERATE_DH()   # <— fresh keypair; old private key is now DELETED
    state.RK, state.CKs = KDF_RK(state.RK, DH(state.DHs.private, state.DHr))
```

Crucially, the old `DHs.private` is destroyed immediately after both derivations. From that moment, an attacker who had copied `state` at time T cannot compute either the new `CKr` or the new `CKs`.

### Out-of-order message handling

`MKSKIPPED` stores message keys derived from `KDF_CK` calls performed to "fast-forward" past gaps. A message arriving out of order is decrypted with its stored key; the max gap is bounded to prevent unbounded dictionary growth. Each stored key is deleted as soon as it is used.

### What a full conversation looks like

```
Alice → Bob: [DH_A1, N=0]  msg0  (Alice's first DH ratchet step: DH(A1, B0))
Alice → Bob: [DH_A1, N=1]  msg1  (same chain key, symmetric ratchet only)
Bob   → Alice: [DH_B1, N=0] reply (Bob's DH ratchet: DH(B1, A1); new CKr for Alice)
Alice → Bob: [DH_A2, N=0]  msg2  (Alice's DH ratchet: DH(A2, B1); fresh entropy)
```

After step 4, an attacker who cloned Alice's state before step 4 cannot compute `msg2`'s key — they don't know `A2.private`.

## Where it breaks

**Requires frequent key exchanges to heal.** Post-compromise security is only regained after a full DH ratchet round-trip. If one party is offline for a long time and the other keeps sending, the attacker who compromised the sending party can follow the entire one-sided chain until the receiver replies. The healing latency equals one round-trip.

**Multi-device is genuinely hard.** The Double Ratchet is point-to-point between two ratchet states. Supporting multiple devices requires running separate ratchet sessions per device, or a more complex multi-party scheme (Signal uses "Sealed Sender" and sender-key groups for this). Group messaging is not directly covered.

**Header data is metadata.** The DH public keys in headers, the message counters, and the previous-chain length `PN` reveal the conversation's message-count structure to anyone who can observe the ciphertext — even without decrypting.

**`MKSKIPPED` is a side-channel.** Storing skipped message keys in persistent storage risks exposing them if the device is later compromised. Implementations must bound its size and age-out entries.

**Initial key establishment is separate.** The Double Ratchet spec takes a shared secret (`SK`) as input. Getting that first shared secret — authenticating the parties, distributing pre-keys, preventing impersonation — is the job of X3DH, which has its own trust assumptions and attack surface.

## Why it works

The Double Ratchet is an instance of a recurring principle in cryptography: **replace a global, long-lived secret with a sequence of short-lived secrets, and use a one-way function to relate them**.

- The **KDF chain** is a forward-secure pseudorandom generator (PRG): `(state, output) = PRG(state)`. The output is a message key; the new state replaces the old one. This is structurally identical to a stream cipher keystream generator, but applied to key material rather than plaintext. Forward secrecy in TLS 1.3 uses the same one-way KDF chain for session keys; the Double Ratchet extends it per-message.

- The **DH ratchet** is an application of the **key compromise impersonation (KCI) gap** filling: long-term static keys can be stolen, but ephemeral keys generated and deleted within a session cannot be retroactively obtained. This is the same insight behind TLS 1.3's forward-secret handshake: ephemeral ECDH means session keys are not derivable from the long-term certificate private key alone. The Double Ratchet makes this happen at the *message* granularity rather than the session granularity.

The combination — symmetric chain for the fast path, DH ratchet for healing — is an example of **composing complementary security properties**: neither mechanism alone achieves both forward secrecy and PCS; together they cover the full threat model with minimal overhead.

There is a deeper parallel to distributed systems. The KDF chain is like a Lamport clock: it advances monotonically, creates an ordering, and is cheap. The DH ratchet is like a physical clock (TrueTime): it injects real-world entropy (new randomness) to provide a bound on what can be inferred. Just as HLC (hybrid logical clocks) combine the two for causal consistency with bounded drift, the Double Ratchet combines the two for message secrecy with bounded heal time.

## Going deeper

1. **The Double Ratchet specification** — the canonical primary source, in the public domain, with full pseudocode and a "Security Considerations" appendix: [signal.org/docs/specifications/doubleratchet](https://signal.org/docs/specifications/doubleratchet/)

2. **Alwen, Coretti, Dodis — "The Double Ratchet: Security Notions, Proofs, and Modularization for the Signal Protocol"** (IEEE S&P 2019) — the formal security proof, showing that the Double Ratchet achieves what it claims under standard cryptographic assumptions; also separates the ACD notion of "continuous key agreement" which generalises the DH ratchet to arbitrary sources of entropy: [eprint.iacr.org/2018/1037](https://eprint.iacr.org/2018/1037)

3. **X3DH (Extended Triple Diffie-Hellman) specification** — covers the key-establishment protocol that bootstraps a Double Ratchet session: identity keys, signed pre-keys, one-time pre-keys, and how to prevent key-reuse attacks: [signal.org/docs/specifications/x3dh](https://signal.org/docs/specifications/x3dh/)
