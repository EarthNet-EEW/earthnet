# 🌎🔴 EarthNet

**Open-source, decentralized earthquake EARLY WARNING for Latin America.**

> It does **not** predict earthquakes — that isn't a thing. EarthNet detects the fast,
> non-destructive **P-wave** near the epicenter and **races an alert ahead of the slow,
> destructive S-wave**, giving you **seconds to take cover**. Seconds save lives.

![status](https://img.shields.io/badge/status-working%20MVP-brightgreen)
![license](https://img.shields.io/badge/license-AGPL--3.0-blue)
![stack](https://img.shields.io/badge/Rust%20·%20Flutter%20·%20Python%20·%20TS-informational)

EarthNet is a **hybrid** network: it fuses **official public seismograph feeds** with a
distributed network of **smartphones** (each phone = sensor + client). **No mandatory
central server** — the network outlives its creators.

---

## How it works

```
seismograph / phone detects the P-wave   (STA/LTA + 2–8 Hz Butterworth band-pass)
        │  signs the observation          (Ed25519, anonymous identity)
        ▼
   earthnet-node  ── windowed phase association: N stations must agree on ONE hypocenter
        │           (a single noisy station never alarms — kills false positives)
        │  estimates epicenter · depth · magnitude (taup travel times + calibrated GMPE)
        ▼
   signed ConfirmedEvent ──▶ earthnet-relay (low-latency fan-out) ──▶ phones
        ▼
   app shows a LIVE countdown to the S-wave  +  expected shaking intensity at your location
```

The **countdown is the life-saving signal** (time to evacuate). Expected intensity is
informational — strongest at the epicenter, weaker with distance.

---

## Status — ✅ working MVP, verified end-to-end

Tested with **real Chilean seismograph data**, alerting on a physical Android phone:

- 🛰️ **284 public seismograph stations** across South America wired in (16 networks: Chile
  C1/C/CX, Ecuador, Colombia, Venezuela, Brazil + global backbone), streamed in real time.
- 📱 **On-device P-wave detection** on Android — the phone reuses the **shared Rust core**
  via `flutter_rust_bridge` (identical, deterministic picks; no ML in the alert path).
- 🤝 **Consensus by phase association** — verified live: 6 real stations triangulated an
  epicenter to ~7 km; isolated noise picks are silently dropped.
- 🔐 **Ed25519-signed events**, anonymous identities, reputation-weighted consensus.
- 📊 Dashboard with events, confirm latency, and sensor reputation.

---

## Repositories

Built in this order — `earthnet-protocol` is the spine everything depends on:

| Repo | What it is | Stack |
|------|-----------|-------|
| 🧬 [**earthnet-protocol**](https://github.com/devjamez/earthnet-protocol) | Signed event format, Ed25519 signing, the shared STA/LTA + band-pass detection core, taup travel-time tables | Rust · Protobuf |
| 🖥️ [**earthnet-node**](https://github.com/devjamez/earthnet-node) | Ingest → fusion → windowed phase-association **consensus** → magnitude (calibrated GMPE) → async TimescaleDB persistence | Rust · axum · sqlx |
| 📡 [**earthnet-relay**](https://github.com/devjamez/earthnet-relay) | Low-latency WebSocket fan-out of confirmed events to subscribers | Rust |
| 📱 [**earthnet-mobile**](https://github.com/devjamez/earthnet-mobile) | Android app: on-device detection, S-wave countdown, expected intensity, multimodal alarm | Flutter · Dart · Rust (FRB) |
| 🌐 [**earthnet-adapters**](https://github.com/devjamez/earthnet-adapters) | Country/network real-time ingestion: 284-station SeedLink registry, ObsPy detection, backtesting & GMPE calibration | Python · ObsPy |
| 📊 [**earthnet-dashboard**](https://github.com/devjamez/earthnet-dashboard) | Network health: events, latency, reputation | Next.js · TypeScript |
| 📖 **earthnet** (this repo) | Project home: design, protocol draft, decisions | Docs |

---

## Design decisions (the guardrails)

- **Latency is king.** pick → alert in **< 1–2 s**. Nothing in the hot path waits on the DB
  (persistence is fire-and-forget).
- **Deterministic first.** STA/LTA + band-pass, no ML in the alert path. AI only where it
  wins measurably and **never** between a pick and an alert.
- **Decentralized.** No mandatory central server; the network survives its creators.
- **Android-first.** iOS can't hold background sockets reliably → later phase.
- **Privacy by design.** Anonymous identities, Ed25519-signed events, location privacy,
  **never raw audio** (FFT only), never PII in the clear.
- **OFFICIAL ≠ our own detection.** Agency EEW feeds are trusted single-pick; our STA/LTA on
  public waveforms goes through **consensus** (N stations must agree).

**Stack:** Rust (core/node/relay) · Flutter+Dart (app) · Python+ObsPy (adapters) · Next.js
(dashboard) · PostgreSQL+PostGIS+TimescaleDB · Protobuf+Ed25519 · SeedLink/FDSN · Docker.

See [`DESIGN.md`](DESIGN.md) for the full architecture and [`PROTOCOL-v0.1-DRAFT.md`](PROTOCOL-v0.1-DRAFT.md) for the wire format.

---

## Quickstart (local stack)

```sh
# 1) protocol is a git dependency of node/relay/mobile — nothing to do, it's public.

# 2) node + TimescaleDB
git clone https://github.com/devjamez/earthnet-node && cd earthnet-node
docker compose up -d            # TimescaleDB
cargo run                       # node on :8080

# 3) relay
git clone https://github.com/devjamez/earthnet-relay && cd earthnet-relay
cargo run                       # relay on :8090

# 4) feed real seismographs (Chile + more)
git clone https://github.com/devjamez/earthnet-adapters && cd earthnet-adapters
pip install -r requirements.txt
python -m earthnet_adapter.network_adapter    # streams the 284-station registry

# 5) the app
git clone https://github.com/devjamez/earthnet-mobile && cd earthnet-mobile
flutter run                     # point it at ws://<host>:8090/subscribe
```

---

## Roadmap

- [x] Protocol, node, relay, mobile, adapters, dashboard — **working MVP**
- [x] On-device detection (Rust core via flutter_rust_bridge)
- [x] Multi-network real-time ingestion (284 SA stations)
- [x] Reputation-weighted consensus + calibrated magnitude
- [ ] wss/TLS + Docker Compose full-stack (one command)
- [ ] P2P gossip between relays (libp2p/gossipsub)
- [ ] Wider GMPE calibration · Play Store release
- [ ] Rollout: South America → Central America → North America

---

## Contributing

We need **Rust, Flutter, seismology, and DevOps** hands. Latin America is underserved by
earthquake early warning — this can change that. See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

Multi-licensed on purpose — copyleft for the network, permissive for the protocol:

| Component | License | Why |
|-----------|---------|-----|
| `earthnet-protocol` | **Apache-2.0** | Anyone can implement the wire format / signing freely |
| `earthnet-node` · `-relay` · `-adapters` · `-dashboard` | **AGPL-3.0** | Network copyleft — improvements to the running service stay open |
| `earthnet-mobile` | **GPL-3.0** | Copyleft for the client app |

This repo: [AGPL-3.0](LICENSE).
