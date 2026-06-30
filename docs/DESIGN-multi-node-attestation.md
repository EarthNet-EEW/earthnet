# Design — Multi-node event attestation (remove the single trust root)

*Roadmap [Issue #1](https://github.com/EarthNet-EEW/earthnet/issues/1). Design only. Carrier is the
`ConfirmedEvent` v0.2 `repeated NodeAttestation` + `tier` fields (see
[`PROTOCOL-v0.2-DRAFT.md`](../PROTOCOL-v0.2-DRAFT.md)).*

## Problem

Today **one** `earthnet-node` signs every `ConfirmedEvent`. A compromised node can **forge an alert
to the entire network** (cause panic) or **suppress** a real one. For a decentralized life-safety
network this is the #1 trust gap. The "decentralized" claim is not honest until no single party can
forge an alert.

## Goal

An **`ALERT`-tier** event is valid to a client **only if M independent, eligible nodes co-attest
it.** No single node — including the originator — can produce a life-safety alert alone.

Requirements:
- **Low added latency** — the alert plane has a <2 s pick→alert budget.
- **Partition-tolerant** — a region must certify locally even if cut off from the rest.
- **Byzantine-aware** — tolerate up to *f* faulty/malicious eligible nodes.
- **Thin-client verifiable** — a phone must verify a certificate cheaply (it already does Ed25519).
- **No heavy new infra or exotic crypto** for v1 (auditability matters for life-safety).

## Scheme options

| Scheme | Size | Client verify | New crypto | Coordination | Verdict |
|---|---|---|---|---|---|
| **M-of-N plain Ed25519 multisig** (each node signs independently) | M×96 B | M× Ed25519 verify | none (reuse) | none (independent) | **✅ start here** |
| BLS aggregate signature | ~96 B (flat) | 1 pairing verify | BLS lib (Rust+Dart), rogue-key/PoP | aggregation step | later, if M grows |
| FROST / threshold Ed25519 | 64 B (looks single) | 1 Ed25519 verify | DKG + threshold lib | DKG + signing round, re-share on churn | ❌ too much op risk now |
| BFT quorum certificate (HotStuff/Tendermint-style) | M votes | M verify | consensus engine | full voting round | ❌ overkill unless we adopt BFT |

### Decision — independent M-of-N plain Ed25519 (v1)

**Why:** zero new crypto (reuses the audited Ed25519 already in protocol/node/app), **no DKG and no
signing round** (attestors sign independently → lowest latency, partition-friendly), thin-client
verifiable with code that already exists, and size is fine for small M (M=3–7 → a few hundred
bytes). It is the **honest, boring, secure** choice for a system where an alert moves people.

Revisit **BLS aggregation** only when M is large enough that certificate size matters — keeping plain
Ed25519 as the interoperable baseline. Avoid **FROST/threshold** until membership is stable enough to
afford DKG + re-sharing on node churn (a real operational burden in an open network).

## Who may attest — the quorum membership problem (the hard part)

A signature is only meaningful if the client knows **which pubkeys are eligible** to count toward the
quorum — otherwise an attacker spins up M Sybil nodes and self-attests. So the **eligible-attestor
set is itself a trust anchor** (chicken-and-egg).

- **Per-cell eligibility:** attestors for an event are the nodes responsible for its geo-cell +
  neighbors (ties to Phase-2 geographic partitioning), gated by reputation/stake.
- **Bootstrapping (honest about it):** v1 uses a **governance-signed root set** of operated nodes
  (a curated, signed eligibility list distributed like the schema registry / a CRL). Decentralizing
  membership (permissionless attestors with proof-of-uniqueness + stake) is the long game (Phase 6
  governance). Saying "fully decentralized" before that root exists would be dishonest.
- **M and N:** start small (e.g., **M=3** of the regional eligible set) to protect latency; raise M
  as the network matures. Forgery cost = "compromise M eligible nodes." For agreement under *f*
  Byzantine faults, M ≥ 2f+1.

## Production flow

1. A node forms a **provisional** event (its own consensus result) and emits it immediately at
   `tier=PROVISIONAL` — *the network warns first, certifies in flight* (see latency trick below).
2. It broadcasts the **canonical signing payload** to the cell's eligible attestors.
3. Each attestor **independently re-validates** (re-runs phase association over the same observation
   set / checks the claim) and, only if it agrees, signs the canonical bytes → returns a
   `NodeAttestation{node_pubkey, signature}`.
4. The originator collects **≥ M** attestations, fills `attestations[]`, sets **`tier=ALERT`**, and
   re-emits (a revision that `supersedes` the provisional).
5. If < M within a deadline → it stays `PROVISIONAL` (research/UI only, never life-safety). Graceful
   degradation: a possibly-real event is never *blocked*, it just isn't *certified*.
6. **Client:** for `tier=ALERT`, verify ≥ M valid `NodeAttestation`s whose pubkeys are in the known
   eligible set; otherwise treat the event as provisional/informational and do not raise a
   life-safety alarm.

### The latency trick (how this stays under 2 s)

Do **not** wait for the quorum to warn. The deterministic single-node event ships **instantly** as
`PROVISIONAL` ("possible event, confirming…"); the attestation round (one regional RTT + a fast
re-validation, M small) runs **in flight**; when ≥ M attestations arrive the app **upgrades** the
card to `ALERT` ("confirmed by N nodes"). Users get the early warning immediately *and* a trust
upgrade seconds later — best of both planes.

## Attacks addressed / residual

- **Single-node compromise:** cannot reach M alone. ✅
- **Forged/fabricated event:** attestors re-validate against the observations and won't sign a
  fabrication. ✅
- **Sybil attestors:** eligibility list + reputation gate. ✅ (strength = honesty of the list)
- **Residual:** compromise of **M** eligible nodes (cost scales with M); and the **eligibility-root
  governance** (rotation, who curates it). These are explicitly the long-game (Phase 6).

## Dependencies on other work

- Replicated **node registry / eligibility list** (ties to Phase-2 replicated reputation/state).
- **Geo-cell quorum assignment** (Phase-2 geographic partitioning).
- **`tier` semantics in the client** (Phase-1 mobile): provisional vs alert rendering + the upgrade.

## Open questions

- Dynamic membership / churn of the eligible set.
- **Cross-cell events** (large quakes span many cells — whose quorum certifies?).
- Incentive to attest; **slashing** for signing a false attestation.
- Governance & rotation of the eligibility root (the bootstrap trust anchor).
