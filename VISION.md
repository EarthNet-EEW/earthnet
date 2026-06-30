# EarthNet — Vision

> **EarthNet is the open infrastructure layer for distributed geophysical intelligence:**
> a worldwide network of **heterogeneous sensors** — phones, IoT, dedicated stations,
> satellite, weather — that, through **distributed AI and continuous learning**, detects,
> correlates, and lets researchers **discover** physical phenomena in real time.
>
> It does **not** replace official systems. It adds an open layer of correlation, discovery,
> and experimentation on top of them. It is to geophysics what BOINC was to computing, and
> what IP was to networks: **a narrow waist anyone can build on.**

---

## What this is (and what it is not)

EarthNet is **not** "an earthquake alert app." Earthquake Early Warning (EEW) is the
**flagship application** that proves the substrate — not the product itself.

The product is a **planetary sensing substrate**:

- **Open** — open source, open data, open API, open to researchers and organizations.
- **Multi-modal** — built for many signal types, not just seismic.
- **Heterogeneous** — millions of phones + IoT + dedicated stations + satellite + weather.
- **Distributed AI** — edge, regional, and global intelligence with continuous learning.
- **Discovery, not just classification** — find correlations and anomalies *not yet known*.
- **Complementary** — augments official agencies (CSN, USGS, JMA, INGV…); never replaces them.
- **Privacy-first** — anonymous identities, signed data, FFT-only (never raw audio/imagery).

EarthNet should evolve into a platform for **Planetary Intelligence** — a foundation on which
researchers and organizations build new models and experiments, rather than a single product
with a single purpose.

## The gap we fill

Every piece exists somewhere. The **combination does not**:

| Effort | Scale | Open? | Multi-modal? | Research platform? |
|--------|-------|-------|--------------|--------------------|
| Google Android Quake Alerts | planetary | ❌ | ❌ (seismic) | ❌ |
| MyShake (Berkeley) | large | ❌ | ❌ (seismic) | ❌ |
| ShakeAlert / JMA / SASMEX | regional | ❌ | ❌ (seismic) | ❌ |
| Raspberry Shake | community | ~ | ❌ (seismic) | ~ |
| Safecast | community | ✅ | ❌ (radiation) | ~ |
| BOINC / Folding@home | global | ✅ | — (compute, not sensing) | ✅ |
| **EarthNet** | **planetary** | **✅** | **✅** | **✅** |

The unoccupied position is the intersection of *open + multi-modal + heterogeneous sensors +
distributed AI + continuous learning + anomaly **discovery** + research API + complement-not-
replace*. A closed giant cannot follow us into **openness and heterogeneity** — that is the moat.

## The BOINC lesson (precisely)

BOINC didn't win because of "distributed computing." It won because of three things, and each
has a geophysical translation:

1. **A trivial volunteer client + a credit/gamification system → retention.** For EarthNet this
   is not UX polish; it is **the fuel**. Without node retention there is no network. Credit for
   uptime and data quality is literally what keeps the infrastructure alive.
2. **A "project" framework** so any scientist could run SETI@home, Rosetta@home, etc. on the
   same infra. EarthNet's equivalent: an **experiment framework** where a researcher deploys a
   detector or correlator that **consumes the substrate (live stream or replay) without touching
   the core**, sandboxed, with attribution of discoveries.
3. **A clean separation** of volunteer infrastructure from science.

The difference — and why it's harder and more valuable — is that BOINC distributes *idle CPU*.
EarthNet distributes **sensing + edge compute + real time + location- and privacy-bound data**.
Nobody built it because it's hard, not because it isn't worth it.

## The architectural soul: a narrow waist, and two planes

Two principles govern every future decision.

**1. Narrow waist (like IP).** A **thin, stable, universal substrate** — the signed multi-modal
*Signal* envelope + a replayable durable backbone + correlation primitives + the open API — with
**rich diversity above** (hazard apps, experiments, models) and **below** (any sensor). The
fatal mistake would be to put hazard-specific logic in the core. *The core does not know about
earthquakes; it knows about signed signals, location, time, and correlation.*

> Governance metric: **if adding a new hazard requires changing the protocol or the node, we
> failed.** It should be: register a schema + deploy an experiment.

**2. Two planes with different SLAs.**

- **Research plane** (the main product, the "BOINC"): open, rich, experimental, best-effort.
  Many sources, replay, third-party models, discovery.
- **Alert plane** (an *application* that consumes the substrate, not the center): hard real-time,
  conservative, **multi-node attested**, clearly labeled as *complementary to official sources*.

> Iron rule: **a researcher's experimental model must NEVER trigger a public evacuation alert
> without passing the conservative, attested gate of the alert plane.**

Conflate the planes and you get both an unreliable alert system and a constrained research
platform. Keep them separate and you get two good products.

## Scientific rigor is the moat, not the AI

"Discover unknown geophysical anomalies" is scientifically seductive and a graveyard: radon,
ionosphere, animal behavior, earthquake lights — decades of non-reproducible "precursors" that
burned credibility. EarthNet's defensible moat is being **the platform that makes discoveries
falsifiable and reproducible**:

- rigorous, continuously-learned baselines per modality and region,
- pre-registration of hypotheses,
- reproducible replay of any result,
- statistical honesty and open peer review,
- the mantra: **we complement, we do not predict.**

This is product, not bureaucracy. Credibility is the only asset a life-safety network has.

## Principles

- **Latency is king** in the alert plane (pick → alert < 1–2 s). Deterministic, no ML in the
  alert hot path. AI enriches and discovers **off** the hot path.
- **Privacy is structural**, declared per modality in the schema (never raw audio/imagery).
- **Decentralized trust**: no single node can forge an alert (multi-node attestation).
- **Open by default**: data, code, API, models, governance.
- **Complement, never replace** the official systems.

---

## Visión (Español)

> **EarthNet es la capa de infraestructura abierta de inteligencia geofísica distribuida:** una
> red mundial de **sensores heterogéneos** —teléfonos, IoT, estaciones dedicadas, satélite,
> clima— que, mediante **IA distribuida y aprendizaje continuo**, detecta, correlaciona y permite
> **descubrir** fenómenos físicos en tiempo real.
>
> **No reemplaza** a los sistemas oficiales: les agrega una capa abierta de correlación,
> descubrimiento y experimentación encima. Es para la geofísica lo que BOINC fue para el cómputo
> y lo que IP fue para las redes: **una cintura angosta sobre la que cualquiera construye.**

**No es "una app de alerta sísmica".** El Early Warning es la **aplicación insignia** que valida
el substrato, no el producto. El producto es un **substrato planetario de sensado**: abierto,
multi-modal, heterogéneo, con IA distribuida, orientado al **descubrimiento** (no solo
clasificación), **complementario** a las agencias oficiales y **privado por diseño**.

**El hueco que llenamos:** cada pieza existe por separado (Google tiene escala pero cerrada y
mono-hazard; MyShake teléfonos pero cerrado; BOINC abierto pero de cómputo, no de sensado). La
**combinación** —abierto + multi-modal + sensores heterogéneos + IA distribuida + aprendizaje
continuo + **descubrimiento** de anomalías + API de investigación + complementar-no-reemplazar—
**no la tiene nadie**. Un gigante cerrado no puede seguirnos a la **apertura y la
heterogeneidad**: ahí está el foso.

**El alma arquitectónica:** (1) **cintura angosta** —un substrato fino, estable y universal (el
sobre de señal firmado multi-modal + backbone con replay + API abierta) con diversidad rica por
encima (hazards, experimentos) y por debajo (cualquier sensor); el core **no sabe de
terremotos**. (2) **Dos planos**: el de **investigación** (abierto, experimental, best-effort) y
el de **alerta** (tiempo real duro, conservador, atestado multi-node, complementario). *Un modelo
experimental nunca puede disparar una evacuación sin pasar la compuerta conservadora del plano de
alerta.*

**La lección de BOINC:** retención del nodo (créditos/gamificación) **es el combustible**;
framework de experimentos para investigadores; separación infra/ciencia. Nosotros distribuimos
**sensado + cómputo edge + tiempo real**, no CPU ociosa — más difícil, más valioso.

**El rigor científico es el foso, no la IA.** "Descubrir anomalías desconocidas" es un cementerio
de pseudociencia (radón, ionosfera…). Nuestro diferencial es ser **la plataforma que hace los
descubrimientos falsables y reproducibles**: baselines rigurosos, pre-registro de hipótesis,
replay reproducible, honestidad estadística, peer review abierto. **Complementamos, no
predecimos.** La confianza es el único activo de una red que salva vidas.

---

See [`docs/ARCHITECTURE-REVIEW.md`](docs/ARCHITECTURE-REVIEW.md) for the full technical analysis
and the 6-phase roadmap toward Planetary Intelligence.
