# PRD-001 — Product Requirements Document

| Field | Value |
| --- | --- |
| Document ID | PRD-001 |
| Title | Server-Side Hierarchical Spatial Filtering Engine for 3D Digital Twin Visualization |
| Version | 0.1.0 |
| Status | Draft |
| Date | 2026-09-14 |
| Owner | Project maintainer |
| Related documents | Downstream: TDD-001 (Technical Design), PLN-001 (Master Execution Sequence). This document is self-contained. |

## 1. Purpose

This document specifies **what** the project must deliver and **how success is judged**. It is the authoritative, self-contained statement of product requirements and acceptance criteria; it does not prescribe implementation. Downstream design and execution documents trace back to it.

The core product is a **Rust server engine**: the real-time spatial filtering and streaming layer for visualization, and one component of a digital-twin stack rather than a complete platform. It keeps a large site's telemetry current and, for each viewer, decides per frame what to send, what to blend into group readings, and what to drop. The design follows one central rule: **the browser draws; the server thinks.** All heavy data work happens once per frame on the server, shared across viewers; the browser only renders a small, relevant slice.

A synthetic simulator, a browser 3D client, and a demo application are demonstration packages that exercise the engine end to end, forming a reproducible showcase in which an ordinary browser renders a live, smooth 3D "digital twin". Benchmarking covers three tiers — a single building (~4,280 devices), an intermediate site (50,000), and a full park (100,000) — while the demonstration application targets the building tier (~4,280 devices).

## 2. Problem Statement

On a large industrial site, up to a hundred thousand continuously reporting devices are each tied to a fixed physical location. At full scale the site produces approximately **300,000 readings per second (~30 MB/s raw)**. A viewer is interested only in the currently visible subset, and a screen can absorb only a small fraction of the stream.

Both obvious approaches fail:

- **Send everything** — overloads the browser and stalls the view.
- **Send slow summaries** — yields a stale view and loses short-lived events.

The site therefore produces roughly a hundred-fold more telemetry than the engine streams to a viewer (K5). The product must reduce the stream, per viewer and per frame, to what is visible and meaningfully changed, while preserving spatial context and live accuracy.

## 3. Product Goals

The project's committed objectives define five product goals:

| ID | Goal | Verified by |
| --- | --- | --- |
| G1 | Sustain the full telemetry stream continuously (≥ 300,000 readings/s). | K1 |
| G2 | Keep every viewer's rendering workload small and constant regardless of site size. | K3, K6 |
| G3 | Keep group summaries exactly accurate. | K4 |
| G4 | Cut network traffic by at least 99% at full scale and 98.5% at building scale versus raw streaming. | K5 |
| G5 | Serve both a single building (~4,280 devices) and a full park (100,000 devices) with the same untouched engine. | §12 (item 4), §9 |

## 4. Users and Personas

| Persona | Description | Primary needs | Serves |
| --- | --- | --- | --- |
| Operator / Analyst | Watches the live 3D site view in a browser and inspects devices. | Smooth 60 FPS, spatial context, click-to-inspect live values, no stale data. | FR-VS-01–FR-VS-08, FR-CD-01–FR-CD-06, K6, K7 |
| Integrator / Developer | Configures site topology and runs the simulator, engine, client, and demo application. | Simple config format, reproducible run order, clear logs and metrics. | FR-TD-01, FR-TD-09, FR-CD-07, FR-CD-08, FR-CD-09 |
| Reviewer / Reproducer | Independently reproduces the benchmark results. | Reproducible benchmark host, scripts, and a benchmark report. | FR-BR-01–FR-BR-04, K1–K11 |

## 5. System Context

```
 [Topology config: shared by simulator, engine, and client]
             │
             ▼
 [Simulator: ~100k devices, ~300k EPS]
             │ readings
             ▼
 SERVER (Rust engine), shared across all viewers
   ├─ keeps the latest reading per device metric
   ├─ groups devices by place (e.g. site → building → room → rack → device)
   ├─ blends readings of groups too small on screen
   └─ selects only what each viewer is looking at
             │  visible slice + selected values
             │  ≤ 5,000 active entries/frame/viewer
             ▼
 BROWSERS (WebGL): draw at 60 FPS, click-to-inspect
             │
             ├─ camera + screen geometry, every frame ─► server
             └─ selected object id, on selection change ─► server
```

Each browser sends its camera position and screen geometry to the server every frame, and the identity of its selected object on selection change, closing the feedback loop that drives server-side visibility selection, blending, and targeted inspection (FR-VS-01, FR-VS-02, FR-CD-04, FR-CD-06). The simulator, the engine, and the client all load the same topology definition, so device and metric identities, positions, units, and thresholds align (FR-TD-09).

The benchmark harness and report complete the deliverables (see §13).

## 6. Functional Requirements

Requirement IDs use `FR-<area>-<nn>`, where `<area>` is one of `TD` (topology and data), `IS` (ingestion and state), `AG` (aggregation), `VS` (visibility and selection), `TR` (transport), `CD` (client and demonstration), or `BR` (benchmarking and reporting).

Priority: **M** = must have, **S** = should have.

### 6.1 Topology and Data

| ID | Requirement | Priority | Source |
| --- | --- | --- | --- |
| FR-TD-01 | The system shall load a site hierarchy — a tree rooted at one site, composed of group nodes (conventionally building → room → rack) with devices only as leaves. Any level may be omitted, and a device attaches to exactly one group, at any level. | M | G5, R1 |
| FR-TD-02 | The system shall support up to 100,000 devices in a single hierarchy. | M | G5, K8 |
| FR-TD-03 | Each device shall have a fixed position and one or more metrics. | M | G5 |
| FR-TD-04 | Each metric shall have a stable identity and declare its unit, noise threshold, freshness timeout, and the aggregate channels it contributes to; multiple metrics may share a unit when their identities differ. | M | G3, K5 |
| FR-TD-05 | The topology shall define a set of aggregate channels — at most 16 — declared once and computed for every group. Each channel shall have a stable identity, a unit, an aggregation reduction from a fixed set (sum, mean, min, max, count), and a noise threshold, and may receive contributions from multiple metrics of its unit. | M | G3 |
| FR-TD-06 | Each group may exclude any of its channels from what it contributes to its parent, so that an ancestor's channel reflects only the sub-groups that contribute to it; every group contributes all of its channels by default. | M | G3 |
| FR-TD-07 | The system shall validate the topology at boot as a single-rooted, acyclic tree in which every device is a reachable leaf with a unique identity. Groups may be empty. It shall refuse to serve on structural errors and report each fault with enough detail to locate it. | M | G5, R1 |
| FR-TD-08 | The system shall validate that metric identities are unique within each device, that each metric identity denotes one quantity and unit throughout the topology, and that every channel's unit matches each metric that contributes to it. It shall refuse to serve on violation. | M | G5, R1 |
| FR-TD-09 | The topology definition shall be the single source shared by the simulator, the engine, and the client. | M | G5 |
| FR-TD-10 | The system shall serve both per-viewer visibility queries and per-group aggregate reads from the site hierarchy, becoming ready to serve within the boot-time budget (K8) and serving queries within the per-frame budgets (K2, K9). | M | G2, G3, K2, K9 |

### 6.2 Ingestion and State

| ID | Requirement | Priority | Source |
| --- | --- | --- | --- |
| FR-IS-01 | The system shall accept a continuous stream of readings and retain exactly one current value per device metric; the most recently received value wins, regardless of its sample timestamp, and no history is retained. | M | G1 |
| FR-IS-02 | The system shall discard invalid readings — those naming an unknown device or metric identity, or carrying a value that is not a finite number valid for that metric — without disturbing existing values, and shall make each accepted reading visible to the next frame. Invalid readings shall not refresh a device's liveness. | M | G1, G3 |
| FR-IS-03 | The system shall track availability per device metric: a metric's value is current only while a reading for it has arrived within that metric's configured freshness timeout, and a stale metric shall be treated as unavailable without affecting the availability of the device's other metrics. | M | G3 |
| FR-IS-04 | The system shall report a device offline when an explicit offline signal is received or when all of its metrics are unavailable, and return it online when a metric reading is next accepted; viewers shall be able to distinguish offline devices and unavailable metrics from live ones. | M | G3 |
| FR-IS-05 | The system shall ingest the offered stream continuously with bounded memory and without unbounded buffering or stalling; at the full-scale tier (100,000 devices) it shall ingest the generated stream at ≥ 300,000 readings/s for ≥ 30 minutes with no unbounded memory growth. | M | G1, K1 |

### 6.3 Aggregation

| ID | Requirement | Priority | Source |
| --- | --- | --- | --- |
| FR-AG-01 | For every group and aggregate channel, the system shall compute one combined value that exactly reflects the channel's configured reduction applied to the group's available contributing readings. | M | G3 |
| FR-AG-02 | An aggregate channel shall combine only readings of its own unit and never combine quantities of different units. | M | G3, R3 |
| FR-AG-03 | Group aggregates shall update once per frame within the aggregation latency target (K9); adding viewers shall not increase aggregation cost, and all viewers of a frame shall observe a consistent snapshot. | M | G2, R5, K9 |
| FR-AG-04 | A group's channel shall include only available readings; unavailable (stale or offline) values shall be excluded. A channel with no available contributing readings shall report no value rather than a value derived from stale or absent data. | M | G3 |
| FR-AG-05 | The aggregate published for a frame shall include every reading accepted before that frame began. | M | G3 |

### 6.4 Visibility and Selection

| ID | Requirement | Priority | Source |
| --- | --- | --- | --- |
| FR-VS-01 | The system shall accept each viewer's camera position and screen geometry (width, height, pixel ratio, field of view) every frame. | M | G2 |
| FR-VS-02 | The system shall accept each viewer's selected object identity on selection change and, while selected, include that object's current values in the viewer's stream every frame until deselection. | M | G2 |
| FR-VS-03 | For each viewer, the system shall determine which devices and groups are visible from the viewer's camera, within the visibility query latency target (K2). | M | G2, K2, R5 |
| FR-VS-04 | For each viewer, the visibility result shall be a set of individual device instances and blended group entries. | M | G2 |
| FR-VS-05 | When a group would appear on screen smaller than the configured on-screen height, the system shall show the group's aggregate channel values instead of its individual devices. | M | G2 |
| FR-VS-06 | The system shall avoid visible flicker when a group crosses the on-screen-size threshold; the threshold band is configurable (default enter 120 px, exit 100 px). | M | G2, R7 |
| FR-VS-07 | The system shall present no more than 5,000 active entries per viewer per frame in the worst case (K3), where an entry is an individual device instance or a blended group entry. When the visible region would exceed that bound, the system shall represent the coarser groups by their aggregate channel values, showing in detail those groups with the largest on-screen size. | M | G2, K3 |
| FR-VS-08 | When a viewer's camera context is absent or late, the system shall reuse the last known context until a configurable timeout, after which it shall hold that viewer's updates. | M | G2 |

### 6.5 Viewer Transport

| ID | Requirement | Priority | Source |
| --- | --- | --- | --- |
| FR-TR-01 | The system shall stream device and group updates to each connected viewer independently over a persistent session, one session per viewer. | M | G4 |
| FR-TR-02 | The system shall send each viewer only changes relative to the values last sent to that viewer, suppressing changes below the applicable noise threshold — each metric's for a device instance, each channel's for a group entry — so that steady-state egress stays within K5 while each displayed value remains within its threshold of the current value. Availability changes and a viewer's selected object (FR-VS-02) shall be transmitted regardless of the threshold. | M | G4, K5 |
| FR-TR-03 | On connect or reconnect, a viewer shall receive a complete, consistent snapshot of its visible set before incremental updates, and shall be able to recover from missed updates. | M | G2, R7 |
| FR-TR-04 | The system shall keep per-session state bounded and shall not accumulate unbounded backlog for a slow client; such a client receives the most recent state rather than every intermediate update. | M | G2, K10 |

### 6.6 Client and Demonstration

| ID | Requirement | Priority | Source |
| --- | --- | --- | --- |
| FR-CD-01 | The browser client shall render the visible device instances and blended group entries as a live 3D scene that distinguishes unavailable/offline devices from live ones, without allocation-driven stalls. | M | G2, K6, K7 |
| FR-CD-02 | The client shall render at a stable 60 FPS with main-thread overhead < 3 ms per frame. | M | G2, K6 |
| FR-CD-03 | The client shall colour each rendered object by one selected value — a metric for a device instance or an aggregate channel for a group entry — using a deterministic default and allowing the viewer to change the selection. | M | G2, K6 |
| FR-CD-04 | The client shall let a viewer pick any rendered object — an individual device instance or a blended group entry — and shall publish the selected identity and each deselection to the server. Only currently rendered objects can be picked; a device that is not rendered must be brought into view first. | M | G2 |
| FR-CD-05 | While an object is selected, the client shall display its current value(s) and unit plus its availability status, refreshing every frame regardless of how little they change, and shall return to normal streaming when deselected. | M | G2 |
| FR-CD-06 | The client shall publish camera position and screen geometry every frame. | M | FR-VS-01 |
| FR-CD-07 | A synthetic telemetry simulator shall emit device-level readings for the shared topology at configurable rates up to full-scale load, with per-metric value models including noise, steps, spikes, and offline transitions, and deterministic traces for reproducible benchmarks. | M | G5 |
| FR-CD-08 | A demo application shall integrate simulator → engine → browser into an end-to-end runnable showcase at the building scale (~4,280 devices). | M | G5 |
| FR-CD-09 | The engine and demonstration packages shall surface status, errors, and run metrics sufficient to diagnose a failing run. | M | G5 |

### 6.7 Benchmarking and Reporting

| ID | Requirement | Priority | Source |
| --- | --- | --- | --- |
| FR-BR-01 | A benchmark harness shall measure all KPIs (K1–K11), with backend KPIs (K1–K5, K8–K10) on a fixed benchmark host and browser-facing KPIs (K6, K7, K11) in the demo application on a mid-range laptop. | M | G5, K1–K11 |
| FR-BR-02 | Benchmarks shall cover building scale (~4,280), intermediate scale (50,000), and full scale (100,000) devices, plus a concurrent-session run. | M | G5 |
| FR-BR-03 | Ablation studies shall isolate the contribution of individual optimisations to measured performance. | S | R1, R5 |
| FR-BR-04 | A benchmark report shall record measured results against every KPI target, with the method, configuration, and host documented so the results are reproducible. | M | G5 |

## 7. Non-Functional Requirements (KPI Targets)

These are the measurable acceptance targets. Throughput is quoted in readings per second (EPS). Backend targets are measured on the fixed benchmark host across all three tiers; browser-facing targets (K6, K7, K11) are measured in the demo application on a mid-range laptop at the building scale (~4,280 devices). The K5 baseline is defined in §7.1.

| KPI | Metric | Target | Verified by |
| --- | --- | --- | --- |
| K1 | Sustained ingestion throughput | ≥ 300,000 EPS for ≥ 30 min with stable memory | Stress run at 100k |
| K2 | Visibility query latency | Server-side: p99 ≤ 1.5 ms per viewer per 60 Hz frame at 100k, across all camera positions, including max zoom-out; excludes encode, transport, and client | Engine instrumentation |
| K3 | Bounded visible workload | ≤ 5,000 active entries/frame worst case (device instances + blended group entries); ~1,000 typical | Per-frame visible-set log |
| K4 | Aggregate correctness | Group aggregates match an exact reference; per-unit separation enforced | Unit + property tests |
| K5 | Steady-state egress | ≤ 0.3 MB/s/client at 100k (≥ 99% cut); ≤ 0.05 MB/s at 4,280 (≥ 98.5% cut), for the benchmark topology's configured channel set | Network accounting |
| K6 | Client frame rate | Stable 60 FPS, main-thread overhead < 3 ms/frame | Browser profiler |
| K7 | Client memory | No monotonic heap growth in steady state; no allocation-driven frame spikes | Browser memory timeline |
| K8 | Boot time | ≤ 30 s from topology file to fully loaded and serving at 100k devices | Boot timer |
| K9 | Group-cluster aggregation | Baseline path ≤ 5 µs per channel per group with up to 1,000 contributing readings (a rack, for example); optimised path ≤ 1 µs (stretch) | Micro-benchmark + ablation |
| K10 | Multi-client scaling | ≥ 10 concurrent clients at 60 Hz, ≤ 50% aggregate server CPU, per-client egress within K5 | Concurrent-session benchmark |
| K11 | Update freshness | p99 ≤ 100 ms from a reading accepted by the engine to the rendered change in the client, at 60 Hz and the building scale (~4,280 devices) | End-to-end instrumentation |

Aggregation cost and steady-state egress scale with the number of channels actually used (at most 16).

Latency KPIs measure distinct intervals: K2 covers engine-side visibility computation only, excluding encoding, transport, and client; K11 is end-to-end, from a reading accepted by the engine to the rendered change in the client.

### 7.1 Baseline Representation (K5)

K5's cuts are measured against a raw-streaming baseline: every reading forwarded to every viewer, unprocessed. For reproducibility, the baseline reading representation is fixed as a full-key JSON object:

```json
{"device_id":"rack-07-node-4213","metric":"temperature","value":23.75,"unit":"C","ts":1757587200123}
```

At ~100 bytes per reading, the baseline follows each tier's aggregate reading rate (the benchmark topology also fixes the metric set per device; state size is the number of device-metric pairs):

| Tier | Devices | Metrics/device | Aggregate readings/s | Raw baseline |
| --- | --- | --- | --- | --- |
| Full park | 100,000 | ~3 | ~300,000 (~3 per device/s) | ~30 MB/s |
| Building | 4,280 | ~3 | ~37,000 (~8.6 per device/s) | ~3.7 MB/s |

At full scale, 300,000 readings/s × 100 B ≈ **30 MB/s** — an unoptimised but realistic baseline, not a worst case. The building tier's higher per-device rate reflects denser high-frequency instrumentation (for example, vibration and power metrics sampled at several hertz), not a larger payload; the per-reading size is identical across tiers.

The measured cut is sensitive to the baseline's format: a compact binary event stream (~12–18 B/reading, ~4–5 MB/s) would imply roughly a 92–94% reduction to the same 0.3 MB/s egress. The baseline is fixed explicitly so the figure is reproducible, and so that the reduction reflects server-side selection, LOD blending, compression, and change suppression rather than the choice of serialisation library.

The engine's byte-level representation of the same values, including exact wire widths, is an implementation detail fixed in the technical design.

## 8. Interfaces and Data Contracts (Product Level)

| Interface | Direction | Content |
| --- | --- | --- |
| Topology config | file → simulator / engine / client | Site hierarchy, device positions, per-metric identity, unit, noise threshold, freshness timeout, and channel contributions; aggregate channel definitions; per-group channel contributions. |
| Telemetry ingest | simulator / devices → engine | Device identity, metric values, and online/offline status at the configured rate. |
| Camera context | browser → engine, every frame | Camera position, screen width/height, pixel ratio, field of view. |
| Inspection target | browser → engine, on selection change | Identity of the rendered object selected for inspection, and deselection. |
| Streamed updates | engine → browser, every frame | Changes, availability, and snapshots for visible device instances and blended group entries (aggregate channel values). |

Exact wire layouts and the configuration schema are implementation details outside this document; this document fixes only the interfaces and their semantics.

## 9. Constraints and Assumptions

- **Language/runtime:** engine in Rust; browser client in Three.js.
- **Browser capability:** a WebGL-capable browser on a mid-range laptop is the rendering target.
- **Transport:** WebSocket, one persistent connection per viewer.
- **Scope tiers:** 4,280 / 50,000 / 100,000 devices are deliberately demanding and testable figures, not measurements from a field study. All three tiers are in scope — the benchmark harness exercises all three, while the demonstration application targets the building tier (~4,280 devices) — and the full-park (100,000-device) tier ships rather than being deferred.
- **Scale unit:** scale is counted in devices (the leaves of the hierarchy); memory and compute also depend on metrics per device and the reading rate (see §7.1).
- **Benchmark host:** a single fixed host, specified before benchmarking begins.
- **Measurement environment:** benchmark runs place the simulator, engine, and harness on the fixed host with browsers on the same local network; the demonstration runs the engine and browser together on the mid-range laptop; wide-area network behavior is out of scope.
- **Topology stability:** the topology is fixed at boot and is not modified while serving.
- **LOD granularity:** level-of-detail depends on intermediate groups; a flat hierarchy still meets K3 but collapses to large blended entries when zoomed out.
- **No production deployment** is assumed.

## 10. Out of Scope

- Integration with real field protocols/drivers (OPC-UA, MQTT, Modbus).
- Long-term telemetry persistence, historical replay, or analytics beyond current state.
- Statistical reductions beyond the supported set (for example median, percentiles, or standard deviation).
- Multiple sites or cross-site federation; the topology is a single-rooted tree (FR-TD-01).
- Alerting, rule engines, anomaly detection, or stored warning/event history; the engine signals only live device state, including offline.
- Occlusion (devices hidden behind other geometry); only in-view selection is performed.
- Authentication, authorization, and multi-tenant security hardening.
- Mobile or native (non-browser) clients.
- Cloud deployment and horizontal scaling; the product is a reproducible showcase, not a production deployment (see §9).

## 11. Product Risks

| ID | Risk | Primary mitigation |
| --- | --- | --- |
| R1 | Spatial structure built incorrectly, yielding wrong visible or aggregate results | Independent correctness reference, property tests, and structural invariant checks |
| R2 | Optimised path produces wrong values | Cross-check the optimised path against a straightforward reference; ship the simple path first |
| R3 | Aggregates computed over the wrong descendants | Automated invariant checks and adversarial/permutation tests |
| R5 | Query latency exceeds its budget | Bound per-frame work; reduce work early; share computation across viewers |
| R6 | Simulator starves server CPU | Provision the demonstration so the simulator cannot starve the engine |
| R4 | Silent client rendering failures | Early rendering spike and a fallback rendering path |
| R7 | Flicker at group transitions | Smooth transitions around the size threshold |
| R9 | Misconfigured topology or channels reach serving | Boot validation refuses to serve on structural or channel/unit errors (FR-TD-07, FR-TD-08) |
| R8 | Schedule slippage | A fixed cut-down ladder, defined before execution |

## 12. Acceptance Criteria

The product is accepted when:

- All Must functional requirements are demonstrated; any requirement without a dedicated KPI is verified by test.
- All eleven KPIs (K1–K11) meet their targets on the fixed benchmark host and demo laptop, as recorded in the benchmark report; the browser KPIs (K6, K7, K11) are demonstrated at the building scale (~4,280 devices).
- The §13 deliverables exist and are reproducible.
- The same engine binary passes both the 4,280-device and 100,000-device benchmarks without code change (G5).
- The end-to-end demo runs the documented sequence simulator → engine → browser.

## 13. Deliverables

- **The filtering engine.**
- **Three demonstration packages:**
  - **Synthetic device simulator.**
  - **Browser 3D client** with click-to-inspect.
  - **Demo application** integrating simulator → engine → browser.
- **The benchmark harness.**
- **A benchmark report** covering all three tiers (4,280 / 50,000 / 100,000 devices) and the concurrent-session run.
