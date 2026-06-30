# EarthNet Protocol v0.2 — Multi-modal Signal envelope (DRAFT / RFC)

*Status: **draft**. This proposes the single most important change toward Planetary Intelligence:
generalizing the earthquake-specific `Observation` into a **modality-agnostic, signed Signal
envelope** — the "narrow waist" of the platform. See [`VISION.md`](VISION.md) and
[`docs/ARCHITECTURE-REVIEW.md`](docs/ARCHITECTURE-REVIEW.md). v0.1 lives in
[`PROTOCOL-v0.1-DRAFT.md`](PROTOCOL-v0.1-DRAFT.md).*

## 1. Why

v0.1's `Observation` (`sta_lta_ratio`, `p_wave_detected`, `estimated_pga`) is cast for earthquakes.
Every new modality (magnetometer, barometer, weather, satellite…) would require touching the
protocol, node, app, adapters and DB schema — open-heart surgery each time. v0.2 makes the
**envelope stable and universal** while modalities evolve **independently** in a schema registry.

> **Narrow-waist governance rule:** adding a hazard or a sensor must NOT change this protocol or the
> node. It must be: *register a schema + deploy a detector/experiment.* If it isn't, the design failed.

## 2. Goals / non-goals

**Goals:** one signed envelope for any signal; modalities added without recompiling the core;
privacy declared structurally per modality; backward-compatible mapping of v0.1; a hook for
multi-node attestation; cross-language byte-determinism preserved (Rust/prost == Python == Dart).

**Non-goals (this RFC):** implementing new sensors; choosing the streaming backbone (separate
design); the on-device AI runtime. We design the *contract*, not the pipeline.

## 3. The envelope: `Signal`

The whole platform's narrow waist. Coarse, stable fields for the **fabric** (routing/partitioning/
trust) + an **opaque, registry-typed payload** for the **semantics**.

```protobuf
message Signal {
  uint32 protocol_version  = 1;   // = 2

  // ---- identity (unchanged scheme from v0.1) ----
  bytes  observation_id    = 2;   // 16 random bytes (idempotency)
  bytes  pubkey            = 3;   // Ed25519, the anonymous identity

  // ---- fabric fields (readable WITHOUT the registry) ----
  ModalityClass modality_class = 4;  // coarse class for routing/sharding (see §6)
  uint64 schema_id         = 5;   // precise {modality, version} -> registry (see §7)
  SourceType  source_type  = 6;   // PHONE | STATION | OFFICIAL | IOT | SATELLITE | DERIVED
  DeviceClass device_class = 7;   // capability tier of the emitter

  // ---- spacetime ----
  Location location        = 8;   // geohash + uncertainty + privacy level (§5)
  int64  captured_at_ns    = 9;   // UTC nanoseconds
  uint32 clock_uncert_ms   = 10;  // clock uncertainty (consensus weights by this)

  // ---- quality ----
  float  quality           = 11;  // 0..1 self-reported SNR/confidence (NOT trusted; a hint)

  // ---- semantics (opaque; meaning lives in the schema referenced by schema_id) ----
  bytes  payload           = 12;  // canonical bytes of the modality-specific schema

  // ---- signature LAST ----
  bytes  signature         = 13;  // Ed25519 over: domain || encode(Signal w/ signature empty)
}
```

### Key design decision — opaque payload, not a `oneof`
We deliberately do **not** put a protobuf `oneof` of all modalities here. A `oneof` couples the
protocol to every modality (recompile to add one). Instead: **`bytes payload` + `uint64 schema_id`**,
where the registry resolves `schema_id → {modality, version, schema, privacy policy}`. This mirrors
CloudEvents (`data` + `dataschema`) and Kafka Schema Registry. *Trade-off:* the envelope loses typed
introspection of the payload (you need the registry to decode it) — **gained:** modalities evolve
with zero core changes. This is the narrow-waist-correct choice and the whole point of v0.2.

The **signature covers the opaque `payload` bytes**, so neither the registry nor a schema upgrade
can retroactively change what a node signed. (Cleaner than signing typed fields.)

## 4. Identity & signing (carried from v0.1, new domain tag)

Unchanged scheme: `signature = Ed25519(domain_tag || deterministic_encode(msg with signature empty))`.
New domain tags (version-bound, replay-separated):
- `DOMAIN_SIGNAL          = "earthnet-signal-v2"`
- `DOMAIN_CONFIRMED_EVENT = "earthnet-event-v2"`
- `DOMAIN_NODE_ATTEST     = "earthnet-attest-v2"`

Determinism rules (unchanged, now law for v0.2): fields in tag order; proto3 omits defaults; the
payload bytes are produced by the modality schema's own canonical encoder. Cross-language vectors
(Rust/prost, Python, Dart) must match byte-for-byte, as in v0.1.

> Lesson already learned (v0.1, mobile): never set a default-valued scalar explicitly when building
> a message to sign — proto3/prost omit it but some encoders emit it, breaking canonical bytes.

## 5. Location & structural privacy

```protobuf
message Location {
  string geohash       = 1;   // truncatable: precision encodes privacy
  uint32 precision_m   = 2;   // declared spatial precision (meters)
  PrivacyLevel privacy = 3;   // EXACT | COARSE | CELL_ONLY | NONE
}
enum PrivacyLevel { PRIVACY_EXACT=0; PRIVACY_COARSE=1; PRIVACY_CELL_ONLY=2; PRIVACY_NONE=3; }
```
Privacy is **structural**, not a promise: each **schema** in the registry declares the *maximum*
fidelity and the *minimum* location-coarsening allowed for that modality (e.g., `audio_fft` →
FFT-only payload, location ≥ COARSE; `image_features` → never raw pixels). The node/app enforce the
schema's policy on emit and on ingest. This is what later lets the research plane request higher
fidelity **only from consenting nodes** without betraying anyone by default.

## 6. ModalityClass (coarse, stable, rarely extended)

For the streaming fabric to **route/shard/prioritize without parsing the payload**:

```
MOTION          // accelerometer, gyroscope, seismometer waveform
FIELD           // magnetometer, electric field
POSITION        // GPS/GNSS displacement, tilt
ATMOSPHERIC     // barometer, temperature, humidity, wind
ACOUSTIC        // microphone FFT, infrasound
IMAGING         // camera-derived features (never raw)
ENVIRONMENTAL   // generic IoT/citizen sensors
REMOTE_SENSING  // satellite (InSAR, thermal), weather feeds
DERIVED         // outputs of detectors/agents (a pick, an anomaly, a fused estimate)
```
This enum is **deliberately coarse and slow-changing** (the fabric depends on it). Precise semantics
and versioning live in `schema_id`. Adding a *kind* of sensor = a new schema (no enum change);
adding a whole new *class* of physics (rare) = one enum value.

## 7. Schema registry

A versioned, signed catalog: `schema_id → { modality_class, name, version, encoding, schema_def,
privacy_policy, status }`.
- **Encoding:** protobuf (default) or CBOR/Arrow for numeric arrays (waveform windows, FFT bins).
- **Versioning:** `schema_id` is immutable; a new version = a new `schema_id` (no in-place mutation,
  so old signatures stay verifiable forever).
- **Distribution:** the registry is itself replicated/cached at the edge (a small, slow-changing
  artifact). Nodes ship with a baseline set; new schemas propagate like CRLs.
- **Governance:** who may register a schema, and the review for any schema that can feed the **alert
  plane** (research-plane schemas are open; alert-plane schemas are curated). Ties into §11.

Example schemas at launch: `seismic.accel.v1` (waveform window), `seismic.pick.v1` (a STA/LTA pick =
today's `Observation` fields), `atmo.baro.v1`, `geo.gnss_disp.v1`, `field.mag.v1`,
`derived.anomaly.v1`.

## 8. Backward compatibility — v0.1 `Observation` becomes a modality

No data is orphaned. The v0.1 `Observation` maps cleanly:
- `modality_class = MOTION`, `schema_id = seismic.pick.v1`.
- The payload schema `seismic.pick.v1` carries `{ sta_lta_ratio, p_wave_detected, estimated_pga,
  reported_magnitude }` — exactly the v0.1 semantic fields.
- Envelope fields (`pubkey`, `source_type`, `location`, `captured_at_ns`, `clock_uncert_ms`,
  `signature`) map 1:1.

A node can **dual-stack**: accept v0.1 `Observation` and v0.2 `Signal` during migration, internally
normalizing v0.1 into a `Signal{seismic.pick.v1}`. The consensus/locate/magnitude code reads the
typed payload of `seismic.pick.v1` — its logic is unchanged; it just gets its inputs from the
payload instead of the top-level message.

## 9. ConfirmedEvent v0.2 (multi-hazard) + attestation hook

The output generalizes from "earthquake" to "a typed hazard event," and gains the **multi-node
attestation** hook (architecture-review gap #1):

```protobuf
message ConfirmedEvent {
  uint32 protocol_version = 1;   // = 2
  bytes  event_id         = 2;
  HazardClass hazard_class= 3;   // SEISMIC | TSUNAMI | VOLCANIC | FIRE | FLOOD | STRUCTURAL | ATMOSPHERIC | ANOMALY ...
  uint64 event_schema_id  = 4;   // typed event payload (epicenter/magnitude live here for SEISMIC)
  bytes  payload          = 5;   // opaque, registry-typed (mirrors Signal)

  int64  origin_time_ns   = 6;
  Location epicenter      = 7;    // generalized: the event's primary location
  uint32 num_observations = 8;
  Confidence tier         = 9;    // RESEARCH | PROVISIONAL | ALERT  (the two-plane gate)
  bytes  supersedes       = 10;

  // ---- decentralized trust ----
  repeated NodeAttestation attestations = 11;  // M-of-N independent co-signatures
  bytes  signature        = 12;   // emitting node's own signature (over fields w/ sig empty)
}
message NodeAttestation { bytes node_pubkey = 1; bytes signature = 2; }
```
- **Two planes encoded in `tier`:** a research-grade event (`RESEARCH`/`PROVISIONAL`) may be emitted
  by one node/model; an **`ALERT`-tier** event MUST carry ≥ M valid `attestations` from independent
  nodes, or clients reject it for life-safety use. This is what makes "no single node can forge an
  alert" real, and keeps the experimental research plane open without endangering anyone.
- Threshold scheme (FROST/BLS aggregate vs simple multi-sig vs quorum certificate) is a **separate
  design** (roadmap Issue #1); this envelope just reserves the carrier so we don't re-break the wire
  later.

## 10. Wire format & determinism

Unchanged from v0.1 in principle: Protobuf, deterministic/canonical encoding, signature over
`domain || encode(empty-signature)`. New: payloads are opaque bytes whose canonical form is the
modality schema's responsibility, and which the envelope signature covers verbatim. Cross-language
conformance vectors must be added for `Signal{seismic.pick.v1}` before any code ships.

## 11. Open questions (RFC)

1. **schema_id allocation:** global registry authority vs namespaced (org-prefixed) IDs vs
   content-addressed (hash of the schema)? *Leaning content-addressed* (self-verifying, no central
   authority — fits decentralization).
2. **Registry trust:** who signs the registry; how edge nodes validate a new schema; revocation.
3. **Alert-plane schema governance:** the curation/review process for any schema allowed to reach
   `tier=ALERT`.
4. **Payload encoding for numeric arrays:** protobuf repeated-float vs CBOR vs Arrow IPC for
   waveform/FFT windows (size + zero-copy matter at scale).
5. **Quality/`quality` field:** self-reported and untrusted — keep as a routing hint only, or drop
   it and derive quality from reputation + cross-validation? *Leaning: hint only, never trusted.*
6. **Privacy enforcement point:** emit-side (app refuses) + ingest-side (node refuses) — both, with
   the schema policy as the single source of truth.

## 12. Migration plan (no big bang)

1. **protocol:** add `Signal`, `Location` (extended), `ModalityClass`, `ConfirmedEvent` v0.2, domain
   tags v2; ship `seismic.accel.v1` + `seismic.pick.v1` schemas; add conformance vectors.
2. **node:** dual-stack ingest (v0.1 `Observation` ⇒ normalize to `Signal{seismic.pick.v1}`);
   consensus/locate/magnitude read the typed payload; emit `ConfirmedEvent` v0.2 with `tier`.
3. **adapters:** emit `Signal{seismic.pick.v1}` (the per-station identity work already done carries
   over unchanged).
4. **mobile:** emit `Signal`, verify `ConfirmedEvent` v0.2 (incl. attestation count for `ALERT`
   tier); the on-device detector becomes one modality producer.
5. **dashboard / data:** store by `modality_class` + `schema_id`; the lake partitions on them.

Deprecate v0.1 once all producers emit v0.2 and the conformance suite is green.

---

*This is the keystone of the platform. Everything else — multi-modal sensors, the agent layer,
research API, multi-hazard apps — assumes this envelope exists. Review and challenge before code.*
