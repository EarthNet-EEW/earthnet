# EarthNet — Diseño (Etapas 1–6)

> Documento de trabajo. Capturado durante el diseño guiado (proceso de 10 etapas). Etapas 1–6 cerradas;
> 7–10 quedan pendientes (ver final). Lenguaje de trabajo: español; README público en inglés.

## Contexto del proyecto
- **Qué es:** red/app OSS de **alerta temprana de sismos** (early warning) para LatAm. NO predicción, NO rescate (en MVP).
- **Quién:** Gastón, en solitario, OSS bien público, sin deadline, autofinanciado. Proyecto pendiente hace 20 años.
- **Independiente** del ecosistema Develone.

---

## Etapa 1 — Problema
En regiones sísmicas de LatAm con poca/nula cobertura de alerta temprana pública, no existe una red **abierta, independiente y resistente** que dé segundos de aviso. Las existentes (Google, ShakeAlert, MyShake) son centralizadas/cerradas o no cubren ciertas zonas.

**Verdad física rectora:** los segundos de aviso tienen **techo físico**, no tecnológico. El aviso útil = diferencia de tiempo entre que la onda P llega a un sensor lejano y la onda S te llega a vos. Sobre el epicentro ≈ 0 seg. El valor está en zonas a decenas/cientos de km de la falla.

## Etapa 2 — Objetivos
- **Primario:** una **app que funciona** (no solo un estándar).
- **Foco:** **ganar segundos de alerta temprana** (detección + difusión rápida).
- **Métricas:** latencia end-to-end medible; tasa de falsos positivos medible; protocolo versionado; validación de campo.
- **No-goals v1 (secuenciados, NO abandonados):** rescate, tsunami, federación global, research experimental, LoRa/mesh, ESP32 dedicado, ML pesado.

## Etapa 3 — Usuarios
| # | Usuario | Rol | MVP |
|---|---------|-----|-----|
| U1 | Ciudadano en riesgo | Recibe alerta; su teléfono es sensor | 🟢 Primario |
| U2 | Operador de nodo/relay | Densifica la red (VPS/Raspberry/teléfono viejo) | 🟡 Fase 2 |
| U3 | Contribuidor OSS | Desarrolla/extiende | 🟡 Transversal |
| U4 | Científico / Defensa Civil / ONG | Consume datos, valida, adopta | 🔵 Fase 3 |

## Etapa 4 — MVP: "El primer aviso real"
Cadena end-to-end híbrida en zona acotada:
```
FUENTES (oficial + teléfonos) → CONFIRMACIÓN (consenso) → DIFUSIÓN (baja latencia) → ALERTA (cuenta regresiva)
```
**Entra en v1:**
1. Ingesta de **feeds oficiales real-time** (solo feeds EEW/waveform; NO catálogo con delay de minutos).
2. **Detección local en teléfono** (STA/LTA + P-wave) → entra en **v1.1** (v1 = feeds oficiales primero).
3. **Fusión + consenso ligero**: dispara si viene de fuente oficial confiable, o ≥N teléfonos cercanos correlacionados.
4. **Difusión de baja latencia** con estimación de segundos hasta la onda S (cuenta regresiva).
5. **Alerta multimodal accesible** (sonido + vibración + visual + segundos).
6. **earthnet-protocol v0.1** (formato de evento firmado + transporte).

## Etapa 5 — Arquitectura
**Principio:** todo gira alrededor de un **evento sísmico firmado** que viaja lo más rápido posible. La red es una **federación de nodos independientes**; ninguno obligatorio; local-first (P2P/LAN/BLE como fallback offline).

```
   FUENTES                      CONFIRMACIÓN            DIFUSIÓN           CLIENTE
 Sismógrafos oficiales ─SeedLink/FDSN─►┐
                                       ├─► earthnet-node ─► earthnet-relay ─gossip/sub-seg─► earthnet-mobile
 Teléfonos (Edge STA/LTA) ─obs firmada─┘   (country adapters,        (fan-out, libp2p)      (alerta + cuenta
                                            fusión + consenso)         P2P/BLE offline        regresiva; sensor;
                                                   │                                          relay opcional)
                                            ConfirmedEvent ─► earthnet-dashboard (ops)
```

**Modelo de confianza:** fuente oficial = alta (dispara sola); teléfono = baja (necesita consenso ≥N). Dos mensajes: `Observation` (pick crudo firmado) y `ConfirmedEvent` (post-fusión, dispara alarma).

**Descentralización:** federación de Nodes+Relays que cualquiera levanta; teléfonos → relay(s) cercano(s); relays gossipean (libp2p/gossipsub); sin servidor central obligatorio.

**Presupuesto de latencia (objetivo end-to-end pick→alerta < 1–2 s):** detección <300ms · nodo/consenso <300ms · relay+red <500ms · render <200ms. Se mide, no se asume.

### Decisiones de arquitectura confirmadas
1. **Topología híbrida:** relays para velocidad + P2P (libp2p) como fallback/anti-censura.
2. **Primer adaptador: Chile (CSN)**; siguientes Argentina (INPRES), Perú (IGP).
3. **v1 = feeds oficiales → alerta**; **detección en teléfono = v1.1**.
4. **Rollout:** Sudamérica → Centroamérica → Norteamérica ⇒ **ingesta por país pluggable desde el día 1**.

## Etapa 6 — Stack
| Componente | Stack | Nota |
|---|---|---|
| earthnet-protocol | **Rust core** + **Protobuf** + firma **Ed25519** | core único reusado en mobile/node/relay |
| earthnet-mobile | **Flutter (Dart)** UI + **Rust core** vía `flutter_rust_bridge` | detección/cripto nativas |
| earthnet-node | **Rust** | fusión + consenso, en budget de latencia |
| earthnet-relay | **Rust** + **libp2p (gossipsub, QUIC)** | rust-libp2p |
| country adapters | **Python + ObsPy** (aislado tras el protocolo) | ObsPy = toolkit FDSN/SeedLink; acelera cada país |
| datos | **PostgreSQL+PostGIS** + **TimescaleDB** | FUERA del hot-path (persistencia async) |
| dashboard | **TypeScript + Next.js** | ops/monitoreo |
| infra | **Docker Compose** (NO k8s) | nodos en VPS barato/Raspberry |
| AI/ML | **nada en v1**; luego TinyML/ONNX on-device p/ P-wave + falsos positivos; Ollama solo research/ops | nunca en el path de aviso |

**Cortes respecto del manifiesto:** Go (Rust lo cubre, menos lenguajes para 1 persona), Kubernetes (Compose alcanza).
**Footprint:** Rust + Dart + TS + Python(adapters).

### Restricción dura (mobile)
Avisar a un teléfono **en <2s con pantalla apagada** ⇒ **FCM/APNs NO sirve** (lento/throttled). Se necesita conexión persistente: **Android foreground service** sosteniendo socket QUIC/libp2p. **iOS no permite** sockets en background ⇒ **Android-first**; iOS en fase posterior con alcance reducido.

### Decisiones de stack confirmadas (por defecto, al avanzar a guardar)
(a) cortar Go + K8s · (b) adapters en Python/ObsPy · (c) Android-first.
> Si alguna no te cierra, corregila en este doc antes de arrancar el desarrollo.

---

## Pendiente — Etapas 7–10 (a cerrar antes/durante el arranque)

### 7. Riesgos (semilla, a refinar)
- 🔴 **Entrega de alerta en background <2s** (el riesgo #1; ver restricción mobile). Mitigación: Android foreground service + QUIC persistente; medir en campo.
- 🔴 **Falsos positivos** → matan la confianza. Mitigación: consenso ≥N, fuentes oficiales con alta confianza, banco de pruebas.
- 🟠 **Latencia de feeds oficiales**: muchos son catálogo (minutos) y NO sirven para avisar. Verificar real-time/EEW por país.
- 🟠 **Cold-start / densidad de teléfonos**: mitigado por feeds oficiales (valor día 1).
- 🟠 **Sybil / nodos maliciosos**: firmas Ed25519, identidades anónimas, reputación, Sybil resistance.
- 🟡 **Sostenibilidad solo-dev**: alcance acotado, comunidad OSS, docs de onboarding.
- 🟡 **Responsabilidad legal** por avisos fallidos/no-avisos: disclaimers, no garantía, marco OSS.

### 8. Roadmap (borrador)
- **Fase 0:** earthnet-protocol v0.1 (schema + firma) + repos + licencias + CI.
- **Fase 1 (MVP):** node con adapter Chile (CSN) → relay → mobile (alerta + cuenta regresiva). Aviso real en Chile.
- **Fase 1.1:** detección en teléfono (Edge STA/LTA) como sensor + consenso.
- **Fase 2:** adapters Argentina/Perú; nodos comunitarios; dashboard; P2P endurecido.
- **Fase 3:** Centroamérica; federación; rescate/tsunami/research.

### 9. División en agentes (a definir en Etapa 9)
Un worker por repo: `W-protocol`, `W-mobile`, `W-node`, `W-relay`, `W-dashboard`, `W-adapters`. Orden: protocol → node+adapter Chile → relay → mobile.

### 10. Desarrollo
Recién al cerrar 7–9. Empezar por `earthnet-protocol` (todo depende de él).

---

## Cómo continuar desde otra consola
1. Abrí Claude Code en `C:\develone\earthnet`.
2. Leé `CLAUDE.md` (contexto y guardrails) y este `DESIGN.md`.
3. Cerrá Etapas 7–10 si querés, o arrancá Fase 0 (protocolo). No escribir código fuera del scope v1 sin revisar los no-goals.
