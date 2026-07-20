# technical architecture

Engineering reference for a **new, from-scratch** driver/traveler-assistance navigation application. This is **not** a plugin into Navit or any other existing navigation binary. The product owns its own core, map stack, routing, sensors, and a sandboxed plugin host.

**Implementation language:** Rust for core, plugin host, and plugin SDK.  
**Plugin sandbox:** WebAssembly (WASM), executed with a host such as **wasmtime**.  
**Portability:** The tree must build and run on multiple CPU architectures and OS targets (see [Multi-architecture builds](#multi-architecture-builds)).

Companion product-idea docs: [README.md](README.md), [PROTOCOLS.md](PROTOCOLS.md), [API.md](API.md), [APRS.md](APRS.md), [CAT.md](CAT.md).

---

## Table of contents

1. [Goals and non-goals](#goals-and-non-goals)
2. [Trust boundary](#trust-boundary)
3. [Multi-architecture builds](#multi-architecture-builds)
4. [System overview](#system-overview)
5. [Crate and module layout](#crate-and-module-layout)
6. [Thread and priority model](#thread-and-priority-model)
7. [Inter-thread communication](#inter-thread-communication)
8. [Core native modules](#core-native-modules)
9. [Plugin host architecture](#plugin-host-architecture)
10. [Capability-based host API](#capability-based-host-api)
11. [Plugin manifest](#plugin-manifest)
12. [Map layers and moving icons](#map-layers-and-moving-icons)
13. [Registering layers, POIs, and overlays from plugins](#registering-layers-pois-and-overlays-from-plugins)
14. [Planned plugin categories](#planned-plugin-categories)
15. [Future Marine Extensions (Not Yet Implemented)](#future-marine-extensions-not-yet-implemented)
16. [Data storage](#data-storage)
17. [Failure, fuel, and hang isolation](#failure-fuel-and-hang-isolation)
18. [Security non-goals and hard denials](#security-non-goals-and-hard-denials)

---

## Goals and non-goals

### Goals

- Own the full navigation stack: sensors, ECU/BMS, routing (rest stops, energy/eco, validation), maps, POI search, history, and UI.
- Keep **trusted code in-process native Rust**; keep **untrusted/optional features in WASM plugins**.
- Guarantee that sensor, ECU, and routing work cannot be stalled by plugins.
- Support product ideas already documented (travel modes, charging stations, APRS, CAT via Hamlib, elevation APIs) as core or as plugins where appropriate.
- Leave the plugin API generic enough for marine AIS, weather overlays, bathymetry, and sonar without core rewrites.
- Compile and run on **multiple architectures** (desktop, embedded, mobile-class SBCs) from one workspace.

### Non-goals

- Embedding into Navit, OsmAnd, Organic Maps, or similar as a binary plugin.
- Loading third-party **native** `.so` / `.dylib` / `.dll` plugins into the core address space.
- Giving plugins raw filesystem, raw sockets, or direct hardware device access by default.
- Real-time hard guarantees suitable for safety-certified vehicle control (this is an assistance / navigation app, not an ECU).

---

## Trust boundary

```text
+------------------------------------------------------------------+
|  TRUSTED CORE (native Rust, no sandbox)                           |
|  sensors | ECU/BMS | Hamlib CAT | routing | maps | POI | SQLite   |
|  plugin-host (wasmtime) | UI compositor                             |
+------------------------------------+-------------------------------+
                                     | capability ABI only
                                     | (versioned WIT / host calls)
                                     v
+------------------------------------------------------------------+
|  UNTRUSTED PLUGINS (WASM)                                          |
|  APRS decode helpers, AIS overlays, weather, bathymetry, sonar UI  |
|  no raw ptr, no host FS/HW unless capability granted in manifest   |
+------------------------------------------------------------------+
```

**Rule:** If a feature needs GPS device handles, SocketCAN, serial OBD, or Hamlib, it belongs in **core** (or a core-owned adapter). Plugins may consume **snapshots** and publish **overlay contributions** through the host API.

---

## Multi-architecture builds

The workspace must be portable across architectures. Design constraints:

| Concern | Approach |
|---------|----------|
| Targets | First-class: `x86_64` and `aarch64` on Linux; keep `armv7` / RISC-V where dependencies allow. Desktop macOS/Windows as secondary via the same Rust sources. |
| Toolchain | Stable Rust; `cargo build --target <triple>` in CI for at least `x86_64-unknown-linux-gnu` and `aarch64-unknown-linux-gnu`. |
| CPU features | Default builds avoid mandatory AVX/NEON-only paths; optional SIMD behind `#[cfg(target_feature = ...)]` or runtime detection. |
| Endianness | Persist little-endian on disk; decode wire protocols with explicit endian readers (OBD, J1939, HGT, CI-V). |
| WASM plugins | Plugins compile to `wasm32-wasip2` (or `wasm32-unknown-unknown` + custom ABI) **once**; the same `.wasm` artifact runs on every host arch under wasmtime. |
| Native deps | Prefer pure-Rust or widely packaged libs (SQLite via `rusqlite`/bundled, Hamlib optional feature). Feature-gate arch-specific stacks (e.g. SocketCAN on Linux only). |
| Cross-compile | Document `cross` or OS sysroots for Hamlib/libusb where needed; core logic must remain buildable with hardware features disabled for CI on foreign targets. |

```text
                    +---------------------------+
                    |  plugin .wasm (portable)  |
                    +-------------+-------------+
                                  |
         +------------------------+------------------------+
         v                        v                        v
  x86_64 Linux             aarch64 Linux              other hosts
  (wasmtime + core)        (wasmtime + core)          (same crate graph)
```

---

## System overview

```text
                         +-------------------+
                         |   app (UI shell)  |
                         +---------+---------+
                                   |
         +-------------------------+-------------------------+
         |                         |                         |
         v                         v                         v
  +-------------+          +---------------+          +--------------+
  | map render  |          | routing engine|          | plugin-host  |
  +------+------+          +-------+-------+          +------+-------+
         |                         |                         |
         |         +---------------+---------------+         |
         |         |         core services bus      |         |
         |         +---+------+------+------+-------+         |
         |             |      |      |      |                 |
         v             v      v      v      v                 v
      tiles/POI     GPS/IMU  ECU   CAT   SQLite         WASM instances
```

Core owns truth for position, vehicle energy, active route, and persisted trip/config state. Plugins subscribe to snapshots and push layer/POI/log contributions.

---

## Crate and module layout

Cargo workspace (illustrative):

```text
driver-break/
  Cargo.toml                 # workspace
  core/                      # trusted native library
    src/
      lib.rs
      sensors/               # GPS, IMU
      ecu/                   # OBD-II, J1939, MegaSquirt, BMS adapters
      radio/                 # Hamlib CAT wrapper
      routing/               # rest stops, energy/eco, validation
      map/                   # load, tiles, layers, moving icons
      poi/                   # R-tree / spatial index
      storage/               # SQLite schema, migrations
      time/                  # GPS/RTC offline elapsed
      bus/                   # channels, snapshots, priorities
  plugin-host/               # wasmtime embedder, fuel, manifests
    src/
      lib.rs
      runtime.rs
      capabilities.rs
      manifest.rs
      scheduler.rs           # lowest-priority plugin workers
  plugin-sdk/                # crates plugins link against (guest side)
    src/
      lib.rs                 # WIT/bindgen or thin ABI wrappers
      prelude.rs
  plugins/                   # optional first-party WASM plugins (examples)
    ais-overlay/
    weather-grib/
    bathymetry/
    sonar-feed/
  app/                       # binary: UI + wiring
    src/main.rs
  docs/                      # may mirror or point at repo-root *.md
```

| Crate | Privilege | Role |
|-------|-----------|------|
| `core` | Trusted | All hardware I/O, routing, maps, SQLite |
| `plugin-host` | Trusted | Loads WASM, enforces fuel/capabilities |
| `plugin-sdk` | Guest | Safe API surface for plugin authors (compiles to WASM) |
| `app` | Trusted | Process entry, windowing, config paths |

Plugins are **separate packages** that build to `.wasm` and are discovered at runtime from a plugin directory — never linked into `core` as native dynamic libraries.

---

## Thread and priority model

Priorities are process-local OS thread priorities (or an equivalent cooperative scheduler with strict budgets). Highest priority must never wait on lower tiers.

| Tier | Role | Priority | Notes |
|------|------|----------|-------|
| T0 | Sensors (GPS, IMU) | Highest | Sample and publish; no DB, no plugins |
| T1 | ECU / BMS | High | OBD / J1939 / MegaSquirt / BMS polls |
| T1b | Radio CAT (Hamlib) | High–medium | Short transactions; never inside sensor loop |
| T2 | UI / map render | High–medium | Consume snapshots; moving-icon redraw |
| T3 | Routing | Medium | Rest-stop, energy cost, validation; multi-core ok |
| T4 | SQLite / persistence | Lower | Serialized writer; async requests from others |
| T5 | Plugin host workers | Lowest | WASM instances; fuel + wall timeout |

```text
Priority (high → low)

  T0  Sensors --------+  never blocks on T3–T5
  T1  ECU/BMS --------+
  T1b CAT ------------+
  T2  UI/Render ------+---- read snapshots / bounded channels
  T3  Routing --------+---- may use N worker cores
  T4  SQLite ---------+---- queue; backpressure to requesters
  T5  Plugins (WASM) -+---- starved first under load
```

**Routing** may spawn a small pool of T3 workers for segment energy walks. **Plugins** never share those cores in a way that preempts T0–T2 under contention; the host schedules T5 only with leftover budget.

---

## Inter-thread communication

### Principles

- Prefer **message passing** over shared mutable state.
- Cross-thread “current state” is published as **immutable snapshots** (`Arc<Snapshot>`) or single-producer slots.
- No plugin-visible mutex that a WASM guest can hold across host calls in a way that blocks T0–T3.

### Channel types

| Channel | Pattern | Backpressure / drop policy |
|---------|---------|----------------------------|
| Sensor → bus | SPSC ring or `watch`/slot | **Drop-oldest** if consumer lags; latest fix always wins |
| ECU → bus | SPSC / watch | Drop-oldest; keep last valid fuel/SoC |
| Bus → UI | watch / broadcast | UI always takes latest frame state |
| Bus → routing | mpsc bounded | **Drop-newest** or coalesce “route dirty” flags if routing is busy |
| Anyone → SQLite | mpsc bounded request/response oneshot | Apply backpressure to caller; never block T0 waiting on commit |
| Core → plugin | snapshot push (copy into guest memory) | If plugin not ready, **skip tick** (drop) |
| Plugin → core | bounded contribution queue | Drop or reject if full; never block T0–T3 on enqueue from host side beyond a try_send |

### Snapshot contents (read-only to plugins)

Typical fields: timestamp, position, speed, course, travel mode, fuel/SoC summary, active route id/hash, viewport (bbox + zoom), offline-elapsed clock.

Plugins must not receive live file descriptors or mutable references into core heaps.

```text
T0/T1 producers          snapshot slot           T2/T3/T5 consumers
  (latest wins)  --->  Arc<WorldSnapshot>  --->  clone Arc / copy into WASM
```

---

## Core native modules

All of the following are **trusted native Rust** inside `core` (not sandboxed).

| Module | Responsibility |
|--------|----------------|
| **Sensors** | GPS (NMEA/gnssd/Android location), IMU fusion inputs; GPS time for RTC-less offline elapsed ([README.md](README.md)) |
| **ECU / BMS** | OBD-II (ELM327), J1939 (SocketCAN), MegaSquirt (serial); EV SoC/power adapters as they mature ([PROTOCOLS.md](PROTOCOLS.md)) |
| **Radio CAT** | Hamlib only ([CAT.md](CAT.md)); tune from parsed QRV frequencies |
| **Routing** | Rest-stop planning, energy/eco segment costs, route validation (forbidden roads, military areas, depth constraints when bathymetry layer present) |
| **Map load/render** | Basemap layers (OSM, Tracestrack Topo, CyclOSM, Cycle Map), tile cache, OSM POI icons |
| **POI search** | Spatial index (R-tree or equivalent) over OSM + runtime categories |
| **Storage** | SQLite for config, rest/fuel/charge history, adaptive fuel samples, APRS/AIS track cache owned by core policy |
| **Time** | Shutdown/boot offline duration; GPS preferred without RTC |

Elevation downloads (Copernicus / Viewfinder / SRTMGL1) are core services using [API.md](API.md) interfaces; progress jobs run at T3/T4 priority, not T0.

---

## Plugin host architecture

### Sandbox choice

- **WASM via wasmtime** (or equivalent bytecode sandbox with fuel).
- No `libloading` of native plugin DSOs into the core process.
- Each plugin instance: own store/engine limits, memory cap, fuel counter, epoch interruption for wall-clock timeout.

### Lifecycle

```text
discover *.wasm + manifest.toml
        ↓
verify version + required capabilities
        ↓
instantiate (deny-by-default host imports)
        ↓
call plugin_init(capabilities)
        ↓
on each schedule tick (T5):
    write snapshot → guest
    call plugin_on_tick / plugin_on_event
    harvest contributions (layers, POIs, logs)
        ↓
fuel exhausted or timeout → trap, unload or quarantine
```

### Scheduling

- Dedicated **lowest-priority** worker pool (size configurable, often 1–2).
- Per-tick **fuel** (wasmtime fuel) and **wall timeout** (e.g. few ms–tens of ms).
- Hung plugin: epoch interrupt, discard in-flight contributions, optional disable until restart.

### Isolation properties

| Threat | Mitigation |
|--------|------------|
| Plugin crash | Trap contained in instance; core continues |
| Infinite loop | Fuel + timeout |
| Memory bomb | Memory limit on store |
| Confused deputy | Capability list from manifest; host checks every call |
| Timing attack on sensors | Plugins never run on T0/T1 threads |

---

## Capability-based host API

Narrow, **versioned** ABI (WIT recommended; bindgen into `plugin-sdk`). Plugins import only what the manifest grants.

### Core capability groups

| Capability | Guest may | Guest may not |
|------------|-----------|---------------|
| `position:read` | Read snapshot lat/lon/speed/time | Write GPS device |
| `poi:query` | Spatial query in bbox/radius | Touch SQLite files |
| `poi:write` | Upsert POIs in **plugin-scoped** namespace | Overwrite core OSM POIs silently |
| `layer:contribute` | Register/update map layer descriptors + tiles/features | Replace basemap engine |
| `track:contribute` | Moving-icon tracks (APRS/AIS-like) | Access radio hardware |
| `log:write` | Structured logs at info/warn/error | Flood without rate limit (host rate-limits) |
| `data:overlay` | Register weather/bathymetry/sonar overlay handles | Bypass routing validation API |
| `route:annotate` | Suggest constraints/hints (min depth, avoid zone) | Force-commit route without core accept |
| `storage:plugin` | Key-value or blob store under plugin id | Read other plugins’ keys / core DB |
| `net:fetch` (optional) | Host-mediated HTTP(S) to allowlisted hosts | Raw TCP/UDP sockets |
| `sensor:subscribe` (optional) | Receive sonar/depth **as plugin-provided** streams registered earlier | Open `/dev/tty*` |

**Versioning:** `host_api = "1"` in manifest; host rejects incompatible majors. Additive minors allowed with feature detection.

### Illustrative host imports (conceptual)

```text
host::get_snapshot() -> Snapshot
host::poi_query(bbox, category_mask) -> [Poi]
host::poi_upsert(namespace, poi) -> Result
host::layer_register(LayerDesc) -> LayerId
host::layer_update_features(LayerId, FeatureBatch) -> Result
host::track_upsert(TrackId, TrackSample) -> Result   # moving icons
host::overlay_register(OverlayDesc) -> OverlayId
host::log(level, msg)
host::plugin_storage_get/set(key, bytes)
host::http_get(url) -> Result<bytes>   # only if net:fetch granted
```

No `malloc` into host heaps, no shared mutable pages, no `mmap` of core databases.

---

## Plugin manifest

Example `manifest.toml` next to the `.wasm`:

```toml
id = "example.ais-overlay"
name = "AIS marine traffic"
version = "0.1.0"
host_api = "1"
wasm = "ais_overlay.wasm"

# Deny by default; only listed capabilities are wired into the linker.
capabilities = [
  "position:read",
  "layer:contribute",
  "track:contribute",
  "log:write",
  "storage:plugin",
  "net:fetch",
]

[limits]
memory_pages = 256          # wasm pages (64 KiB each) — host-enforced cap
fuel_per_tick = 5_000_000
timeout_ms = 20
max_http_per_minute = 30

[net.allowlist]
hosts = ["api.example-ais.invalid"]

[ui]
# Optional: human description for capability approval screen
description = "Shows nearby vessels as moving icons."
```

First run (or capability change): **app UI prompts** for approval of requested capabilities. Denied caps are not imported; plugin must handle missing caps gracefully.

---

## Map layers and moving icons

### Basemap layers (core)

Owned by core map module; not plugins:

1. Standard OSM (default)  
2. Tracestrack Topo  
3. CyclOSM  
4. Cycle Map  

POI icons from OSM tag mapping (including public restrooms).

### Layer stack

```text
z-high  plugin overlays (weather raster, bathymetry contours, sonar HUD)
        plugin tracks (AIS vessels, optional APRS if plugin-hosted)
        core tracks (APRS if implemented in core — same TrackStore API)
        core POIs + rest/charge suggestions
        basemap tiles
z-low
```

### Moving-icon / track system (generalized)

One **TrackStore** in core supports any callsign/MMSI-keyed moving entity:

| Field | Use |
|-------|-----|
| `track_id` | Stable id (APRS callsign-SSID, AIS MMSI, …) |
| `kind` | `aprs`, `ais`, `other` |
| `position`, `course`, `speed`, `updated_at` |
| `symbol` / icon key | APRS table+code, AIS ship type → icon |
| `label_flags` | e.g. QRV `*` prefix |
| `detail_text` | Full message; shown only if zoom > 14 (APRS policy) |

**APRS** and **AIS** are both producers into `TrackStore` (core APRS ingest and/or plugins with `track:contribute`). Range policy for RF APRS remains hardcoded 50–150 km ([APRS.md](APRS.md)); AIS plugins may apply their own maritime range rules via contribution filtering before upsert, still subject to host rate limits.

---

## Registering layers, POIs, and overlays from plugins

Plugins extend the map **without modifying core code** by calling host registration APIs during `plugin_init` or later ticks.

### Map layer

1. `layer_register(LayerDesc { id, z_order, kind: Vector | RasterTiles | Contours })`  
2. Push data with `layer_update_features` or `layer_set_tile(url_or_blob)`.  
3. Core renderer composites by `z_order`.  
4. On plugin unload, host drops that `LayerId`.

### POI category

1. Declare category in manifest or via `poi_category_register("plugin.ns:marina")`.  
2. `poi_upsert` into plugin namespace.  
3. Core index inserts into the spatial R-tree with namespace tag; search filters can include/exclude plugin namespaces.  
4. Icons: plugin may supply icon bytes **or** map to an OSM icon key; core validates size/format.

### Data overlay (weather, bathymetry)

1. `overlay_register(OverlayDesc { kind: WeatherGrib | Bathymetry | SonarTrace | Generic })`.  
2. Plugin streams grids/tiles/polylines through host.  
3. Routing may query overlays through **core** helpers (`depth_at(lat,lon)`, `weather_hazard(lat,lon)`) implemented by core reading the last accepted overlay snapshot — plugins cannot be invoked synchronously from the routing hot path.

```text
plugin (WASM) --async contribute--> OverlayStore (core)
routing (T3)  ----read only------> OverlayStore snapshot
```

---

## Planned plugin categories

These are first-class **future** (or example) plugins; the ABI must not be APRS-specific.

| Category | Contribution type | Notes |
|----------|-------------------|-------|
| **Marine traffic (AIS-style)** | `track:contribute` + optional layer | Moving icons for vessels; same TrackStore as APRS; identity = MMSI |
| **Weather overlays** | `layer:contribute` / `data:overlay` | GRIB2 decode in WASM or host-mediated decode; radar/satellite tiles as raster layer |
| **Bathymetry** | `data:overlay` + route annotate | Depth/contours; core route validation may enforce min depth when overlay present |
| **Sonar** | plugin-owned realtime stream + overlay | Live/logged depth / fishfinder-style traces; **not** a core hardware module (optional/specialized); plugin supplies samples via approved capability, core displays/records under plugin storage |

APRS may live in core (RF/SDR/NMEA proximity to radio hardware) or as a plugin for network-only feeds; either way it uses the shared moving-icon path. CAT and ECU remain **core-only**.

---

## Future Marine Extensions (Not Yet Implemented)

> **Status: non-binding roadmap / design intent only.**  
> Nothing in this section is scheduled, specified for v1, or committed. It records *why* the core/plugin boundary should stay shaped the way it is (SocketCAN and CAT in core; overlays and domain cost functions as plugins) so later marine work does not force a redesign. **Do not treat this as current scope.**

Reasoning pattern (same as the rest of this document):

- **Core** — safety- or urgency-critical, or structurally identical to existing trusted telemetry / geofence / radio paths.  
- **Plugin** — domain-specific or optional data, overlays, or cost functions; no new plugin-host mechanism required, only new **capability declarations** in the manifest (and possibly new overlay/cost kinds on existing APIs).

### Core candidates

| Item | (a) Description | (b) Placement | (c) Why | Thread / priority tier |
|------|-----------------|---------------|---------|------------------------|
| **NMEA 2000 / NMEA 0183 instrument bus** | Ingest wind, speed log, depth sounder, autopilot heading from the boat instrument network. | **Core** | CAN-based path is structurally parallel to **J1939 over SocketCAN** already planned for ECU data; reuse the same trusted SocketCAN (and serial NMEA 0183) integration rather than exposing raw bus access to WASM. | Same tier as **ECU / BMS (T1)** |
| **MOB (Man Overboard) marking** | One-tap capture of the casualty position and immediate route-back / return guidance. | **Core** | Urgency-critical assistance; must not depend on plugin fuel, scheduling, or sandbox availability. | Same tier as **sensor thread (T0)** for mark capture; route-back calculation on **routing (T3)** using the frozen MOB fix |
| **Anchor watch / drag alarm** | Monitor live position against an anchor point plus swing radius; alarm on drag-out. | **Core** | Same geofence-style radius math and urgency class as building / glacier distance checks already owned by trusted routing/safety logic. | Position samples from **sensors (T0)**; watch evaluation on a trusted path at **T1 or T2** (never T5) |
| **Grounding / shallow-water alerts** | Proactive depth warnings from bathymetry (and/or sounder) vs configured vessel draft. | **Core** (alert engine) | Structurally like eco-routing config (`Cd`, mass): vessel draft is a core vehicle parameter; the *alert decision* is safety-critical. Bathymetry *data* may still arrive from a plugin overlay (see below); core reads the last accepted overlay snapshot, same as route validation must not call into WASM on the hot path. | Alert evaluation with **routing / safety (T3)**; sounder samples if from NMEA bus on **ECU tier (T1)** |
| **VHF DSC distress call integration** | Trigger or assist DSC distress via the radio, building on Hamlib CAT. | **Core** | Extension of existing **Hamlib CAT** ownership; hardware and regulatory sensitivity stay out of plugins. **Flag: complex / high-effort** (DSC protocol, certification, UX, failure modes)—roadmap only. | Same tier as **CAT / Hamlib (T1b)** |

### Plugin candidates

| Item | (a) Description | (b) Placement | (c) Why | Extends existing plugin category | Manifest capabilities (illustrative) |
|------|-----------------|---------------|---------|----------------------------------|--------------------------------------|
| **Nautical chart overlays** | S-57/S-63 vector charts or raster chart tiles as a basemap-adjacent layer, distinct from OSM styles. | **Plugin** | Domain-specific chart formats and licensing; optional for non-marine users. | **Map layer** (`layer:contribute`) | `layer:contribute`, `storage:plugin`, optional `net:fetch` |
| **Tide and current data** | Tide tables/predictions and current flow fields to pair with bathymetry for time-aware depth. | **Plugin** | Optional marine dataset; same overlay/store pattern as weather/bathymetry. | **POI / data overlay** (`data:overlay`) | `data:overlay`, `storage:plugin`, optional `net:fetch` |
| **Marine weather routing** | Wind/wave/current-optimized path costs as an alternate to terrain eco routing. | **Plugin** | Domain cost function plugged into the existing routing engine via annotate/cost hooks—not a second router in core. | **Routing cost function** (`route:annotate` / cost contribution) | `route:annotate`, `position:read`, `data:overlay` (read weather/current overlays) |
| **Marine-specific GRIB fields** | Wave height/period, swell direction, sea state on top of generic weather GRIB. | **Plugin** | Field-set extension of the generic **weather overlay** plugin category already planned. | **Data overlay / weather** (`data:overlay`, `layer:contribute`) | Same as weather plugin; no new host mechanism |
| **Storm / squall alerting** | Lightning or storm-cell tracking as a real-time overlay (and optional hints). | **Plugin** | Optional realtime meteorological overlay; not structurally required for land modes. | **Map layer + data overlay**; optional `route:annotate` for hints | `layer:contribute`, `data:overlay`, `log:write`, optional `net:fetch` |
| **TSS / no-wake / restricted zones** | Traffic separation schemes, no-wake, and restricted areas as avoid/constraint geometry. | **Plugin** (zone data) | Zone datasets are domain-specific; validation is analogous to military-area avoidance already in core—core applies constraints from an accepted overlay snapshot, same pattern as bathymetry depth. | **Data overlay** + core consumes via existing validation; plugin supplies geometry | `data:overlay`, `route:annotate` |
| **Boat-specific fuel burn curves** | Planing vs displacement hull consumption models feeding adaptive fuel / range. | **Plugin** | Hull-specific models are optional and domain-specific; core keeps adaptive learning *framework*, plugin supplies the cost/curve function. | **Routing / energy cost function** (adaptive fuel extension) | `route:annotate` and/or a future `energy:model` capability declaration only—no new sandbox type |

### Design implication (for implementers of v1)

When choosing SocketCAN, CAT, TrackStore, overlay snapshots, and `route:annotate` shapes **now**, prefer APIs that can later accept NMEA 2000 PGNs, MOB fixes, anchor geofences, draft-vs-depth checks, nautical layers, and marine cost functions **without** inventing a second plugin runtime. **Building these marine features is out of scope until explicitly scheduled.**

---

## Data storage

### Core SQLite (trusted)

| Data | Why in core |
|------|-------------|
| App config, travel mode, rest parameters | Single source of truth |
| Rest / fuel / charge stop history | Cross-feature timeline |
| Adaptive fuel / energy samples | Routing + range |
| Session state, last shutdown time | Offline elapsed |
| Elevation tile index metadata | Download jobs |
| Approved plugin capability grants | Security |
| Optional: APRS/AIS track cache with TTL | Shared TrackStore persistence policy |

### Plugin-managed (via `storage:plugin` or host cache dirs)

| Data | Owner |
|------|--------|
| Cached weather tiles / GRIB slices | Weather plugin namespace |
| Bathymetry tileset cache | Bathymetry plugin |
| Sonar logs / ring buffers | Sonar plugin |
| Plugin-private settings | That plugin only |

Host may place plugin blobs under `data/plugins/<plugin_id>/` with quota. Plugins never receive path strings to arbitrary FS locations.

### What never goes in plugin storage as authoritative truth

Vehicle fuel/SoC, legal rest clocks for truck mode, allemannsretten / glacier rejection decisions — core routing/safety owns those.

---

## Failure, fuel, and hang isolation

| Event | Core behaviour |
|-------|----------------|
| WASM trap | Log, drop contributions, restart or disable plugin |
| Fuel out | As trap; count toward strike policy |
| Wall timeout | Epoch interrupt; same as hang |
| Contribution queue full | Drop plugin messages; increment metric |
| Plugin asks for denied cap | Error to guest; no host side effects |
| SQLite busy | Retry with backoff on T4; callers use oneshot timeout |

**Invariant:** loss of all plugins leaves sensors, ECU, basemap, and core routing usable.

---

## Security non-goals and hard denials

Plugins are **never** allowed to:

- Load native code into the core process.
- Open raw GNSS, IMU, CAN, serial ECU, or Hamlib handles.
- Read or write the core SQLite file directly.
- Block T0–T3 threads (no sync RPC from sensor/ECU into WASM).
- Bypass capability checks by calling undocumented imports.
- Allocate unbounded host memory or spawn OS threads inside core.
- Perform unconstrained network I/O (only host-mediated, allowlisted `net:fetch` when granted).
- Elevate their scheduler priority above T5.

---

## Summary

Driver Break is a **standalone Rust navigation core** plus a **wasmtime plugin host**. Hardware, routing, maps, and SQLite stay trusted and multi-arch portable; optional marine, weather, bathymetry, and sonar features plug in through a versioned capability API at the lowest thread priority, consuming snapshots and contributing layers/tracks without shared mutable access to core state.
