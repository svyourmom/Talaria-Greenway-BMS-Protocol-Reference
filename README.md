# Talaria xXx — Greenway BMS Protocol Reference

Complete connection and protocol notes for the battery management system fitted to
the **Talaria xXx (TL2500)** electric dirt bike.

Everything here was obtained by **observing real hardware**. Values marked
*confirmed* have at least one independent cross-check; values marked *unconfirmed*
are working hypotheses. Nothing is copied from vendor documentation.

| | |
|---|---|
| **Pack** | Greenway `DM4471608` |
| **Cells** | `LGM50LT_B` (LG INR21700 M50LT), **16S** |
| **Designed capacity** | 38.896 Ah @ 60.0 V nominal |
| **Interfaces** | CAN 250 kbit/s · Bluetooth LE · RS485 (same application layer as BLE) |

There are **two independent ways to read this pack**, and the Bluetooth one is
substantially better. If you only implement one, implement Bluetooth.

---

## 1. Physical connections

### 1.1 The 6-wire battery harness

Modern packs use a **solid-coloured** 6-conductor harness. (Older factory
schematics show *striped* colours — Blue/White and Green/White for CAN. Those do
not match this harness. Identify the CAN pair by resistance, never by colour.)

| Wire | Signal | Notes |
|---|---|---|
| **white** | CAN (one side) | confirmed by two days of traffic, 0 error frames |
| **green** | CAN (other side) | confirmed |
| **black** | **Ground** | measured 0.00 Ω to pack negative |
| **red** | **60 V, unswitched** | source for the enable circuit |
| **orange** | **Enable / S1** | must be held at 60 V |
| **yellow** | **Enable / S2** | must be held at 60 V |

> ### ⚠️ TRAP: two different green wires
> On the **pack side** of the connector, `green` is **CAN**.
> On the **bike side**, an older-convention `green` is **ground** and mates to the
> harness `black`. Always say which side you mean.

CAN polarity (which of white/green is CAN_H) did not need to be determined for
these tests — CAN is differential and the transceiver tolerated the pairing used.
Verify against your own adapter if it matters to you.

### 1.2 THE ENABLE IS A TWO-WIRE CIRCUIT — the most important practical finding

**Both `orange` (S1) and `yellow` (S2) must be held at ~60 V, continuously, for
the entire time the pack is in use.**

This is the single thing most likely to cost you days:

* **Drive only one of them** and the pack *appears* to work — it latches, reports
  ready, and delivers voltage — but it **interrupts its own output for 1.06 s
  every 4.17 s, forever**. On a meter this looks like a rhythmic sawtooth that
  makes the pack unusable. The BMS reports **no fault** while doing it.
* **Drive both** and the output is rock solid: 100 % settled, zero interruptions.

Measured, driving one wire vs both:

| | one wire (S1 only) | both wires |
|---|---|---|
| Output interruptions | every **4.173 s** (stdev 0.048) | **none** |
| Interruption width | **1.056 s** | — |
| `0x201` byte 7 = `0x07` | 71 % | **100 % (425/425)** |
| Cell sag during interruption | 2.9 mV — *the battery is fine* | — |
| Fault reported | **none** | none |

**It is not momentary.** Release the enable and the output drops in **0.11 s**.
There is no grace period, no latch, and no CAN keepalive that substitutes for it.
The factory design confirms this independently: the bike can be powered off
mid-ride by both the power button and the side-kickstand sensor, which is only
possible if the enable is a continuously-held signal.

Factory topology: an NFC/RFID control unit switches 60 V onto the enable line
while the bike is on. Replacing it with a key switch (or a permanent jumper, which
is the community bypass for a lost NFC card) is electrically equivalent.

### 1.3 CAN bus electrical

| Property | Value |
|---|---|
| **Bitrate** | **250 000 bit/s** |
| Sample point | 0.875 |
| Frame format | standard 11-bit IDs, all payloads 8 bytes |
| Byte order | **little-endian** throughout |
| Termination | 120 Ω inside the pack |

**Resistance across the CAN pair**, measured with **both ends completely dead**
(pack disconnected *and* any adapter unplugged):

| Reading | Meaning |
|---|---|
| **~137 Ω** | one terminator — the pack's, alone |
| **~60 Ω** | two terminators — a second node is present and terminated |
| **~40 Ω** | three terminators |
| open | not connected |

> ⚠️ **A powered CAN transceiver biases both lines to ~2.5 V and destroys these
> readings.** With termination off, a live bus reads ~650 Ω, which is meaningless.
> Power everything down before measuring resistance.

---

## 2. CAN interface

### 2.1 The pack is silent until polled

The BMS **transmits nothing unprompted** — ever. Not when idle, not when enabled,
not when delivering power. Days can be lost sniffing a bus that will never speak.

**A single zero-length data frame at ID `0x202` triggers a telemetry burst:**

```sh
cangen can0 -x -I 202 -L 0 -g 100 -n 1     # one frame -> 185 frames back
```

| | |
|---|---|
| Burst length | **5.25 s**, **185 frames** (~35 frames/s) |
| To hold a continuous stream | poll `0x202` at ≥ 1 Hz |
| Cold-start wake latency | **~6 s** from deep sleep |
| Warm wake latency | **~5 ms** |

`0x201` and every other standard ID produce **zero** response — `0x202` is the
only request ID in the entire 11-bit space.

**The receiver is duty-cycled.** A single un-retried frame lands only ~40 % of the
time. Use normal CAN retry (not one-shot) to wake it, or poll repeatedly. Never
conclude the pack is dead from one unanswered frame.

### 2.2 Message map

| ID | Period | Contents |
|---|---|---|
| `0x101` | 53 ms | pack voltage, current, a constant |
| `0x201` | 106 ms | cell extremes + indices, enable/ready bitfield |
| `0x301` | 212 ms | unknown — **all 64 bits always zero** |
| `0x401` | 530 ms | 4 temperatures, remaining capacity, cycle count |

Periods are harmonics of ~53 ms — one broadcast schedule.

#### `0x101` — pack summary (53 ms)

```
7E 02 00 00 E8 03 00 00
```

| Bytes | Field | Encoding | Status |
|---|---|---|---|
| 0–1 | **Pack voltage** | u16 LE, 0.1 V | *confirmed* |
| 2–3 | **Pack current** | **s16** LE, 0.1 A | *confirmed* |
| 4–5 | ~~State of charge~~ | always `1000` | **NOT SoC — see below** |
| 6–7 | unknown | always 0 | — |

> ⚠️ **Bytes 4–5 are not state of charge.** They read exactly `1000` (0x03E8)
> permanently — across an 18-hour span and through a measured discharge. The pack's
> real SoC, read over Bluetooth at the same moment, was **79 %**. Treating this
> field as "100.0 %" will tell you a three-quarters-full pack is full.

#### `0x201` — cell extremes and enable state (106 ms)

```
8F 0F 92 0F 09 01 02 07
```

| Bytes | Field | Encoding |
|---|---|---|
| 0–1 | **Max cell voltage** | u16 LE, mV |
| 2–3 | **Min cell voltage** | u16 LE, mV |
| 4 | **Max cell index** | u8, **1-based** (1–16) |
| 5 | **Min cell index** | u8, 1-based |
| 6 | unknown | always 2 |
| 7 | **Enable/ready bitfield** | see below |

**Byte 7 bitfield:**

| Bit | Mask | Meaning |
|---|---|---|
| 0 | `0x01` | unknown; set in steady state, clears briefly during transitions |
| 1 | `0x02` | **Both enable inputs satisfied** (S1 *and* S2) |
| 2 | `0x04` | **Ready / output enabled** — validated 774/774 against the physical wires |

Common values: `0x07` fully enabled and settled · `0x05` enabled but only one
enable input satisfied (the 4.17 s interruption) · `0x01` not enabled.

Latency versus the physical wires: bit 2 sets ~1.27 s after the enable is applied
and clears ~1.66 s after it is removed.

#### `0x301` — unknown (212 ms)

All eight bytes are **zero in every state ever observed** — disabled, enabled,
interrupting, marginally enabled, deeply asleep. Presumed to be a status or fault
word, but **nothing has ever set a bit in it.**

> ⚠️ **Do not read all-zero here as "no fault".** There is no confirmed fault
> indicator on the CAN link at all.

#### `0x401` — temperatures and lifetime (530 ms)

```
4F 49 49 4A 41 0B A3 00
```

| Bytes | Field | Encoding | Status |
|---|---|---|---|
| 0 | **Temperature — BMS board / FETs** | u8, 1 °C/LSB, **offset −40** | *confirmed* |
| 1 | **Temperature — cell sensor A** | u8, 1 °C/LSB, **offset −40** | *confirmed* |
| 2 | **Temperature — cell sensor B** | u8, 1 °C/LSB, **offset −40** | *confirmed* |
| 3 | **Temperature — cell sensor C** | u8, 1 °C/LSB, **offset −40** | *confirmed* |
| 4–5 | **Remaining capacity** | u16 LE, 10 mAh/LSB | *confirmed vs BLE* |
| 6–7 | **Cycle count** | u16 LE | *confirmed* |

Byte 0 sits a consistent **+6 °C** above the cell sensors — it is on the BMS board
and self-heats. Bytes 1 and 2 read *exactly equal* on a thermally-soaked pack,
which makes them the reliable ambient reference.

> ⚠️ **The −40 offset applies to CAN only.** The same temperatures read over
> Bluetooth are **direct °C with no offset**. Do not apply −40 to BLE values.

Cross-check: `0x401` bytes 4–5 read 2881 (28.81 Ah); Bluetooth
`RemainingCapacity` read **28.794 Ah** at the same time.

### 2.3 Decode cross-check

Two unrelated messages, different units and scales, reconcile exactly:

```
0x101 word0  ->  63.76 V   (pack)
0x201 cells  ->  3985 mV   (mean cell)
63.76 / 3.985 = 16.00 series
```

**16.00** against a 16S pack. Combined with 1-based cell indices spanning 1–16 and
a cycle count matching the manufacturer's own app, the field assignments are not
curve-fitting.

### 2.4 What CAN cannot give you

| Missing | Where to get it |
|---|---|
| **Individual cell voltages** | Bluetooth `CellVoltages1` |
| **Real state of charge** | Bluetooth `BatteryPercent` |
| **Capacity, SoH, identity** | Bluetooth |
| **Current below 0.1 A** | Bluetooth (1 mA resolution) |
| **Terminal voltage** | a meter — `0x101` measures the *internal* cell stack |
| **Any write or control command** | **does not exist** |

**The CAN link is read-only.** An exhaustive search found no command that
influences the pack: 182 distinct stimuli across every protocol family
(all opcodes `0x40`–`0x96`, write-shaped frames, CANopen NMT/SYNC/SDO/heartbeat,
ESC-style presence frames, rolling counters, the full `0x202` payload space,
charger frames, OBD, UDS) changed nothing. Independently, the same vendor's serial
protocol contains **no write command at all** — 3156 captured packets from a
working bike contain only reads.

**The output is controlled by the enable wires and nothing else.**

---

## 3. Bluetooth LE interface — the better link

The pack carries a sealed BLE dongle. It is a **transparent bridge onto the BMS's
own serial protocol**, which means the full parameter set is available wirelessly
with no wiring at all.

### 3.1 Module and GATT

| | |
|---|---|
| Module | **BEKEN BK-BLE-1.0** (`Manufacturer` = `BEKEN SAS`) |
| Advertised name | the pack's **serial number**, e.g. `A236KD0011277` |
| Advertised service UUID | `fff0` |

| Service | Characteristic | Properties | Use |
|---|---|---|---|
| `0000fff0-…` | **`0000fff1-…`** | read, write, **notify** | **data pipe — everything happens here** |
| `0000180a-…` | standard | read | Device Information (module identity, not pack) |
| `f000ffc0-0451-…` | `ffc1`, `ffc2` | write, notify | cloned TI OAD firmware-update service — **not data** |

The dongle is powered from the pack's enable circuit. **If the enable is not
driven, it will not advertise.**

Identify the right device by matching the advertised name against parameter 35
(`SerialNumber`) — they are identical.

### 3.2 Frame format

Write a request to `fff1`; the response arrives as one or more **notifications** on
the same characteristic and may be split across packets — reassemble by length.

```
Request    46  <addr:u16 LE>  <param>  <len>  <cksum>
Response   47  <addr:u16 LE>  <param>  <len>  <data …>  <cksum>

BMS address = 0x0116   (on the wire: 16 01)
cksum       = sum of all preceding bytes & 0xFF
```

> **Credit:** this framing, the `0x0116` address and the parameter identifiers in
> §3.3 are documented in [**patagonaa/surron-light-bee**](https://github.com/patagonaa/surron-light-bee),
> an independent open-source reverse engineering of the same vendor's BMS on
> Sur-Ron hardware. Recognising that the Bluetooth dongle speaks that protocol is
> what made this parameter map readable in an afternoon rather than a month. That
> project is released under **CC0 1.0** and requires no attribution — this credit
> is given because it is deserved.

Worked example — read `BatteryVoltage` (parameter 9, 4 bytes):

```
->  46 16 01 09 04 6A
<-  47 16 01 09 04  10 F9 00 00  74

    0x0000F910 = 63760  ->  63.760 V
```

Opcodes: `0x46` = ReadRequest, `0x47` = ReadResponse. **No write opcode is known
to exist.**

### 3.3 Parameter map

Lengths must be correct in the request. Live values below are a real sample.

| ID | Name | Len | Encoding | Example |
|---|---|---|---|---|
| 0 | Unknown_0 | 4 | — | `46 00 00 00` |
| 7 | Unknown_7 | 1 | — | *no response* |
| **8** | **Temperatures** | 8 | u8[8], **direct °C**, 0 = unused | `[34,33,33,0,36,36,36,0]` |
| **9** | **BatteryVoltage** | 4 | u32 LE, mV | 63.756 V |
| **10** | **BatteryCurrent** | 4 | **s32** LE, mA | −0.033 A |
| **13** | **BatteryPercent** | 1 | u8, % | **79 %** |
| **14** | **BatteryHealth** | 4 | u32, % | 94 % |
| **15** | **RemainingCapacity** | 4 | u32 LE, mAh | 28.794 Ah |
| **16** | **TotalCapacity** | 4 | u32 LE, mAh | 36.570 Ah |
| 17 | Unknown_17 | 2 | — | *no response* |
| 20 | Unknown_20 | 4 | — | zero |
| 21 | Statistics | 12 | undecoded | `da 8e 00 00 bf ab 55 00 bc 5b 00 00` |
| 22 | BmsStatus | 10 | **static config word** | `e0 03 00 …` |
| **23** | **ChargeCycles** | 4 | u32 LE | 163 |
| **24** | **DesignedCapacity** | 4 | u32 LE, mAh | 38.896 Ah |
| **25** | **DesignedVoltage** | 4 | u32 LE, mV | 60.000 V |
| 26 | Versions | 8 | undecoded | `1f 01 00 03 55 41 32 35` |
| **27** | **ManufacturingDate** | 3 | u8 yy, mm, dd | 2023-06-19 |
| 28 | Unknown_28 | 4 | — | zero |
| **29** | **RtcTime** | 6 | u8 yy,mm,dd,hh,mm,ss | — |
| 30 | Unknown_30 | 6 | undecoded | `00 00 ee 0e a1 01` |
| **32** | **BmsManufacturer** | 16 | ASCII, NUL-padded | `GREENWAY` |
| **33** | **BatteryModel** | 32 | ASCII | `DM4471608` |
| **34** | **CellType** | 16 | ASCII | `LGM50LT_B` |
| **35** | **SerialNumber** | 32 | ASCII | `A236KD0011277` |
| **36** | **CellVoltages1** | 32 | **u16 LE × 16, mV** | all 16 cells |
| 37 | CellVoltages2 | 32 | second bank | empty on a 16S pack |
| 38 | History | 14 | undecoded | `d9 78 fe ff 24 4d 00 00 91 10 00 0c` |

> ⚠️ **`BmsStatus` (22) is not a fault register.** It reads `E0 03 00 …` constantly
> — identical across independent captures including hard riding and regen. It is a
> static configuration word. **Neither interface exposes a working fault
> indicator.**

### 3.4 Consistency checks

Sampled together, the parameters agree with each other and with CAN:

```
CellVoltages1 sum      63.754 V     vs  BatteryVoltage  63.756 V   (2 mV)
Remaining / Total      28.794 / 36.570 = 78.7 %   vs BatteryPercent 79 %
RemainingCapacity      28.794 Ah    vs  CAN 0x401 word2  28.81 Ah
ChargeCycles           163          vs  CAN 0x401 word3  163
Temperatures           33, 33 °C    vs  CAN 0x401 (−40)  33, 33 °C
DesignedCapacity       38.896 Ah    vs  nameplate        38.9 Ah
```

Note **TotalCapacity 36.570 Ah** against **DesignedCapacity 38.896 Ah** — the pack
has lost 2.3 Ah, consistent with the reported 94 % SoH.

---

## 4. Practical notes

**Use Bluetooth unless you specifically need CAN.** It requires no wiring, gives
per-cell voltages, correct SoC, capacity, health and identity, and 100× better
current resolution.

**Poll `0x202` at ≥1 Hz** to hold a CAN stream. Nothing arrives unprompted.

**Never use terminal voltage to decide whether the output is on.** The output node
carries roughly 10–30 µF; with the FETs open it holds near pack voltage and decays
only slowly through a meter's ~10 MΩ input. Use `0x201` bit 2 or `0x101` current.

**Beware transmit-queue jamming.** A pack that is asleep acknowledges nothing, so
every CAN frame retries forever and a default 10-deep queue fills within
milliseconds — after which sends return `ENOBUFS` having transmitted **nothing**,
which looks exactly like an unresponsive pack. Serialise transmissions while the
pack is dark; flood only once it is acknowledging. Always verify sends against the
driver's `tx_packets` counter.

**`one-shot` mode is not a fix for that.** Without retry, frames land only ~40 % of
the time against the duty-cycled receiver and the pack cannot be woken at all.

**Bus-off may latch** on adapters whose driver lacks `restart-ms`; recovery is a
manual interface down/up. Allow ~3 s between down and up or the error state
persists.

---

## 5. Reference implementation

Minimal Bluetooth read, Linux + BlueZ over D-Bus:

```python
ADDR = 0x0116
def request(param, length):
    b = [0x46, ADDR & 0xFF, ADDR >> 8, param, length]
    return bytes(b + [sum(b) & 0xFF])

# write request(9, 4) to characteristic fff1, then reassemble notifications:
#   response[0]   == 0x47
#   response[4]   == length
#   response[5:5+length] == data
#   response[-1]  == sum(response[:-1]) & 0xFF
```

---

## Provenance

This document was produced by **clean-room reverse engineering**: every value was
derived from direct observation of privately-owned hardware — CAN captures,
Bluetooth GATT exchanges, and multimeter measurements. No vendor firmware was
decompiled, no proprietary documentation was consulted, and no confidential
material was used.

### Prior art used

The serial application layer — `0x46`/`0x47` framing, the `0x0116` BMS address and
the parameter identifiers listed in §3.3 — is **not original to this document**. It
was cross-referenced against:

> **[patagonaa/surron-light-bee](https://github.com/patagonaa/surron-light-bee)**
> — the most complete public reverse engineering of a Greenway BMS, documented on
> Sur-Ron Light Bee hardware. Released under **[CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/)**
> (public domain dedication — no attribution required).

That project provided the protocol framing, the parameter map, and six real
traffic captures which were used here to validate a parser (3189 frames, 3156
checksum passes; the 33 failures are capture noise, not protocol). It also
supplied the decisive negative result that the vendor's protocol contains **no
write command** — 3156 packets from a working bike, every one a read.

Recognising that this pack's **Bluetooth dongle speaks that same serial protocol**
is what turned an opaque BLE characteristic into a fully-decoded parameter map in
a single session. Credit is given here because it is owed, not because the licence
demands it — CC0 asks for nothing.

**Original to this document:** the CAN link in its entirety (bitrate, the `0x202`
wake mechanism, all four message decodes, the enable bitfield), the physical
harness pinout, the two-wire enable requirement and its failure signature, and the
Bluetooth GATT mapping.

Findings were reproduced across multiple sessions with explicit controls. Several
early conclusions were retracted when controls failed — where a claim here is
marked *unconfirmed*, it means exactly that.

Reverse engineering performed with **Claude Code** assistance.

### Licence

**This document is released under [CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/)
— the same terms as the prior art it builds on.**

To the extent possible under law, the authors have waived all copyright and
related rights to this work. **You may copy, modify, distribute and use it,
including commercially, without permission and without attribution.**

Attribution is not required. If this saved you time, crediting it — and
[patagonaa/surron-light-bee](https://github.com/patagonaa/surron-light-bee) —
helps the next person find both.

### Disclaimer

This describes a **~60 V lithium-ion traction battery**. That is enough voltage to
injure or kill, and lithium cells can vent, burn, or fail catastrophically if
mistreated.

Everything here is offered **as-is, with no warranty of any kind**. Values were
observed on one pack, on specific dates, with the equipment described — yours may
differ in firmware, revision, or wiring. **Verify against your own hardware before
relying on any of it.** Working on a battery, or defeating a safety interlock, is
done entirely at your own risk.

Nothing here should be read as encouragement to bypass battery protection
features. The BMS exists for a reason.
