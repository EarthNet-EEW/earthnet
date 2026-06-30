# EarthNet — Architecture Review & 5-Year Technical Strategy

*A CTO-level review of the current system and the path from Earthquake Early Warning to an open
platform for **Planetary Intelligence**. See [`../VISION.md`](../VISION.md) for the positioning.*

Status of this document: **strategic baseline**. It questions current decisions, compares
alternatives, and identifies debt, risk, and opportunity. It is not a backlog; the roadmap at the
end is the actionable distillation.

---

## 0. Executive verdict

What exists today is an **excellent EEW MVP as a product proof, but a prototype in topology — not
a distributed system**. Three uncomfortable truths first:

1. **It is not decentralized yet.** It is a classic star: sensors → **one** node → **one** relay →
   apps. The node is a monolith with in-memory state (single point of failure + hard scaling
   bottleneck). The libp2p/QUIC/gossip from the original stack is **not implemented**. The claim
   *"no central server / the network outlives its creators"* is currently aspirational.
2. **The data model is earthquake-specific, and that blocks the vision.** The `Observation`
   protobuf (`sta_lta_ratio`, `p_wave_detected`, `estimated_pga`, geohash) is cast for earthquakes.
   The multi-modal "Planetary Intelligence" future needs a **generic, typed Signal envelope**.
   Strategically this is the single most important change in this document.
3. **There is a single cryptographic trust root.** The node signs `ConfirmedEvent`s. A compromised
   node **forges alerts to the whole network**. For a "decentralized" life-safety network this is
   security gap #1; it requires **multi-node attestation (M-of-N threshold signing)**.

The good news — and it's substantial — is that the **low-level decisions are sound**: determinism
in the hot path, a **shared Rust detection core reused on node + phone** (single source of truth,
numerically verified against ObsPy), Ed25519-signed events, **consensus by phase association**
(not naive counting), reputation-weighted consensus, async persistence off the hot path. **The
problem is not code quality; it is that the topology and data model were designed for "an EEW app,"
not for "a planetary platform."** The migration is **evolutionary, not a rewrite**: the
deterministic core and the signed protocol are the correct foundation; what changes is everything
*around* them.

Two principles govern the whole strategy (from `VISION.md`):
- **Narrow waist** — a thin, stable, universal substrate; hazard logic lives in applications, never
  in the core.
- **Two planes** — a **research plane** (open, experimental, best-effort) and an **alert plane**
  (hard real-time, conservative, multi-node attested). They never mix.

---

## 1. Architecture: modularity, coupling, layering

**Strengths (do not break):**
- Clean separation into 7 components with independent CI.
- **Shared deterministic detection core** (`earthnet-protocol::detect`) reused by node and phone
  via flutter_rust_bridge — one definition of a "pick", with verified numerical parity to ObsPy.
  This is a rare, excellent pattern; preserve it.
- Deterministic hot path with fire-and-forget persistence (mpsc `try_send`) — respects latency.
- Pluggable per-network adapters from a station registry.

**Debt & coupling (what limits us):**
- **The node is a monolith with in-memory state.** Ingest + verify + fusion + locate + grid-search
  + magnitude + consensus + reputation + persistence + forward, all in one process, with
  observation buffers in RAM. Consequence: **no horizontal scale** (consensus state is local) and
  it is a **SPOF**. Phase association coupled to in-memory buffers is the coupling that prevents
  sharding.
- **Single-instance relay, in-memory subscribers.** No clustering. Practical ceiling ~tens of
  thousands of WS connections per box.
- **Reputation in a TSV file + DB mirror.** Fine for one node; with N nodes you need **replicated /
  shared reputation state**, or each node has a different view of trust.
- **No internal contracts beyond protobuf** and no messaging backbone → everything is direct
  point-to-point HTTP.
- **The earthquake-specific data model threads through everything** (protocol, node, app, adapters,
  DB) — the most expensive coupling to reverse if left to grow.

**Verdict:** high extensibility *within* a component, low *at the system level*. Adding a second
modality today would touch protocol + node + app + adapters + DB schema — the signal that a
**Signal-abstraction layer** is missing.

---

## 2. Scalability: 100 → 100M nodes, and where it breaks

| Scale | State | Primary bottleneck |
|---|---|---|
| **100** | ✅ works as-is | none real |
| **1,000** | ✅ works | ingest durability starts to matter (mpsc drops under bursts) |
| **100,000** | ⚠️ strained | **single WS relay** (connections), **node CPU** (grid-search per ingest, RAM buffers), **DB writes** (1 insert per observation) |
| **10,000,000** | ❌ breaks | single node **cannot** hold global buffers or run global association; star topology unviable; needs geographic sharding + streaming backbone + regional aggregation |
| **100,000,000** | ❌ different architecture | edge tier (local clusters doing pre-consensus), regional aggregation, **geo-distributed alert mesh (CDN-like PoPs, not fan-out from one box)**, data tiering to object storage |

**Concrete bottlenecks of the current design:**
1. **Centralized phase association in one process with RAM buffers** → not shardable without
   redesign. *#1.*
2. **Hypocenter grid-search in the ingest path** → CPU-bound; collapses at high pick rate. It
   should be a stream-processing stage, not inline in the handler.
3. **Relay = in-memory fan-out of one instance** → not clusterable.
4. **1 obs = 1 HTTP POST = 1 insert** → no batching, no real backpressure (drops), no durable queue.
5. **Local reputation** → inconsistent across nodes.

**Redesign principle: geographic partitioning as a first-class citizen.** Every observation
belongs to a cell (geohash of N chars). Cells map to partitions/nodes. Consensus is **local to the
cell + neighbors** (earthquakes are local). This turns a global O(N²) problem into many local O(k)
problems — the only way to reach 100M.

---

## 3. Is this really a distributed system?

**Today: no.** It is client-server with signed sensors. Missing to become one:
- **Discovery & membership** of nodes (gossip / DHT). Today: none.
- **State replication** (events, reputation) across nodes. Today: none.
- **Consensus between nodes** (not only between sensors within one node). Today: intra-node only.
- **Multi-node event attestation** (without it there is no decentralized trust).
- **Partition tolerance** (CAP): today if the node dies, the region goes blind.

**To question / what is superfluous:**
- The stack named **libp2p/gossipsub + QUIC** and never used it. **Decide consciously:** commit to
  real P2P (libp2p) or an operated mesh (NATS supercluster / geo-distributed MQTT brokers)?
  Recommendation: **hybrid** — operated backbone for durability/order (Kafka/NATS) + P2P **only at
  the edge for resilience when networks fail** (BLE/Wi-Fi Direct mesh between phones in a disaster
  zone, where infra collapses exactly when it's needed most). Pure P2P for everything is romantic
  and operationally expensive; P2P **where it buys survival** is correct.
- **Go and Kubernetes were cut.** At 1M+ scale, k8s (or Nomad) **becomes necessary again**.
  Revisit in Phase 4.

**What should change:** the unit of deployment stops being "the node" and becomes "a region"
(cluster of services + cell partition + replica). Today's node becomes the **regional/edge node**.

---

## 4. AI: is the architecture ready for research?

**State:** zero AI today (only deterministic STA/LTA) — **and that is correct for the hot path.**
The guardrail *"AI never between pick and alert"* is one of the project's best decisions; keep it
as law. AI lives on the **research plane**, off the hot path.

But the research platform you want **does not exist in any blueprint**. To add it without touching
the hot path:
- **Edge AI:** capability already proven (FRB runs native Rust on-device → can run **TFLite/ONNX**).
  Missing: an inference runtime, model distribution, a feature loop. Immediate use case: **local
  sensor fusion + anomaly detection** deciding *when it's worth sending data* (saves battery and
  bandwidth — critical at scale).
- **Anomaly detection / probabilistic models:** run **off-path** over streams. The deterministic
  event ships fast; models enrich/correct/discover afterward.
- **Continuous learning:** impossible today — no data lake, no labeling (**confirmed events are
  free, valuable labels**, currently dumped to a table with no pipeline), no feature store, no
  model registry, no retraining, no shadow deployment.
- **Federated learning:** the architecture is *compatible in spirit* (devices with data + compute +
  stable identity + privacy-first FFT-only fits FL perfectly) but everything is missing: model
  channel, local training, secure aggregation. Phase 5 — but the substrate design must **leave the
  door open** (stable per-device identity already exists ✅).
- **Multi-agent reasoning:** see §6.

**Verdict:** the deterministic base is right; what's missing is **the entire data + MLOps machinery
around it**, strictly off-path. "Add new AI prediction models" is marketing without a data lake +
feature store + registry + reproducibility.

---

## 5. Sensors: does the architecture allow multi-modal?

**Today: no, without redesigning the protocol.** This is **the strategic bottleneck.**

**Recommendation (the most important design decision here):** generalize to a **multi-modal Signal
envelope**:
- Common fields: signed identity, location (with uncertainty + privacy level), timestamp (with
  clock uncertainty), device class, **quality/SNR**, modality.
- Payload **typed per modality** via `oneof` or `bytes + schema_id`, with a versioned **schema
  registry** per modality (accel, gyro, mag, baro, gps, audio_fft, image_features, weather, iot,
  satellite…).
- The current seismic `Observation` becomes **one modality** of the new envelope.

With this, adding magnetometer/barometer/microphone(FFT)/camera(features)/weather/IoT/satellite is
**registering a schema + a detector** — no change to transport or consensus. **Without it, every
new modality is open-heart surgery.** We don't implement the sensors now, but **the generic
envelope must be designed in Phase 2** because it conditions everything else.

Privacy note: *"never raw audio, FFT only; never raw imagery"* must be **part of the schema
registry** (each modality declares what may be transmitted). This makes privacy **structural**, not
a promise — and it is what later enables research without betraying the user.

---

## 6. Agents: evolve toward multi-agent?

**Yes — it is the natural way to organize the multi-modal off-path** — but with discipline, not as
fashion. The proposed decomposition maps to a **stream-processing topology reframed as agents**:
each agent **consumes topics and produces topics**, decoupled by the event bus.

| Agent | Role | Consumes → Produces |
|---|---|---|
| **Motion** | Movement detection (today's `detect`) | accel signals → picks |
| **Sensor Fusion** | Combine modalities of one device | multi-signal → node state |
| **GPS Correlation** | Spatial co-movement, anti-spoofing | picks + location → validated clusters |
| **Weather** | Atmospheric context | weather feeds → features |
| **Satellite** | InSAR / thermal / other | sat feeds → anomalies |
| **Anomaly** | Unsupervised cross-modal detection | all streams → anomaly candidates |
| **Scientific** | Models/hypotheses, reproducibility | data → versioned findings |
| **Alert** | Decision and distribution (deterministic) | confirmed events → alerts |
| **Knowledge** | Maintains the **knowledge graph / world-state** | everything → platform memory |

**Serious architecture (not superficial):**
- Agents are **stream-native services** (not "LLMs chatting"). Most are specialized models/algorithms.
- **The Alert Agent and the deterministic path are NOT "intelligent" agents in the hot path** —
  they stay fast deterministic code. AI agents **enrich and discover off-path**; they may only
  *raise* confidence or *trigger investigations*, never *block or delay* a deterministic alert.
- Coordination via **event bus + the knowledge graph** (the Knowledge Agent is shared memory), not
  centralized orchestration (which would be another SPOF).
- **Risk to watch:** over-engineering and latency. Governance metric: *if an agent enters the
  critical alert path, it is misplaced.*

True **multi-agent reasoning** (LLM/planner) belongs in `Scientific`/`Anomaly` for **discovery and
hypotheses** (correlate a seismic swarm + magnetic anomaly + satellite thermal → "possible volcanic
reactivation, investigate"), not in real-time operation.

---

## 7. Mobile as a distributed node

**Today: sensor + client, not a node.** For millions of always-on sensors, **battery is the
product** (if it drains, they uninstall and coverage evaporates). Gaps:
- **No store-and-forward offline.** If the node is unreachable, observations are lost (there is a
  cooldown, not a queue). In a **disaster** — when it matters most — the network fails and the app
  goes silent. Critical.
- **No batching or compression** (1 pick = 1 POST). Inefficient on network and battery at scale.
- **No adaptive duty-cycling** (sampling by battery/charging/motion). Today fixed sampling, `detect`
  every 500 ms over a 12 s buffer.
- **No priority queues** (alert > observation > telemetry).
- **No local mesh** (BLE/Wi-Fi Direct) to propagate alerts phone-to-phone when there is no infra →
  a **survival differentiator** in a disaster zone.
- **iOS excluded** (correct today due to background sockets, but at planetary scale it's half the
  world — revisit with push/background processing in Phase 4).

**Recommendation:** turn the app into a **real edge node**: durable queue (SQLite/Drift) + batching
+ compression (protobuf/CBOR + zstd) + energy-aware duty-cycling + priorities + WorkManager for
reliability + **opt-in local mesh for disasters**. This is Phase 1–2 and it is where scale is won
or lost.

---

## 8. Backend: queues, streaming, transport (justified, no fashion)

**Today:** direct HTTP POST to the node + WS from a single relay + fire-and-forget mpsc persistence.
**It is a prototype, not a backbone.**

Per-layer recommendation, justified:
- **Edge ingest (millions of devices): MQTT** (EMQX/VerneMQ). *Over HTTP/WebSocket:* designed for
  massive constrained-device fleets, pub/sub, **QoS**, persistent/offline sessions, last-will,
  minimal overhead. HTTP-per-pick does not survive 1M devices. *Not AMQP/RabbitMQ:* built for
  enterprise messaging, not IoT-scale fan-in.
- **Durable processing backbone: Kafka** (or Redpanda). *Why:* durability + **replay** (gold for
  ML/backtesting/research — retrain over years of data), **partition by geohash** (= your geographic
  sharding for free), high throughput. The *replay* property is what enables the research platform.
- **Geo-distributed alert mesh: NATS (JetStream) / MQTT.** *Why NATS:* low latency,
  **geo-distributed superclusters** (fits the planetary mesh + PoPs), simpler ops than Kafka.
  Alerts go to *per-cell subjects* at PoPs near the user, not fan-out from one box. *Kafka vs NATS:*
  Kafka for the durable/analytic pipeline; NATS for real-time/edge — **each where it shines.**
- **WebSocket:** stays for the **app** (alert subscription, browser/mobile friendly), behind the
  scalable mesh (MQTT-over-WS or NATS gateway), not the current single relay.
- **Cache:** Redis for shared hot state (reputation, identity dedupe, rate limits, presence) —
  absent today and needed the moment there is >1 node.
- **Stream processing:** association/locate/anomaly move from "inline in the handler" to **stream
  stages** (Flink, or custom Rust processors over Kafka/NATS) — gets grid-search out of the ingest
  hot path.

**No-fashion justification:** nothing is added "just because." MQTT enters at ~10⁴–10⁵ devices;
Kafka when you need durability/replay/sharding (Phase 2); NATS when alert distribution must be
multi-region (Phase 4). Below those scales it is **over-engineering** and the current HTTP+WS is
fine for product validation.

---

## 9. Data: does it hold years of multi-modal data?

**TimescaleDB is a good choice** for hot time-series (hypertables for events/observations). But most
of the strategy is missing:
- **No retention policy, no continuous aggregates (downsampling), no native compression configured,
  no tiering.** The `observations` table explodes at scale.
- **No data lake.** Years of multi-modal data + waveforms + ML features need **object storage
  (S3/MinIO) + Parquet**, partitioned by modality/geo/time. TimescaleDB is the *hot tier*, not the
  archive.
- **No feature store, no model registry, no dataset versioning** (DVC/lakeFS) or model versioning
  (MLflow). Without these there is no reproducible "continuous learning."
- **Audit/provenance:** signatures give per-record provenance (good), but **event lineage** (which
  observations → which event, reproducible) and an **append-only audit log** are missing. For a
  network that fires public alerts, "why did this alert fire" traceability is a legal/scientific
  requirement.

**Recommendation:** a tiered data architecture — **hot** (Timescale, days/weeks, with compression +
continuous aggregates) → **warm/cold** (object storage + Parquet, years) → **feature store +
registry** for ML → append-only **audit/lineage**. The **confirmed events are the project's most
valuable labeled dataset**; today they are not captured with enough context to train. Phase 3.

---

## 10. Security: the most serious chapter

Good cryptographic foundations (Ed25519, domain-tag, 1-pubkey-1-vote, reputation-weighted consensus
with decay). But at adversarial scale there are grave gaps:

- **Sybil:** identity is **free and unlimited**. Phase-association consensus + reputation raise the
  cost *over time*, but **cold-start is weak** and a coordinated Sybil fleet with physically
  plausible picks can fabricate events. **Missing: proof-of-uniqueness/device** — Android Key
  Attestation / Play Integrity, iOS DeviceCheck/App Attest → bind identity to real hardware. Without
  it, "millions of nodes" is "millions of potential fake identities."
- **GPS spoofing:** **no defense.** Location is self-asserted inside the signed obs — an attacker
  signs any geohash. Needed mitigations: cross-check with network geo (cell/IP), **neighbor
  coherence**, sensor fusion (GPS+accel+baro must be consistent), and reputation weighted by
  *location-consistency history*. For consensus, require **spatial diversity that is hard to fake**
  without many physically distributed spoofers.
- **Fake data / malicious nodes:** phase association is the primary defense (good design); reinforce
  with **anomaly detection on submission patterns**, rate-limiting per identity/cell, and proof-of-
  location.
- **Alert manipulation / single trust root:** **gap #1.** One node signs events → compromise it =
  forge alerts to everyone. Solution: **M-of-N threshold signing** — an event is valid only if M
  independent nodes co-attest. This **removes the single trust root** and is what makes the
  "decentralized" claim honest.
- **Transport:** today `ws://` and `http://` in the clear. **wss/TLS everywhere, now** (Phase 1).
- **Model governance:** when AI enters, who approves a model that fires alerts. Needs governance +
  canary/shadow.

**Mental frame:** EarthNet is **critical infrastructure that triggers physical actions (evacuate)**.
The threat model is not "spam"; it is "an adversary who wants to cause panic or suppress a real
alert." That demands defense-in-depth: hardware identity + proof-of-location + spatial consensus +
multi-node attestation + audit.

---

## 11. Observability: today you are flying blind

Nearly nonexistent. `tracing` for some logs and `/health`, nothing else. Unviable at scale.
- **No metrics** (Prometheus): ingest rate, association latency, consensus fire rate, **false-
  positive rate**, DB lag, **how many observations the `try_send` drops** (you don't know today), WS
  connections, per-region health.
- **No distributed tracing** (OpenTelemetry) adapter→node→relay→app. Impossible to debug "why did
  this alert take 4 s."
- **No SLOs.** The project lives and dies by pick→alert latency; it should have a **monitored p99 <
  2 s SLO** with alerts when violated.
- The current "dashboard" is for **seismic events**, not **system health** — two different products.

**Recommendation (Phase 1):** OTel + Prometheus + Grafana + structured logs with correlation IDs +
SLOs. Cheap now, very expensive to retrofit. **You cannot operate a planetary network you cannot
measure.**

---

## 12. UX: node retention IS the product

Today the app is "receive alerts." For a network of millions of always-on sensors, **the core
product challenge is node retention** (uninstalls → lost coverage → the asset evaporates). UX must
sell *participation*, not just *consumption*:
- **Scientific participation:** "your phone helped detect today's M5.2," your contributions, impact.
- **Gamified node quality & uptime:** quality score, streaks, badges, regional leaderboards. **Not
  fluff:** gamifying "uptime + data quality" **directly increases coverage and density**, which
  improves accuracy (proven: more stations → more precise epicenter).
- **Network state:** live coverage map, your cell's density, "your area needs more nodes."
- **Research contribution:** opt-in to datasets, see discoveries.

This turns passive users into **a community that keeps the infrastructure alive** — the
Folding@home/SETI@home model applied to geophysics.

---

## 13. Technical debt / Risks / Opportunities

**Technical debt (by cost-to-reverse):**
1. Earthquake-specific data model (threads through everything). *Reverse it late = rewrite.*
2. Monolithic node with in-memory state (blocks sharding/HA).
3. Single cryptographic trust root (node signs alone).
4. No streaming backbone (direct HTTP).
5. Local, non-replicated reputation.
6. No observability.
7. No store-and-forward on mobile.
8. ws/http in the clear.

**Risks:**
- **Existential:** a security incident (a false panic alert, or suppression of a real one) destroys
  public trust — and trust is the only asset of an alerting system. Mitigate: multi-node attestation
  + audit.
- **Product:** battery drain → uninstall → coverage collapse.
- **Scale:** discovering bottlenecks in production at 100k instead of in design.
- **Scope:** attempting multi-hazard on the earthquake-specific model → repeated surgeries.
- **Scientific credibility:** overclaiming "precursors"/predictions → the pseudoscience graveyard.
  Mitigation is the rigor tooling in `VISION.md` ("complement, not predict").

**Opportunities (the differentiators):**
- The **shared deterministic core + edge compute (FRB)** is a rare, excellent base for edge AI.
- **Signed confirmed events = a labeled, provenanced, reproducible dataset** → a unique research
  asset.
- **Structural privacy (FFT-only) + federated learning** = a privacy-respecting AI story almost no
  one has.
- **Disaster mesh** (peer-to-peer alerts when infra fails) = a differentiator official networks
  don't offer.

---

## 14. Roadmap (6 phases)

**Phase 1 — Critical problems (harden the MVP, close the gaps that kill).**
wss/TLS everywhere · secrets management · **baseline observability** (metrics + tracing + SLOs) ·
**multi-node event attestation (threshold signing)** to remove the single trust root ·
**store-and-forward + batching + duty-cycling on mobile** · TimescaleDB retention/compression ·
make the "decentralized" claim honest. *Goal: reliable, secure production of the EEW app as-is.*

**Phase 2 — Architecture (become a platform and truly distributed).**
**Multi-modal Signal envelope + schema registry** · **geographic partitioning (geohash) as a
first-class citizen** · streaming backbone (**MQTT edge + Kafka durable**) · **decompose the node**
into stream-native services · **relay → scalable mesh (NATS/MQTT)** · replicated reputation
(Redis/shared state). *Goal: the planetary substrate, with EEW running on top unchanged. Establish
the narrow waist and the two planes.*

**Phase 3 — AI (the data machinery, off-path).**
Data lake (object storage + Parquet) + feature store + model registry + versioning · **capture
confirmed events as labels** · edge AI runtime (TFLite/ONNX via FRB) for local fusion/anomaly ·
anomaly detection + probabilistic models over streams · **agent framework** (the specialized agents
as stream processors) preserving the deterministic hot path. *Goal: a reproducible research
platform.*

**Phase 4 — Scalability (to 10M+).**
Edge clustering / **hierarchical consensus** · regional aggregation · **geo-distributed alert mesh
(PoPs)** · data tiering · **revisit k8s/Nomad** (cut earlier, now necessary) · load/chaos testing at
1M+ simulated nodes · revisit iOS. *Goal: survive growth without surprises.*

**Phase 5 — Research.**
Federated learning + secure aggregation · continuous learning · **multi-agent reasoning for
discovery** (cross-modal anomaly correlation) · open API/datasets for scientists · **experiment
framework** (researchers deploy models against stream/replay, sandboxed, with attribution) · new
models (tsunami, volcanic, structural, flood) as modules. *Goal: discover, not just detect.*

**Phase 6 — Planetary Intelligence.**
Full multi-modal platform · satellite/weather/IoT integration · anomaly-discovery engine · **global
knowledge graph** · marketplace of hazard applications · **network governance** (how a valid alert is
decided, who operates nodes, incentive model/DAO) · multi-hazard early warning as a planetary public
good.

---

## The one thing to keep nailed

EarthNet's success will **not** be decided by detector quality (already good). It will be decided by
three things that barely exist today: **(1) decentralized trust** (multi-node attestation — without
it you are not credible as a "network that saves lives"), **(2) the multi-modal data envelope**
(without it there is no "planetary," only "an EEW app with features"), and **(3) node retention**
(without it there is no network). If the next big decisions serve those three, you are heading
toward Planetary Intelligence. If they serve "a better EEW app," you stay in EEW.
