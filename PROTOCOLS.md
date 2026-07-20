# Vehicle, ECU, and charging-station protocols

Reference for transports and data open source navigation units can use for fuel, energy, and charge state. Ideas are not limited to Navit: any host that can talk to vehicles, maps, and charging infrastructure should be able to apply the same mappings. Covers ICE backends (OBD-II, J1939, MegaSquirt), electric cars, electric cycles, electric motorcycles, and **compatibility with charging stations**.

Read-only vehicle telemetry by default: subscribe or request data; do not change ECU or BMS configuration from the navigation unit. Station interaction stays within documented, user-consented charge/session APIs.

---

## Table of contents

1. [Common goals](#common-goals)
2. [OBD-II (ELM327 and ISO diagnostic stack)](#obd-ii-elm327-and-iso-diagnostic-stack)
3. [SAE J1939 (heavy vehicles)](#sae-j1939-heavy-vehicles)
4. [MegaSquirt family](#megasquirt-family)
5. [Electric cars](#electric-cars)
6. [Charging stations (required compatibility)](#charging-stations-required-compatibility)
7. [Electric cycles (e-bikes / pedelecs)](#electric-cycles-e-bikes--pedelecs)
8. [Electric motorcycles and scooters](#electric-motorcycles-and-scooters)
9. [Mapping to host fields](#mapping-to-host-fields)
10. [Backend design notes](#backend-design-notes)

---

## Common goals

For liquid-fuel vehicles, the plugin needs at least:

- Instantaneous fuel rate (L/h), or enough inputs to estimate it.
- Optional tank level / remaining fuel (L or %).

For electric vehicles, the analogous signals are:

- Instantaneous power (kW) or current and pack voltage.
- State of charge (SoC, %).
- Optional: remaining energy (kWh), state of health (SoH), charging status.

Energy-based routing and range estimates use these the same way fuel rate and tank level are used for ICE.

---

## OBD-II (ELM327 and ISO diagnostic stack)

### Role

Standard light-vehicle diagnostics for petrol, diesel, and flex-fuel cars (and many motorcycles with an OBD-II port, commonly Euro 3 and later). Typical adapter: ELM327-compatible over UART/USB/Bluetooth.

### Physical and transport layers

| Layer | Typical standard |
|-------|------------------|
| Connector | SAE J1962 (16-pin) |
| Classic signalling | ISO 15765-4 CAN (most modern vehicles); older: ISO 9141-2, ISO 14230 (KWP2000), SAE J1850 |
| Bit rates (CAN) | Often 500 kbit/s; some 250 kbit/s |
| Addressing | 11-bit or 29-bit CAN IDs |
| Application | OBD modes/services (e.g. Mode 01 current data); enhanced diagnostics via UDS (ISO 14229) on many ECUs |

ELM327 speaks an AT command set (init, protocol select, PID request) and returns hex responses that encode mode, PID, and payload bytes.

### Useful standard PIDs (Mode 01 examples)

| PID | Name | Use |
|-----|------|-----|
| `0x2F` | Fuel tank level | Remaining fuel (%) |
| `0x5E` | Engine fuel rate | Direct L/h when supported |
| `0x10` | MAF air flow rate | Fallback fuel-rate estimate |
| `0x52` | Ethanol fuel % | Flex-fuel AFR / density |
| `0x0C` | Engine RPM | Sanity checks, injector models |
| `0x0D` | Vehicle speed | Trip / adaptive learning |

When `0x5E` is missing, fuel rate is often estimated from MAF and AFR/density (see `README.md` formulas).

### Limits

- Emissions-oriented PIDs; not every vehicle exposes fuel rate.
- Manufacturer-specific (enhanced) PIDs vary by brand and year.
- Adapter absent or non-responsive: fall back to adaptive estimation; other plugin features continue.

### Mutual exclusion

OBD-II and MegaSquirt typically share one serial port — only one serial backend active at a time.

---

## SAE J1939 (heavy vehicles)

### Role

In-vehicle network for trucks, buses, and many heavy machines. Used in truck mode when a SocketCAN (or equivalent) interface is available.

### Physical and transport layers

| Layer | Typical standard |
|-------|------------------|
| Bus | CAN (often 250 kbit/s on classic J1939; newer fleets may use J1939-22 / CAN FD) |
| Addressing | 29-bit CAN identifiers |
| Naming | Parameter Group Numbers (PGNs), Suspect Parameter Numbers (SPNs) |
| Host API (Linux) | SocketCAN (`can0`, etc.) |

The plugin listens for broadcast PGNs; it does not need to be a full J1939 stack for fuel monitoring.

### Fuel-related PGNs / SPNs (plugin use)

| PGN | Common name | SPN | Scaling (classic) |
|-----|-------------|-----|-------------------|
| 65266 (FEEA) | Fuel Economy / Engine Fuel Rate | 183 Engine Fuel Rate | 0.05 L/h per bit, offset 0 |
| 65276 (FEF4) | Dash Display | 96 Fuel Level | 0.4 % per bit, range typically 0–250 % |

Fuel level percent converts to litres with configured tank capacity:

`fuel_current_L = (fuel_level_% / 100) * tank_capacity_L`

### Notes

- Many fleets also expose engine hours, ambient temp, and other SPNs useful for trip context.
- Electric or hybrid commercial vehicles may reuse J1939 with different SPNs for battery/power; treat those as a separate mapping layer when supported.

---

## MegaSquirt family

### Role

Aftermarket / performance ECUs: MS1, MS2, MS3, MS3-Pro, MicroSquirt. Common on kit cars, race engines, and custom installs.

### Physical and transport layers

| Layer | Typical setup |
|-------|---------------|
| Link | Serial / USB-serial |
| Settings | Often 115200 8N1 |
| Access | Realtime data block (e.g. command `A` on many firmwares) |

### Data used for fuel rate

From the realtime block (and config):

- Injector pulse width (ms)
- Engine RPM
- Cylinder count
- Injector flow (cc/min), configured as `fuel_injector_flow_cc_min`

Approximate fuel rate (see `README.md`):

`fuel_rate_L_h = (pw_ms * rpm * n_cyl * flow_cc_min) / 2_000_000`

Skip updates when RPM, pulse width, or computed rate are outside sanity limits.

### Related aftermarket pattern

Other ECUs (Haltech, Link, AEM, ECUMaster, etc.) often broadcast similar channels over CAN or serial. Same mapping target: fuel rate L/h and optional tank level. Prefer the vendor’s documented broadcast; do not implement full tuning.

---

## Electric cars

There is no single “fuel rate” PID for all EVs. Useful energy telemetry comes from several layers.

### Diagnostic / in-vehicle (navigation unit side)

| Protocol / stack | What it is | Typical useful data |
|------------------|------------|---------------------|
| **OBD-II Mode 01** | Same port as ICE on many EVs | Speed, some temps; SoC/energy often **not** in generic PIDs |
| **ISO 15765-4 CAN** | Diagnostic transport under OBD | Request/response frames |
| **UDS (ISO 14229)** | Application diagnostics | Manufacturer DIDs: pack voltage/current, SoC, SoH, cell data |
| **OEM enhanced PIDs** | Brand-specific Mode 22 / DID maps | SoC, power, temperatures (varies widely by make/model/year) |
| **CAN FD** | Higher payload diagnostics | Large BMS responses that do not fit classic 8-byte CAN |

Practical reality: a generic ELM327 Mode 01 poll is often insufficient for accurate EV range. SoC and traction power usually need UDS/OEM maps or a documented OEM CAN stream. Many brands firewall HV/BMS data; support is per-vehicle profiles, not one universal PID list.

### Examples of OEM diagnostic patterns (illustrative, not exhaustive)

- Mode 22 / UDS DID ranges differ by brand (pack V/I, SoC, SoH, module temps).
- Some groups use 11-bit CAN diagnostics; others use 29-bit.
- Cell-level detail may be unavailable without OEM tools or reverse-engineered DBC/PID tables.

---

## Charging stations (required compatibility)

Navigation hosts must treat charging stations as first-class stops for electric modes: discover, filter, plan, navigate to, and log them the same way fuel stations are handled for ICE.

### What the navigation unit must support

- **Map / POI discovery** of charging stations (OSM and other open sources): location, operator, access, opening hours when tagged.
- **Compatibility filters** against the vehicle profile: connector type (e.g. Type 2, CCS2, CHAdeMO, NACS/J3400, GB/T, Schuko for light AC), max AC/DC power, phase count, and membership or roaming network where known.
- **Charge-stop planning** from SoC, consumption model, and station power — insert stops before range runs out; prefer stations that match the vehicle.
- **Session history** — charge stops in the same timeline as rest and fuel stops.
- **Optional live station data** when the host can use open roaming or operator APIs (availability, power, tariff), without locking the design to one proprietary network.

### Vehicle–charger protocols (physical charge path)

These run mainly between vehicle and charger. The navigation unit does not replace the charge controller, but must understand them enough to label stations and plan correctly:

| Protocol | Scope |
|----------|--------|
| **IEC 61851** | Basic AC/DC charging control (PWM on Control Pilot, etc.) |
| **DIN SPEC 70121** | CCS DC high-level communication (earlier) |
| **ISO 15118** | Smart charging, Plug & Charge, optional V2G; PLC on Control Pilot |
| **CHAdeMO** | DC fast charge protocol (own signalling stack) |
| **GB/T 27930** | Chinese DC charging communication |
| **SAE J2847** family | PEVs and off-board chargers (related to DIN 70121 / ISO 15118 ideas) |

### Station / backend protocols (infrastructure)

| Protocol | Scope |
|----------|--------|
| **OCPP** | Charger-to-backend management; relevant if a host or companion service talks to infrastructure, not as in-vehicle ECU telemetry |
| **OCPI / similar roaming models** | Exchange of location, tariff, and session data between MSPs and CPOs — useful for open clients that show live availability |

Drive telemetry for energy still comes from **vehicle SoC / power** (diagnostic or OEM CAN/BLE). Station compatibility layers add **where and how** to charge, not a substitute for BMS data.

### Mapping for eco / range

- Instantaneous electrical power ≈ pack voltage × pack current (sign for regen).
- Remaining energy ≈ SoC × usable pack capacity (kWh).
- Charge-stop history is the EV analogue of fuel-stop history.

---

## Electric cycles (e-bikes / pedelecs)

No universal OBD-II equivalent. Systems are fragmented by motor/BMS vendor.

### Common interfaces

| Interface | Where it appears | Typical data |
|-----------|------------------|--------------|
| **Proprietary CAN** | Mid-drive systems (many Bosch, Shimano STEPS, Brose, Yamaha ecosystems) | Assist mode, cadence, speed, battery SoC, error codes — often encrypted or undocumented on the public bus |
| **UART / TTL serial** | Many hub motors and open controllers | Current, voltage, speed limit, PAS level |
| **LIN** | Some sensor/display links | Low-bandwidth sensor/display traffic |
| **Bluetooth LE / ANT+** | Displays, aftermarket computers, some batteries | SoC, range estimate, assist level — vendor GATT/ANT profiles |
| **CANopen** | Some industrial / open builds | Standardised object dictionary when used |
| **EnergyBus** (legacy) | Older open e-bike efforts | Power and battery concepts; limited modern adoption |

### Practical approach for the navigation host

- Prefer a documented vendor API or BLE/ANT profile when the bike exposes SoC and power.
- For open controllers, UART/CAN decode of voltage, current, and speed is enough for energy-based routing and charge-stop hints.
- Do not assume a J1962 OBD port exists.
- Treat display buses as read-only; avoid injecting commands that could alter assist limits or unlock firmware.

### Signals to capture when available

- Battery SoC (%) and/or voltage
- Motor or battery current / power
- Speed and assist level (for adaptive energy models)
- Optional: cadence, torque (for effort models)

---

## Electric motorcycles and scooters

Mix of automotive-like and scooter-OEM designs.

### What you may find

| Path | Notes |
|------|--------|
| **OBD-II / ISO 15765** | Some street-legal e-motorcycles expose a diagnostic connector; generic PIDs still rarely include traction SoC |
| **OEM CAN** | Dominant for BMS, inverter, charger, dash — DBC or reverse-engineered IDs per brand |
| **UDS over CAN** | Dealer-level diagnostics on higher-end bikes |
| **Proprietary serial / BLE** | Common on scooters and small bikes for dash and phone apps |
| **Charger protocols** | Onboard AC charger vs DC CCS/CHAdeMO on a minority of models — same family as cars when present |

### Practical approach

- Same field mapping as electric cars: SoC, pack V/I or power, optional SoH.
- Prefer SocketCAN (or USB-CAN) sniffing of published OEM streams when available.
- Fall back to adaptive energy learning from trip power estimates if live BMS data is locked.
- Motorcycle break intervals remain time/fatigue based; energy data improves range and charge-stop planning, not rest-law rules.

---

## Mapping to host fields

Unify backends into a small set of internal quantities any open source navigation host can adopt:

| Concept | ICE / MegaSquirt / J1939 | Electric |
|---------|--------------------------|----------|
| Instantaneous use | `fuel_rate_l_h` | Power (kW) or equivalent energy rate |
| Remaining energy store | `fuel_current` (L) | SoC (%) and/or remaining kWh |
| Stop history | Fuel stop | Charge stop (same timeline as rest stops) |
| Adaptive learning | L/100 km samples | Wh/km or kWh/100 km samples |

Configuration flags per backend (examples): `fuel_obd_available`, `fuel_j1939_available`, `fuel_megasquirt_available`, plus future `ev_*` / `ebike_*` flags. Only one backend may own a given serial port or CAN interface at a time.

---

## Backend design notes

1. Open the device (serial, SocketCAN, BLE, etc.).
2. Decode using the protocol documented for that backend.
3. Write only physically plausible values; skip garbage frames.
4. If the source is missing, leave adaptive estimation and rest-stop logic running.
5. Run ECU/sensor reads on high-priority threads so DB and routing work cannot stall live data (see concurrency section in `README.md`).
6. Persist samples and stop history with correct wall time (GPS time preferred when there is no RTC).
7. Keep charging-station discovery and filters host-agnostic so other open source navigation units can reuse the same station model.
