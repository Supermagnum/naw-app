# CAT — computer control of radios

Computer Aided Transceiver (CAT) lets a navigation host set frequency (and related mode/PTT options) on an amateur radio from software. Used here when an APRS comment carries **QRV/qrv** and a frequency (see [APRS.md](APRS.md)): parse the frequency, then optionally tune the radio over CAT.

**CAT commands are supported via Hamlib.** The navigation host should use Hamlib (library API and/or `rigctld`) as the supported CAT backend. Hamlib already implements vendor dialects (Kenwood text, Icom CI-V, Yaesu, Elecraft, and many others): the host sends Hamlib operations such as set/get frequency and mode; Hamlib translates those into the correct CAT bytes on the wire.

This document still lists common baud rates, serial formats, and vendor command families so integrators can choose Hamlib model IDs, match radio menu settings, and debug the link. Direct raw CAT without Hamlib is not the preferred path.

Read-only or tune-only by default. Do not enable transmit paths unless the operator is licensed and explicitly configures TX.

---

## Table of contents

1. [Hamlib support (required path)](#hamlib-support-required-path)
2. [Role in this project](#role-in-this-project)
3. [Physical link](#physical-link)
4. [Common baud rates](#common-baud-rates)
5. [Common serial formats](#common-serial-formats)
6. [Frequency encoding](#frequency-encoding)
7. [Kenwood-style text CAT](#kenwood-style-text-cat)
8. [Icom CI-V](#icom-ci-v)
9. [Yaesu CAT (text / binary families)](#yaesu-cat-text--binary-families)
10. [Other vendors](#other-vendors)
11. [Hamlib host API](#hamlib-host-api)
12. [QRV → CAT flow](#qrv--cat-flow)
13. [Safety and configuration](#safety-and-configuration)

---

## Hamlib support (required path)

| Item | Detail |
|------|--------|
| Support statement | All CAT command use in this project is **through Hamlib**. |
| Why | One API covers many radios; Kenwood / Icom / Yaesu / others are already mapped inside Hamlib. |
| How the host talks | Link `libhamlib` and call `rig_*` functions, **or** run `rigctld` and send rigctl network/text commands. |
| What the host does *not* do | Re-implement full vendor CAT tables for each radio when Hamlib already supports that model. |
| Vendor sections below | Reference only (baud, 8N1, frame shapes) for configuration and troubleshooting under Hamlib. |

Hamlib maps a stable set of operations onto each radio’s CAT command set, including at least:

- Open / close the CAT port (model, device, baud, serial format)
- Set frequency / get frequency
- Set mode / get mode (e.g. FM after a QRV parse)
- Optional VFO and level controls when the model backend supports them

If a radio is listed in Hamlib’s supported models, use that model number in the host profile; Hamlib emits the correct CAT commands.

---

## Role in this project

| Step | Action |
|------|--------|
| 1 | APRS decoder finds `QRV`/`qrv` + frequency (`146.500 MHz`, `145,500 mhz`, …). |
| 2 | Normalise to Hz; validate amateur VHF/UHF band. |
| 3 | User confirms (or auto-tune if enabled). |
| 4 | Host asks **Hamlib** to open the CAT port and set frequency (and usually mode FM for 2 m voice). |
| 5 | Optional: Hamlib read-back frequency to verify. |

---

## Physical link

| Medium | Typical use |
|--------|-------------|
| USB–serial (FTDI, CP210x, CH340, built-in USB) | Most modern HF/VHF mobiles and HTs with “CAT” or “clone” cables |
| RS-232 DE-9 | Older base / mobile rigs |
| Bluetooth SPP | Some HTs and aftermarket interfaces |
| Network (rigctld) | Hamlib daemon; host talks TCP instead of raw serial |

Device path examples: `/dev/ttyUSB0`, `/dev/ttyACM0`, `COM3`.

---

## Common baud rates

CAT baud is radio- and menu-dependent. The host must match the rig’s CAT/CI-V baud setting.

| Baud | Where it is common |
|------|--------------------|
| **4800** | Older Kenwood / Yaesu defaults; some mobiles |
| **9600** | Very common default (Kenwood, many Yaesu, Icom CI-V) |
| **19200** | Frequent modern default / optional menu value |
| **38400** | Common on newer Kenwood, Yaesu, Icom when enabled |
| **57600** | Some Yaesu / computer-control setups |
| **115200** | High-speed CAT on newer SDR-capable or USB-native rigs |

Suggested host defaults when auto-detect is unavailable: try **9600**, then **38400**, then **19200**, then **4800**. Persist the working rate per radio profile.

---

## Common serial formats

Most CAT ports use **8 data bits, no parity, 1 stop bit**:

| Parameter | Typical value | Notes |
|-----------|---------------|--------|
| Data bits | **8** | Almost universal |
| Parity | **None (N)** | Dominant; some legacy gear odd/even (rare for CAT) |
| Stop bits | **1** | Standard |
| Shorthand | **8N1** | Default assumption for new profiles |
| Flow control | **None** | Usual; hardware RTS/CTS only if the cable/rig requires it |
| Duplex | Full | Independent RX/TX on the UART |

Less common: **8E1** / **8O1** on obscure industrial clones — only if the manual says so.

Line endings for text protocols:

- Kenwood-style: commands end with **`;`** (semicolon), not CR/LF (many Kenwood commands).  
- Some older text dialects use **CR** (`\r`) or **CRLF**.  
- CI-V and many Yaesu binary frames are binary — no line ending.

---

## Frequency encoding

Hosts should keep an internal frequency in **integer Hz**, then format per protocol:

| Example on-air | Hz |
|----------------|-----|
| 146.500 MHz | 146500000 |
| 145,500 mhz | 145500000 |
| 144.800 MHz | 144800000 |

For FM voice after a QRV parse, also set mode to **FM** (or the rig’s narrow-FM equivalent) when the CAT dialect supports it.

---

## Kenwood-style text CAT

Reference for what Hamlib sends on Kenwood (and Kenwood-like) backends. Used by many Kenwood HF/VHF radios (and some clones). ASCII commands, often terminated with `;`. Hosts should still drive these radios through Hamlib, not by hand-rolling `FA;` strings unless debugging.

### Serial profile (typical)

- Baud: 4800 / 9600 / 19200 / 38400 / 57600 (menu)  
- Format: **8N1**, no flow control  

### Useful commands

| Command | Meaning | Example |
|---------|---------|---------|
| `FA;` | Read VFO A frequency | Reply like `FA000144800000;` |
| `FA` + 11 digits + `;` | Set VFO A frequency (Hz, zero-padded) | `FA000146500000;` → 146.500 MHz |
| `FB;` / `FB…;` | Read / set VFO B | Same width as FA |
| `MD;` | Read mode | |
| `MD0;` … | Set mode (map depends on model; FM often a fixed digit — check model manual) | |
| `IF;` | Read status / frequency / mode block | Long status reply |
| `AI0;` / `AI1;` / `AI2;` | Auto-information off / on variants | Reduce unsolicited traffic with `AI0;` when polling |

Notes:

- Digit width for `FA`/`FB` is model-dependent (often 11 digits).  
- Always consult the specific Kenwood CAT manual for mode tables and exact digit counts.  
- After set, read `FA;` back to confirm.

---

## Icom CI-V

Reference for Icom backends inside Hamlib. Icom uses a binary **CI-V** bus (often via CT-17-style or USB CI-V). Multiple radios can share an address. Set the CI-V address in the Hamlib/rig profile to match the radio menu.

### Serial profile (typical)

- Baud: **4800**, **9600**, **19200**, **38400** (CI-V baud in menu)  
- Format: **8N1**  
- Framing: binary packets with preamble `0xFE 0xFE`, addresses, command, data, end `0xFD`

### Frame sketch

```text
FE FE [to_addr] [from_addr] [cmd] [sub] [data…] FD
```

| Field | Role |
|-------|------|
| `FE FE` | Preamble |
| to_addr | Radio CI-V address (model-specific, e.g. often in 0x58–0xA4 range — check manual) |
| from_addr | Controller address (commonly `0xE0`) |
| cmd / sub | Operation |
| data | BCD frequency, mode bytes, etc. |
| `FD` | End of message |

### Frequency set (typical pattern)

- Command to set operating frequency (commonly **0x05**): data is frequency in **BCD**, usually **5 bytes**, **least-significant digit pair first** (order is CI-V convention — implement from the model’s CI-V reference).  
- Command to read frequency (commonly **0x03**).  
- Mode set (commonly **0x06**) with a mode code byte (FM value is model-specific).

Example conceptual payload for 146.500000 MHz in BCD (byte order per Icom doc): encode 146500000 as BCD nibbles into the data field of cmd `0x05`.

Echo / OK: radio may reply with the same command echo or `FB`/`FA`-style CI-V OK/NG codes depending on transceiver.

---

## Yaesu CAT (text / binary families)

Reference for Yaesu backends inside Hamlib. Yaesu has several generations:

### Newer text CAT (FT-991A, FTDX, many late models)

- ASCII commands similar in spirit to Kenwood (`;` terminated on many models).  
- Frequency set often `FA` / `FB` style or model-specific `OP`/`SQ` families — **use the model CAT manual**.  
- Baud: commonly **4800–38400**, **8N1**.

### Older binary CAT (e.g. some FT-817 / FT-857 / FT-897 era)

- Fixed-length binary frames (often 5-byte command blocks).  
- Frequency as BCD in the payload.  
- Baud historically **4800 8N2** on some models (note **2 stop bits**), others 8N1 — verify per model.

### Example operations (check model doc)

| Goal | Approach |
|------|----------|
| Set frequency | Model-specific set-freq opcode + BCD or ASCII Hz |
| Set mode FM | Mode opcode / status byte |
| Read frequency | Status or read-freq opcode |

Because Yaesu dialects diverge sharply by era, treat “Yaesu” as a family of profiles, not one command table.

---

## Other vendors

| Vendor / stack | Notes |
|----------------|--------|
| **Elecraft** | Text CAT; Kenwood-compatible subset on many models; 8N1; baud menu |
| **Flex / Anan / SDR** | Often network CAT or proprietary; Hamlib/rigctld preferred |
| **Mobile Chinese OEMs** | Mix of Kenwood-like ASCII clones; test against the bundled PC program’s baud/format |
| **TM-D710 / TH-D72 style APRS radios** | CAT for frequency; APRS TNC may be a separate serial/USB function |

---

## Hamlib host API

**Supported CAT path for this project.** The navigation unit issues Hamlib calls; Hamlib issues the vendor CAT commands.

| Operation | Hamlib API / rigctld | Purpose |
|-----------|----------------------|---------|
| Open | `rig_open` / `rigctld` | Port, Hamlib model ID, baud, 8N1 |
| Set frequency | `rig_set_freq` / `F` | Hz from QRV parse → CAT set-freq |
| Set mode | `rig_set_mode` / `M` | FM (or NFM) for VHF QRV → CAT set-mode |
| Get frequency | `rig_get_freq` / `f` | Verify via CAT read-back |
| Close | `rig_close` | Release port |

Example rigctld-style commands (text protocol to the daemon):

```text
F 146500000
M FM 0
f
```

(`F` set freq Hz, `M` set mode, `f` get freq — exact syntax follows rigctld.) Hamlib converts these into Kenwood, CI-V, Yaesu, or other CAT command sequences for the selected model.

Host config fields: **Hamlib model ID**, device path, baud, databits/parity/stopbits (default 8N1), CI-V address (Icom), optional RTS/DTR PTT quirks (usually unused for QRV tune-only).

---

## QRV → CAT flow

```text
APRS comment: "qrv 146.500 MHz"
        ↓
parse → 146500000 Hz
        ↓
validate band / user confirm
        ↓
Hamlib: set mode FM (if supported)  →  vendor CAT
        ↓
Hamlib: set frequency 146500000     →  vendor CAT
        ↓
Hamlib: read-back frequency
```

Frequency text accepted by the APRS side (for reference):

- `146.500 MHz`, `146.500 mhz`  
- `145,500 MHz`, `145,500 mhz` (comma decimal)

See [APRS.md](APRS.md) for `*` label and zoom > 14 full-message rules.

---

## Safety and configuration

| Rule | Detail |
|------|--------|
| Consent | Default: confirm before CAT tune; optional auto-tune off by default |
| Licence | Operator must be authorised for the frequency and radio |
| TX | Do not assert PTT from QRV parsing; tune only unless a separate, explicit TX feature exists |
| Port sharing | CAT UART must not fight APRS TNC/SDR on the same device without multiplexing |
| Timeout | Serial I/O on a medium-priority thread; never block GPS |
| Profiles | Store per-radio: **Hamlib model ID**, baud, 8N1 (or override), CI-V address |
| Backend | CAT commands go through **Hamlib** only in the supported design |

### Suggested defaults for a new radio profile

```text
backend        = hamlib
hamlib_model   = <rig model id>
baud           = 9600
data_bits      = 8
parity         = none
stop_bits      = 1
flow_control   = none
set_mode_on_qrv = FM
auto_tune      = false
```
