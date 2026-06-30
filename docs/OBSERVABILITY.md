# Observability

Phase-1 baseline (roadmap [Issue #1](https://github.com/EarthNet-EEW/earthnet/issues/1)).
**You cannot operate a network you cannot measure** — the node and relay expose Prometheus
metrics; a local Prometheus + Grafana stack is in [`../monitoring/`](../monitoring/).

## Run

```sh
# with a node (:8080) and relay (:8090) running
docker compose -f monitoring/docker-compose.yml up -d
# Prometheus → http://localhost:9090   ·   Grafana → http://localhost:3001 (anon admin)
```

## Metrics

### Node — `GET :8080/metrics`
| Metric | Type | Meaning |
|---|---|---|
| `earthnet_observations_ingested_total{source_type}` | counter | verified observations ingested |
| `earthnet_events_emitted_total{evidence}` | counter | ConfirmedEvents emitted (official/consensus) |
| `earthnet_ingest_errors_total{kind}` | counter | rejects: `decode` / `signature` / `bad_fields` |
| `earthnet_ingest_seconds` | histogram | ingest handler latency (the node-local slice of pick→alert) |
| **`earthnet_persistence_dropped_total`** | counter | **records dropped because the async persistence channel was full** — was invisible before; a non-zero rate means the DB writer is backlogged |

### Relay — `GET :8090/metrics`
| Metric | Type | Meaning |
|---|---|---|
| `earthnet_relay_events_forwarded_total` | counter | events fanned out |
| `earthnet_relay_subscribers` | gauge | live WebSocket subscribers |
| `earthnet_relay_messages_sent_total` | counter | messages delivered to subscribers |
| **`earthnet_relay_lagged_total`** | counter | **slow subscribers that lagged and dropped events** — the signal that a single-instance broadcast relay is saturating |
| `earthnet_relay_ingest_errors_total{kind}` | counter | rejects: `decode` / `signature` |

## What to watch

- **`persistence_dropped_total` rate > 0** → DB writer can't keep up (raise channel capacity / batch / shard). This is the first thing that breaks under load.
- **`relay_lagged_total` rate > 0** → the relay is saturating; time for the scalable mesh (NATS/MQTT, roadmap Phase 2/4).
- **`ingest_errors{kind="signature"}`** spiking → possible spoofing / a misconfigured producer.

## SLO

The project lives by **pick → alert latency**. The end-to-end budget is **p99 < 2 s**.
`earthnet_ingest_seconds` measures only the node-local slice; full end-to-end requires
**distributed tracing** (OpenTelemetry adapter→node→relay→app), which is the next observability
slice (not yet implemented). Until then, alert on the node histogram and the two drop counters
above.

## Not yet (follow-ups)

- OpenTelemetry distributed tracing (end-to-end pick→alert spans).
- A curated Grafana dashboard JSON (provisioned), per-region health, and Alertmanager rules.
- Mobile/adapter client-side metrics.
