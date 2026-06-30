# earthnet-protocol v0.1 — BORRADOR

> Estado: **borrador para revisión** (no implementado). Cierra las decisiones que el schema fuerza.
> Licencia del protocolo: **Apache-2.0**. Servicios: AGPL-3.0. Mobile: GPL-3.0.
> Una vez aprobado → mover a repo `earthnet-protocol` y arrancar Fase 0.

## Principios de diseño del wire format
- **Dos mensajes, dos confianzas:** `Observation` (pick crudo firmado, baja confianza) y `ConfirmedEvent` (post-fusión, dispara alarma, alta confianza).
- **Todo firmado Ed25519.** La pubkey (32B) ES la identidad anónima. Sin registro, sin PII.
- **Nada en el hot-path espera a la DB.** El wire format es autocontenido y verificable sin lookups.
- **Privacidad por defecto:** ubicación de teléfonos como geohash truncado; nunca waveform/audio crudo (solo features).
- **Versionado explícito:** `protocol_version` primer campo en cada mensaje.

---

## Esquema de firma (normativo)

```
signing_payload = domain_tag || deterministic_encode(message with signature = empty)
signature       = Ed25519_sign(privkey, signing_payload)   # 64 bytes
```

- `domain_tag`: ASCII, sin null terminator.
  - `Observation`  → `"earthnet-obs-v1"`
  - `ConfirmedEvent` → `"earthnet-evt-v1"`
- `deterministic_encode`: protobuf con serialización determinística (campos por tag ascendente,
  repeated packed, **sin campos desconocidos** → si llega un unknown field, la verificación falla).
- Verificación: poner `signature` en vacío, re-encodear determinístico, anteponer `domain_tag`, `Ed25519_verify`.
- **Pre-1.0:** cualquier cambio de schema ⇒ bump de `protocol_version`. No hay promesa de compat de firma en 0.x.

---

## earthnet.proto (v0.1)

```proto
syntax = "proto3";
package earthnet.v1;

// Ubicación con control de precisión para privacidad.
// Teléfonos: geohash de 5 chars (~±2.4 km). Estaciones oficiales: exacto (público).
message Location {
  string geohash    = 1;  // base32 geohash
  uint32 precision_m = 2; // precisión declarada en metros (info para el consumidor)
}

enum SourceType {
  SOURCE_TYPE_UNSPECIFIED = 0;
  SOURCE_TYPE_OFFICIAL    = 1; // estación oficial vía adapter (alta confianza, dispara sola)
  SOURCE_TYPE_PHONE       = 2; // teléfono (baja confianza, requiere consenso >=N)
}

// Pick crudo firmado emitido por un sensor (teléfono o estación oficial).
message Observation {
  uint32     protocol_version = 1;  // = 1 en v0.1

  bytes      observation_id   = 2;  // 16B aleatorios, para dedup
  bytes      pubkey           = 3;  // Ed25519 pubkey del emisor (32B) = identidad

  SourceType source_type      = 4;
  string     source_id        = 5;  // id de estación oficial (ej. "CSN:GO01"); vacío si teléfono

  // Tiempo: UTC nanosegundos. La sincronización de reloj es crítica para early warning.
  int64      captured_at_ns   = 6;  // wall-clock UTC ns del pick
  uint32     clock_uncert_ms  = 7;  // incertidumbre estimada del reloj (NTP/GNSS)

  Location   location         = 8;

  // Features del pick. NUNCA waveform crudo ni audio. Solo derivados.
  float      sta_lta_ratio    = 9;   // ratio del detector STA/LTA
  bool       p_wave_detected  = 10;
  float      estimated_pga    = 11;  // PGA estimado (g), opcional (0 si N/D)
  float      reported_magnitude = 12; // solo fuentes oficiales EEW; 0 si N/D

  bytes      signature        = 15;  // Ed25519(64B) sobre signing_payload (ver arriba)
}

enum EvidenceKind {
  EVIDENCE_KIND_UNSPECIFIED = 0;
  EVIDENCE_KIND_OFFICIAL    = 1; // disparado por fuente oficial
  EVIDENCE_KIND_CONSENSUS   = 2; // disparado por consenso de teléfonos
  EVIDENCE_KIND_BOTH        = 3;
}

// Evento confirmado post-fusión. DISPARA la alarma en el cliente.
message ConfirmedEvent {
  uint32   protocol_version = 1;  // = 1 en v0.1

  bytes    event_id         = 2;  // 16B
  bytes    pubkey           = 3;  // pubkey del nodo que confirmó (32B)

  int64    origin_time_ns   = 4;  // tiempo de origen estimado del sismo (UTC ns)
  int64    issued_at_ns     = 5;  // cuándo el nodo emitió este evento (UTC ns)

  Location epicenter        = 6;
  float    depth_km         = 7;  // opcional, 0 si N/D
  float    magnitude        = 8;
  float    magnitude_uncert = 9;

  EvidenceKind evidence     = 10;
  uint32   num_observations = 11; // cuántas observations alimentaron la fusión
  repeated bytes obs_ids    = 12; // observation_id de los picks usados (auditoría)

  bytes    supersedes       = 13; // event_id previo que este revisa/reemplaza; vacío si primero

  bytes    signature        = 15; // Ed25519(64B) sobre signing_payload
}
```

---

## Notas de diseño
- **`captured_at_ns` / `origin_time_ns` en int64 UTC ns:** el cliente calcula la cuenta regresiva
  S-wave localmente (epicentro + origin_time + su propia ubicación). El cliente NO envía ubicación para esto.
- **`supersedes`:** los EEW oficiales revisan magnitud/epicentro en segundos; el cliente actualiza la cuenta
  regresiva sin re-disparar la alarma.
- **`obs_ids`:** trazabilidad para el dashboard y análisis de falsos positivos. No se usa en hot-path.
- **geohash-5 para teléfonos:** suficiente para "N teléfonos correlacionados en una región" (consenso v1);
  la localización fina viene de feeds oficiales. Tradeoff documentado; revisar en v1.1 (detección en teléfono).
- **Tag 15 reservado para `signature`** en ambos: queda en 1 byte de tag y separa firma del payload semántico.

## Decisiones abiertas (no bloquean v0.1, anotar)
- Rotación de claves / reputación por pubkey → diseñar en Fase 2 (endurecimiento P2P/Sybil).
- Transporte (QUIC/libp2p framing) → spec separado; el wire format de mensajes es independiente del transporte.
- Compresión de batches de Observations en el relay → optimización posterior, medir primero.
```
