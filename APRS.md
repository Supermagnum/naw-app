# APRS — moving icons and station tracking

Ideas for Automatic Packet Reporting System (APRS) support on open source navigation units: decode packets, keep a station database, and draw **moving icons** on the map as positions update. Not limited to one host; any nav stack that can place dynamic map items can apply the same model.

---

## Table of contents

1. [Core idea](#core-idea)
2. [Moving icons](#moving-icons)
3. [Beacon intervals and station timeout](#beacon-intervals-and-station-timeout)
4. [Architecture](#architecture)
5. [Packet sources](#packet-sources)
6. [Protocol layers](#protocol-layers)
7. [Decoder: water, QRV, and CAT frequency](#decoder-water-qrv-and-cat-frequency)
8. [Symbols](#symbols)
9. [Range filtering (hardcoded)](#range-filtering-hardcoded)
10. [Frequencies (receive)](#frequencies-receive)
11. [Configuration sketch](#configuration-sketch)
12. [Legal note](#legal-note)

Related: [CAT.md](CAT.md) — CAT commands are supported through **Hamlib** (baud rates, 8N1, vendor notes).

---

## Core idea

APRS stations (vehicles, hikers, balloons, fixed digipeaters, etc.) beacon positions over amateur radio (and sometimes network feeds). The navigation unit should:

- Ingest AX.25 / APRS frames from RF (SDR), NMEA/TNC serial, or network.
- Store callsign, position, symbol, course/speed, comment, weather-related fields, QRV/frequency flags, and last-heard time.
- Show each station as a **map icon that moves** when a new position arrives.
- Expire stale stations after a timeout **no greater than 3600 seconds** (see below).

A persistent database keeps last known positions between beacons so icons do not vanish between updates, and so the user can still navigate toward a station that has paused.

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
| QRV mark | If the station’s comment/message carries a parseable QRV + frequency, show a `*` prefix on the label (see [Decoder](#decoder-water-qrv-and-cat-frequency)). |
| Label zoom | Full message/comment text is shown only when map zoom level is **greater than 14**; at lower zoom show callsign (and `*` if applicable) only. |
| Stale | After timeout without packets (max 3600 s), remove or dim the icon. |
| Threading | Packet ingest and DB writes must not block GPS/UI; map item updates marshalled to the graphics thread (see concurrency notes in `README.md`). |

### Why a database

- Bridging long beacon gaps for slow or stopped mobiles (intervals can stretch toward ~3000 s).
- Range queries within the hardcoded VHF/UHF-useful window.
- Restoring icons after app restart until they expire.
- Future: route-to-station when the host supports routing to dynamic targets.
- Holding parsed water and QRV/frequency fields for UI and CAT control.

---

## Beacon intervals and station timeout

A lot of APRS transmitters change how often they send based on how fast the attached object is moving. Intervals can vary widely — for example from about **3000 seconds** down to about **20 seconds**.

Therefore:

- The station database and map must tolerate long gaps without clearing a moving station too early.
- Station expiration / display timeout is configurable only up to a hard maximum of **3600 seconds**.
- **Any timeout above 3600 seconds is not allowed** (reject or clamp in config and UI).
- Suggested default: at or below 3600 s (e.g. 3600 s to cover the long end of mobile beacon spacing).

---

## Architecture

Split into two modules (same idea as a dedicated APRS host + optional SDR frontend):

### Core APRS module

Responsibilities:

- AX.25 / APRS parse and position validation  
- Comment parsing: water information, QRV + frequency → CAT  
- SQLite (or equivalent) station store  
- Moving map items / icon updates (`*` prefix, zoom-gated full text)  
- Station expiration (timeout ≤ 3600 s) and hardcoded range filtering  
- NMEA waypoint-style input (`$GPWPL`, `$PGRMW`, `$PMGNWPL`, `$PKWDWPL`, etc.)  
- Packet ingestion API for external sources  
- Optional CAT output when user confirms tuning to a parsed QRV frequency  

Exports (conceptual):

- `process_packet(frame)` — ingest one decoded AX.25 frame  
- `register_packet_source(callback)` / `unregister_packet_source` — SDR or TNC plugs in here  
- `update_map_items()` — refresh visible moving icons from DB  
- `parse_qrv_frequency(comment)` — extract Hz for CAT  
- `cat_set_frequency(hz)` — host/radio bridge via **Hamlib** (Hamlib emits vendor CAT)  

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

Symbol table and code sit in that field (here `/` and `-`). Free-text comment after the position can carry weather/water notes and QRV lines.

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

## Decoder: weater, QRV, and CAT frequency

The decoder must inspect the APRS information field / comment, not only lat/lon and symbol.

### Weater information

Parse and store weater-related content when present, for example:

- APRS weatherfields where the packet type carries them.
- Free-text mentions useful to hikers and boaters (weater, flood, tide, well, spring, etc.) so the UI can flag or filter stations that advertise weater context.
- Keep raw comment text; attach structured flags/fields when recognition is confident.

### QRV and frequency → CAT

Detect comments/messages that contain:

1. The token **`QRV`** or **`qrv`** (amateur Q-code: ready / listening on a frequency), and  
2. A frequency written like:
   - `146.500 MHz`
   - `145,500 mhz`
   - Same patterns with `.` or `,` as decimal separator; case-insensitive `MHz` / `mhz`.

Parsing rules (idea):

- Case-insensitive search for `qrv` as a whole word.  
- Nearby or in the same comment: number with `.` or `,` decimal, optional spaces, then `mhz`.  
- Normalise to Hz (e.g. `146.500 MHz` → 146500000; `145,500 mhz` → 145500000).  
- Validate against a sane VHF/UHF amateur range before offering CAT tune (reject garbage parses).

**CAT control:** the parsed frequency can be sent to a radio over a CAT interface (e.g. Hamlib or vendor serial/USB CAT) so the radio is set to the frequency shown in the message. Tuning should be explicit/user-confirmed unless the user enables auto-tune. Command dialects, baud rates, and serial formats are documented in [CAT.md](CAT.md).

### Map label rules for QRV messages

If such a QRV + frequency message is sent by an APRS transmitter shown on the map:

- Place a **`*`** in front of the station label (e.g. `*LA1ABC-9`).  
- The **full message** (complete comment / QRV text) is visible only when the map **zoom level is greater than 14**.  
- At zoom ≤ 14: show callsign with `*` only (and icon); do not clutter the map with full QRV text.

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

Note: the label prefix `*` for QRV is separate from the APRS aircraft symbol code `*`.

### Host rendering

Map items need a custom-icon path (or equivalent): each moving station carries `icon_src` from the symbol lookup so the renderer can draw the correct PNG at the live coordinates. Overlay/layout config must allow dynamic icon substitution for these items. Label layer applies `*` and zoom > 14 full-text rules above.

---

## Range filtering (hardcoded)

VHF/UHF APRS has limited practical reach. Showing stations beyond that is pointless and clutters the map.

**Hardcoded policy (not user-overridable outside this band):**

| Bound | Value | Rule |
|-------|-------|------|
| Minimum display range | **50 km** | Do not configure a filter below 50 km as the product default window; useful local floor of the allowed band. |
| Maximum display range | **150 km** | Never show stations farther than 150 km from the device. |
| Unlimited / global | **Forbidden** | No “0 = no limit” mode for over-the-air APRS display. |

Within 50–150 km the host may expose a single range setting (still clamped to that interval); anything outside is rejected or clamped. Default suggestion: **150 km**.

Distance uses the Haversine great-circle formula:

```text
a = sin²(Δlat/2) + cos(lat1) * cos(lat2) * sin²(Δlon/2)
distance = 2 * R * atan2(√a, √(1-a))
R ≈ 6371 km
```

Process: load candidates from DB → distance from live GPS centre → keep only stations with `50 km ≤ d ≤ 150 km` (or `d ≤ selected_range` with `selected_range` ∈ [50, 150]) → create/update moving icons. Centre updates as the vehicle moves.

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

Modulation on these channels: Bell 202 / AFSK 1200 bps; ~6 kHz occupied bandwidth typical. Check local band plans before transmit; this design is aimed at **receive and display** unless the operator is licensed and the host explicitly supports TX. QRV-parsed frequencies for CAT may be any valid channel the comment advertises (not only the APRS digi frequency).

---

## Configuration sketch

| Setting | Role | Limits |
|---------|------|--------|
| Database path | Persistent station store (or in-memory) | — |
| Timeout (s) | Expire / remove stale moving icons | **1–3600 only; > 3600 forbidden** |
| Distance (km) | Range filter from GPS | **Hardcoded band 50–150 km** |
| Centre | Lat/lon for range | Live GPS |
| RX frequency | Regional APRS channel for SDR | Regional table |
| Gain / PPM | SDR front-end | — |
| Serial baud/parity | NMEA/TNC input | — |
| CAT device | Optional radio control for QRV tune | User consent |
| Auto-tune QRV | Optional | Off by default |

Runtime UI changes should apply without restart when possible, still enforcing timeout and range clamps.

---

## Legal note

Receiving APRS for map display is common practice where allowed; **transmitting** on amateur frequencies requires a valid licence and compliance with local law. Keep TX paths disabled unless explicitly implemented and legally operated. CAT control of a radio likewise assumes the operator is authorised to use that equipment and frequency.
