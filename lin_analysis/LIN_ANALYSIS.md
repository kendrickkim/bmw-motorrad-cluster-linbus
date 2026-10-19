# BMW Motorrad LIN Signal Analysis

[한국어](LIN_ANALYSIS-kr.md)

> **Reference bike:** BMW S1000XR (2016)  
> Captures and decode notes for other models / years are welcome.

For parsing and UI implementation, see:

- [Cluster application](https://github.com/kendrickkim/bmw-motorrad-cluster-application)
- [ESP32 firmware](https://github.com/kendrickkim/bmw-motorrad-cluster-esp32)

---

## Contents

1. [Frame notation](#frame-notation)
2. [Message ID overview](#message-id-overview)
3. [ID `0x14` — Left handle / Wonder Wheel](#id-0x14--left-handle--wonder-wheel)
4. [ID `0x20` — Ignition / rear speed / fuel range / ambient light](#id-0x20--ignition--rear-speed--fuel-range--ambient-light)
5. [ID `0x2B` — Multiplexed payloads / odometer](#id-0x2b--multiplexed-payloads--odometer)
6. [ID `0x2E`](#id-0x2e)
7. [ID `0xE2` — RPM / gear / throttle / engine temperature](#id-0xe2--rpm--gear--throttle--engine-temperature)
8. [ID `0xE9` — Average speed / speed / voltage / faults](#id-0xe9--average-speed--speed--voltage--faults)

---

## Frame notation

LIN frames are treated as **1-byte Message ID + 7 data bytes**.

### Byte index

| Index | Name | Description |
|:-----:|:----:|-------------|
| 0 | Message ID | Frame identifier |
| 1 | Data-0 | Payload |
| 2 | Data-1 | Payload |
| 3 | Data-2 | Payload |
| 4 | Data-3 | Payload |
| 5 | Data-4 | Payload |
| 6 | Data-5 | Payload |
| 7 | Data-6 | Payload |

In the capture viewer, column `D1` is Data-0 and `D2` is Data-1. `Dn` is Data-`(n - 1)`.

Some fields split a byte into **nibbles (4 bits)**. Invalid sentinels (`0xFF` for a byte, `0xF` for a nibble) are left blank.

### Bit order

Bit **7 = MSB**, bit **0 = LSB**.

Example: value `0xA1`

| 7 | 6 | 5 | 4 | 3 | 2 | 1 | 0 |
|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| 1 | 0 | 1 | 0 | 0 | 0 | 0 | 1 |

- Bits 7–4 → high nibble (MSB nibble) = `0xA`
- Bits 3–0 → low nibble (LSB nibble) = `0x1`

---

## Message ID overview

| ID | Status | Contents |
|:--:|:------:|----------|
| `0x14` | Decoded | Left-handle buttons, Wonder Wheel In/Out, rotation, and direction |
| `0x20` | Decoded | Ignition, rear speed, fuel range, fuel gauge, ambient light |
| `0x2B` | Decoded | Several payloads in one frame. Data-1 selects the kind. Odometer (km) when Data-1 is `0x25`. |
| `0x2E` | Unknown | — |
| `0xE2` | Decoded | Engine RPM, gear, throttle, engine temperature, ASC (Automatic Stability Control), oil |
| `0xE9` | Not accurate | Average speed, speed, voltage, fuel fault, general fault. This decode is not accurate. |

---

## ID `0x14` — Left handle / Wonder Wheel

| Data | Status | Description |
|:----:|:------:|-------------|
| Data-0 | Unknown | — |
| Data-1 | Decoded | Left Handle Controller 1 (button bitmap) |
| Data-2 | Decoded | Left Handle Controller 2 (button bitmap) |
| Data-3 | Decoded | Wonder Wheel In / Out |
| Data-4 | Unknown | — |
| Data-5 | Decoded | Wonder Wheel rotation value |
| Data-6 | Unknown | — |

### Data-1 — Left Handle Controller 1

Bike: S1000XR and similar

| 7 | 6 | 5 | 4 | 3 | 2 | 1 | 0 |
|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| Horn | Turn cancel | Turn right | Turn left | Cruise | Hazard | ABS | ESA |

Turn signals also use the whole byte, not only one bit:

| Value | Signal |
|:-----:|:------:|
| `0x10` | Left |
| `0x20` | Right |
| `0x40` | Center (cancel) |
| other | Off |

A set bit (`1`) is `on`. A clear bit is `off`.

### Data-2 — Left Handle Controller 2

Bike: S1000XR and similar

| 7 | 6 | 5 | 4 | 3 | 2 | 1 | 0 |
|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| ? | ? | Trip | Info | Cruise Set | Cruise Reset | High beam | ? |

### Data-3 — Wonder Wheel In / Out

Bike: S1000XR and similar

| Value | Meaning |
|:-----:|---------|
| `0xFD` | Out, wheel button Left |
| `0xFE` | In, wheel button Right |
| other | Wheel button off |

### Data-5 — Wonder Wheel rotation value

Bike: S1000XR and similar

- Range: `0x00` – `0xFF`
- Compared with the previous frame, wrapping through 8-bit overflow: an increase is `up`, a decrease is `down`.

---

## ID `0x20` — Ignition / rear speed / fuel range / ambient light

| Data | Status | Description |
|:----:|:------:|-------------|
| Data-0 | Unknown | — |
| Data-1 | Decoded | Ignition, high nibble |
| Data-2 | Decoded | Rear speed, low 8 bits |
| Data-3 | Decoded | Low nibble = rear-speed high 4 bits; high nibble = fuel-range low digit |
| Data-4 | Decoded | Fuel range, remaining digits; gauge = range / 250 on S1000XR 2016 |
| Data-5 | Decoded | Ambient light, low nibble |
| Data-6 | Unknown | — |

### Data-1 — Ignition

Use the high nibble, `(Data-1 >> 4) & 0x0F`.

| Value | Meaning |
|:-----:|:-------:|
| `0x0`–`0x3` | Off |
| `0x4`–`0x7` | On |
| other | Blank |

`0x7F` is On (`0x7`).

### Data-2 / Data-3 — Rear speed

Blank when Data-2 is `0xFF`. When the low nibble of Data-3 is `0xF`, the data is not accurate.

```text
rear_wheel_rpm = Data-2 | ((Data-3 & 0x0F) << 8)
```

**Tire assumption (rear 190/55ZR17)**

| Item | Value |
|------|-------|
| Effective diameter | 642 mm |
| Distance per revolution | 2017 mm |

```text
speed_kmh = (rear_wheel_rpm × 60 × 2017) / 1000 / 1000
```

The bike appears to use a fixed factor of `0.14` instead of this tire-derived scale (`60 × 2017 / 1_000_000 ≈ 0.121`).

### Data-3 / Data-4 — Fuel range

When the high nibble of Data-3 is `0xF` or Data-4 is `0xFF`, the data is not accurate.

```text
fuel_range = ((Data-3 >> 4) & 0x0F) + ((Data-4 & 0x0F) × 16) + (((Data-4 >> 4) & 0x0F) × 256)
```

On the S1000XR 2016, a full tank is about 250 km of range, so the fuel gauge is:

```text
fuel_gauge = fuel_range / 250
```

### Data-5 — Ambient light

When the low nibble is `0xF`, the data is not accurate.

```text
ambient_light = Data-5 & 0x0F
```

---

## ID `0x2B` — Multiplexed payloads / odometer

`0x2B` carries several payloads in one frame. In this capture Data-0 is `0x55`, and Data-1 selects the kind. The odometer applies only when Data-1 is `0x25`.

| Data | Odometer (`Data-1 = 0x25`) |
|:----:|----------------------------|
| Data-0 | `0x55` in this capture |
| Data-1 | `0x25` |
| Data-2 | Low byte |
| Data-3 | Middle byte |
| Data-4 | High byte |
| Data-5 | Not part of the distance |

```text
0x2B  0x55  0x25  Data-2  Data-3  Data-4  Data-5
odometer_km = (Data-4 × 65536) + (Data-3 × 256) + Data-2
```

Equivalent to `(Data-4 << 16) | (Data-3 << 8) | Data-2`.

> Indexes are **Data-n** (excluding Message ID). In a raw buffer that is `buf[3]…buf[5]`.

---

## ID `0x2E`

| ID | Status |
|:--:|:------:|
| `0x2E` | Unknown |

---

## ID `0xE2` — RPM / gear / throttle / engine temperature

| Data | Status | Description |
|:----:|:------:|-------------|
| Data-0 | Decoded | RPM low byte |
| Data-1 | Decoded | High nibble = gear; low nibble = RPM high digit |
| Data-2 | Decoded | Throttle position |
| Data-3 | Decoded | Engine temperature |
| Data-4 | Decoded | High nibble = ASC; low nibble = oil |
| Data-5…6 | Unknown | — |

### Data-0 / Data-1 — Engine RPM

When Data-0 is `0xFF` or the low nibble of Data-1 is `0xF`, the data is not accurate.

```text
rpm = (Data-0 + ((Data-1 & 0x0F) × 255)) × 5
```

### Data-1 — Gear (high nibble)

Blank when the nibble is `0xF` or is not in the table.

| Value | Gear |
|:-----:|:----:|
| `0x1` | 1 |
| `0x2` | N |
| `0x4` | 2 |
| `0x7` | 3 |
| `0x8` | 4 |
| `0xB` | 5 |
| `0xD` | 6 |

### Data-2 — Throttle

Blank when Data-2 is `0xFF`. `36` is 0% and `236` is 100%. Results below 0% or above 100% are clamped.

```text
throttle_percent = ((Data-2 - 36) × 100) / (236 - 36)
```

### Data-3 — Engine temperature

When Data-3 is `0xFF`, the data is not accurate.

```text
engine_temp_c = (Data-3 × 0.75) - 25
```

### Data-4 — ASC (Automatic Stability Control, high nibble)

When the nibble is `0xF` or matches none of the rows below, the data is not accurate.

| Value | ASC |
|:-----:|:---:|
| `0x1`, `0x9` | Intervention |
| `0x2`, `0x5`, `0x6`, `0x7`, `0xA`, `0xD`, `0xE` | Error |
| `0x3`, `0xB` | Self-diagnosis |
| `0x8` | Deactivated |

### Data-4 — Oil (low nibble)

When the nibble is `0xF`, the data is not accurate.

| Value | Oil |
|:-----:|:---:|
| `0x2`, `0x6`, `0xA`, `0xE` | Low |
| other | Normal |

---

## ID `0xE9` — Average speed / speed / voltage / faults

The `0xE9` decode below is not accurate.

| Data | Status | Description |
|:----:|:------:|-------------|
| Data-0 | Not accurate | Average speed, both nibbles |
| Data-1 | Not accurate | Low nibble = average-speed high digit |
| Data-2 | Not accurate | Speed |
| Data-3 | Not accurate | Voltage |
| Data-4 | Not accurate | High nibble = fuel fault; low nibble = general fault |
| Data-5…6 | Unknown | — |

### Data-0 / Data-1 — Average speed

When Data-0 is `0xFF` or the low nibble of Data-1 is `0xF`, the data is not accurate.

```text
avg_speed = (((Data-0 >> 4) & 0x0F) × 2) + ((Data-0 & 0x0F) × 0.125) + ((Data-1 & 0x0F) × 32)
```

### Data-2 — Speed

Blank when Data-2 is `0xFF`.

```text
speed = Data-2 × 2
```

### Data-3 — Voltage

When Data-3 is `0xFF`, the data is not accurate.

```text
voltage = Data-3 / 10
```

### Data-4 — Fuel fault (high nibble)

Blank when the nibble is `0xF`.

| Value | Fuel fault |
|:-----:|:----------:|
| `0x2`, `0x6`, `0xA`, `0xE` | Fault |
| other | Normal |

### Data-4 — General fault (low nibble)

When the nibble is `0xF` or no flag below is set, there is no fault. More than one flag can be active.

| Flag | Values |
|:----:|:------:|
| Flashing yellow | `0x1`, `0x5`, `0xD` |
| Steady yellow | `0x2`, `0x6`, `0xA`, `0xE` |
| Flashing red | `0x4`, `0x5`, `0x6`, `0x7`, `0x9` |
| Steady red | `0x8`, `0x9`, `0xA`, `0xB` |

---

## Contributing

- Include capture logs, bike model/year, Message ID, Data index, bit/mask, and interpretation.
- Leave unknown fields as `?` — do not guess button semantics.
