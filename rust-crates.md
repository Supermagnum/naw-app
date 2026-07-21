# Useable Crates

Reference list of existing Rust crates that cover major subsystems, so core logic can build on them instead of reimplementing from scratch. Grouped by domain; each entry notes what it does and where it fits.

---

## Elevation data

| Crate | What it does | Notes |
|---|---|---|
| [`geotiff`](https://crates.io/crates/geotiff) | Reads GeoTIFF raster files — the format used by Copernicus DEM GLO-30. Part of the georust ecosystem; built specifically for importing elevation models into a routing library. | Recently rewritten on top of the `tiff` crate; expect breaking changes pre-1.0, pin the version. |
| [`srtm_reader`](https://github.com/jarjk/srtm_reader) | Reads NASA/Viewfinder Panoramas `.hgt` files (SRTM1 1 arc-sec, SRTM3 3 arc-sec). Actively maintained fork of the older, stalled `srtm` crate. | Use this over the original `srtm` crate — that one is 6+ years stale with an unmerged PR. |
| [`htg`](https://docs.rs/htg) | `.hgt` reader with built-in filename-from-lat/lon lookup and an LRU-cached `SrtmTile` service. | Caching pattern is a good reference even if you write your own service layer. |
| [`gdal`](https://crates.io/crates/gdal) | Full Rust bindings to libgdal — reads/writes dozens of raster and vector geospatial formats. | Heavier: links against native C GDAL. Only reach for this if you outgrow the pure-Rust readers above. |

**Not covered by any crate:** authenticated fetch/download logic for Copernicus DEM, Viewfinder Panoramas, or NASA SRTMGL1 specifically. Build a thin `reqwest`-based fetcher per source; feed downloaded tiles into `geotiff` / `srtm_reader`.

---

## IMU / sensor fusion

| Crate | What it does | Notes |
|---|---|---|
| [`uf-ahrs`](https://lib.rs/crates/uf-ahrs) | `no_std`, allocator-free orientation estimation from gyro + accelerometer + magnetometer. Implements Mahony, Madgwick, and VQF filters, validated against the BROAD dataset. | Actively developed (as of Feb 2026); API may still shift. Good default if targeting embedded/no_std. |
| [`imu-fusion`](https://crates.io/crates/imu-fusion) | Rust port of the well-known xioTechnologies **Fusion** AHRS library (Madgwick's revised algorithm, ch. 7 of his thesis). | Ports a proven C implementation; includes the C lib as a submodule for test comparison. |
| [`ahrs2`](https://docs.rs/ahrs2) | Attitude/heading determination plus GPS + dead-reckoning fusion; includes UBX-NAV-PVT (u-blox) parsing helpers. | Useful if your GPS module speaks UBX directly — bundles GPS fusion, not just IMU. |
| [`bno055`](https://crates.io/crates/bno055) | `embedded-hal` driver for the Bosch BNO055 9-DoF IMU (has onboard sensor fusion in NDOF mode). | Driver, not a fusion algorithm — use if that's your physical chip. |
| [`bno08x_rs`](https://docs.rs/bno08x-rs) | Driver for the Bosch BNO08x family over SPI (SHTP protocol), exposes fused rotation vectors directly from the chip. | Same category as above — hardware-fusion chip driver. |
| [`bmi160-rs`](https://github.com/eldruin/bmi160-rs) | Platform-agnostic driver for the Bosch BMI160 6-axis accel+gyro. | Raw sensor driver; pair with `uf-ahrs`/`imu-fusion` for software fusion if the chip has no onboard fusion. |

**Pick one of two paths:** (a) a chip with onboard fusion (BNO055/BNO08x) and use its driver's output directly, or (b) a raw 6/9-axis chip driver (e.g. `bmi160-rs`) feeding a software fusion crate (`uf-ahrs` or `imu-fusion`).

---

## Routing

| Crate | What it does | Notes |
|---|---|---|
| [`ferrostar`](https://github.com/stadiamaps/ferrostar) | Cross-platform navigation core: route-following, off-route detection, distance-to-next-maneuver, state machine. Upstream examples often talk to Valhalla/OSRM; **in this project Ferrostar is used only as the on-device guidance layer** against a locally computed route — never to fetch routes from a hosted service. | Navigation *shell* logic — reuse for follow/off-route; pair with on-device `routx` (or equivalent), not a network router. |
| [`routx`](https://lib.rs/crates/routx) | Converts OSM `.pbf` data into a weighted directed graph and runs A* for shortest paths. Supports one-way streets, access tags, turn restrictions, custom profiles. | On-device routing *engine* for this project. Also ships C/C++ bindings. |
| [`osm4routing2`](https://github.com/rust-transit/osm4routing2) | Extracts a routable street network (nodes/edges) from OSM `.pbf` into CSV, or usable as a library. Rust rewrite of the original `osm4routing`. | Graph *extraction* only — no pathfinding; pair with your own A*/Dijkstra or `routx`. |
| [`osm_graph`](https://crates.io/crates/osm_graph) | Builds road networks from OSM data and generates isochrones (reachable-area-within-time-limit) for a given point. | Useful if you ever want "reachable POIs within N minutes" type features. |
| [`pathfinding`](https://crates.io/crates/pathfinding) | General-purpose pathfinding algorithms (A*, Dijkstra, BFS, etc.) over arbitrary graphs, not OSM-specific. | Good building block if you construct your own graph structure from `osm4routing2` output. |

**Practical pairing (this project):** on-device `osmpbf` + `routx` (and/or `osm4routing2` + `pathfinding`) for route **computation**; `ferrostar` for on-device route **following** only. **No** hosted Valhalla/OSRM (or similar) for routing — see [architecture.md](architecture.md#offline-routing-no-hosted-backends).

---

## Database

| Crate | What it does | Notes |
|---|---|---|
| [`rusqlite`](https://crates.io/crates/rusqlite) | Thin, safe wrapper around SQLite's native C library. Synchronous, minimal overhead, direct SQL. | Best fit for this project: single-device embedded storage, no async server needed, matches the "queued writes on a lower-priority DB thread" design already discussed. Requires the native SQLite C library at build time. |
| [`sqlx`](https://crates.io/crates/sqlx) | Async SQL toolkit (not a full ORM) with compile-time-checked queries. Supports SQLite, Postgres, MySQL. | Consider only if the app grows an async multi-backend need (e.g. syncing history to a server) — adds async complexity you likely don't need for local-only storage. |
| [`diesel`](https://diesel.rs/) | Full ORM with compile-time SQL safety, mature (1.0 since 2018). Sync by default (`diesel-async` bolts on async). | Heavier tooling/compile times; worth it mainly for large schemas with many relations — likely overkill for a stop/fuel/charge history schema. |
| [`sea-orm`](https://www.sea-ql.org/SeaORM/) | Async-first ORM built on `sqlx`, ActiveRecord-style API. | Same reasoning as `sqlx` — nice ergonomics, unnecessary async overhead for local embedded storage. |
| [`geo`](https://crates.io/crates/geo) + [`rstar`](https://crates.io/crates/rstar) | Not a database, but: `geo` gives core geometry types/algorithms, `rstar` gives an R-tree for spatial indexing. | Use alongside `rusqlite` for in-memory spatial POI queries (radius search for water/cabins/chargers) without needing SpatiaLite. |

**Recommendation for this project:** `rusqlite` for the core history/config store (matches your single-device, thread-priority-aware design), plus `rstar` for in-memory spatial POI indexing rather than pulling in SpatiaLite unless you specifically need persisted spatial queries.

---

## Supporting georust crates (used by several of the above)

| Crate | What it does |
|---|---|
| [`geo`](https://crates.io/crates/geo) | Core geometry types (points, lines, polygons) and algorithms (distance, area, intersection). |
| [`geojson`](https://crates.io/crates/geojson) | GeoJSON read/write. |
| [`gpx`](https://crates.io/crates/gpx) | GPX track/route/waypoint file read/write. |
| [`osmpbf`](https://crates.io/crates/osmpbf) | Low-level OSM `.pbf` file parsing (underlies several of the routing crates above). |

---

## Summary — suggested starting stack

- **Elevation:** `geotiff` (Copernicus DEM) + `srtm_reader` (SRTM/Viewfinder Panoramas fallback) + custom fetcher; optional Open-Meteo Elevation when online ([API.md](API.md)).
- **IMU:** **Android reference:** OS `TYPE_ROTATION_VECTOR` via JNI/UniFFI ([architecture.md](architecture.md#reference-sensor-source-android-gps-and-imu)). Embedded/non-Android: hardware driver (`bno055`/`bno08x_rs` if onboard fusion, else `bmi160-rs` + `uf-ahrs`).
- **Routing:** on-device only — `osmpbf` + `routx` (and/or `osm4routing2` + `pathfinding`) for pathfinding; `ferrostar` for guidance/follow. **No** hosted Valhalla/OSRM.
- **Database:** `rusqlite` + `rstar` for spatial POI indexing.
- **OSM data:** Geofabrik region/country `.osm.pbf` downloads ([API.md](API.md)).
