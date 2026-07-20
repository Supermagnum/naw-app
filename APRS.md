# APRS — moving icons and station tracking

Ideas for Automatic Packet Reporting System (APRS) support on open source navigation units: decode packets, keep a station database, and draw **moving icons** on the map as positions update. Not limited to one host; any nav stack that can place dynamic map items can apply the same model.

---

## Table of contents

1. [Core idea](#core-idea)
2. [Moving icons](#moving-icons)
3. [Architecture](#architecture)
4. [Packet sources](#packet-sources)
5. [Protocol layers](#protocol-layers)
6. [Symbols](#symbols)
7. [Range filtering](#range-filtering)
8. [Frequencies (receive)](#frequencies-receive)
9. [Configuration sketch](#configuration-sketch)
10. [Legal note](#legal-note)

---

## Core idea

APRS stations (vehicles, hikers, balloons, fixed digipeaters, etc.) beacon positions over amateur radio (and sometimes network feeds). The navigation unit should:

- Ingest AX.25 / APRS frames from RF (SDR), NMEA/TNC serial, or network.
- Store callsign, position, symbol, course/speed, comment, and last-heard time.
- Show each station as a **map icon that moves** when a new position arrives.
- Expire stale stations after a configurable timeout so the map stays useful.

Many trackers change beacon interval with speed (e.g. tens of seconds when moving, much longer when stopped). A persistent database keeps last known positions between beacons so icons do not vanish between updates, and so the user can still navigate toward a station that has paused.

---

## Moving icons

This is a required capability of the overall idea set: the map must support **dynamic, relocatable icons**, not only static POIs.

### Behaviour

| Requirement | Detail |
|-------------|--------|
| Create | On first valid position for a callsign, add a map item with the APRS symbol icon. |
| Move | On later packets for the same callsign, update coordinates (and optional course/speed) in place — the icon translates on the map. |
| Identity | Key stations by callsign (and SSID); do not spawn a new icon per packet. |
| Symbol | Icon from APRS symbol table + code (see [Symbols](#symbols)); fallback to a default POI icon if unknown. |
| Stale | After timeout without packets, remove or dim the icon; keep DB history as configured. |
| Threading | Packet ingest and DB writes must not block GPS/UI; map item updates marshalled to the graphics thread (see concurrency notes in `README.md`). |

### Why a database

- Bridging long beacon gaps for slow or stopped mobiles.
- Range queries (“stations within N km”).
- Restoring icons after app restart until they expire.
- Future: route-to-station when the host supports routing to dynamic targets.

Suggested timeout range: about 30–180 minutes (example default ~90 minutes / 5400 s), configurable in the UI.

---

## Architecture

Split into two modules (same idea as a dedicated APRS host + optional SDR frontend):

### Core APRS module

Responsibilities:

- AX.25 / APRS parse and position validation  
- SQLite (or equivalent) station store  
- Moving map items / icon updates  
- Station expiration and range filtering  
- NMEA waypoint-style input (`$GPWPL`, `$PGRMW`, `$PMGNWPL`, `$PKWDWPL`, etc.)  
- Packet ingestion API for external sources  

Exports (conceptual):

- `process_packet(frame)` — ingest one decoded AX.25 frame  
- `register_packet_source(callback)` / `unregister_packet_source` — SDR or TNC plugs in here  
- `update_map_items()` — refresh visible moving icons from DB  

### Optional APRS SDR module

Responsibilities:

- RTL-SDR (or similar) open, tune, gain, PPM  
- I/Q capture  
- Bell 202 2FSK demodulation at 1200 bps  
- AX.25 frame extract → deliver to core via `process_packet`  

Devices commonly supported: RTL-SDR Blog V3 / V4, Nooelec NESDR, generic RTL2832U. On Android, USB Host / OTG permissions apply; a powered hub may be needed.

If SDR is absent, core still works with NMEA, TNC, file, or network feeds. If core is absent, SDR may keep running but discard frames until core registers.

---

## Packet sources

1. **SDR** — direct RF → demod → AX.25 (preferred for over-the-air receive).  
2. **NMEA serial** — USB/serial adapters; baud/parity configurable.  
3. **External** — TNC, APRS-IS-style network feed, recorded files, custom producers using the ingestion API.

---

## Protocol layers

High to low:

```text
APRS information field (payload)
        ↓
AX.25 UI frame (addresses, control=0x03, PID=0xF0, info, FCS)
        ↓
HDLC framing (0x7E flags, bit stuffing)
        ↓
NRZI line coding
        ↓
Bell 202 2FSK (mark 1200 Hz, space 2200 Hz, 1200 baud)
        ↓
RF / I/Q sampling (e.g. RTL-SDR)
```

### APRS payload (application)

Position and symbol live in the information field (compressed or uncompressed formats per the APRS spec). Example uncompressed style:

```text
!5132.00N/00007.00W-Test
```

Symbol table and code sit in that field (here `/` and `-`).

### AX.25 UI

- Dest / source addresses (shifted encoding + SSID)  
- Control `0x03` (UI), PID `0xF0`  
- Info = APRS payload  
- FCS CRC-CCITT  

### HDLC / NRZI / Bell 202

- Flags `0x7E`; bit-stuff after five `1` bits in data  
- NRZI: `0` = transition, `1` = no transition  
- Mark = 1200 Hz, space = 2200 Hz at 1200 baud  

SDR DSP sketch: mix to baseband → DC block → FM discriminator → bit PLL → NRZI decode → HDLC de-stuff → frame callback.

---

## Symbols

Icons follow the Updated APRS Symbol Set (e.g. Rev H), commonly distributed as the hessu/aprs-symbols set (OH7LZB / aprs.fi authoring lineage).

### Encoding

Two characters:

| Part | Values | Role |
|------|--------|------|
| Symbol table | `/` primary or `\` alternate | Which table |
| Symbol code | one character | Which icon |

Example: `/` + `>` → vehicle.

### Files

- PNG icons, typically **48×48**  
- Layout: `…/aprs-symbols/48x48/primary/` and `…/alternate/`  
- Generated from sprite sheets at build/install time (do not need to vendor huge binary trees in git)  
- Lookup: `(table, code)` → filename; missing → default POI icon  

Roughly ~95 symbols per table (primary and alternate).

### Common examples

| Table + code | Meaning |
|--------------|---------|
| `/` `>` | Vehicle |
| `/` `-` | House |
| `/` `*` | Aircraft |
| `/` `S` | Boat |
| `/` `B` | Bike |
| `/` `H` | Hospital |
| `/` `P` | Police |

### Host rendering

Map items need a custom-icon path (or equivalent): each moving station carries `icon_src` from the symbol lookup so the renderer can draw the correct PNG at the live coordinates. Overlay/layout config must allow dynamic icon substitution for these items.

---

## Range filtering

Limit which stations become visible moving icons, using distance from a centre (usually the device GPS position).

Haversine great-circle distance:

```text
a = sin²(Δlat/2) + cos(lat1) * cos(lat2) * sin²(Δlon/2)
distance = 2 * R * atan2(√a, √(1-a))
R ≈ 6371 km
```

Process: load candidates from DB → compute distance → keep only those within max range → create/update map icons. Example: show stations within 50–150 km; `0` = no limit. Centre updates as the vehicle moves so the moving-icon set follows the user.

---

## Frequencies (receive)

Common 2 m APRS channels (configure the SDR/TNC; the app does not replace the radio licence):

| MHz | Regions (typical) |
|-----|-------------------|
| 144.390 | North America; parts of South America and SE Asia |
| 144.575 | New Zealand |
| 144.640 | China, Taiwan, Japan (main) |
| 144.660 | Japan (Kyushu area) |
| 144.800 | Europe (IARU R1), much of Africa/Russia use |
| 144.930 | Argentina, Uruguay, Panama |
| 144.990 | Some Canadian secondary / handheld use |
| 145.175 | Australia |
| 145.570 | Brazil |

Modulation on these channels: Bell 202 / AFSK 1200 bps; ~6 kHz occupied bandwidth typical. Check local band plans before transmit; this design is aimed at **receive and display** unless the operator is licensed and the host explicitly supports TX.

---

## Configuration sketch

| Setting | Role |
|---------|------|
| Database path | Persistent station store (or in-memory) |
| Timeout (s) | Expire / remove stale moving icons |
| Distance (m) | Range filter; 0 = unlimited |
| Centre | Lat/lon for range (usually live GPS) |
| RX frequency | Regional APRS channel for SDR |
| Gain / PPM | SDR front-end |
| Serial baud/parity | NMEA/TNC input |

Runtime UI changes should apply without restart when possible.

---

## Legal note

Receiving APRS for map display is common practice where allowed; **transmitting** on amateur frequencies requires a valid licence and compliance with local law. Keep TX paths disabled unless explicitly implemented and legally operated.
