# Contributing to EarthNet

Latin America is underserved by earthquake early warning. A few seconds of warning is the
difference between being caught off guard and getting to safety. If you can help, you're
welcome here.

## Where we need hands

- **Rust** — the protocol, node (fusion/consensus), and relay.
- **Flutter / Dart** — the Android app (UX, background reliability, alerting).
- **Seismology / Python (ObsPy)** — detection tuning, GMPE calibration, new country/network
  adapters, backtesting against real catalogs.
- **DevOps** — Docker Compose full-stack, wss/TLS, deployment, CI.
- **Frontend / TS** — the dashboard.

## Ground rules (non-negotiable)

These are the project's guardrails — PRs that break them won't merge:

- **Latency is king.** Nothing in the pick → alert path may block on I/O or the DB.
- **Deterministic in the alert path.** STA/LTA + band-pass, no ML between a pick and an alert.
- **Privacy by design.** Anonymous identities, signed events, never raw audio (FFT only),
  never PII in the clear.
- **`OFFICIAL` is for real agency feeds only.** Our own waveform detections go through
  consensus (N stations must agree).
- Don't weaken tests to make them pass — fix the root cause.

## Getting started

1. Read [`README.md`](README.md) and [`DESIGN.md`](DESIGN.md).
2. Pick a repo (see the table in the README) — each has its own README, tests, and CI.
3. Run the local stack (Quickstart in the README), make your change, keep CI green.
4. Open a PR with a clear description and the reasoning. Small, focused PRs merge fastest.

## Good first issues

- Add a country/network adapter (see `earthnet-adapters/earthnet_adapter/network_adapter.py`).
- Improve the app's background-service reliability on more Android OEMs.
- Widen the GMPE magnitude calibration with a broader catalog.
- Docker Compose that brings up node + relay + TimescaleDB + dashboard in one command.

Questions or ideas? Open an issue. Thanks for helping build open early warning. 🌎
