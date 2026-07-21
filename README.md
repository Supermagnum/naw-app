# ideas

Concept notes for mode-aware rest stops, safety filters, POI discovery, fuel and charge monitoring, and optional energy-based (eco) routing app or plugin. The product is optional a **new, from-scratch** navigation app (not a Navit plugin); see [architecture.md](architecture.md) for the Rust core, thread model, and WASM plugin host. Idea-level behaviours may still interest other open source navigation projects.

Electric modes must stay **compatible with charging stations**: discover suitable chargers along the route, plan charge stops with the same history model as fuel stops, and align with common station/vehicle charge interfaces where the unit can participate (connector types, power levels, and high-level protocols documented in [PROTOCOLS.md](PROTOCOLS.md)).

The map must support **moving icons** (dynamic positions that update in place), especially for APRS station tracking — see [APRS.md](APRS.md). Elevation download APIs are in [API.md](API.md). Radio CAT (via Hamlib) is in [CAT.md](CAT.md).

## Documents

| Document | Contents |
|----------|----------|
| [README.md](README.md) | This file — product ideas overview |
| [architecture.md](architecture.md) | Technical architecture: core, threads, WASM plugins, multi-arch |
| [PROTOCOLS.md](PROTOCOLS.md) | OBD-II, J1939, MegaSquirt, electric vehicles, charging stations |
| [API.md](API.md) | Geofabrik OSM extracts; elevation APIs; external data sources table |
| [APRS.md](APRS.md) | Moving icons, APRS decode, range, QRV/CAT, symbols |
| [CAT.md](CAT.md) | Radio CAT via Hamlib; baud rates, serial formats, vendor notes |

## Table of contents

1. [Technical architecture](architecture.md)
2. [Core idea](#core-idea)
3. [Concurrency: multi-core and priority threads](#concurrency-multi-core-and-priority-threads)
4. [Travel mode ideas](#travel-mode-ideas)
5. [POI discovery ideas](#poi-discovery-ideas)
6. [Map layers](#map-layers)
7. [Moving icons (APRS and similar)](#moving-icons-aprs-and-similar)
8. [Offline routing](#offline-routing)
9. [Safety and legality ideas](#safety-and-legality-ideas)
10. [Configurable rest parameters](#configurable-rest-parameters-idea-sketch)
11. [Historical basis: rast and vei](#historical-basis-rast-and-vei)
12. [Network and priority ideas](#network-and-priority-ideas)
13. [Water safety ideas](#water-safety-ideas-hiking--cycling)
14. [Route validation ideas](#route-validation-ideas-hiking--cycling)
15. [Energy-based routing](#energy-based-routing-eco-mode-idea)
16. [Elevation idea](#elevation-idea-srtm-family)
17. [Fuel and charge monitoring ideas](#fuel-and-charge-monitoring-ideas)
18. [History, UI, and configuration](#history-ui-and-configuration-ideas)
19. [Mathematical formulas](#mathematical-formulas)
20. [Vehicle / ECU protocols](PROTOCOLS.md)
21. [External data APIs](API.md)
22. [APRS / moving icons](APRS.md)
23. [CAT radio control](CAT.md)

---

## Core idea

Plan a route that already knows when and where you should stop, using different rules per travel mode:

- **Hiking** — water refills, tent/cabin overnight spots, official hiking or pilgrimage networks; keep off motorways, trunks, and other roads unsafe on foot.
- **Cycling** — same stop types (water, overnight, optional national cycling/pilgrimage networks); avoid roads unsafe for cyclists.
- **Car** — rest stops with something to do (cafe, restaurant, museum, gallery, zoo, viewpoint, etc.); fuel monitoring and optional fuel-efficient routing from terrain.
- **Electric vehicles (car, motorcycle, cycle)** — same rest logic as the matching travel mode, plus charge-stop planning at compatible charging stations (power, connector, and access constraints).
- **All modes** — no overnight camps too close to glaciers (ice avalanche risk); keep a minimum distance from buildings for right-to-roam rules such as Norwegian *allemannsretten*; optional energy-based routing for least effort, fuel, or electrical energy.

When a compatible vehicle ECU or BMS is present, live fuel or energy use feeds better route cost estimates. Charge stops and fuel stops share one stop history.

---

## Concurrency: multi-core and priority threads

Must support **multiple CPU cores** and **multiple threads with different priorities**, so time-critical I/O and UI stay responsive while heavy work runs elsewhere. Workloads should be isolatable onto separate threads (and, where useful, pinned or scheduled across cores) rather than sharing one blocking loop.

Suggested thread roles and relative priorities (highest first):

| Thread / role | Priority | Responsibility |
|---------------|----------|----------------|
| **Sensors (GPS, IMU)** | Highest | Continuous position, heading, and motion samples; must not stall for map or DB work. **Reference:** Android fused location + rotation vector ([architecture.md](architecture.md#reference-sensor-source-android-gps-and-imu)). |
| **ECU data** | High | Live fuel / engine reads (OBD-II, J1939, MegaSquirt); poll and decode without waiting on routing or UI. |
| **Graphical / UI** | High–medium | Map redraw, display, menus, moving-icon updates; keep interaction smooth while background jobs run. |
| **Routing calculations** | Medium | Rest-stop planning, energy cost walks, route validation; CPU-heavy, parallelizable across cores when segment work allows. |
| **Database access** | Lower | Persistent history, config, fuel samples, trip summaries; queued writes/reads so DB latency does not block sensors, ECU, or graphics. |

Design constraints:

- Sensor and ECU threads must not block on database locks or long routing passes.
- Routing and energy work may use multiple worker cores; cancel or deprioritize when a new position or route invalidates in-flight results.
- Database access is serialized or otherwise concurrency-safe, with async or batched I/O so higher-priority threads keep running.
- Shared state (position, fuel rate, route, session break state) is published with clear ownership or lock-free / short critical sections so priorities are not inverted by long holds.

---

## Travel mode ideas

| Mode | Idea |
|------|------|
| **Motorcycle** | Like car, but shorter break intervals (e.g. every 2 h) and shorter max riding periods — higher fatigue. Same POIs and fuel logic; OBD-II when available (common Euro 3+); otherwise adaptive fuel estimation. |
| **Car** | Soft/max driving hours, break interval (e.g. 4–4.5 h), break duration (e.g. 15–45 min). |
| **Truck** | Mandatory rest/driving rules aligned with EU EC 561/2006 (break after 4.5 h, 45 min break, daily/weekly rest limits, etc.). |
| **Hiking** | ~40 km suggested max per day; rests at 11.295 km (main) or 2.275 km (alt); optional SRTM + water/cabin POIs. |
| **Bicycle** | ~100 km suggested max per day; rests at 28.24 km (main) or 5.69 km (alt); same rast/vei concept as hiking, scaled up. |

---

## POI discovery ideas

Search radii configurable (examples: water 2 km, cabins 5 km, general POI 15 km, network huts 25 km). **POI icons must come from OSM** (standard OpenStreetMap icon set / tag-matched icons), including on every basemap layer below.

- **Water** — drinking water tap, fountain, spring (hiking/cycling refills).
- **Public restrooms** — toilets / public conveniences (`amenity=toilets` and equivalents); included in discovery for all relevant travel modes.
- **Cabins / huts** — wilderness hut, alpine hut, hostel, camping; optional DNT/network prioritization.
- **Car / truck** — cafe, restaurant, museum, gallery, zoo, aquarium, viewpoint, picnic site, tourist attraction, similar amenities.
- **Charging stations** — for electric modes: public and semi-public chargers filtered by connector type, power (kW), access (membership/payment), and vehicle compatibility; usable as planned charge stops along the route.

---

## Map layers

Basemap selection (numbered); default is layer 1. All layers use **OSM POI icons** for discovered POIs (restrooms, water, chargers, etc.).

| Number | Layer | Role |
|--------|-------|------|
| **1** (default) | Standard OSM | General navigation |
| **2** | Tracestrack Topo | Topographic / terrain-oriented |
| **3** | CyclOSM | Cycling-oriented rendering |
| **4** | Cycle Map | Classic cycle map style |

---

## Moving icons (APRS and similar)

The host must render **moving icons**: map markers whose coordinates update when new telemetry arrives, without creating a duplicate marker per update. Primary use is APRS (callsign-keyed stations with symbol-table icons that track mobiles as they beacon).

Hard rules for APRS display (see [APRS.md](APRS.md)):

- Show stations only within a **hardcoded 50–150 km** window (VHF/UHF reach); no unlimited range.
- Station timeout **must not exceed 3600 s** (beacon intervals may run ~20–3000 s with speed).
- Decoder handles **QRV/qrv + frequency** (e.g. `146.500 MHz`, `145,500 mhz`) for optional CAT radio tune ([CAT.md](CAT.md)); label those stations with a leading `*`; full message only at **zoom > 14**.
- APRS **WX** (weather-station) beacons are a **future** optional source for weather-along-route — unrelated to drinking-water POIs; see [APRS.md](APRS.md).

---

## Offline routing

Route computation is **fully on-device** against locally downloaded OSM (and local DEM for energy costs). No hosted Valhalla/OSRM (or similar) call is used to build a route. Guidance/follow logic (e.g. Ferrostar) runs on-device only. OSM region/country downloads: [API.md](API.md). Details: [architecture.md](architecture.md#offline-routing-no-hosted-backends).

### Distance from buildings (camping / allemannsretten)

Overnight candidates too close to buildings or dwellings are filtered out. **Hardcoded default: 150 m** for Norwegian *allemannsretten* and similar right-to-roam rules.

### Distance from glaciers (overnight)

Reject overnight spots too close to glaciers (ice avalanche, rockfall, meltwater flood risk). **Minimum distance: 1000 m**, hardcoded as the default floor. This check is **overridden** when the candidate is (or is at) a recognised camping site, hut, cabin, or similar overnight facility — those established sites remain allowed even if nearer a glacier than 1000 m.

---

## Configurable rest parameters (idea sketch)

- **Car** — soft limit hours, max hours, break interval (h), break duration (min).
- **Truck** — mandatory break after (h), break duration (min), max daily driving hours.
- **Hiking / cycling** — main and alternative break distances (km), max daily distance (km).
- **General** — rest interval range, POI radii, min distance from buildings, min distance from glaciers for overnight.

---

## Historical basis: rast and vei

Suggested hiking/cycling defaults draw on old Scandinavian length units:

- A **rast** (also **vei**) was roughly how far you walked before needing a rest (often tied to a *mil* / ell length; regional and historical variation).
- A **stone’s throw** was **120 ells** (also called a “great hundred”) — about **56.88 m** (200 feet).
- There were **4 stone’s throws** in an **arrow’s flight**, so about **480 ells** — **227.52 m** (800 feet) around the year **900**.
- Later in the Middle Ages, **10 arrow shots** made up a **fjerding** of a mile — **2,275.2 m** (8,000 feet), a quarter of a younger Norse mile.
- That younger Norse mile (**rast** / **vei**) was **9,100.8 m** (32,000 feet). The same order of magnitude appears in 12th-century expressions such as 16,000 ells.
- A **dagsvei** (day’s journey) was commonly ~40 km on foot.

Plugin defaults follow that tradition (hiking main/alternative breaks at 11.295 km / 2.275 km and ~40 km day; cycling scaled from the same idea).

---

## Network and priority ideas

### DNT / network hut priority

Optional preference for network huts with a configurable search radius. Set radius from typical nearest-neighbor spacing; raise toward the max in remote areas.

**Networked cabin spacing (sample nearest-neighbor stats, for search-radius tuning):**

| Region | Sample | Avg | Median | Max |
|--------|--------|-----|--------|-----|
| Norway (DNT, OSM relation 1110420) | 449 huts | 10.56 km | 8.83 km | 100.45 km |
| Sweden (wilderness/alpine hut) | 439 | 12.31 km | 8.24 km | 83.85 km |
| Sweden STF only | 42 | 14.47 km | 11.50 km | 83.85 km |
| Finland (wilderness/alpine hut) | 324 | 11.72 km | 6.68 km | 64.32 km |
| Finland Metsähallitus only | 108 | 16.05 km | 5.31 km | 247 km |
| Germany (wilderness/alpine hut) | 261 | 12.98 km | 9.72 km | 119.76 km |
| Switzerland (wilderness/alpine hut) | 328 | 4.40 km | 3.82 km | 23.70 km |
| Austria (wilderness/alpine hut) | 330 | 5.30 km | 3.56 km | 102.51 km |

**Open / non-networked huts** (no network tag; not DNT/STF/DAV/SAC/OeAV/Metsähallitus etc.):

| Region | Sample | Avg | Median | Max |
|--------|--------|-----|--------|-----|
| Germany | 235 | 14.29 km | 10.22 km | 119.76 km |
| Switzerland | 261 | 4.93 km | 4.03 km | 23.70 km |
| Austria | 287 | 5.71 km | 3.65 km | 102.51 km |
| Sweden | 395 | 12.45 km | 7.85 km | 64.86 km |
| Finland | 206 | 17.00 km | 12.33 km | 75.75 km |

Practical ranges: ~5–15 km in the Alps; ~10–20 km in Scandinavia/Germany/Finland for open huts. Use at least typical spacing (~10–12 km); raise toward max remotely.

### Hiking / pilgrimage priority

Optional preference for official hiking and pilgrimage routes when validating or suggesting stops.

---

## Water safety ideas (hiking / cycling)

**Prefer treated / reliable sources**

- Marked drinking water taps (`amenity=drinking_water`) in towns, trailheads, staffed cabins.
- Staffed huts and lodges usually supply treated or reliably clean water.

**Treat natural sources before drinking**

- Springs, streams, rivers: treat even if they look clean (grazing, vegetation, upstream activity; *Giardia*, *Cryptosporidium*, etc.).
- Options: boil (1 min, or 3 min above 2,000 m), certified filter (0.1 µm or finer), chemical tablets, UV pen.
- Avoid stagnant water when possible; boil if used.
- Near farms, roads, mining, or glacial runoff: chemical/heavy-metal risk may survive filters and boiling — find another source.

Running water in North European national parks is generally considered drinkable untreated, but stay aware of rodent years even though rodent-borne illness cases are scarce.

**Rough daily water needs**

- Hiking: ~0.5 L per hour of activity (more in heat / altitude).
- Cycling: ~0.5–0.75 L per hour.
- Extra for cooking and camp use.

---

## Route validation ideas (hiking / cycling)

- Avoid forbidden road types (motorway, trunk, primary unsafe for foot/bike).
- Avoid military areas: `landuse=military` and `military=danger_area` (routes and rest/overnight candidates).
- Report share of priority paths (footway, path, track, steps, bridleway; pilgrimage/hiking tags when priority is on).
- Warn on high forbidden-road % or low priority-path %.

---

## Energy-based routing (eco-mode) idea

Model physical effort / energy per segment from weight, rolling resistance, air drag, elevation, and downhill recuperation. With a live ECU, fold measured fuel into cost so flatter / more efficient alternatives win when they exist.

Configurable via host UI: drag coefficient `Cd`, frontal area (m²), total mass (kg).

---

## Elevation idea (SRTM family)

On-demand elevation for better stops and energy routing. Scope options:

- **Region** — tiles covering a selected map region or bounding box.
- **Entire country** — all tiles for a chosen country in one download job.

Try sources in order:

1. Copernicus DEM GLO-30  
2. Viewfinder Panoramas  
3. NASA SRTMGL1  

Downloads pausable/resumable/cancellable; progress tracked per region or per country job. How each DEM source is listed, named, authenticated, and fetched — and how OSM region/country `.pbf` files are obtained from Geofabrik — is documented in [API.md](API.md).

---

## Fuel and charge monitoring ideas

Live ECU/BMS data for tracking and (with energy routing) better costs. Must work with **charging stations** for electric modes: suggest and log charge stops at compatible stations, and keep station metadata (connector, power, access) usable across open source navigation hosts. Protocol details for OBD-II, J1939, MegaSquirt, electric vehicles, and charging-station interfaces are in [PROTOCOLS.md](PROTOCOLS.md). Three ICE backends:

1. **OBD-II (ELM327)** — petrol/diesel/flex cars and light vehicles; fuel level, fuel rate, ethanol when available; MAF-based estimate if no direct fuel-rate PID; fallback to adaptive estimation.
2. **J1939 (SocketCAN)** — trucks/heavy vehicles; engine fuel rate and fuel level on CAN; auto in truck mode when a CAN interface exists.
3. **MegaSquirt** — MS1/MS2/MS3/MS3-Pro/MicroSquirt over serial; injector-based consumption; fallback if ECU missing.

### Adaptive fuel learning

Without ECU/BMS data, build a persistent consumption model from samples and trip summaries. Fuel stops and charge stops are logged alongside rest-stop history so a trip shows rest, refuel, and charge events in one timeline.

### Time across power-off

The navigation unit must know how much time elapsed while it was turned off (for driving/rest clocks, mandatory truck breaks, trip summaries, and history timestamps). Preference order for wall-clock time:

1. **GPS time** — preferred when the unit has no reliable RTC; use the time from a GNSS fix as soon as available after boot.
2. **RTC** — use a hardware real-time clock when present and trusted.
3. **Last-known + monotonic** — if neither GPS nor RTC is available yet, keep relative elapsed time with the monotonic clock and backfill absolute timestamps once GPS (or RTC) is valid.

On shutdown, persist last shutdown time (and session state). On boot, compute offline duration before continuing driving/rest accounting so a powered-down overnight stop is not counted as continuous driving.

---

## History, UI, and configuration ideas

- Persist rest / fuel / charge history, consumption samples, trip summaries, and config across sessions. Fuel and charge stops appear in the same history view as rest stops.
- Browse/clear history in the host UI; menu changes save automatically.
- Host UI actions: suggest rest stop, suggest charge/fuel stop, history, start/end break, configure intervals per profile, overnight settings (buildings, glaciers, POI radii), charging-station filters (connector, power, access), **download OSM region/country** (Geofabrik — [API.md](API.md)), download elevation region/country.
- Track session state: driving time, break in progress, mandatory break required — including time while powered off (see above).
- Vehicle type selected via host configuration.
- Keep behaviour portable: other open source navigation units should be able to host the same stop logic, energy model, and station compatibility without depending on one proprietary stack.

---

## Mathematical formulas

Readable reference for how an implementation derives numbers (concrete code may live in C or the host’s native language).

### OBD-II — MAF-based fuel rate

When PID `0x5E` (engine fuel rate) is unavailable, estimate from mass air flow (PID `0x10`):

\[
\text{Fuel rate (L/h)} = \frac{\text{MAF (g/s)} \times 3600}{\text{AFR} \times \rho \times 1000}
\]

- **MAF** — air mass flow (g/s)
- **AFR** — air–fuel ratio (e.g. ~14.7 for petrol; adjusted by fuel type / ethanol from PID `0x52`)
- **ρ** — fuel density (kg/L, e.g. ~0.745 for petrol)

### J1939 — fuel rate and fuel level

- **SPN 183** (PGN 65266 / FEEA): 0.05 L/h per bit, offset 0

\[
\text{fuel\_rate\_L\_h} = \text{raw\_spn183} \times 0.05
\]

- **SPN 96** (PGN 65276 / FEF4): 0.4 % per bit, range 0–250 %

\[
\text{fuel\_level\_\%} = \text{raw\_spn96} \times 0.4
\]

\[
\text{fuel\_current\_L} = \frac{\text{fuel\_level\_\%}}{100} \times \text{tank\_capacity\_L}
\]

### MegaSquirt — injector-based fuel rate

Inputs: injector pulse width `pw_ms` (ms), engine `rpm`, cylinder count `n_cyl`, injector flow `flow_cc_min` (cc/min).

\[
\text{fuel\_rate\_L\_h} =
\frac{\text{pw\_ms} \times \text{rpm} \times n_{\text{cyl}} \times \text{flow\_cc\_min}}{2\,000\,000}
\]

Rationale sketch: pulses/min scale with RPM; each pulse delivers `flow_cc_min / 60` cc/s at 100 % duty; duty cycle is `pw_ms / 1000` of the cycle. Out-of-range RPM / pulse width / rate updates are skipped.

### Range estimation

From current fuel and average consumption (`fuel_avg_consumption_x10` → L/100 km):

\[
\text{range\_km} =
\frac{\text{fuel\_current\_L} \times 100}{\text{consumption\_L\_per\_100km}}
\]

Short-term adaptive averages from live samples can refine this.

### Energy-based routing (eco-mode)

Segment energy cost is proportional to:

\[
E \approx
(F_{\text{rolling}} + F_{\text{drag}}) \times d
+ m g \Delta h
\]

- \(F_{\text{rolling}}\) — rolling resistance  
- \(F_{\text{drag}}\) — aerodynamic drag  
- \(d\) — segment length  
- \(m\) — total mass  
- \(g\) — gravity  
- \(\Delta h\) — height difference  

Drag from user `Cd` and frontal area \(A\), with sea-level air density \(\rho \approx 1.225\,\mathrm{kg/m^3}\):

\[
F_{\text{drag}} \propto \tfrac{1}{2}\,\rho\,C_d A\,v^2
\]

Temperature and elevation still adjust the coefficient inside each segment. Rolling resistance uses a fixed coefficient times weight in the model initializer. Live ECU fuel can be compared to energy predictions so lower-energy (and thus lower-fuel) routes are preferred.
